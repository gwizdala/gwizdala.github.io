---
draft: false
date: '2024-02-22'
title: 'API to API Communication in PingOne Advanced Identity Cloud'
description: 'Connecting APIs with OAuth while restricting Scope'
summary: Enable systems such as microservices to communicate with one another via standards such as OAuth while restricting scope per relationship between systems
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "JavaScript", "OAuth"]
types: ["Coding"]
cover:
  image:
  alt:
  caption:
  relative: false
ShowToc: true
author: ["David Gwizdala", "John Kimble"]
---

# Summary

## Objective

Enable systems such as microservices to communicate with one another via standards such as OAuth while restricting scope per relationship between systems.

## Goals

* Enable the Client to be able to have a relationship with multiple other Resource Servers  
* Ensure that the Resource Server can validate the audience in the claim  
* Ensure that the Resource Server receives a minimally-scoped token  
* Scale across hundreds of services in a maintainable and centralized manner  
* Enable fine-grained control of scopes by Identity (e.g. Organization, Group, Role, Attribute) and Application (e.g. API)

## Problem Statement {#problem-statement}

In a multi-system architecture (e.g. microservice design), multiple applications may be utilizing the same set of APIs minimally scoped to their own unique set of permissions. The example use case may look like the following:

`API1` is the Consumer API  
`API2` is the Provider API

`API1` is looking to consume (i.e. use) `API2`. In order to do so, it will need to have a scope that `API2` knows *and* will need to provide the appropriate audience (i.e. `API2`) so that `API2` can validate that they are the recipient of the token.

But this is a small snapshot: let’s scale this up to hundreds of microservices with thousands of scopes.

We don’t want to store every single scope permutation on `API1`, as they may not be unique, would require the client to have knowledge of scopes unrelated to itself, and would contain the wrong audience (i.e. `API1`) when calling other services.

We don’t want to put multiple audiences in the claim, because we want to specify only the recipient of the particular scope request and don’t want to accidentally over-scope an audience (back to that non-unique token issue).

So what can we do to connect APIs together in a maintainable, consistent way within PingOne Advanced Identity Cloud that adheres to the OAuth spec?

# The Approach {#the-approach}

Ping Access Management (PingAM, PingOne Advanced Identity Cloud) includes something called the Policy Engine, which allows an administrator to create permission rules grouped by Subjects, Resources, and Actions. The Policies contained within a Policy Set can be evaluated by Access Management at time of token issuance, allowing the Authorization Server to programmatically validate and modify the returned access token to match the appropriate Client and their Scopes.

In the example below, we will be using the Client `scopeDrivenAPI1Client` as the Consumer API and `scopeDrivenAPI2Client` as the Provider API, and that the Provider API has a scope of `read`. If you are following along, change these values with that of your own Clients.

Note that every action here can (and should\!) be accessed via API, which greatly expedites setup and management at a large scale.

## Prerequisite: Policy Validation {#prerequisite:-policy-validation}

To enforce our Provider policies from a request made through the Consumer, we’ll create a secure connection to the Policy Engine and then automate that validation method with two scripts: a **Scope Validation Script** and a **Token Modification Script**. These scripts will move validation of the token from the API Consumer Client into the Policy Engine and then return an updated token with the Provider API as the audience.

Once policy validation is set up, it can be reused across all future Policy Sets and Service configurations.

The scripts used in this section are found at the following location: 

