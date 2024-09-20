---
layout: post
title: E-Commerce Project
subtitle: Step-by-Step Development Process (Scroll down for details)
author: Hamza Gedikkaya
categories: E-Commerce
banner:
  video: /assets/videos/tech-back.mp4
  loop: true
  volume: 0.5
  image: /assets/images/post_imgs/e_comm.gif
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Ruby Rails E-Commerce
top: 1
sidebar: []
---

# E-Commerce Project: Step-by-Step Development Process

In this blog post, you will find detailed documentation on how the [e-commerce project](https://github.com/hamzagedikkaya/E-Commerce-Project) on GitHub was developed step by step. This document serves as a comprehensive guide for those looking to build an e-commerce application from scratch. It will also provide information about the various tools and technologies used in the project (such as specific gems), and show you how to integrate these solutions into other projects.

Through this blog series, you will not only learn how to build a fully functional e-commerce project but also gain a solid resource on how to incorporate the necessary technologies into your own projects.

Technologies used in the project:
- **Ruby**: 3.3.0
- **Rails**: 7.2.1

For a quick start on any project, you can use this repository: [rails_template](https://github.com/hamzagedikkaya/rails_template). This is a **quick_start_template** that I have written. Since I will be starting the project with this template, I will first explain the structures within the template and how they work. Let's start the project with the following command:

```bash
rails new e_commerce -j esbuild -c bootstrap -d postgresql
```