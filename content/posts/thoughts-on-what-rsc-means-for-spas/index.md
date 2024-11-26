---
title: "Thoughts on What RSC Means for SPAs"
subtitle: "Wanna try Waku?"
date: 2024-11-26T21:00:00+09:00
tags: ["react", "rsc", "framework"]
---

## Introduction

RSC stands for React Server Component, but in this post, I’ll use RSC to refer to a broader architecture consisting of two key aspects:

- The core feature, which is the ability to serialize and deserialize React components and other values, and
- The best practice based on the core feature, which I kind of think is still to be explored.

SPAs, or Single Page Applications, can often be deployed as static files. While a separate server may exist, it typically doesn’t serve the SPA itself. In this context, an SPA can have multiple pages, but let's not argue about the precise definition.

Now, as the name suggests, RSC usually requires a runtime server, and it will be more powerful with a server. But, it's often said that we can use RSC without a server.

In this post, I would like to share my random brief thoughts on this topic.

To focus the discussion, I would like to exclude SSR (Server-Side Rendering) from the scope. SSR adds a capability to generate HTML on the runtime server or at build time, and it's just one additional feature that's valuable in any case.

## How can RSC benefit SPAs right away?

Even without a runtime server, RSC offers specific advantages for SPAs.

If we don't have a runtime server, there seems to be no benefit of RSC for SPAs. However, the serialization capability of RSC has a merit for SPAs if we want to reduce the JS bundle size. If parts of the component tree are somewhat static, we can render the components at build time and serialize them.

The RSC payload is text and can be served as a static file. We don't need to send the JS code to generate the RSC payload. For example, a markdown renderer can run at build time, and if we don't need to render markdown on the client, we don't need to include the markdown renderer in the JS bundle.

The static files of RSC payloads don't need to be loaded initially. SPAs can fetch them on demand, just like we do lazy loading for JS bundles. So, we can basically offload some client work to the build process.

In this context, SPAs don't necessarily mean client-first. The component tree would be server-first, and then we add client components. Even if an app is mostly built with client components, we would still have some server components as server components and client components can be mixed in the component tree.

One drawback, in my mind, is the mental shift. Now, we need to think about what should be server components and what should be client components. With pure SPAs, we don't need to worry about it. So, this is a trade-off.

## What can RSC do if we have an API server for SPAs?

If we had an API server with an SPA, we could use RSC. Typically, we would use JSON for the API response, but that can be replaced with an RSC payload.

The benefits of using an RSC payload instead of plain JSON are:
- We can send rendered components in addition to data: Again, this avoids sending JS bundles to render the components.
- We can stream data: Data can be sent in chunks over time, allowing users to see it as it arrives.
- We don't need to define the data format. The RSC payload is just renderable on the client.

So, basically, it's just another wire protocol. But, it's for React. We typically use the `"use server"` directive, which allows seamless integration with the server and helps with type checking.

## Anything else?

A router library will play an important role in integrating RSC capabilities. They provide user-facing APIs and hide some of the complexity of RSC. It doesn't have to be a router. There can be another library to hide the complexity of RSC. It's just that the router seems the most common library for now.

## Closing notes

These were just my initial reflections on how RSC might benefit SPAs. There would be more to uncover, and probably I should revisit this topic with some working examples. [Waku](https://waku.gg) can be used to create such examples that use RSC for static sites. If you are curious about how this works in practice, I encourage you to try it out.