[PingOne Advanced Identity Cloud: API to API Policy Evaluation · GitHub](https://github.com/gwizdala/lib-ping/tree/main/How-Tos/api-to-api-communication-with-aic) 

### Creating the Policy Evaluator {#creating-the-policy-evaluator}

In order to securely connect to the Policy Engine, we’ll want to create a specialized Agent that authenticates with AM permissions. We’ll use an Agent identity to do this since it can pass values as an SSO token with the appropriate scope while locking out the request endpoint from other Identity types like Users, Groups, or Organizations.

To create a Gateway, navigate to Gateways & Agents and select “New Gateway/Agent”.

![Screenshot of Creating a Gateway in AIC](/img/api-to-api-communication-with-aic/new-gateway-agent.png)

_Creating a Gateway in AIC_

Inside the dialog, select “Identity Gateway”, and then provide a unique Agent ID and Agent Password. Save these values for later \- you’ll be storing them as an ESV. You won’t need to set up an Identity Gateway for this.

We’ll now want to create a way for the Agent to authenticate. Navigate to Journeys, select “Import”, and upload the `AgentLogin.json`. The journey will contain a Username and Password field validated with an Agent Data Store Decision node, which only allows authentication by Agents.

![Screenshot of Agent Login Journey](/img/api-to-api-communication-with-aic/agent-login-journey.png)

_The Agent Login Journey_

### Setting the ESVs {#setting-the-esvs}

In order for the provided scripts to run, we’ll be setting some environment secrets and variables. This way you can create different configurations per tenant, environment, and realm.

Navigate to the Tenant Settings using the top right navigation and select “Environment Secrets & Variables” \> “Add Secret”, adding in the following secrets to your tenant. Once they’ve been added, make sure to push the changes to the tenant and wait up to 10 minutes for them to populate.

| Secret | Value |
| :---- | :---- |
| `tenant-env-fqdn` | The fully qualified domain name of the tenant, e.g. `openam-example.forgeblocks.com` |
| `cookie` | The tenant cookie, used as the header alongside the SSO token |
| `policy-gateway-id` | The ID of the Gateway being used to generate an SSO token capable of evaluating policies |
| `policy-gateway-secret` | The Secret of the Gateway being used to generate an SSO token capable of evaluating policies |

### Creating the Scripts {#creating-the-scripts}

Under Native Consoles \> AM \> Scripts, select “New Script” and create a Legacy JavaScript entitled `scopeDrivenConsumerClient Scope Validation` with Script Type `OAuth2 Validate Scope`.

In the Script Body, paste `scopeDrivenConsumerClientScopeValidation.js`. This script allows us to additionally validate scope using the Policy Engine instead of evaluating scope on the client, enabling the inclusion of the policies we made without duplicating those scopes in the client configuration directly.

Save this script, and then create another script entitled `PolicyEvalMod`, Language Legacy JavaScript with Script Type `OAuth2 Access Token Modification`, and then paste `policyEvalMod.js`. This script queries the Policy Engine based on the audience-scoped tokens and will return an access token with the audience of the Provider API if valid.

## Defining Policies {#defining-policies}

Once we have set up the validation scripts, we can start setting up Policies based on the permission relationships we want between each service. This section outlines how to set up the relationship between `scopeDrivenAPI1Client` as the Consumer API and `scopeDrivenAPI2Client` as the Provider API, but the principles stay the same no matter what configuration you are looking to create. 

> **Important:** Make sure to note the IDs of your APIs as they’re shown in AM (Applications → OAuth2.0 → Clients, and the name that appears at the top of the page) as these will be used to tie an application back to their policies.

### Creating the Resource Type {#creating-the-resource-type}

Policies and Policy Sets will validate Resources based on Resource Types, coupled rulesets of patterns and actions, configured in AM.

To create a Resource Type, go to Native Consoles \> AM \> Authorization \> Resource Types and select “New Resource Type”.

![Screenshot of Resource Types](/img/api-to-api-communication-with-aic/new-resource-type.png)

_Creating a New Resource Type_

For your resource, use the name of the Provider API/service you’re looking to protect, the pattern you’d like to use to differentiate the policy, audience, and scope (we’re using `api://{Your Provider API ID}/*`  in this example), and the action of `GRANT`. This will allow you to call this resource within a policy set and define different scopes easily for that client.

Note that you can use whatever pattern you’d like as long as it’s consistent and unique \- your policy validation script will be following this pattern for every request coming through your services.

![Screenshot of Configuring the Resource Type](/img/api-to-api-communication-with-aic/setting-resource-pattern.png)

_Setting Up the Pattern_

### Creating the Policy Set {#creating-the-policy-set}

Now that we have a Resource Type, we can define a Policy Set. We’ll want to create a different Policy Set per API/service so that we can easily review and modify permissions on an API by API basis.

To create a Policy Set, go to Native Consoles \> AM \> Authorization \> Policy Sets and select “New Policy Set”.

![Screenshot of Policy Sets](/img/api-to-api-communication-with-aic/new-policy-set.png)

_Creating a New Policy Set_

Set the ID of the Policy Set to the **ID of your Provider API**, the name to “{Provider Name} Policies” (for ease of filtering), and set the Resource Type to the matching Resource Type we created in the previous step.  

![Screenshot of Configuring the Policy Set](/img/api-to-api-communication-with-aic/setting-policy-set.png)

_Setting Up the Policy Set_

Once you hit “Create”, you’ll be taken to the Policy Set where you can start defining policies.

### Creating a Policy {#creating-a-policy}

Let’s add an example permission where we provide the `read` permission for the requesting subject of our Consumer Client. Note that each policy can have multiple Permissions, Actions, and Subjects \- meaning you can create more complex variations of API permissions (such as a “read/write” grouping of APIs, or a scope allowed only to a particular subset of users or organizations).

Inside your created Policy Set, select the “Add a Policy” button. On the New Policy screen, set the name to something easily understandable (such as “read”), select the Resource Type, and add the resource of `read`. This operates as your scope, and will be what the Policy Engine will be validating.

![Screenshot of Configuring the Policy](/img/api-to-api-communication-with-aic/setting-policy.png)

_Configuring the Policy_

You’ll next be taken to the Policy screen itself. Here you can define who this policy applies to (the Subjects) and what response should be made based on the request (the Actions). Let’s set these to GRANT access to the subject of the Consumer Application.   
Hit the edit button for Subjects and in the specified condition select type of “OpenID Connect/JWT Claim”, the Claim Name of “subject”, and the Claim Value of the name of your Consumer API.

![Screenshot of Configuring the Subject](/img/api-to-api-communication-with-aic/setting-subject.png)

_Configuring the Subject_

Hit the checkmark, “Save Changes”, and then return to the Summary page. Select the edit button for Actions, add the Action “GRANT”, and select Save Changes.

![Screenshot of Configuring the Action](/img/api-to-api-communication-with-aic/setting-action.png)

_Configuring the Action_

When you return to the Summary page, your Policy should look like this:  

![Screenshot of Policy Summary Page](/img/api-to-api-communication-with-aic/policy-summary.png)

_The Policy Summary_

### Protecting an API {#protecting-an-api}

We now have an active policy set evaluating permissions to a Provider API based on a Consumer API subject and a set of scopes. We’ll now want to use the policy to validate the requests made by the Consumer API.

Go to your Consumer API’s management screen by navigating to Native Consoles \> AM \> Applications \> OAuth 2.0 \> Clients \> Consumer API Name, and then to the “OAuth2 Provider Overrides” tab.

In this section, set the following configuration:

| Setting | Value |
| :---- | :---- |
| Enable OAuth2 Provider Overrides | `True` |
| Access Token Modification Plugin Type | `SCRIPTED` |
| Access Token Modification Script | `PolicyEvalMod` |
| Scope Validation Plugin Type | `SCRIPTED` |
| Scope Validation Script | `scopeDrivenConsumerClient Scope Validation` |

These should look familiar to you \- they’re the Scripts we made back in the Prerequisite steps\! By adding these scripts, we are now directing additional scopes not defined on the consumer’s Client to be validated with the Policy Engine.

# Testing the Approach {#testing-the-approach}

To test this approach, we’ll generate an access token using the Consumer API’s client credentials with the Provider API’s scope and then introspect the token.

```bash
# Using the following constants:
# tenant-fqdn: e.g. openam-example.forgeblocks.com
# realm: e.g. realms/root/realms/alpha
# client-secret: The secret you used when creating your Consumer Client

# Access Token Request
curl --location 'https://{tenant-fqdn}/am/oauth2/{realm}/access_token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'scope=api://scopeDrivenAPI2Client/read' \
--data-urlencode 'grant_type=client_credentials' \
--data-urlencode 'client_id=scopeDrivenAPI1Client' \
--data-urlencode 'client_secret={client-secret}'
```

```bash
# Using the access_token contained in the response,

# Token Introspection
curl --location 'https://{tenant-fqdn}/am/oauth2/{realm}/introspect' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'token={access_token}' \
--data-urlencode 'client_id=scopeDrivenAPI1Client' \
--data-urlencode 'client_secret={client-secret}'
```

After introspecting your generated token, you’ll see that the scope in the response is set to `read` and that the `aud` of the request is set to `scopeDrivenAPI2Client` with a `policyResult` of `true`.

```json
{
    "active": true,
    "scope": "read",
    "realm": "/alpha",
    "client_id": "scopeDrivenAPI1Client",
    "user_id": "scopeDrivenAPI1Client",
    "username": "scopeDrivenAPI1Client",
    "token_type": "Bearer",
    "exp": 1700540274,
    "sub": "scopeDrivenAPI1Client",
    "iss": "https://{tenant-fqdn}:443/am/oauth2/{realm}",
    "subname": "scopeDrivenAPI1Client",
    "authGrantId": "...",
    "auditTrackingId": "...",
    "aud": "scopeDrivenAPI2Client",
    "policyResult": true
}
```

# Evaluating by User Context

There are cases in which you may want to present tokens not only by the client application sending the request but by the user (or data on that user) who initiated the request. An example could be that while `API1` (the Consumer API) has `read` and `write` scopes to `API2`, the user interacting with `API1` should only have the `read` scope. Dynamically scoping based on the _user_ or _user attribute data_ allows us to pass user context downstream to a microservice or API layer architecture without requiring the user to be directly associated to that layer.

Fortunately, the approach we’ve taken here already accounts for this concept - if the user is the one retrieving the token, we will have their `subject` in the request in the same way as we retrieved the API name in a client credentials request.

Firstly, let’s update our Policy to include the user’s UUID as a valid subject. To get the UUID, go to the “Raw JSON” tab on your user management page (Identities > Manage > Users > Your User) and copy the `_id` (it’s the same ID in the url of that page too).

![Screenshot of the User's UUID](/img/api-to-api-communication-with-aic/getting-uuid.png)

_The User's UUID_

Next, make sure that your Consumer application includes the `code` **response type** and the `Authorization Code` **grant type** for this example. Optionally, associate the user with the Consumer OAuth Application (under the “Applications” tab) - this will help you organize users and apps as well as show the app in the user’s self-service UI.

![Screenshot of the User associated with the scopeDrivenAPI1Client](/img/api-to-api-communication-with-aic/scoping-user.png)

_Associating the User to the ScopeDrivenAPI1Client_

Finally, add the `_id` as a subject within the policy you created in the [Creating a Policy](#creating-a-policy) section of this walkthrough. It’s important to note that you should switch the Logical Operator from “All of” to “Any of” in this example so that you can test both API and User as the Subject. Your configuration will look something like this:

![Screenshot of the Policy Summary incorporating the User](/img/api-to-api-communication-with-aic/policy-summary-with-user.png)

_Allow-listing the User's ID as a Subject in the Policy_

The neat part about policies like this is that you can have multiple logical groupings with different purposes - for example, you could have a Subject set that checks for an API as a subject, a User as a subject with a specific API (or groups of APIs!) as its client, or any custom claim passed into the request that may pass any other criteria regarding the subject requesting authorization (say, an IoT device’s location, a department/brand that the user is coming from, a risk score, etc).

Once the policy has been updated, let’s run an Authorization Code Grant flow and introspect the token to see the user’s permissions.

```bash
# Using the following constants:
# tenant-fqdn: e.g. openam-example.forgeblocks.com
# realm: e.g. realms/root/realms/alpha
# username: Your user's username
# password: Your user's password
# cookie: Your tenant's cookie, which you'll put your user's SSO token into
# client-secret: The secret you used when creating your Consumer Client

# Retrieve the user's SSO Token (you can use whatever approach you want - this is an example using Password Grant)
curl --location --request POST 'https://{tenant-fqdn}/am/json/{realm}/authenticate?authIndexType=service&authIndexValue=PasswordGrant' \
--header 'X-OpenAM-Username: {username}' \
--header 'X-OpenAM-Password: {password}' \
--header 'Content-Type: application/json' \
--header 'Accept-API-Version: resource=2.1'

# generated-sso-token: the tokenId contained in the response

# Retrieve the Authorization Code using the SSO token
curl --location 'https://{tenant-fqdn}/am/oauth2/{realm}/authorize' \
--header 'Cookie: {cookie}={generated-sso-token}' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'scope=api://scopeDrivenAPI2Client/read' \
--data-urlencode 'response_type=code' \
--data-urlencode 'client_id=scopeDrivenAPI1Client' \
--data-urlencode 'redirect_uri={scopeDrivenAPI1Client-redirect_uri}' \
--data-urlencode 'decision=allow' \
--data-urlencode 'csrf={generated-sso-token}'

# generated-authorization-code: the code contained in the response

# Exchange the Authorization Code for an Access Token
curl --location 'https://{tenant-fqdn}/am/oauth2/{realm}/access_token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'grant_type=authorization_code' \
--data-urlencode 'code={generated-authorization-code}' \
--data-urlencode 'redirect_uri={scopeDrivenAPI1Client-redirect_uri}' \
--data-urlencode 'client_id=scopeDrivenAPI1Client' \
--data-urlencode 'client_secret={client-secret}'

# Using the access_token contained in the response,

# Token Introspection
curl --location 'https://{tenant-fqdn}/am/oauth2/{realm}/introspect' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'token={access_token}' \
--data-urlencode 'client_id=scopeDrivenAPI2Client' \
--data-urlencode 'client_secret={client-secret}'
```

After introspecting your generated token, you’ll see that the scope in the response is set to `read` and that the `user_id` of the request is set to the user’s UUID with a `policyResult` of `true`. You’ll note that as per spec you won’t be able to introspect with `scopeDrivenAPI1Client` since its client_id doesn’t match the audience in the token.

```json
{
    "active": true,
    "scope": "read",
    "realm": "/alpha",
    "client_id": "scopeDrivenAPI2Client",
    "user_id": "{Your user's _id}",
    "username": "{Your user's _id}",
    "token_type": "Bearer",
    "exp": 1700540274,
    "sub": "scopeDrivenAPI1Client",
    "iss": "https://{tenant-fqdn}:443/am/oauth2/{realm}",
    "subname": "scopeDrivenAPI1Client",
    "authGrantId": "...",
    "auditTrackingId": "...",
    "policyResult": true
}
```

# Conclusion {#conclusion}

By using the Policy Engine, we can create Policy Sets grouped by API or Service with fine-grained control over scopes by Subject or Subjects. Scopes can be controlled in a centralized location and be updated on the fly without requiring the Consuming Client to have any knowledge of the additional permissions or storing multiple Client IDs and Secrets.

# Next Steps {#next-steps}

While this document provides a demonstrative approach to this concept, there are a few considerations to make when looking to productionalize.

1. Within the `PolicyEvalMod` script:  
   1. A new Agent SSO Token is generated per execution. For performance reasons, it is advisable to consider caching the token to reduce network calls.  
   2. If multiple audience-driven scopes are set in the access token request, the first audience found only is evaluated. Your business logic may dictate the precedence of which audience to select.  
2. Note that while we demonstrated creating policies that evaluate permissions for the subject of the request, you can create (and combine) any attributes in your claim \- for example, the example Policy Evaluation is already set up to handle the claim types of the subject and the referring client application (look for `"subject":{"claims":{"sub":"id=${POLICY_GATEWAY_ID},ou=agent,o=alpha,ou=services,ou=am-config","subject":"${subjectName}", "client": "${clientId}"}}}` inside `PolicyEvalMod`).  
3. As mentioned earlier in this document, it’s suggested to use Ping’s AM APIs to generate the API Client and create/update the Policies and Policy Sets. Not only can this increase the time to value, it also reduces the potential of mistakes like misspelling a Client ID.