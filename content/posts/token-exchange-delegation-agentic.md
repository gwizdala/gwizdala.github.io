---
draft: false
date: '2026-06-01'
title: 'Token Exchange with Delegation in the Agentic Era'
description: A primer on OAuth Token Exchange and delegation in both traditional and agentic systems
summary: Learn how token exchange and delegation enable secure, auditable access delegation in modern systems. Explore the concepts with real-world examples and then see how they apply to both traditional applications and AI-powered agentic systems. 
categories: ["Ping Identity"]
tags: ["OAuth"]
types: ["Coding"]
cover:
  image:
  alt:
  caption:
  relative: false
ShowToc: true
author: David Gwizdala
---
# Introduction

If you’re familiar with OAuth, you’ve likely come across [**RFC 8693 - Token Exchange**](https://datatracker.ietf.org/doc/html/rfc8693). An often-overlooked capability within the OAuth specification, token exchange gives us the ability to communicate between multiple services and applications, re-scope requested tokens, and **delegate** authority to services downstream.

Delegation in particular has become a hot topic as of late due to the rise of **Agentic Systems**. With the rise of AI tooling deciding to perform actions in a non-deterministic manner it is becoming more and more important to delegate those actions instead of giving them broad overarching permissions without auditability.

If some - or all - of that went over your head, you’re not alone and you’re in the right place. 

This Guide aims to give you a high-level overview of what Token Exchange is, how Delegation applies, and what that means in agentic systems. We’ll do that by taking a closer look at what Delegation means in the context of application, and then agentic, identity. 

It’s split into the following parts:

1. [Understanding](#what-is-token-exchange-with-delegation)
2. [Application](#applying-token-exchange-with-delegation)

By the end of this Guide you should have a high-level understanding of what Token Exchange is, why it and delegation is important, and how it applies to both an agentic and non-agentic system. You’ll also gather a deeper understanding of why identity is so important to this system.

# What is Token Exchange with Delegation? 

## Overview

We use OAuth to **secure access to resources**. It allows us to define **who owns what** and **what permissions they have**.

OAuth also allows us to **share access to resources securely**. Given what we know about ownership and permissions, we can **permit others access** - forever or temporarily (**lifetime**) and with specific permissions (**scopes**).

When we share resources, we want to be very specific about **what we share** and **with whom** we share that resource with. We also likely want to know, once we’ve shared, **who is doing what and who are they doing that action for**.

Token Exchange with Delegation provides us that detail.

Let’s define some key players in exchange: the **Subject**, the **Actor**, and the **Authorization Server**.

* The **Subject** is the primary entity that owns the resource. This could be an end-user, a microservice, an embedded system, or otherwise - it’s the **primary identity** that this exchange is for.
* The Subject passes their token to the **Actor**, who is the entity that exchanges the token. This could be an application, secondary microservice, or even another identity who will receive the exchanged token and (likely) perform an action **on behalf of the Subject**.
* The **Authorization Server (AS)** performs the exchange. There could be more than one AS depending on if the exchange is occurring cross-domain with trusted systems or all be performed on a single AS if the Subject and Actor live in the same domain. The AS will ultimately decide what scopes the Actor is able to receive based on the request that they made.

At the end of the day, the **Actor** exchanges the **Subject’s Token** with the **Authorization Server** to get a new token modified to perform the specific task the Actor wants to take. And **exchanges can be chained** - some services may call additional services, and along the way their permissions and Subject/Actor detail updates accordingly.

**Token Exchange** is the process of **transforming the scope**. **Delegation** is the “**on behalf of” relationship** tied to the exchange itself. While you can perform Token Exchange without Delegation, **it’s critical to delegate when we want a clear audit trail of who performed the action and who they performed it for**.

## Example

Getting clearer? Let’s break this down with a non-technical example: paying someone from your bank account.

In this example, you want to pay your friend $10. You could go to your bank, give them your ID and PIN, withdraw $10, and then give it to your friend.

Let’s say you can’t get to the bank. Instead, you could give your friend your ID and PIN, have them go to the bank, and have them withdraw the $10.

But wait! If they have your ID and PIN, who’s to say they won’t transfer more than $10, change your PIN, or close your account? By sharing your credentials directly you’ve given your friend far too many permissions.

So instead, you issue a temporary PIN that lets your friend go to the bank and withdraw only $10. To make that PIN you had to prove yourself to your bank and because the PIN is tied to the $10 transaction its blast radius has been significantly reduced (one might even say “downscoped”). But that PIN is missing some key information - we have no idea who withdrew the $10. In fact, in the eyes of the bank, YOU withdrew it. **We just performed Token Exchange without delegation**.

So let’s clear this up further. Rather than a temporary PIN you give your friend a check. That check has your bank routing number on it, the exact amount your friend will get from your account, AND (notably) your friend’s information on it. In fact, before depositing that check the friend is going to have to **validate themselves** by endorsing the check and the bank will **validate the transaction** before letting your friend cash it. 

**You are the Subject, your friend is the Actor, and the bank is the Authorization Server.** You validated yourself by using a valid check, you have downscoped your access to withdrawal only and for a specified amount, and your friend is acting on behalf of you to withdraw using the permissions that they exchanged at the bank who approved the whole thing.

In the case of chaining requests, we are going to be a little imaginative with this example. 

Let’s say that your friend employed a couple other people to help with your task. You don’t know who they are but you do know that you owe your friend $10. Chaining exchanges lets your friend create an invoice that pays their employees from the $10 you gave them access to in your bank account. This would let you, the bank, and your friend be able to see that your friend’s employees got paid through your friend using your money. **A chained token exchange gives you full visibility into each actor before it interacts with the resource.**

# Applying Token Exchange with Delegation 

Now that we have a good baseline, let’s apply this to a non-agentic system and then evolve it into an agentic one.

## Delegation in a Non-Agentic System

A common system may look like what you see here.

![A diagram of a user pointing to an application and then a service/resource](/img/token-exchange-delegation-agentic/basic-non-agentic.png)
*A Basic, Non-Agentic, System*

A **user** uses an **application** to interact with a **service** or **resource**.

In the most traditional sense, this could be an employee logging into a document site to edit a presentation, or a customer listing their previous orders. But an application is really an interface to the service – it could just as easily be a telephony product embedded in a helpdesk system or an embedded system receiving and interacting with analytics data relevant to the end user.

At the end of the day, your infrastructure likely relies on something – be it a person, system, or otherwise – interacting with something else to receive or act upon a service.

And in all reality, your application is likely communicating with more than one service. Microservice architectures, SaaS platforms, and specified tools have given rise to the implementation of an API Gateway that receives, validates, and routes requests to the appropriate service given the context of the user’s – and the application’s – request.

![A diagram of a user pointing to an application, then an API Gateway, which connectes to multiple services](/img/token-exchange-delegation-agentic/gateway-non-agentic.png)
*Adding an API Gateway*

**So how does identity play into all of this?**

![A diagram where an Identity Provider has pointed to the User (authentication), the Application (Token Exchange), the gateway (Introspection, Authorization, Token Exchange) and then the services](/img/token-exchange-delegation-agentic/identity-non-agentic.png)
*Layering an Identity Provider Over a Non-Agentic System*

Your identity layer sits on top of every stage in the flow from a user down to the resource.

* Your **user authenticates**
* The **application exchanges the user’s token** with the appropriate scope to perform an action for the user
* The API Gateway **introspects the exchanged token** to ensure its validity and optionally **exchanges again** to provide additional context and scoping on the call.

In short, the identity provider helps you answer the following questions:

1. **Who is this request for?**
2. **Who is performing the request?**
3. If they’re able, **What can they do?**

![An overlaid set of questions on the aforementioned diagram, which are outlined in this document](/img/token-exchange-delegation-agentic/questions-non-agentic.png)
*Questions an Identity Provider Answers*

Let’s break this down in more detail:

1. **Who is this request for?** In this example, it’s your end-user. Not only do you want to know who this action should be performed on, but you want to ensure that they approved this action to have taken place. In our flow we have this level of assurance because your user **authenticated** with their identity provider and **initiated** the request with their **valid access token**.
2. **Who is performing the request?** Your application, upon receiving the user’s token, exchanged it for their own. This tied the application to the request as the **actor** – they are performing an action **on behalf of** the user. Ultimately, this gives you a **clear audit trail** of **who is doing what and for who**. The API gateway can also exchange the request from the Application to scope the request to themselves and track that the gateway is acting not only on behalf of the application but also the user.
3. If they’re able, **What can they do?** Now that you have all of this information – the actions being requested (the scopes), the resources they want to access, the user that requested and the applications that are acting on behalf of that user, you can use an **authorization engine** to **strongly verify** that all of this data matches what you expect.

Your identity provider plays a part in all of these interactions: It **authenticates the user**, **exchanges the tokens**, and **validates the requests** based on the policies you’ve defined.

## Delegation in an Agentic System

Securing an Agentic interface isn’t too different from the concepts you just reviewed.
![A diagram in which a user interface within an application is talking to an agent which communicates with an LLM, an MCP gateway, MCP, and then services](/img/token-exchange-delegation-agentic/agentic.png)
*An Agentic System*

That same application you were using before now likely contains an interface to interact with your agent. Your agent is listening to that interface. **The agent isn’t the reasoning engine**: it is a deterministic system that surfaces common, repeatable actions you want your non-deterministic system to take.

That agent can be plugged into one, or many, LLMs – Large Language Models (like Claude, ChatGPT, etc.) which use the context of the user interface and the actions surfaced by the agent to make decisions based on what it is understanding and learning from.

But an agent by itself isn’t particularly useful. It, like your application, needs to interact with services and resources based on the actions it’s been told to do. You could surface the raw API calls directly but there are common interfaces, like [MCP (Model Context Protocol)](https://modelcontextprotocol.io/specification/2025-11-25) that have defined standards for how to surface tools for your Agent to discover and subsequently use.

Just like before, you likely have more than one set of services and as such likely have more than one MCP server. To easily surface all tools, and make it simple to interact with them, you probably have an MCP Gateway to field and route those requests.

And, as we saw in the first example, your identity lives across the entire operational flow. The only difference is since we have added an additional layer of action, we have added an additional layer of exchange.

![The identity provider performing the same authentication exchanges, and introspection/authorization but now with an additional step of exchanging at the agent](/img/token-exchange-delegation-agentic/identity-agentic.png)
*Layering an Identity Provider Over an Agentic System*

This answers the exact same questions as before - **Who is this request for**, **Who is performing the request,** and If they’re able, **What can they do?**

**![The questions overlaid on the identity system connecting to the agentic workflow](/img/token-exchange-delegation-agentic/questions-agentic.png)**
*Questions an Identity Provider Answers in an Agentic System*

And that token exchange, that delegation, is key to this approach – we are now tracking, auditing, and validating that the application, the downstream agent, and optionally the gateway have the appropriate permission to perform the actions they’ve been instructed to do.

# Conclusion

You should now have a high-level understanding of Token Exchange with Delegation, along with a view into how to apply it in both non-agentic and agentic use cases. You should understand the importance of knowing who is performing what and for whom, and that this relationship can be chained through more than one identity. Finally, you should recognize the incredible importance of identity in managing this system.

This Guide is your primer, 101 course into the architectural basics. With it, you can begin designing authorization strategies that utilize the information provided by the delegating parties in your identity stack.

