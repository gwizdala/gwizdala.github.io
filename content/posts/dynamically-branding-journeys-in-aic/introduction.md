---
draft: false
date: '2024-11-16T03:00:00+00:00'
title: 'Dynamically Branding in PingOne Advanced Identity Cloud: Introduction'
description: 'Introduction of the 2-part series on how to dynamically brand User Experience Journeys and Email Templates in PingOne AIC'
summary: Learn how to use PingOne Advanced Identity Cloud to create custom-themed experiences for each and every user
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud"]
types: ["Coding"]
cover:
  image:
  alt:
  caption:
  relative: false
ShowToc: true
author: David Gwizdala
---

# Why Dynamic Branding?

A key consideration when designing your Identity and Access Management (IAM) strategy is **experience**. If your customer experience is difficult to use, or doesn’t match the branding of your organization, customers lose patience and trust in your platform and tend to walk away. If your employee experience is the same, you risk inadvertently relaxing your employee’s tolerance to scam sites and phishing operations.

But experience isn’t one-size fits all - every brand, department, franchise, and user could require different theming and branding to identify their intent and actions when they enter into your platform.

The PingOne Advanced Identity Cloud (AIC) platform easily enables you to dynamically control branching paths based on who you are, what you have, and what you know. And with [Hosted Pages](https://docs.pingidentity.com/pingoneaic/latest/end-user/customize-login-enduser-pages.html) and [Email Templates](https://docs.pingidentity.com/pingoneaic/latest/tenants/email-templates.html), you can design how these experiences **look and feel** without requiring any updates to your own websites’ or applications’ source code.

Since there’s plenty of documentation on how to design and use Hosted Pages and Email Templates (Check out the [Customization Use Cases](https://docs.pingidentity.com/pingoneaic/latest/use-cases/preface-pages/customization.html)), we’re going to focus on swapping them dynamically within Journeys. As such, this walkthrough expects a level of familiarity with Hosted Pages, Email Templates, and Journeys to get started.

# Dynamically Branding Journeys and Emails

This two-part How-To series will cover:

1. [Dynamically Changing the Theming]({{< ref "1-dynamic-theming-aic" >}})
2. [Selectively Changing the Theme Styling]({{< ref "2-selective-theme-styling-aic" >}})

Each section contains Themes, Journey Configurations, and/or Scripts that you can use either standalone or as part of a larger Journey. Since branding and theming is different per organization, the sections are structured in a way that you can pick and choose which apply to you. If you’d like a version that consolidates all features into one Journey, you can jump to The Completed Journey in the [Recap]({{< ref "conclusion" >}}) section to download a copy for your tenant. Otherwise, follow along with the How-To.

This How-To expects a basic understanding of AIC Journeys and JavaScript. If you haven’t scripted within Identity Cloud before, it may be wise to review some precursory blogs in the [documentation](https://docs.pingidentity.com).

---

# Further Reading

- **Introduction**
- [Part 1: Dynamically Changing the Theming]({{< ref "1-dynamic-theming-aic" >}})
- [Part 2: Selectively Changing the Theme Styling]({{< ref "2-selective-theme-styling-aic" >}})
- [Conclusion & Recap]({{< ref "conclusion" >}})