---
draft: false
date: '2024-11-16T00:00:00+00:00'
title: 'Dynamically Branding in PingOne Advanced Identity Cloud: Conclusion'
description: 'Conclusion of the 2-part series on how to dynamically brand User Experience Journeys and Email Templates in PingOne AIC'
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

# Recap

Hosted Pages and Email Templates are an excellent out of the box way to cater your User Experience to match each and every individual using your platform. But configuring themes at the Journey or Node level becomes tough to scale as you onboard more and more brands and partnerships. By dynamically identifying and building our theming we can simplify our administrative experience while providing the greatest level of personalization to our end-users.

We were able to customize theming in two ways:

1. [Dynamically Changing the Theming]({{< ref "1-dynamic-theming-aic" >}})
2. [Selectively Changing the Theme Styling]({{< ref "2-selective-theme-styling-aic" >}})

# The Completed Journey

The Complete Journeys for each section can be found here:

1. [ChangeThemeByOrg.json](https://gist.github.com/gwizdala/a81ae8ac9fcf2621473404b1307fdc00#file-changethemebyorg-json)
2. [ChangeThemeCustomizationByOrg.json](https://gist.github.com/gwizdala/a81ae8ac9fcf2621473404b1307fdc00#file-changethemecustomizationbyorg-json)

We have seen them in action separately, but we can combine the two together into a singular Journey as well.

The Combined Journey, using the Journeys developed in the previous parts, can be found here:

[ChangeBrandingByOrg.json](https://gist.github.com/gwizdala/a81ae8ac9fcf2621473404b1307fdc00#file-changethemecustomizationbyorg-json)

This Journey allows us to **Override** the theme and **Customize** components of that theme. With these building blocks in place, you have all the tools you need to make your experience the best it can be for all users you support.

---

# Further Reading

- [Introduction]({{< ref "introduction" >}})
- [Part 1: Dynamically Changing the Theming]({{< ref "1-dynamic-theming-aic" >}})
- [Part 2: Selectively Changing the Theme Styling]({{< ref "2-selective-theme-styling-aic" >}})
- **Conclusion & Recap**