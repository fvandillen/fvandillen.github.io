---
title: "Your pipeline said it deployed. Your app crash-looped anyway."
slug: "verify-azure-container-apps-deployment-health"
date: 2026-07-03T09:00:00+02:00
draft: false
author: ["Florian van Dillen"]
categories: [ "Azure", ".NET" ]
tags: [ "GitHub Actions", "Container Apps", "DevOps" ]
description: "az containerapp update only waits for ARM, not for your app to actually start. Here's a small GitHub Actions step that catches crash-looping deployments before your users do."
---

## The 2 AM feeling, at 10 AM

I recently shipped a small feature to an internal tool: profile picture uploads backed by Azure Blob Storage. Locally, everything worked — Aspire, Azurite emulator, the whole local dev loop was green. I pushed to `main`, the GitHub Actions pipeline ran, every step turned green, and I went on with my day.

A little while later a colleague tried uploading a picture and got a `404 Not Found`. Odd — the endpoint existed, I'd tested it. I opened the Container App logs in the portal and found this instead:

{{< highlight text >}}
Unhandled exception. System.ArgumentException: Format of the initialization string does not conform to specification starting at index 0.
   at System.Data.Common.DbConnectionOptions.GetKeyValuePair(...)
   at Aspire.Azure.Storage.Blobs.AzureBlobStorageContainerSettings.ParseConnectionString(String connectionString)
   at Program.<Main>$(String[] args) in /home/runner/work/.../Program.cs:line 63
{{< /highlight >}}

The API had been crash-looping since the deploy. Every single request was hitting a container that never finished starting up. And yet the GitHub Actions run for that deployment was **100% green**.

## Why your pipeline doesn't notice

If you deploy to Azure Container Apps from a workflow, there's a good chance your rollout step looks something like this:

{{< highlight bash >}}
az containerapp update \
  --name my-api \
  --resource-group my-rg \
  --image myregistry.azurecr.io/my-api:abc1234
{{< /highlight >}}

This command waits for the *Azure Resource Manager* deployment to reach `Succeeded`. But that only means Azure accepted your request and created a new revision — it says **nothing** about whether the container inside that revision actually started, bound to its port, and started serving traffic. An unhandled exception during startup, a missing environment variable, a bad connection string — all of these crash the container after ARM has already reported success. Your pipeline goes green, your app goes down, and you find out from a colleague (or a customer) instead of from CI.

The fix is simple: after the rollout, ask Container Apps directly whether the revision you just shipped is actually healthy, and fail the job if it isn't.

## A full, working example

Below is a complete, minimal GitHub Actions workflow for "an arbitrary .NET solution" — swap in your own project names, registry, and resource group. It:

1. Builds and publishes a container image directly from the `.csproj` using the .NET SDK's built-in container support (no `Dockerfile` needed).
2. Pushes it to Azure Container Registry.
3. Rolls it out to an existing Azure Container App.
4. **Verifies the new revision is actually running and healthy before letting the job succeed.**

