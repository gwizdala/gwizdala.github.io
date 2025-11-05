---
draft: false
date: '2025-11-05'
title: 'Simplifying Callbacks with Library Scripts in PingOne Advanced Identity Cloud or PingAM'
description: 'Create and manage custom inputs (locally and dynamically, from APIs/CDNs/otherwise) with ease using Library Scripts'
summary: Learn how to use the Callbacks Library to speed up and simplify developing custom inputs in PingOne AIC and PingAM
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "PingAM", "JavaScript"]
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

[Callbacks](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/callbacks-supported.html) are the backbone of Orchestration (Journeys and Trees) in PingOne Advanced Identity Cloud (AIC) and PingAM. When you hit a [Hosted Page](https://docs.pingidentity.com/pingoneaic/latest/end-user/identity-cloud-hosted-pages.html), use the [SDK](https://docs.pingidentity.com/sdks/latest/sdks/index.html), or [target a Journey with an API call](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/login-using-rest.html), the information that is presented and the data you return are each wrapped inside of a callback. **Callbacks represent the interaction between the end-user and your orchestration.**

While out of the box nodes determine which callbacks to use for you, there may be cases in which you want to create functionality that doesn’t exist in those nodes. Fortunately, you can [invoke the very same callbacks using PingAM/AIC’s scripting engine](https://docs.pingidentity.com/pingoneaic/latest/am-scripting/scripting-api-node.html#scripting-api-node-callbacks) - meaning that you can build [custom scripts](https://docs.pingidentity.com/auth-node-ref/latest/scripted-decision.html) or [custom nodes](https://docs.pingidentity.com/pingoneaic/latest/journeys/node-designer.html) to serve and respond to whatever data, inputs, and methods that you need.

That being said, invoking these callback methods can be a bit confusing. Each callback type has their own getters and setters, and the way that they set and get information is slightly different depending on what they do. And while there is documentation on how the callback will present the information (important for the SDK and the API) there’s very little documentation on how to set them up in the script in the first place.

That’s where [Library Scripts](https://docs.pingidentity.com/pingoneaic/latest/am-scripting/library-scripts.html) come in.

By using a Library Script, we can wrap all of that complexity surrounding callbacks with a simple and reusable function that handles the logic of rendering and retrieving each type for you.

# Simplifying Callbacks

This How-To focuses on how to use the `library_callbacks` script within Scripted Decision Nodes. This script, along with an example Scripted Decision Node, can be found here:

* Library Script: [library\_callbacks](https://github.com/gwizdala/lib-ping/blob/main/Library%20Scripts/library_callbacks.js)  
* Example Scripted Node (outcomes `Success` and `Error`): [library\_callbacks\_example](https://github.com/gwizdala/lib-ping/blob/main/Library%20Scripts/library_callbacks_example.js)

The document is broken down into the following parts:

1. [Using the Callbacks Library](#using-the-callbacks-library)  
2. [Example Use Cases](#example-use-cases)

By the end of this How-To you’ll have a library script that allows you to more easily interact with interactive callbacks and some example use cases to apply it.

This How-To expects a level of familiarity with Journeys and Journey Scripting.

# Using the Callbacks Library

The Library Script can be found here: [https://github.com/gwizdala/lib-ping/blob/main/Library%20Scripts/library\_callbacks.js](https://github.com/gwizdala/lib-ping/blob/main/Library%20Scripts/library_callbacks.js) 

## Importing the Library Script

If you have [FroDo]({{< ref "a-love-letter-to-frodo-cli" >}}) you can import the script using the function provided in the header of the file.

Alternatively, if you’d like to import via the UI:

To create a Library Script, go to Scripts → Auth Scripts, Select “New Script”, and then select “Library” as the Script Type (under the “Other” category).

Let’s call this new script `library_callbacks`. I like starting all of the library scripts with the `library_` prefix because it is easy to search and quickly tells me what the script is for.

In the code section, copy/paste the library script. Once saved, the library script is now available throughout Journeys using Scripted Decision Nodes. 

## Using the Functions

Using the Callbacks Library is as simple as using its setter and its getter functions. These functions are universally supported across every implementation. There is an optional formatter function that works out of the box in hosted pages or in the SDK if you support the [ScriptTextOutputCallback](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/callbacks-backchannel.html#scripttextoutputcallback).

A common example of using these functions would look something like this:

```javascript
var libraryCallbacks = require("library_callbacks");

if (callbacks.isEmpty()) {
    // Setter
    libraryCallbacks.renderInteractiveCallbackOptions(INPUTS, callbacksBuilder);
    // Optional formatter
    libraryCallbacks.formatInteractiveCallbackOptions(INPUTS, callbacksBuilder);
} else {
    // Getter
    var responses = libraryCallbacks.gatherInteractiveCallbackResponses(INPUTS, callbacks);
    // Use the responses.
}
```

Every function takes a list of **inputs**, which define what callbacks should be presented to the user and then gathered once they continue. Both `callbacksBuilder` and `callbacks` are available by default within your node and can be passed as-is into the library script.

### The Inputs Object

The `inputs` parameter represents the callbacks you are looking to render, the data within those callbacks, and the order in which they appear. It’s best practice to define your inputs in a constant at the top of your file.

You’ll represent your inputs as an array of objects. The following keys are available in those objects:

**Available in all functions**
| Key | Type | Description | Required |
| ----- | ----- | ----- | ----- |
| `id` | String | The key returned in the responses gathered from `gatherInteractiveCallbacks`. As an example, if my `id` was `nameInput`, I could get the value using `responses.nameInput`. The ID should be a unique value. | Yes |
| `type` | String | The type of [Interactive Callback](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/callbacks-interactive.html) to render/gather. | Yes |
| `label` | String | The label that appears along with the input (e.g. “Username” or “Email Address”). The label isn’t required for callbacks that don’t render an input onscreen, such as the [HiddenValueCallback](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/callbacks-interactive.html#hiddenvaluecallback) (hidden input). | Yes for on-screen inputs |
| `value` | String | A default value to provide to the input. | No |
| `delimiter` | String | Support multi-value inputs when rendering a [NameCallback](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/callbacks-interactive.html#NameCallback) (text input). The delimiter determines what string value splits the input provided by the user, and will present the result as an array of strings. | No |
| `choices` | Array[Object] | If rendering a [ChoiceCallback](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/callbacks-interactive.html#ChoiceCallback) (dropdown list), the choices determine what value will be displayed and collected. The object contains two keys: `label`: The value to display in the dropdown to the user, and `value`: The value to collect when selected. Complex objects, such as arrays or JSON, are supported.  | ChoiceCallbacks only |

**Requires `formatInteractiveCallbackOptions`**
| Key | Type | Description | Required |
| ----- | ----- | ----- | ----- |
| `required` | Boolean | Whether or not the input is required. | No |
| `description` | String | An additional description that appears below the input in smaller text. | No |
| `tooltip` | String | A clickable “?” icon that provides additional information. HTML strings are supported. | No |

# Example Use Cases

The following use cases illustrate some common ways you can use the Callbacks library.

## Simplifying Management of Custom Forms

Let’s say that we want to create a custom form that lets a user pass in a client ID and secret for a [PingOne Worker Application](https://docs.pingidentity.com/pingone/applications/p1_configurerolesforworkerapplication.html) in order to generate an access token for that application. We’ll want to provide the client ID and environment ID as a text input, the secret as a password field, and will need a way for the user to specify their deployment region since the API URL changes.

Using standard callbacks methods, we might make something like this:

```javascript
var NodeOutcome = {
    SUCCESS: "Success",
    ERROR: "Error"
};

(function() {
    outcome = NodeOutcome.SUCCESS;
    try {
        if (callbacks.isEmpty()) {
            callbacksBuilder.nameCallback("Client ID");
            callbacksBuilder.passwordCallback("Client Secret", false);
            callbacksBuilder.nameCallback("Environment ID");
            callbacksBuilder.choiceCallback(
                "PingOne Regional Domain", 
                ["North America (Non-Canada)", "Canada", "European Union", "Asia-Pacific"], 
                0, 
                false
            );
        } else {
            var responses = {
                clientId: callbacks.getNameCallbacks().get(0),
                clientSecret: callbacks.getPasswordCallbacks().get(0),
                environmentId: callbacks.getNameCallbacks().get(1),
                region: callbacks.getChoiceCallbacks().get(0) // This equals the index of the choice, not the value in the label
            };
            // load the worker here
        }
    } catch(e) {
        logger.error(e);
        // Set the error message in a callback so that we can show it back to the user
        callbacksBuilder.textOutputCallback(2, JSON.stringify(e));
        outcome = NodeOutcome.ERROR;
    }
    action.goTo(outcome);
}());

```

Already this is getting a bit hairy. Each callback has its own method and you have to get each callback type (and reference their individual index order) separately. If you change the order of the above callbacksBuilder calls you’ll have to redo the indices in the get methods to match. On top of that, the labels in the choice collector likely don’t match the underlying data that you want to use in your functions (there’s a reason why selects in HTML have labels *and* values) so you’ll probably have to create a mapping that matches the array of labels to something meaningful. As you add code into your main script understanding, managing, and troubleshooting callbacks becomes painful.

Let’s simplify this.

Using the Callbacks Library, the same script would look something like this:

```javascript
var libraryCallbacks = require("library_callbacks");

var NodeOutcome = {
    SUCCESS: "Success",
    ERROR: "Error"
};


var INPUTS = [
    {
        id: "clientId",
        type: "NameCallback",
        label: "Client ID"
    },
    {
        id: "clientSecret",
        type: "PasswordCallback",
        label: "Client Secret"
    },
    {
        id: "environmentId",
        type: "NameCallback",
        label: "Environment ID"
    },
    {
        id: "region",
        type: "ChoiceCallback",
        label: "PingOne Regional Domain",
        choices: [
            {
                label: "North America (Non-Canada)",
                value: "https://api.pingone.com"      
            },
            {
                label: "Canada",
                value: "https://api.pingone.ca"
            },
            {
                label: "European Union",
                value: "https://api.pingone.eu"
            },
            {
                label: "Asia-Pacific",
                value: "https://api.pingone.asia"
            }
        ]
    }
];

(function() {
    outcome = NodeOutcome.SUCCESS;
    try {
        if (callbacks.isEmpty()) {
            libraryCallbacks.renderInteractiveCallbackOptions(INPUTS, callbacksBuilder);
        } else {
            var responses = libraryCallbacks.gatherInteractiveCallbackResponses(INPUTS, callbacks);
            // load the worker here
        }
    } catch(e) {
        logger.error(e);
        // Set the error message in a callback so that we can show it back to the user
        callbacksBuilder.textOutputCallback(2, JSON.stringify(e));
        outcome = NodeOutcome.ERROR;
    }
    action.goTo(outcome);
}());
```

Now all of your inputs are statically defined at the top of your file and are defined in a consistent way. Moving an input around is as simple as putting it in a different spot in the list and there aren’t any downstream implications on how the data is being referenced later in your script. 

The library script lets you handle your functionality separately from your design.

## Dynamically Rendering Form Data

Since the Callbacks Library controls content in a consistent way, you can quickly and easily render dynamic content retrieved from external sources, for example AM, IDM, a CDN, or an API call.

Let’s say that rather than generating an access token, we want to register that PingOne App as a [Social Identity Provider](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/social-idp-client-reference.html). You could manually create each individual  input, but with over 20 required inputs to structure even with the inputs array it could take quite some time. And besides, the information needed can be retrieved from API \- so who’s to say we can’t use that instead?

{{<details title="Example Dynamic Social Identity Provider Loader (Click to View)">}}
```javascript
/*
Load OIDC inputs for the user to fill out. If data was already provided from shared state it will be auto-applied.

This script does not need to be parametrized. It will work as-is.
This script expects the `library_callback` library script to function.

This script expects the following values in shared state:
-  accessToken: An access token of a PingOne AIC Service Account with fr:am:* privileges

This script expects the following outcomes:
- Success
- Error

author: @gwizdala
*/

//// CONSTANTS
var CONSTANTS = {
    HOST: requestHeaders.get("host").get(0),
    API_VERSION: "v1",
    SHARED_STATE_KEY: "OIDCInputValues",
    SHARED_STATE_RESPONSE_KEY: "OIDCServiceResponse",
    ERROR_MESSAGE: 'ErrorMessage',
    OIDC_CONFIG_TYPE: "oidcConfig"
};

var NodeOutcome = {
    SUCCESS: "Success",
    ERROR: "Error"
};

var libraryCallbacks = require('library_callbacks');

//// HELPERS
//// API Helpers
/** 
 * Return Schema details for the selected Social Identity Provider Secondary Configuration Type
 * Requires fr:am:* permissions
 * @param {String} accessToken - The access token for authorization.
 * @param {String} configTypeId - The ID of the configuration type to retrieve schema details for.
 * @returns {Object} - The schema details for the specified configuration type.
 * @throws {Error} - If the API call fails or returns an error status.
 */
function getSocialIdentityProviderConfigurationTypeSchema(accessToken, configTypeId) {
     var headers = {
        "Accept": "application/json, text/plain, */*",
        "Content-Type": "application/json",
        "Accept-API-Version": "protocol=1.0,resource=1.0"
    };

    var options = {
        method: "POST",
        headers: headers,
        token: accessToken
    };

    var requestURL = `https://${CONSTANTS.HOST}/am/json/realms/root/realms/alpha/realm-config/services/SocialIdentityProviders/${configTypeId}?_action=schema`;

    var response = httpClient.send(requestURL, options).get();

    if (response.status === 200) {
        var payload = JSON.parse(response.text());
        return payload.properties;
    } else {
        throw({
            status: response.status,
            statusText: response.statusText,
            text: response.text()
        });
    }
}

//// MAIN
(function() {
    outcome = NodeOutcome.SUCCESS;
    try {
        var accessToken = nodeState.get('accessToken');
        var oidcInputValues = nodeState.get(CONSTANTS.SHARED_STATE_KEY);
        var inputs = [];
        if (!oidcInputValues) {
            var schema = getSocialIdentityProviderConfigurationTypeSchema(accessToken, CONSTANTS.OIDC_CONFIG_TYPE);
            // Get the properties, sort them, and then create input fields for each property
            // Each object has a "propertyOrder" key that dictates the order in which they appear
            var sortedKeys = Object.keys(schema).sort(function(a, b) {
                return schema[a].propertyOrder - schema[b].propertyOrder;
            });

            sortedKeys.forEach(function(key) {
                var field = schema[key];
                var input = {
                    id: key,
                    label: field.title,
                    value: !!field.default ? field.default : (!!field.exampleValue ? field.exampleValue : ""),
                    type: "NameCallback",
                    tooltip: !!field.description ? field.description : "",
                    required: !!field.required
                };

                if (field.type == "string" && !!field.enum) {
                    input.type = "ChoiceCallback";
                    var choices = [];
                    for (var i = 0; i < field.enum.length; i++) {
                        choices.push({
                            label: field.enumNames[i],
                            value: field.enum[i]
                        });
                    }
                    input.choices = choices;
                } else if (field.type == "string" && field.format == "password") {
                    input.type = "PasswordCallback";
                } else if (field.type == "integer") {
                    input.type = "NumberAttributeInputCallback";
                } else if (field.type == "array") {
                    input.delimiter = ' ';
                    input.label = field.title + " (space-separated)";
                } else if (field.type == "boolean") {
                    input.type = "BooleanAttributeInputCallback";
                    input.value = !!field.default;
                }

                inputs.push(input);
            });
            nodeState.putShared(CONSTANTS.SHARED_STATE_KEY, JSON.stringify(inputs));
        } else {
            inputs = JSON.parse(oidcInputValues);
        }

        if (callbacks.isEmpty()) {
            var errorMessage = nodeState.get(CONSTANTS.ERROR_MESSAGE);
            if (!!errorMessage) {
                callbacksBuilder.textOutputCallback(2, errorMessage);
            }
            
            libraryCallbacks.renderInteractiveCallbackOptions(inputs, callbacksBuilder);
            libraryCallbacks.formatInteractiveCallbackOptions(inputs, callbacksBuilder);
        } else {
            var responses = libraryCallbacks.gatherInteractiveCallbackResponses(inputs, callbacks);
            nodeState.putShared(CONSTANTS.SHARED_STATE_RESPONSE_KEY, JSON.stringify(responses));
            inputs.forEach(function(input) {
                if (responses[input.id]) {
                    input.value = responses[input.id];
                }
            });
            nodeState.putShared(CONSTANTS.SHARED_STATE_KEY, JSON.stringify(inputs));
        }
    } catch(e) {
        logger.error(e);
        // Set the error message in a callback so that we can show it back to the user
        var errorMessage = typeof e === 'string' ? e : (e.text ? e.text : JSON.stringify(e));
        callbacksBuilder.textOutputCallback(2, errorMessage);
        outcome = NodeOutcome.ERROR;
    }
    action.goTo(outcome);
}());
```
{{</details>}}

Running this script with a valid access token gives us the following page:

![A screenshot of some of the dynamically rendered inputs from above](/img/simplifying-callbacks-aic/dynamic-oidc.png) 
*And that’s not even half of the inputs!*

In about 150 lines we accomplished something that could manually take four to five times that number to accomplish. And if you swapped out a different endpoint, all you need to do is map the values to the input array that you’re looking to show to the user.

With this strategy you can make short work of rendering data from custom objects, localized values in a CDN, and external APIs without the need to resort to hacking in a `custom_` attribute on a user to do so.

## Rendering Journeys from Other Tenants/Deployments

If the Callbacks Library renders callbacks, and callbacks are returned when targeting a Journey via an API call, who’s to say it can’t render callbacks presented from another Journey?

With the advent of [no-session Journeys](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/configure-authentication-trees.html#configure-nosession-tree) we can create experiences that provide us information from inputs without returning an sso token - meaning we can perform actions with user input interacting with another tenant/deployment without resorting to an IDM call or generating an unnecessary and unmanaged session. And with the library, rather than building a static set of inputs that has to match the other tenant and then populating the value for the call, you can natively and dynamically render the callbacks presented from another Journey.

As an example, let’s say that I had two tenants: Tenant A and Tenant B. Tenant A has all of my customers and Tenant B has my employees. Our helpdesk employees log into Tenant B but also have some permissions to manage users within Tenant A, say resetting a password.

Without requiring our helpdesk employee to have two accounts in two tenants, we could create a Journey in Tenant B that:

1. Checks the validity of the initial request using a key (say, a JWT/PEM/secret passed through the header stored in both Tenant A and B’s ESVs)  
2. Surfaces users to the helpdesk agent based on the permissions of that key  
3. Performs additional verification (akin to backchannel) to the user, such as Identity Verification or push with a second factor like PingID  
4. Returns a list of actions available for the helpdesk user to perform  
5. Performs the selected action

Within Tenant A, you’d need a Journey that passes that initial request, and upon return it’d get the callbacks it needs to render. Now, you could hardcode Tenant A to *always* look for a user list, and then a verification screen, and then a list of actions, and so on - but what if Tenant B’s security posture changes? Rather than relying on your list, you can render the data dynamically given the callbacks provided.

And now why would we want to run this Journey in Tenant A, instead of hitting Tenant B directly? Because to do so you would either have to add your helpdesk users to Tenant B,  give all of your helpdesk agents access to that header key, or expose the Journey in a way that anyone could target it. By running Tenant B’s Journey *inside* of a Journey in Tenant A, you have complete control over locking down that interaction - including who can access that Journey given their permissions in Tenant A.

# Conclusion

With this How-To you now have a powerful toolset that can greatly reduce the time to value when working with custom inputs.

# Enhancements

The callbacks library is built to be extended so that you can fit it to match your specific use cases. Use the examples below as a guide on implementing some common enhancements.

## Adding More Callbacks

I’ve added in the interactive callbacks that I find myself using the most frequently when building inputs. That being said, there are plenty more that can be added.

To add a new callback, 

1. In `renderInteractiveCallbackOptions` add a new case that uses the name of the callback you’re adding. Inside that case statement, provide the appropriate callbacksBuilder function and include any input values that you need (e.g. `input.label` or `input.value`)  
2. In `gatherInteractiveCallbackResponses` add a new case that uses the name of the callback you’re adding. Inside that case statement, set `responses[input.id]` equal to how you’d like to retrieve the value (for the most part, it’s going to be `currentCallback` but sometimes that value needs to be formatted)

If you decide to add Backchannel or Read-Only callbacks, it may be to your advantage to create a new setter as you likely won’t need to get anything from those values.

## Adding more Formatting

All formatting is done via `formatInteractiveCallbackOptions`. Update the CSS or change the HTML options there to change how your custom formatting is rendered.