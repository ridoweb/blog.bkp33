---
published: false
layout: post
date: 2022-04-03T00:00:00.000Z
title: Validate Jekyll in Azure Static Web Sites before publishing to GitHub Pages
categories: blog
---
## Validate Jekyll in Azure Static Web Sites before publishing to GitHub Pages

I love GitHub Pages, and I love Jekyll as my blogging enging. However, as I'm still learning Jekyll I wanted to validate any changes to the blog, content or features, before publishing to production, if possible using Pull Requests.

To do so, I've started a `dev` branch, where I can push content. Then I configured Azure Static Web Sites, following the [Tutorial: Publish a Jekyll site to Azure Static Web Apps](https://docs.microsoft.com/en-us/azure/static-web-apps/publish-jekyll).

Now I can push to `dev`, wait for the GH action to push changes to staging, and validate before merging to `main`.
