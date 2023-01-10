---
title: "Microfrontends: Web components (part 1)"
date: 2023-01-10T19:00:00+01:00
draft: false
author: ["Florian van Dillen"]
tags: ["Frontend", "Vue", "Web components"]
description: Diving into web components to unblock frontend teams.
toc: true
comments: false
---

# Introduction
For one of our customers we are in the process of rebuilding the frontend due to growth in the IT organization. In the past year or two, the single team of developers responsible for frontend development has grown into a more feature-oriented organization. This means that one feature can be owned by team A and another feature can be owned by team B. This creates challenges when working on a single frontend that contains legacy as well.

For instance, the current application can only be deployed as a whole. This means releases of new versions have to be coordinated between the different teams. This makes releasing more labour intensive than it should be.

Also testing is an issue. Ideally you want to run automated UI tests so you are certain the frontend behaves in the way you expect it to. However when you have to run the entire test suite (which takes 3+ hours) when you change a single comma, this becomes an issue when you want to release quick and often.

# Microfrontends as a solution
Luckily, we have tools at our disposal nowadays to overcome these issues. One of the methods is called microfrontends. When going this route, you essentially create a shell application that will be responsible for loading the microfrontends (or MFEs) and routing the user to the correct MFE. The shell also functions as a coordination layer between the MFEs and will address any cross-cutting concerns such as authentication and translations.

In our case the shell application will be written in Vue 3. This Javascript framework is widely used at our customer and most developers are familiar with it already. The plan they drafted was to expose the current Vue 2 application as an MFE and consume it in the new shell application. This will allow the teams to gradually recreate those pages and features in more modern web technology and eventually get rid of the old Vue 2 application entirely.

The major upside of having multiple MFEs is that we can now have multiple teams working on them! Each team has their own repositories and pipelines which means they can deploy and test independently.

# Module federation
So how can we expose our MFEs to the shell application and consume them? To achieve this, we decided to use an approach called module federation. In essence, this allows us to remotely load Javascript modules from a URL and mount them into our shell. Because the modules are fetched remotely every time the shell launches, they can be deployed independently.

However, this has proven to not be a very easy task. For one, the shell runs Vue 3 which means you cannot (easily) mount Vue 2 components inside it. In the first iteration, we tried to wrap our Vue 2 components with all of their dependencies before exposing it to the shell via module federation. This quickly became messy and didn't work well:
```typescript
    export function bootstrapComponentRemote(WrapperComponent: any, wrapperId: string) {

        // Get a correctly bootstrapped Vue instance with dependencies configured.
        const vueInstance = bootstrapComponent(WrapperComponent);

        // Overwrite the render method with our own logic.
        vueInstance.render = (x) => {
            return x(WrapperComponent, {
                on: this.$attrs,
                attrs: this.$attrs,
                props: this.$props,
                scopedSlots: this.$scopedSlots
            })
        };

        // Return the Vue2 instance with our Vue2 component wrapped as a Vue3 component.
        return {
            mounted() {
                vueInstance.$mount(`#${wrapperId}`);
            },
            props: WrapperComponent.props,
            render() {
                vueInstance && vueInstance.$forceUpdate();
            },
        };
    }
```

The above approach worked fine when initially routing to the remote component, but subsequent routing would result in the component not being rendered at all. Oops! Quite a big deal when you're working with a single page application...

# Web components to the rescue!
What if we didn't have to deal with all the UI framework complexities and could just send a standardized (and even native) module to our shell? That sounds like black magic, but fortunately the tech is already here. It's called [web components](https://www.webcomponents.org/introduction) and all the major browsers [already support it](https://caniuse.com/?search=web%20components). The tech is based on open standards and uses ES modules to wire it all together.

Essentially, this turns any module/component into a reusable piece of Javascript that we can then use in any framework or even in plain HTML if that is your cup of tea. Think of it as Docker containers for the World Wide Web. 😎😎

What we did is wrap the Vue 2 component as a web component before exposing it via module federation. To achieve this, we use the plugin [Vue web component wrapper](https://github.com/vuejs/vue-web-component-wrapper). This plugin allows us to easily wrap any Vue 2 component as a web component:
```typescript
import Vue from 'vue'
import wrap from '@vue/web-component-wrapper'

const Component = {
  // any component options
}

const CustomElement = wrap(Vue, Component);
```

The wrapped component is exposed via module federation (more on that in part 2 of these series). Once the module is consumed in the shell, it can be registered as a custom element:

```typescript
import CustomElement from 'mfe/customElement';
window.customElements.define('my-element', CustomElement)
```

Once registered, a new wrapper component can be made in the shell that contains a template that uses the custom element:

```html
<template>
  <my-element></my-element>
</template>
```

# Conclusion
The custom element behaves exactly like it should. It can be routed to, attributes can be added to it as a means of passing arguments. Events can be emitted by it and the browser tools can be used to fully inspect it. The web component achieves full encapsulation from the DOM by using a shadow DOM. Therefore it is (partly) immune from specific CSS styles that affect the regular DOM.

All in all, this provides us with a very nice way of exposing our legacy components with the use of open standards. We are not limited by specific frameworks and could even use React or Angular components in the shell. This creates a very solid foundation for the rest of our adventures with our customer.

I intend to continue writing about this and will go in-depth on module federation in a next installment. Stay tuned!