It assumes the Container App and Container Registry already exist (created once via Bicep/Terraform/`az cli`, and that the workflow authenticates via OIDC federated credentials — no stored client secrets.

### The .NET project

Nothing special is required here — this works for any ASP.NET Core project. Just make sure it targets Linux/x64 so the SDK can build a container image for it:

{{< highlight xml "linenos=table" >}}
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
{{< /highlight >}}

{{< highlight bash >}}
# Sanity check locally before you ever touch CI - this is exactly
# what the pipeline will run to build the image.
dotnet publish src/MyApp.Api/MyApp.Api.csproj \
  -c Release \
  --os linux --arch x64 \
  /t:PublishContainer \
  -p:ContainerRegistry="myregistry.azurecr.io" \
  -p:ContainerRepository="my-api" \
  -p:ContainerImageTags="\"local-test\""
{{< /highlight >}}

### The GitHub Actions workflow

{{< highlight yaml "linenos=table" >}}
name: Deploy to Azure

on:
  workflow_dispatch:
  push:
    branches: [main]

permissions:
  id-token: write   # required for OIDC az login
  contents: read

env:
  RESOURCE_GROUP: my-rg
  CONTAINER_APP_NAME: my-api
  IMAGE_REPOSITORY: my-api

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: ACR login
        run: az acr login --name myregistry

      - name: Build & push image
        run: |
          tag="${GITHUB_SHA::8}"
          dotnet publish src/MyApp.Api/MyApp.Api.csproj \
            -c Release \
            --os linux --arch x64 \
            /t:PublishContainer \
            -p:ContainerRegistry="myregistry.azurecr.io" \
            -p:ContainerRepository="$IMAGE_REPOSITORY" \
            -p:ContainerImageTags="\"$tag\""
          echo "API_IMAGE=myregistry.azurecr.io/$IMAGE_REPOSITORY:$tag" >> "$GITHUB_ENV"

      - name: Roll out new image
        uses: azure/cli@v2
        with:
          azcliversion: latest
          inlineScript: |
            az containerapp update \
              --name "$CONTAINER_APP_NAME" \
              --resource-group "$RESOURCE_GROUP" \
              --image "$API_IMAGE" \
              -o none

      # `az containerapp update` only waits for the ARM-level provisioning of
      # the new revision to succeed — it does NOT verify the container
      # actually started (e.g. an unhandled startup exception causes a crash
      # loop that ARM still reports as "Succeeded"). Poll the revision's own
      # health/running state so a broken image fails the pipeline instead of
      # silently going live.
      - name: Verify new revision is healthy
        uses: azure/cli@v2
        with:
          azcliversion: latest
          inlineScript: |
            set -euo pipefail

            revision=$(az containerapp show \
              --name "$CONTAINER_APP_NAME" \
              --resource-group "$RESOURCE_GROUP" \
              --query properties.latestRevisionName -o tsv)
            echo "Watching revision: $revision"

            max_attempts=30
            attempt=0
            while true; do
              attempt=$((attempt + 1))
              state=$(az containerapp revision show \
                --name "$CONTAINER_APP_NAME" \
                --resource-group "$RESOURCE_GROUP" \
                --revision "$revision" \
                --query "{running:properties.runningState, health:properties.healthState}" -o json)
              running=$(echo "$state" | jq -r '.running')
              health=$(echo "$state" | jq -r '.health')
              echo "Attempt $attempt/$max_attempts: runningState=$running healthState=$health"

              if [ "$running" = "Running" ] && [ "$health" = "Healthy" ]; then
                echo "Revision $revision is healthy."
                break
              fi

              if [ "$running" = "Failed" ] || [ "$health" = "Unhealthy" ]; then
                echo "::error::Revision $revision failed to start (runningState=$running, healthState=$health)."
                echo "--- Recent container logs ---"
                az containerapp logs show \
                  --name "$CONTAINER_APP_NAME" \
                  --resource-group "$RESOURCE_GROUP" \
                  --revision "$revision" \
                  --tail 200 || true
                exit 1
              fi

              if [ "$attempt" -ge "$max_attempts" ]; then
                echo "::error::Timed out waiting for revision $revision to become healthy."
                echo "--- Recent container logs ---"
                az containerapp logs show \
                  --name "$CONTAINER_APP_NAME" \
                  --resource-group "$RESOURCE_GROUP" \
                  --revision "$revision" \
                  --tail 200 || true
                exit 1
              fi

              sleep 10
            done
{{< /highlight >}}

## Walking through the health check

The two Container Apps-specific commands do all the work:

- `az containerapp show --query properties.latestRevisionName` finds the name of the revision that was just created by the update — you need this to look at *that specific* revision rather than whatever happened to be active before.
- `az containerapp revision show` returns, among other things, `properties.runningState` (`Running`, `Processing`, `Failed`, `Degraded`, ...) and `properties.healthState` (`Healthy`, `Unhealthy`, `None`). Together they tell you whether the platform managed to start your container *and* whether it's currently considered healthy.

The loop polls both values every 10 seconds for up to 5 minutes (`max_attempts=30`). Three outcomes are possible:

1. **Healthy** — `Running` + `Healthy` — the step succeeds and the job moves on.
2. **Clearly broken** — `Failed` or `Unhealthy` — no point waiting any longer, so it fails fast and dumps the last 200 lines of container logs straight into the workflow output. This is exactly the exception trace that took me a trip to the Azure Portal to find manually — now it's right there in the Actions log.
3. **Timeout** — after 5 minutes without a clear answer, the step gives up, fails, and still dumps the logs for you.

That's it. No extra infrastructure, no extra services, just two `az` calls wrapped in a small polling loop.

## Why not just curl a health endpoint?

An alternative is to expose a `/health` endpoint and curl the app's public URL after deploying. It works, but it has downsides for this use case:

- It requires your app to expose an *unauthenticated*, public health endpoint — something you may not want in production, and ASP.NET Core's own health check middleware documentation actively warns about the security implications of enabling it outside development.
- It tells you the app answered HTTP requests, but not *why* it didn't if it doesn't — you're back to digging through logs manually.
- It doesn't work at all if your app has no HTTP-reachable health surface (background workers, for example).

Asking the Container Apps control plane directly sidesteps all three: it works for any container regardless of what's running inside it, and a failure comes with the logs already attached.

## Conclusion

"The pipeline is green" and "the app is running" are two different claims, and it's worth spending five extra minutes making your pipeline actually check the second one. It would have saved me a `404` in production and a slightly awkward Slack message from a colleague. Now it just fails the build and shows me the stack trace instead — much better start to the day.
