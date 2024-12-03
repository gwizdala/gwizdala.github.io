---
draft: false
date: '2024-11-16T02:00:00+00:00'
title: 'Dynamically Branding in PingOne Advanced Identity Cloud, Part 1: Changing the Theme'
description: 'Part 1 of 2 in the series Dynamically Branding in PingOne Advanced Identity Cloud'
summary: Learn how to dynamically swap the Hosted Page and Email Template dynamically inside a PingOne AIC Journey
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "JavaScript", "PingIDM"]
types: ["Coding"]
cover:
  image:
  alt:
  caption:
  relative: false
ShowToc: true
author: David Gwizdala
---

# Dynamically Changing the Theme within a PingOne AIC Journey

There’s a high likelihood that your organization requires more than one experience for your users. This could be due in part to:

1. Owning/managing multiple brands  
2. Business partnerships linking out to your service  
3. Sub-departments or specialty services (e.g. a pharmacy login for a grocery store)  
4. Franchises/Coalitions/Leagues/Teams each  
5. Regional advertising, branding, and localizations

PingOnce Advanced Identity Cloud (AIC) easily enables you to create unique themes, both for the end-user experiences ([Hosted Pages](https://docs.pingidentity.com/pingoneaic/latest/end-user/customize-login-enduser-pages.html)) and notifications/messaging ([Email Templates](https://docs.pingidentity.com/pingoneaic/latest/tenants/email-templates.html)). While you can assign these themes at a Default, Journey, and Page level this method of assignment can become difficult to manage as more and more themes are required for your business. To handle a larger scale, enabling branding teams to create and apply theming without having to update a single Journey, we need to consider a more dynamic approach.

This guide shows you how to quickly and dynamically change the theme inside an AIC journey based on the context we know about the user entering into the Journey. The example provided will get this data according to what is set on the Organization the user is interacting with at the time, however the same concept applies to any method you retrieve the appropriate theming, for example:

* An authorization decision returning theming information  
* The host that the user has redirected from  
* The client that initiated the request  
* Information on the user directly, e.g. their email domain, location, saved preferences, etc.

## Setup

In this example, we are managing the theme by setting the appropriate Hosted Page and Email Templates on an Organization. Organizations are a built-in construct in Advanced Identity Cloud that let us set up delegated, templated data which can be associated with users and referenced throughout the platform (in our case, a Journey) - which makes them a perfect place to store unique configurations like theming.

### Configuring the Managed Object

Let’s configure our Organization to hold information about the Hosted Page theme and Email Template we’d like to use. We’ll call these Theme **Overrides**.

To start, head over to your `Alpha_Organization` managed object config in your IDM native console (under Native Consoles → Identity Management → Configure → Managed Objects) and create a new attribute with the following properties:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `themeOverrides` | Theme Overrides | Object | False |

Select that attribute and add the following properties to it:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `themeName` | Theme Name | String | False |
| `emailVerification` | Verification Email Template Name | String | False |

Your `themeOverrides` object should look like this:

![The Theme Overrides Object Attribute](../images/1-themeOverrides.png)

_The Object Attribute inside Organization Object_

With this configuration in place, when we head over to an Organization we’ve created we’ll see a new tab entitled “Theme Overrides” with the two options we’ve configured.

![The Theme Overrides Object inside the "Example" Organization](../images/1-themeOverridesEditor.png)

_The Theme Overrides in Action_

> **Note:** When setting up attributes it’s advisable to add Descriptions (which you’ll see in the screenshot above). Since it isn’t necessary for this example, feel free to add them later if you’d like.

### Building the Journey

Journeys are the way we create interactive experiences in PingOne Advanced Identity Cloud. As such, we’ll use Journeys to build a new Login experience that leverages the themes we’ve set on the Organization.

Navigate to “Journeys” inside of your tenant and create a new Journey, using the following defaults:

| Input | Value |
| ----- | ----- |
| Name | ChangeThemeByOrg |
| Identity Object | Alpha realm - Users `managed/alpha_user` |
| Description (optional) | An example journey showcasing how you can dynamically change the theme based on the metadata set on the Organization. |

In this example, we’re going to have a basic username and password login. Drag in a Page Node with a Platform Username and Platform Password node, tied to a Data Store Decision. This should end up looking like a simplified version of the default “Login” Journey.

![The Simplified Login Journey inside the Journey Editor](../images/1-basicLogin.png)

_Basic Login_

Copy the Preview URL and open up the Journey in a Guest Window, Incognito Window, or separate browser. You should be able to login using an existing user, all with the Default Theme defined in your tenant.

![The Simplified Login Journey Rendered in Preview within a Browser Window](../images/1-basicLoginPreview.png)

_Basic Login, In Action_

This Journey has its own theme, but it’s in no way dynamic. Let’s change that.

## Changing Hosted Page Theming

In the same Journey drag in a Scripted Decision Node, creating a new (next-gen) script entitled `Set Hosted Page Overrides by Organization Metadata` with the outcomes `Success` and `Error`. In the code section, add the following script:

*Set Hosted Page Overrides by Organization Metadata*

```javascript
/*
Utilizing the organization metadata provided, set the hosted page.
This script does not need to be parametrized. It will work properly as is.
 
 The scripted decision node needs the following outcomes defined:
    - Success
    - Error
 
 Author: @gwizdala
 */

//// CONSTANTS
var REALM = "alpha";
var NodeOutcome = {
  SUCCESS: "Success",
  ERROR: "Error"
};

//// DEFAULTS
var HOSTED_CONFIG = 'Contrast';

//// MAIN
(function () {
  outcome = NodeOutcome.SUCCESS;

  try {
    var orgId = nodeState.get('orgId');

    if (null != orgId) {
        var queryFilter = `_id eq "${orgId}"`;
        var fields = 'themeOverrides';
        
        var orgMetadata = openidm.query(`managed/${REALM}_organization`, { 
          "_queryFilter": queryFilter,
          "_fields": fields
        }).result;
      
        if (null != orgMetadata && orgMetadata.length === 1) {
            var orgMetadataJson = JSON.parse(orgMetadata)[0];
            var themeOverrides = orgMetadataJson.themeOverrides;
            
            // Get theme data
            if (null != themeOverrides && null != themeOverrides.themeName) {
                HOSTED_CONFIG = themeOverrides.themeName;
            }
        }   
    }

    // Set Theme
    if (HOSTED_CONFIG && callbacks.isEmpty()) {
        var stage = "themeId="+encodeURI(HOSTED_CONFIG);
        callbacksBuilder.pollingWaitCallback("100", "Please wait ...");
        action.goTo(outcome).withStage(stage);
    } else {
        action.goTo(outcome);
    }
  } catch (e) {
logger.error(e);
        action.goTo(NodeOutcome.ERROR);
  }
}());
```

This example uses the `Contrast` Hosted Page that comes with your tenant as the default, meaning that if no Hosted Page is provided it will “fall back” to `Contrast`. If you would like to use a different default Hosted Page, change the `HOSTED_CONFIG` located at the top of your script (it’s in the `DEFAULTS` section).

> **Note:** This script expects a valid Hosted Page ID to be provided. Make sure to provide a Hosted Page name that exists inside your tenant.

Connect the `Success` and `Error` outcomes of this node to the Page Node. In a real-world scenario, we should route the Failure outcome to some sort of error handling and logging but in the case of this example we’ll let the Journey continue with the theme associated with that browser session.

Finally, let’s set some context in the Journey - we need a way to inform the Journey that there is an Organization associated with this session so that we can change the theme accordingly. In our example, we will gather the Organization ID from a query parameter appended to the Journey URL.

Drag in another Scripted Decision Node and create a new next-gen script with the name `Get Org ID by Query Parameter` with the outcomes `Found`, `Missing`, and `Error`. In the code section, add the following:

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

Connect the Start Node to this new Scripted Decision Node, the `Found` and `Missing` outputs to your Set Theme script, and the `Error` output to the Failure node. Your complete Journey should look something like this:

![A screenshot of the dynamic hosted pages journey](../images/1-dynamic-hosted-pages-journey.png)

_The Dynamic Hosted Pages Journey_

### How it Works

Journeys can take a query parameter that references the Hosted Page ID (its name), specifically `themeId`. Once set, the theme will change and the query parameter disappears from the URL. 

Within our `Get Org ID from Query Param` script we do the following:

1. Look to see if the query param specified (that’s the `QUERY_PARAM` constant) exists in our url.  
2. If the query param exists, store it in the specified shared state value (that’s the `SHARED_STATE_KEY` constant)

The above script is intentionally genericized - feel free to make it reusable as a [Library Script](https://docs.pingidentity.com/pingoneaic/latest/am-scripting/library-scripts.html)!

Within our `Set Hosted Page` script we do the following:

1. Take the organization ID stored in Shared state and query to see if that organization exists and if it has theme overrides set.  
2. If the theme override has been set, and there is a specified Hosted Page ID, build a redirect to continue the journey with the `themeId` query parameter specified (that’s the `stage`).  
3. Direct the Journey to go to the redirect (`action.goTo(outcome).withStage(stage)`) after giving the UI time to load (that’s the [PollingWaitCallback](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-read-only-callbacks.html#pollingwaitcallback)) to prevent screen flickering.

### Testing

To test, you’ll have to have created an Organization. I’ll be using the “Example Org” shown earlier in this How-To.

Refresh the Guest/Incognito window that has the preview URL to your Journey. The theme should default to the the `HOSTED_PAGE` default set in the script (the above example being `Contrast`).

![A screenshot of the rendered "Contrast" Hosted Page theme](../images/1-contrast-fallback.png)

_The Default Theme_

Next, modify the URL to contain the query parameter orgId where the value of that parameter is the GUID for your Organization (you can find this in the URL when you are managing the org in your tenant, i.e. `https://{your-domain}/platform/?realm=alpha#/managed-identities/managed/alpha_organization/{the-org-id}`)

Your full journey URL will look something like this:

```
https://{your-domain}/am/XUI/?realm=/alpha&authIndexType=service&authIndexValue=ChangeThemeByOrg&orgId={the-org-id}
```

When you go to this URL route, nothing should have changed. You haven’t set any Theme Overrides on the Organization yet.

Now, in your Managed Organization, change the theme name to another Hosted Page you have in your Tenant. I’m using Highlander.

![A screenshot of the "Highlander" Theme being added to the "Example" Organization](../images/1-setting-highlander.png)

_Setting Highlander as the Hosted Page Theme_

Save the Organization, and then refresh the page with your Journey. The theme has changed!

![A screenshot of the Journey changing to the Highlander Hosted Page theme](../images/1-highlander-set.png)

_Dynamic Hosted Pages in Action_

## Changing Emails

If we are dynamically changing the theme of the pages the user is interacting with, it’s likely that we want that same theming to additionally show up in the emails that the user receives.

Back inside your Journey, drag and drop in another Scripted Decision Node, creating a new (next-gen) script entitled `Set Email Overrides by Organization Metadata` with the outcomes `Success` and `Error`. This script should look pretty similar to you, since it’ll be gathering the same types of information as we saw within the Hosted Page Overrides.

*Set Email Overrides by Organization Metadata*

```javascript
/*
Utilizing the organization metadata provided, set the email(s).
This script does not need to be parametrized. It will work properly as is.
 
 The scripted decision node needs the following outcomes defined:
    - Success
    - Error
 
 Author: @gwizdala
 */

//// CONSTANTS
var REALM = "alpha";
var NodeOutcome = {
  SUCCESS: "Success",
  ERROR: "Error"
};

//// DEFAULTS
// Email Configurations. These are pulled from themeOverrides -> "email" + {name of config}
var EMAIL_CONFIGS = {
    Verification: { // Re-verifying an account after X days inactivity
        emailSuspendMessage: { // the localizable message shown to your users in your Journey
            en: "An email has been sent to the address on file. Click the link in that email to verify your account."
        },
        emailTemplateName: "registration", // The default email template that will be sent
        identityAttribute: "userName",
        emailAttribute: "mail"
    }
};

//// MAIN
(function () {
  outcome = NodeOutcome.SUCCESS;

  try {
    var orgId = nodeState.get('orgId');

    if (null != orgId) {
        var queryFilter = `_id eq "${orgId}"`;
        var fields = 'themeOverrides';
        
        var orgMetadata = openidm.query(`managed/${REALM}_organization`, { 
          "_queryFilter": queryFilter,
          "_fields": fields
        }).result;
      
        if (null != orgMetadata && orgMetadata.length === 1) {
            var orgMetadataJson = JSON.parse(orgMetadata)[0];
            var themeOverrides = orgMetadataJson.themeOverrides;
            
            // Update email configs from Identity Object
            Object.keys(EMAIL_CONFIGS).forEach(function(emailConfigKey) {
                var emailConfig = EMAIL_CONFIGS[emailConfigKey];
                var emailTemplateName = (!!themeOverrides && themeOverrides[`email${emailConfigKey}`]) ? themeOverrides[`email${emailConfigKey}`] : null;
                if (emailTemplateName) {
                    emailConfig.emailTemplateName = emailTemplateName;
                    EMAIL_CONFIGS[emailConfigKey] = emailConfig;
                }
            });
        }
    }

    // Set Email Configs in Shared State
    Object.keys(EMAIL_CONFIGS).forEach(function(emailConfigKey) {
        var emailConfig = EMAIL_CONFIGS[emailConfigKey];
        nodeState.putShared(`email${emailConfigKey}Config`, emailConfig);
    });
  } catch (e) {
logger.error(e);
        action.goTo(NodeOutcome.ERROR);
  }
}());
```

This example uses the `Registration` email template that comes with your tenant as the default, meaning that if no email template is provided it will “fall back” to `Registration`. If you would like to use a different default email template, change the `emailTemplateName` under the `EMAIL_CONFIGS` located at the top of your script (it’s in the `DEFAULTS` section).

> **Note:** This script expects a valid Email Template to be provided. Make sure to provide an Email Template name that exists inside your tenant.

Wire up the `Success` and `Error` outcomes of your `Set Hosted Pages` script to this new Node and the `Success` and `Error` outcomes of this script to your Page Node. The start of your Journey should now look like this:

![A screenshot of the beginning part of the Journey containing both the Set Hosted Page and Set Email Scripted Nodes](../images/1-dynamic-emails-journey-start.png)

_The Beginning of the Journey_

Drag in an Identify Existing User Node with the following values:

| Input | Value |
| ----- | ----- |
| Identifier | mail |
| Identity Attribute | userName |

This will ensure that we have the email address of the user stored in state, which we will use to send the email. Wire the `False` outcome to the Failure Node.

Next, drag in a Configuration Provider Node. These nodes allow us to dynamically render existing Journey nodes using data we’ve found from a Journey.

Set the Node Type to `Email Suspend Node` and create a new script entitled `Branded Verification Email`. We’ll only need one line of code inside this config provider- where we pass in the configuration we built in the previous “Set Email” Scripted Decision Node.

```javascript
config = nodeState.get("emailVerificationConfig").asMap();
```

Connect the `True` outcome of your Data Store Decision node to your Identify Existing User Node, the `True` outcome of your Identify Existing User Node to your Config Provider Node, the `Outcome` of your Config Provider Node to the Success Node, and finally the `Configuration Failure` outcome of the Config Provider to the Failure Node. Your Journey should now look like this:

![A screenshot of the entire Journey containing both the Set Hosted Page and Set Email Scripted Nodes](../images/1-dynamic-emails-journey-complete.png)

_The Complete Journey_

### How it Works

The approach to customizing emails is very similar to customizing hosted pages. 

Within our `Set Emails` script we do the following:

1. Take the organization ID stored in Shared state and query to see if that organization exists and if it has theme overrides set.  
2. If the theme override has been set, and there is a specified email that matches the defined options in the `EMAIL_CONFIGS`, update that config to match the email name specified.  
3. Store the email configs in Shared State to be rendered by the Configuration Provider.

You may be wondering why we are looping through a config object for emails rather than just returning a single object or rendering in the Config Provider itself. This way, if we want to add a different email, say for MFA, it’s as simple as two steps:

1. Create a string attribute in your `themeOverrides` with the name `emailMFA`  
2. Add an object to the `EMAIL_CONFIGS` called `MFA` with its own suspend message and default email template, which would probably look something like this:

```
MFA: { // Additional factor
        emailSuspendMessage: { // the localizable message shown to your users in your Journey
            en: "An email has been sent to the address on file. Click the link in that email to continue."
        },
        emailTemplateName: "mfa", // The default email template that will be sent
        identityAttribute: "userName",
        emailAttribute: "mail"
    }

```

Now, rather than building and managing a bunch of custom objects and calls, we have a standard method to onboarding new Email Templates into an Organization.

### Testing

To test, make sure you have a user with an email address that you can receive emails from and that you have the default email template entitled `Registration` inside your tenant. 

Go back to the base Journey Preview with no Organization ID passed into the query parameter. You’ll get the default branding both in the Hosted Pages and in your Email (in my case, the `Contrast` Hosted Page and `Registration` Email Template).

![A screenshot of the default "Contrast" Hosted Page theme](../images/1-contrast-theme-testing.png)
![A screenshot of the default email suspend node page with "Contrast" theme](../images/1-contrast-theme-testing-sent.png)
![A screenshot of the default "registration" email in a mail viewer](../images/1-default-registration-email.png)

_The Default Journey with No Organization Set_

Now, let’s add the `orgId` query parameter like we did when we changed the Hosted Page theme. To start, you should have the page theming you set but the email should still be default.

Now, back in your Organization add the name of another email template. I’ll use the default `forgottenUsername` template for the sake of this example.

![A screenshot of setting the Verification Email Template Name to "forgottenUsername"](../images/1-setting-email-template.png)

_Overriding the Email Template_

Re-running your Journey, you’ll see that the email has changed!

![A screenshot of the "forgottenUsername" in an email browser](../images/1-viewing-email-template.png)

_The Email Override in Action_

## Summary

The completed Journey can be found in the link below. It additionally contains a version of the Set Theming Script that combines both Hosted Page and Email setting into one script.

[ChangeThemeByOrg.json](https://gist.github.com/gwizdala/a81ae8ac9fcf2621473404b1307fdc00#file-changethemebyorg-json)

This is a basic approach to dynamically changing the theme and should provide a foundation to managing theming for your users. Some ways that you can extend this concept include:

* Loading themes based on a user’s metadata, like location, preferences, or risk profiles  
* Loading more than one email template to handle branching paths (e.g. different levels of risk, Registration, MFA, or Account Linking)  
* Establishing a “default theme” that your users are overridden to if no value is set rather than relying on the theme cached in their browser

But what if we need to set up branding more granularly? Continue in this series to learn how to [Selectively Change the Theme Styling]({{< ref "2-selective-theme-styling-aic" >}}) on Hosted Pages and in Email Templates, such as colors and logos.

---

# Further Reading

- [Introduction]({{< ref "introduction" >}})
- **Part 1: Dynamically Changing the Theming**
- [Part 2: Selectively Changing the Theme Styling]({{< ref "2-selective-theme-styling-aic" >}})
- [Conclusion & Recap]({{< ref "conclusion" >}})