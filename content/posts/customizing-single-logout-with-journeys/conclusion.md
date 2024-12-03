---
draft: false
date: '2024-03-15T00:00:00+00:00'
title: 'Customizing Single Logout Using Journeys: Conclusion & Recap'
description: 'Conclusion of the 4-part series Customizing Single Logout Using Journeys'
summary: View the completed Journeys and Recap what we've learned from Customizing Single Logout
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

By using Journeys, we are able to capture a user’s session, control external sessions via API calls, terminate their PingOne AIC session, and redirect them to custom endpoints. Effectively, Journeys let us create Universal Single Logout regardless of the implementation from an external application.

While building this universal logout, we additionally learned the following skills:

1. [Capturing the user’s existing browser session]({{< ref "1-capture-browser-session" >}})
2. [Sending out an external API request to terminate an external session]({{< ref "2-terminating-external-session-rest" >}})
3. [Invalidating the user’s PingOne AIC session]({{< ref "3-invalidating-users-aic-session" >}})
4. [Redirecting the user to a custom URL]({{< ref "4-redirecting-custom-url" >}})

These toolsets should give you the confidence and power to create an endless number of interesting and useful Journeys for your users and your team.

# The Completed Journey

The complete Journey, using the Journeys developed in the previous parts, can be found here:

[Customizing Single Logout Using Journeys, Complete](https://gist.github.com/gwizdala/b2ab2b41949933545f6fff7ba97a723e#file-universalsinglelogout_example-json)

A consolidated version of the Journey, without inner trees or testing output, can be found here:

[Customizing Single Logout Using Journeys, Consolidated](https://gist.github.com/gwizdala/b2ab2b41949933545f6fff7ba97a723e#file-universalsinglelogout-json)

# Enhancements/Next Steps

- As noted in previous sections, consider using [Library Scripts](https://docs.pingidentity.com/pingoneaic/latest/am-scripting/library-scripts.html) for common/reusable actions such as gathering cookie data, accessing IDM attributes, and calling external APIs. This will greatly improve your code reusability across your Journeys.
- Depending on your use case, consider updating your error messages to more robustly inform the user (and you!) what went wrong during the execution of the Journey.
- To more easily manage and understand attributes on your Identities when referencing in code and in the administrative console, consider adding [custom attributes](https://docs.pingidentity.com/pingoneaic/latest/identities/identity-cloud-identity-schema.html#create-custom-attributes).

---

# Further Reading

- [Introduction]({{< ref "introduction" >}})
- [Part 1: Capturing the user’s existing browser session]({{< ref "1-capture-browser-session" >}})
- [Part 2: Sending out an external API request to terminate an external session]({{< ref "2-terminating-external-session-rest" >}})
- [Part 3: Invalidating the user’s PingOne AIC session]({{< ref "3-invalidating-users-aic-session" >}})
- [Part 4: Redirecting the user to a custom URL]({{< ref "4-redirecting-custom-url" >}})
- **Conclusion & Recap**