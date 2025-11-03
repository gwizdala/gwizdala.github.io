---
draft: false
date: '2025-09-24'
title: 'Brand Copy Localization in PingOne Advanced Identity Cloud'
description: 'Set up copy for different inputs, callbacks, pages, Journeys, etc. off of any identity object and load the content that matches the language preferences set by your user, browser, or API call.'
summary: Learn how to model, store, and reference dynamic copy per brand, partnership, organization, or user.
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "PingAM", "PingIDM", "JavaScript"]
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

PingOne Advanced Identity Cloud (AIC) enables you to build rich user experiences that dynamically update localization depending on context such as the end-user’s language preferences.

But what if you want to provide different messaging, **in the same language**, depending on factors such as the brand or partnership that the user is interacting with? And, rather than having your centralized administration control all of the localizations, what if you’d like each brand to control their own copy?

## Managing Copy Localization by Brand

This How-To will cover extending your Identity (IDM) schema (specifically, the Organization Model) to hold localized copy, and then walk through applying that copy to specific pages within Journeys. It’s broken down into two parts:

1. [Defining the Localization Properties](#defining-the-localization-properties)  
2. [Controlling Journey Localization with your Dynamic Properties](#localizing-journeys)

By the end of this How-To you’ll have a Journey that can dynamically swap the Header, Description, and Footer of a Page Node with content provided from an Organization. You’ll additionally have a template within your Organization that you could extend for different copy, or to place within an entirely separate managed object entirely.

This How-To expects a basic level of familiarity with Journeys, Journey Scripting, and IDM Managed Objects.

# Defining the Localization Properties

To start, we’ll need a place to store our localization. I’m using the [Organization Model]({{< ref "understanding-the-org-model-aic" >}}) since it has some built-in delegated administration, but you could just as easily use another Managed Object (including a custom model). If you decide to use another managed object, you’ll want to decide how to define an [Internal Role](https://docs.pingidentity.com/pingoneaic/latest/identities/roles-assignments.html#internal_roles) that restricts what copy your brand managers can view.

Head over to your `Alpha_Organization` managed object config in your IDM native console (under Native Consoles → Identity Management → Configure → Managed Objects) and create a new attribute with the following properties:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `locale` | Localization | Object | False |

Select that property and add the following properties to it, all with “Required” set to false:

| Name | Label (Optional) | Type |
| ----- | ----- | ----- |
| `headers` | Headers | Array |
| `descriptions` | Descriptions | Array |
| `footers` | Footers | Array |

![A screenshot of the IDM Managed Locale Object Property within the Alpha Organization](/img/brand-copy-localization-aic/idm-property-locale.png) 
*The Locale Object*

Finally, inside of each of those new properties, set the “Item Type” to “Object” and add the following properties with “Required” set to true:

| Name | Label (Optional) | Type | Description |
| ----- | ----- | ----- | ----- |
| `language` | Language | String | ISO 639 Code (e.g. "en", "en-us") |
| `message` | Message | String | The localized message |

![A screenshot of the IDM Managed Headers Object Property within the Locale Object](/img/brand-copy-localization-aic/idm-property-headers.png) 
*Inside the Headers Object*

At this point we’ve created a means to store both the language code and the localized message for the header, the description, and the footer. In the Platform UI, head over to an Organization we’ve created to see what that looks like for our delegated admins.

![A screenshot of a example organization in which the Locale Array and its object properties are visible](/img/brand-copy-localization-aic/example-org-unpopulated.png)  
*The Example Organization with Localization Properties*

Right now we don’t have any localizations defined. Let’s add in some messages specific to this Organization. Since we’ve defined each property to be an array, we can add as many language options as we want.

![A screenshot of a example organization in which a header, description, and footer message have been applied](/img/brand-copy-localization-aic/example-org-populated.png)   
*Applying Localized Messages to the Example Organization*

You now have localized messages stored on individual Organizations (e.g. brands, departments, partnerships, etc.). Continue on to the next section when you’re ready to apply these values in your user experiences, otherwise if you want to do some more extensions:

* [Learn how to delegate permissions to manage the Localization object]({{< ref "setting-delegated-admin-privileges-in-aic" >}})

# Localizing Journeys

Let’s build out a Journey that we can populate with the values we’ve gathered from our Organization.

## Adding Default Values

Navigate to “Journeys” inside of your tenant and create a brand-new Journey with the following details:

| Input | Value |
| ----- | ----- |
| Name | DynamicLocalization |
| Identity Object | Alpha realm \- Users `managed/alpha_user` |
| Description (optional) | An example journey showcasing how you can enforce dynamic localization based on managed object properties |

Your Journey editor will start out empty. Connect your Start Node to a Page Node and inside of it add a Platform Username and Platform Password Node, as if we’re building a login screen. Your Journey editor and page preview (the URL is found in the top-right corner of your screen) should look something like this:

![A screenshot of the Journey editor which has a Page Node, a Platform Username Node, and a Password Node](/img/brand-copy-localization-aic/journey-initial.png)  
![A screenshot of the end-user view of the original Journey in which no copy has been applied to the page node](/img/brand-copy-localization-aic/journey-initial-rendered.png)   
*The Starting Journey*

Right now we don’t have any copy at all. Let’s define some defaults that our Page Node can fall back on if nothing is provided by our brands.

In your editor, click on the Page Node. You’ll see that there are input sections for Page Header, Page Description, and Page Footer, and that you add key-value pairs to define the language code and the message for each section. This should look familiar \- it’s the same setup we made in the Organization Object.

![A screenshot of the Page Node property view in which custom values are being added - in this case a Spanish description](/img/brand-copy-localization-aic/journey-defaults.png)
*Adding Default Values*

After adding some values for our header, description, and footer, we can see these values applied to our page.

![A screenshot of the default values applied to the end-user rendered view of the page](/img/brand-copy-localization-aic/journey-defaults-rendered.png)  
*Rendering Defaults*

## Localizing with a Script

Now that we have a default message, let’s add a way to override those defaults with the messages we’ve stored on our brands.

Back within our Journey, let’s add two new Scripted Decision Nodes: one which will find what Organization we’re looking for by a query parameter we’ll pass in and another that retrieves the Locale Object from that Organization. If you’ve read the How-To on [Dynamic Branding]({{< ref "dynamically-branding-journeys-in-aic" >}}) this will be very familiar to you.

The first scripted decision node, entitled “Get Org ID by Query Parameter”, will have the outcomes `Found`, `Missing`, and `Error`. 

*Get Org ID by Query Parameter*

```javascript
/*
 Capture the URL Query Param specified
 
 This script does not need to be parametrized. It will work properly as is.
 
 The scripted decision node needs the following outcomes defined:
 - Found
 - Missing
 - Error
 
 Author: @gwizdala
 */
//// CONSTANTS
// Change the query parameter below if you want to gather a different value.
// Ideally, this sort of functionality could be handled in a library script.
var QUERY_PARAM = "orgId";
var SHARED_STATE_KEY = "orgId";
// Request Params
var HOST = requestHeaders.get("host").get(0); // e.g. openam-example.forgeblocks.com

var NodeOutcome = {
    FOUND: "Found",
    MISSING: "Missing",
    ERROR: "Error"
};

//// MAIN
(function () {
    try {  
      // Gather the Query Parameter & pass into shared state
      if (requestParameters.get(QUERY_PARAM)) {
        nodeState.putShared(SHARED_STATE_KEY, decodeURI(requestParameters.get(QUERY_PARAM).get(0)));
        outcome = NodeOutcome.FOUND;
      } else {
		outcome = NodeOutcome.MISSING;
      }
    } catch (e) {
        logger.error(e);
        outcome = NodeOutcome.ERROR;
    }
    
    action.goTo(outcome);
}());
```

The next scripted decision node, entitled “Get Locale From Organization Metadata”, will have the outcomes `Success` and `Error`. 

```javascript
/*
Utilizing the organization ID provided gather the locale information

This script does not need to be parametrized. It will work properly as is.
 
 The scripted decision node needs the following outcomes defined:
    - Success
    - Error
 
 Author: @gwizdala
 */

//// CONSTANTS
var REALM = 'alpha';
var NodeOutcome = {
  SUCCESS: "Success",
  ERROR: "Error"
};

//// MAIN
(function () {
    outcome = NodeOutcome.ERROR;

    try {
        var orgId = nodeState.get('orgId');

        if (null != orgId) {
            var queryFilter = `_id eq "${orgId}"`;
            var fields = 'name,locale';
            
            var orgMetadata = openidm.query(`managed/${REALM}_organization`, { 
              "_queryFilter": queryFilter,
              "_fields": fields
            }).result;
  
            if (null != orgMetadata && orgMetadata.length === 1) {
                var orgMetadataJson = JSON.parse(orgMetadata[0]);
                var locale = orgMetadataJson.locale;
                nodeState.putShared('locale', locale);
                outcome = NodeOutcome.SUCCESS;
            } else {
                throw('No Organization metadata found.');
            }
        } else {
            throw('No Organization ID found in shared state');
        }
    } catch(e) {
        logger.error(e);
        outcome = NodeOutcome.ERROR;
    }

    action.goTo(outcome);
}());
```

Attach the `Missing` outcome of the Get Org ID from Query Parameter Node and the `Success` outcome of your Get Locale Node to your Page Node \- that way your page renders whether or not an organization is found and will fall back to your default value. Connect the `Error` outcomes to the Failure Node.

Finally, drag and drop in a Scripted Decision Node **into your Page Node**, give it the name “Set Locale on Page” and the outcome `Success`.

```javascript
/*
Checks to see if locale is stored in shared state.
If so, it will use the locale provided to update the current page node description and header to match.
If not, it will leave the page as is.
Additionally, the user's langauage preference is checked to ensure that the right locale is used.
If the user's language preference is not found, the default locale is used.

This script expects the following optional values in shared state:
-  locale: The locale to use for the current user session.

This script does not need to be parametrized. It will work properly as is.
 
 The scripted decision node needs the following outcomes defined:
    - Success

 While this script may fail, it will error silently and not impact the user experience.

author: @gwizdala
*/

//// CONSTANTS
var CONSTANTS = {
    HOST: requestHeaders.get("host").get(0)
};

var NodeOutcome = {
    SUCCESS: "Success"
};

/**
 * Gets the current locale and then checks to see if the provided locales have a matching language.
 * If a matching language is found, it will return that locale.
 * If not, it will return null.
 * @param {Array} localeMessages - An array of locale objects to check against.
 * @returns {Object|null} - The matching locale object or null if no match is found.
 */
function getCurrentLocale(localeMessages) {
    if (!localeMessages || !Array.isArray(localeMessages) || localeMessages.length === 0) {
        return null;
    }

    var userLangs = requestHeaders.get("accept-language").get(0).split(",");
    for (var j = 0; j < userLangs.length; j++) {
        var userLang = userLangs[j].trim().split(';')[0].toLowerCase();
        for (var i = 0; i < localeMessages.length; i++) {
            if (localeMessages[i].language.toLowerCase() === userLang) {
                return localeMessages[i];
            }
        }
    }
    return null;
}

/**
 * Updates the header, description, and footer of the current node to match the locale provided in shared state.
 * @param {Object} locales - An object containing arrays of locale message objects.
 * @returns {null}
 */
function renderLocale(locales) {
    var header = getCurrentLocale(locales.headers);
    var description = getCurrentLocale(locales.descriptions);
    var footer = getCurrentLocale(locales.footers);

    // Note we are using "textContent" to avoid XSS issues.
    // If you want to allow HTML, you can use "innerHTML" instead, but be aware of the risks.
    var script = `\
        function updateHeader(message) {\
            if (!message) return;\
            const headerBlock = document.querySelector('.card-header.login-header');\
            if (headerBlock) {\
                const header = headerBlock.querySelector('h1');\
                if (header) {\
                    header.textContent = message;\
                }\
            }\
        }\
        function updateDescription(message) {\
            if (!message) return;\
            const descriptionBlock = document.querySelector('.card-header.login-header');\
            if (descriptionBlock) {\
                const description = descriptionBlock.querySelector('p');\
                if (description) {\
                    description.textContent = message;\
                }\
            }\
        }\
        function updateFooter(message) {\
            if (!message) return;\
            const footerBlock = document.querySelector('.card-footer');\
            if (footerBlock) {\
                footerBlock.textContent = message;\
            }\
        }\
        function updateLocales() {\
            updateHeader(${header && header.message ? '`' + header.message.replace(/`/g, '\\`') + '`' : 'null'});\
            updateDescription(${description && description.message ? '`' + description.message.replace(/`/g, '\\`') + '`' : 'null'});\
            updateFooter(${footer && footer.message ? '`' + footer.message.replace(/`/g, '\\`') + '`' : 'null'});\
        }\
        if (document.readyState === 'loading') {\
            document.addEventListener('DOMContentLoaded', updateLocales);\
        } else {\
            updateLocales();\
        }\
    `;

    callbacksBuilder.scriptTextOutputCallback(script);
}

//// MAIN
(function() {
    outcome = NodeOutcome.SUCCESS;
    try {
        // check if we've already loaded this callback
        if (callbacks.isEmpty()) {
            // gather the locale
            var locales = nodeState.get("locale");
            if (!locales) {
                logger.debug("No locales found in Node State.");
            } else {
                // render the locale using a scripttextoutputcallback
                renderLocale(locales);
            }
        }
    } catch(e) {
        logger.error(`Error while setting locale: ${e}`);
    }
    action.goTo(outcome);
}());
```

Your Journey should look like this:  

![A screenshot of the Journey editor which the three scripts have been applied](/img/brand-copy-localization-aic/journey-scripted.png)  
*Getting and Setting Locale*

## Testing Localization

Open the Preview URL. By default, since no Organization is set, you should see the same default message you saw before.

Now, let’s target some branding based on our Organization. 

On the window with your Journey, modify the Journey URL to contain the query parameter `orgId` where the value of that parameter is the GUID for your Organization (you can find this in the URL when you are managing the org in your tenant, i.e.   
`https://{your-domain}/platform/?realm=alpha#/managed-identities/managed/alpha_organization/{the-org-id}`)

Your full Journey URL will look something like this:

```
https://{your-domain}/am/XUI/?realm=/alpha&authIndexType=service&authIndexValue=DynamicLocalization&orgId={the-org-id}
```

Loading this URL, you’ll find that the Header, Footer, and Description have changed to the values set in the matching language on your Organization.

![A screenshot of the rendered journey in which the custom english copy has been applied](/img/brand-copy-localization-aic/journey-scripted-rendered-english.png)
*Rendering the English Copy*

Now, what if your Organization doesn’t have any custom copy in the user’s language? The script “Set Locale on Page” only overrides the default messages if there’s a matching message keyed to the correct language code, rendered in order of the user’s language preferences. To test this, change your browser setting to a language that you don’t have stored on your Organization object or remove the messages that match your browser preferences on that Organization.

In my example, I’ve set my browser language to Spanish. I have defaults in Spanish on my page node but only have an override for the Description within my organization. When rendering the Journey, I see my defaults in the Header and Footer and my override in the Description.

![A screenshot of the rendered journey in which the custom spanish copy has been applied to parts of the page](/img/brand-copy-localization-aic/journey-scripted-rendered-spanish.png) 
*Branding Portions of the Page*

An important note: If the user’s browser is set to a language that neither the overrides or the defaults have, the Page Node will fall back to English by default. This has nothing to do with the Scripted Decision Node you’ve added \- it’s default behavior for the page itself.

# Conclusion

We now can update local copy based on configuration managed on each individual Organization/Brand/Partnership.

The Journey that we built can be found here:

* [DynamicLocalization.json](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/brand-copy-localization-aic/DynamicLocalization.json)

This same paradigm can be extended to apply to different pages and user experiences \- just add more object arrays that you can reference within your Journey. If localizations are managed by a centralized marketing team, consider managing the copy in a separate managed object and linking it to your brand or user via a relationship.

With this How-To you have created a reusable, extensible way to set localized copy per individual brand, partnership, user, or otherwise. You also have the means to modify your schema to match who should control this data and what pages this data should update.