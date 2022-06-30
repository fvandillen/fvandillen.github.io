---
title: "Hello World"
date: 2022-06-30T13:56:48+02:00
draft: false
author: ["Florian van Dillen", "other author", "third author"]
summary: A very cool article testing out functionality with code blocks.
---

Hello, world! I recently worked with some Azure functions and needed to configure dependency injection. I did that by including the following `Startup.cs` in my project:

{{< highlight Csharp "linenos=table" >}}
[assembly: FunctionsStartup(typeof(Example.Startup))]

namespace Example;

public class Startup : FunctionsStartup
{
    public override void Configure(IFunctionsHostBuilder builder)
    {
        builder.Services.AddScoped<ApiClient>();
        builder.Services.AddAutoMapper(typeof(AutoMapperProfile));

        JsonConvert.DefaultSettings = () => new JsonSerializerSettings
        {
            NullValueHandling = NullValueHandling.Include,
        };
    }
}
{{< /highlight >}}