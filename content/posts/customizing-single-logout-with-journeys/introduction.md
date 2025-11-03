---
draft: false
date: '2024-03-15T05:00:00+00:00'
title: 'Customizing Single Logout With Journeys: Introduction'
description: 'Introduction of the 4-part series Customizing Single Logout Using Journeys'
summary: Learn how to use PingOne Advanced Identity Cloud to create Single Logout experiences for systems that don't support it out-of-the-box
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

# What is Single Logout?

Single Logout is a powerful feature of the SAML2 Spec that enables you to push a single user logout event across multiple applications and services. Effectively, when a user ends their session in one place, you can terminate it everywhere else, too.

When configuring [Federated Single Logout (SLO)](https://docs.pingidentity.com/pingoneaic/latest/am-saml2/saml2-sso-slo.html) in PingOne Advanced Identity Cloud (AIC) with Ping as the Identity Provider (IdP), your setup may look something like this:

![Single Logout Diagram](/img/slo-diagram.png "A typical Single Logout (SLO) flow with Ping as the IdP.")

_A typical Single Logout (SLO) flow with Ping as the IdP._

In this example,

1. Your User requests logout from Application 1 (a Service Provider, or SP), perhaps by clicking a “logout” button
2. Application 1 reaches out to Ping (the Identity Provider, or IdP) with a Logout Request
3. Ping sends a Logout Request to Application 2 (another SP) indicating that this application should terminate their user session
4. Application 2 terminates their session
5. Application 2 responds back to Ping with a Logout Response indicating that the user session has ended

Ping repeats steps 3-5 for every configured Service Provider (steps 6-8), invalidates the user session on itself if it exists (step 9), and then responds back to the initial application (step 10) who will respond to the user (step 11) - possibly by redirecting them to a login page.

# The Real-World Problem

If every one of your applications and services are configured as Service Providers, and they all support the Single Logout endpoint, then this flow will work exactly as described. But this might not always be the case, for example:

- Your initiating application (Application 1 in the diagram) only supports redirects upon logout
- Your user’s identities are federated to Ping from a separate external IdP (e.g. a B2B2C scenario) that is connected to applications you don’t manage and their sessions also need to be terminated
- Your application does not support the Logout Endpoint and/or uses a bespoke API endpoint to terminate a session

Given that SLO isn’t provided with many SPs out of the box (or at all), you may run into this situation more than you’d expect.

So then, how do you log users out of Ping and all of their connected applications when SAML Single Logout standards can’t be used?

# Creating Custom SLO with Journeys

In instances where the standard SLO doesn’t work, we can use Journeys to capture the initiating user, send out custom API requests, invalidate the user’s PingOne AIC session, and redirect them to the appropriate logout redirect. Since Journeys can be invoked by API or directly through a redirect URL, you can initiate this flow in a variety of ways - such as through a standard SAML SLO configuration, a call from within the SDK, a back-end request, or even a redirect after logging out on the initial application.

This four-part How-To series will cover:

1. [Capturing the user’s existing browser session]({{< ref "1-capture-browser-session" >}})
2. [Sending out an external API request to terminate an external session]({{< ref "2-terminating-external-session-rest" >}})
3. [Invalidating the user’s PingOne AIC session]({{< ref "3-invalidating-users-aic-session" >}})
4. [Redirecting the user to a custom URL]({{< ref "4-redirecting-custom-url" >}})

Each section contains a Journey Configuration and/or Script that you can use either standalone or as part of a larger Journey. There’s no one-size-fits-all for Single Logout, so the sections are structured in a way that you can pick and choose which apply to you. If you want to follow along with a working Journey during your reading, or if you’d like a version that consolidates all features into one script, you can jump to The Completed Journey in the [Recap]({{< ref "conclusion" >}}) section to download a copy for your tenant. Otherwise, follow along with the How-To.

This How-To expects a basic understanding of AIC Journeys, REST APIs, and JavaScript. If you haven’t scripted within Identity Cloud before, it may be wise to review some precursory blogs in the [documentation](https://docs.pingidentity.com).

That's it for the overview: let's get started!

---

# Further Reading

- **Introduction**
- [Part 1: Capturing the user’s existing browser session]({{< ref "1-capture-browser-session" >}})
- [Part 2: Sending out an external API request to terminate an external session]({{< ref "2-terminating-external-session-rest" >}})
- [Part 3: Invalidating the user’s PingOne AIC session]({{< ref "3-invalidating-users-aic-session" >}})
- [Part 4: Redirecting the user to a custom URL]({{< ref "4-redirecting-custom-url" >}})
- [Conclusion & Recap]({{< ref "conclusion" >}})