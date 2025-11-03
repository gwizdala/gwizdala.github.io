---
draft: false
date: '2024-11-16T01:00:00+00:00'
title: 'Dynamically Branding in PingOne Advanced Identity Cloud, Part 2: Changing the Styling'
description: 'Part 2 of 2 in the series Dynamically Branding in PingOne Advanced Identity Cloud'
summary: Learn how to dynamically change granular details on Hosted Pages and Email Templates in a PingOne AIC Journey
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

# Selectively Changing the Theme Styling

As we saw in the [previous section]({{< ref "1-dynamic-theming-aic" >}}), Hosted Pages and Email Templates are a great way to quickly spin up and style theming for each of your individual partners, brands, and use cases. But manually changing these themes doesn’t easily scale. For example:

1. What if your company is composed of hundreds, if not thousands, of different sub-brands? Common examples here include food services, sports leagues, and manufacturing.  
2. What if your customers are businesses or business partnerships, and they want to ensure that their employees and/or customers know that they’re not only logging in with *you* but also with *your partner*? Common examples here include enterprise software, employee management systems, and insurance partnerships.

We don’t want to give these groups access to the Hosted Page or Email Template editor, because then they’d have visibility into *everyone’s* branding. We also likely want to *restrict what they can change*, so that they don’t remove things like your logo, footer copyright/disclaimer, or key branding/text elements that inform the user that they are interacting with you securely and correctly in the first place.

So let’s make a **global brand** that can be **selectively configured** by the Organizations - so rather than overriding the themes and emails each time, we are granularly specifying what the administrators can configure.

# Setup

## Configuring the Global Hosted Page

To start, we’ll need to create a global brand. In our case, we are going to clone the Hosted Page Starter Theme and default Email Templates that come with your tenant.

Duplicate the “Starter Theme” theme from your Hosted Pages (or create a new one from scratch), naming it “Global”. We’re going to configure this theme with some new styling so that it loads in a container where we can put our Partner’s logos alongside the main logo.

Underneath the “Journey” section of the editor, turn off the logo. Since we’ll be adding a new logo container to this style, we’ll hide this one.

![A screenshot of diabling the logo in the "Global" hosted page](/img/2-hide-logo.png)

_Hiding the Logo in your Global Hosted Page_

Next, underneath the “Layout” section, all the way at the bottom, enable and add the following Script Tag:

*Create Logo Container*

```html
<!-- Main and partner logo concatenation -->
<script type="text/javascript">
  // The base logo that will show up in this configuration. This is likely the parent company, the product owner, or the main brand.
  var BASE_ID_LOGO_URL = "https://upload.wikimedia.org/wikipedia/en/0/01/PingIdentity_logo.png";

  // Example CSS
  // Since hosted pages override at the element level, you may need to use !important
  // OR (better) create and add custom classes to elements.
  var CSS = `\
  .logo-wrapper {\
    display: flex;\
    justify-content: center;\
    align-items: center;\
  }\
  \
  .logo-wrapper img {\
    padding: 0.5em;\
    border-right: 3px solid #000000;\
    align-self: center;
  }\
  \
  .logo-wrapper img:last-child {\
    border: none; /* remove border  */\
  }\
  `;

  var createLogo = function(id, src, alt) {
    var logo = document.createElement('img');
    logo.id = id;
    logo.src = src;
    logo.alt = alt;
    logo.classList.add('ping-logo');
    logo.classList.add('mb-4');
    logo.classList.add('mt-2');

    return logo;
  }

  var addLogos = function() {
    var header = document.getElementsByClassName("login-header")[0].childNodes[0];
    
    var fragment = document.createDocumentFragment();
    // Containers
    var logoContainer = document.createElement('div');
    logoContainer.classList.add('logo-wrapper');
    logoContainer.id = "partnerLogoWrapper";

    // logos
    var mainLogo = createLogo('mainLogo', BASE_ID_LOGO_URL, 'Main Logo');
    
    fragment.appendChild(mainLogo);
  
    logoContainer.appendChild(fragment);
    header.prepend(logoContainer);
    document.head.appendChild(document.createElement("style")).innerHTML = `${CSS}`;
  };
  
  // Adding HTML to the page
  if (document.readyState === "loading") {
    // loading hasn't finished yet
    document.addEventListener("DOMContentLoaded", addLogos);
  } else {
    // `DOMContentLoaded` has already fired
    addLogos();
  } 
</script>
```

Your hosted page editor should look something like this:

![A screenshot pointing to the "Script Tags" section of the Hosted Page editor where the script has been added and enabled](/img/2-script-tag.png)

_Adding and Enabling the Script Tag_

While this script adds a default logo, the above setup **won’t show up inside the editor** - the editor doesn’t execute any JavaScript added through the Script Tag. To test that your script is working, load a Journey using this Hosted Page to see the logo magically added:

![A screenshot of the default "parent" logo showing up in the Hosted Page](/img/2-script-tag-test.png)

_There's Our Image!_

You’ll see upon inspecting the page that not only is there the main logo, but there’s also a container added with the ID `partnerLogoWrapper` surrounding that logo - this is what we’re going to target when passing our Partner’s logo into the theme.

## Configuring the Global Email Template

Go ahead and duplicate the “Registration” email template, name it to “globalRegistration” and switch to the HTML editor. We’re going to use the HTML (“Advanced”) editor to take advantage of Handlebars notation for not only our page text but also for our page styling, as the CSS will come bundled into the HTML in your editor.

![A screenshot of the 'globalRegistration' email switched to the advanced editor](/img/2-global-registration-email.png)

_The global registration email in the advanced editor. The names `globalRegistration`, `Global Registration`, and `global_registration` will work as your title._

Once you’ve switched to the HTML editor, update your template with the following:

```html
<html>
   <head>
      <style>
         .background {
         background-color: {{#if object.backgroundColorHex}}{{object.backgroundColorHex}}{{else}}#324054{{/if}};
         }
         .button {
         background-color: {{#if object.primaryColorHex}}{{object.primaryColorHex}}{{else}}black{{/if}};
         border: 1px solid {{#if object.primaryColorHex}}{{object.primaryColorHex}}{{else}}black{{/if}};
         border-radius: 5px;
         color: white!important;
         padding: 15px 32px;
         text-align:center;
         text-decoration:none;
         display:inline-block;
         font-size:16px;
         }
         .button:hover {  
         background-color: white;
         color: {{#if object.primaryColorHex}}{{object.primaryColorHex}}{{else}}black{{/if}}!important;
         }
      </style>
   </head>
   <body class="background" style="color:#5e6d82;padding:60px;text-align:center">
      <div class="content" style="background-color:#fff;border-radius:4px;margin:0 auto;padding:48px;width:600px">
         <div class="logo-wrapper" style="text-align:center;width:100%"> {{#if object.orgLogoUrl}}                         
            <img src="{{object.orgLogoUrl}}" alt="{{object.orgName}} Logo" style="padding:0.5em;border-right:3px solid #000000;align-self:center;max-height:100px" /> {{/if}}                         
            <img alt="Main Logo" src="https://upload.wikimedia.org/wikipedia/en/0/01/PingIdentity_logo.png" style="padding:0.5em;border-right:3px solid #000000;align-self:center;max-height:100px;border:none" />
         </div>
         <h3>This is your registration email.</h3>
         <div>
            <a href="{{object.resumeURI}}" class="button">Email verification link</a>
         </div>
      </div>
   </body>
</html>
```

We’ve added in some quality of life features, like a card and button class, as well as set up a means to reference our metadata we’ll be storing on the Organization object.

Go ahead and send yourself a test email. You’ll see that you receive the default theming, which should look similar to the Global Hosted Page you created (even including the same logo wrapper you saw above).

![A screenshot of the 'globalRegistration' email sent to the user](/img/2-global-registration-email-sent.png)

_The Sent Global Email_

## Configuring the Managed Object

Just like how we configured our Organization object to contain Theme **Overrides**, let’s configure it to hold Theme **Customization**.

To start, head over to your `Alpha_Organization` managed object config in your IDM native console (under Native Consoles → Identity Management → Configure → Managed Objects) and create a new attribute with the following properties:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `themeCustomization` | Theme Customization | Object | False |

Select that attribute and add the following properties to it:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `logoUrl` | Logo URL | String | False |
| `backgroundColorHex` | Background Color (Hex) | String | False |
| `primaryColorHex` | Primary Color (Hex) | String | False |

> **Note:** Normally, when creating properties that are constrained to a particular format (e.g. a URL or a Hex Code), it’s wise to enforce that constraint using [Validation Policies](https://docs.pingidentity.com/pingoneaic/latest/idm-objects/configuring-default-policy.html) - but since that’s not the topic of this article we’re going to leave the attributes be for now.

Your `themeCustomization` object should look like this:

![A screenshot of the 'Theme Customization' Object Attribute](/img/2-theme-customization.png)

_The Theme Customization Object Configuration_

With this configuration in place, when we head over to an Organization we’ve created we’ll see a new tab entitled “Theme Customization” with the three options we’ve configured.

![A screenshot of the 'Theme Customization' attributes within the "Example" organization](/img/2-theme-customization-org.png)

_The Theme Customization inside an Organization_

> **Note:** When setting up attributes it’s advisable to add Descriptions (which you’ll see in the screenshot above). Since it isn’t necessary for this example, feel free to add them later if you’d like.

## Building the Journey

Journeys are the way we create interactive experiences in PingOne Advanced Identity Cloud. As such, we’ll use Journeys to build a new Login experience that leverages the theme customizations we’ve set on the Organization.

Navigate to “Journeys” inside of your tenant and either create a new Journey or duplicate the `ChangeThemeByOrg` Journey from the previous section using the following defaults:

| Input | Value |
| ----- | ----- |
| Name | ChangeThemeCustomizationByOrg |
| Identity Object | Alpha realm \- Users `managed/alpha_user` |
| Description (optional) | An example journey showcasing how you can dynamically change the theme customization based on the metadata set on the Organization. |
| Override Theme | True (Checked) |
| Theme | Global |

In this example, we’re going to have a basic username and password login. If it’s not there already, drag in a Page Node with a Platform Username and Platform Password node, tied to a Data Store Decision. This should end up looking like a simplified version of the default “Login” Journey.

![The Simplified Login Journey inside the Journey Editor](/img/1-basicLogin.png)

_Basic Login_

Copy the Preview URL and open up the Journey in a Guest Window, Incognito Window, or separate browser. You should be able to login using an existing user, all with the Global Theme defined in your tenant.

Now that we’ve set up our Global brand and created a means to reference the customizations on that brand, let’s put it all to action!

# Changing the Hosted Page Styling within a Journey

Let’s create a Journey Node that captures the Organization’s theming customizations and updates the page accordingly.

Drag in a Scripted Decision Node and create a new Script entitled “Set Theme Styling by Organization Metadata” with the outcomes `Success` and `Failure`. In the script editor, paste the following:

*Set Hosted Page Styling by Organization Metadata*

```javascript
/*
Utilizing the organization metadata provided, set the logo and colors of the brand on the page.
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

//// HELPERS

function generateUI(name, logoUrl, backgroundColorHex, primaryColorHex) {

    var CSS = '';

    if (backgroundColorHex) {
        CSS += `\
        .fr-body-image-branded {\
          background-color: ${backgroundColorHex}!important;\
        }`;
    }

    if (primaryColorHex) {
      CSS += `\
        button[id^='brandedButton_'] {\
          background-color: ${primaryColorHex}!important;\
          border-color: ${primaryColorHex}!important;\
        }`;
    }
    
  var html = `\
        var logoUrl = "${logoUrl ? logoUrl : ''}";\
        var logoName = "${name ? name : 'Partner'} Logo";\
        var logoId = "partnerLogo";\
        var logoContainer = document.getElementById("partnerLogoWrapper");\
        var preexistingLogo = document.getElementById(logoId);\
        var backgroundImages = document.getElementsByClassName("fr-body-image");\
        var buttons = document.getElementsByClassName("btn-primary");\
        \
        for (var i = 0; i < backgroundImages.length; i++) {\
            var backgroundImage = backgroundImages[i];\
            backgroundImage.classList.add('fr-body-image-branded');\
        };\
        \
        for (var j = 0; j < buttons.length; j++) {\
            var button = buttons[j];\
            button.id = 'brandedButton_'.concat(j);\
        };\
        \
        document.head.appendChild(document.createElement("style")).innerHTML = "${CSS}";\
        \
        if (logoUrl != "" && logoContainer != null && preexistingLogo == null) {\
            var logo = document.createElement('img');\
            logo.id = logoId;\
            logo.src = logoUrl;\
            logo.alt = logoName;\
            logo.classList.add('ping-logo');\
            logo.classList.add('mb-4');\
            logo.classList.add('mt-2');\
            console.log(logo);\
            logoContainer.insertBefore(logo, logoContainer.firstChild);\
        }\
        `;
  
  return html;
}  

//// MAIN
(function () {
    outcome = NodeOutcome.SUCCESS;

    try {
        var logoUrl = "";
        var name = "";
        var backgroundColorHex = "";
        var primaryColorHex = "";

        var orgId = nodeState.get('orgId');

        if (null != orgId) {
            var queryFilter = `_id eq "${orgId}"`;
            var fields = 'name,themeCustomization';
            
            var orgMetadata = openidm.query(`managed/${REALM}_organization`, { 
              "_queryFilter": queryFilter,
              "_fields": fields
            }).result;
  
            if (null != orgMetadata && orgMetadata.length === 1) {
                var orgMetadataJson = JSON.parse(orgMetadata)[0];
                name = orgMetadataJson.name;
                var customization = orgMetadataJson.themeCustomization;
                if (customization != null) {
                    logoUrl = customization.logoUrl != null ? customization.logoUrl : '';
                    backgroundColorHex = customization.backgroundColorHex ? customization.backgroundColorHex : "";
                    primaryColorHex = customization.primaryColorHex ? customization.primaryColorHex : "";
                }
            }
                
            if (callbacks.isEmpty()) {
                callbacksBuilder.scriptTextOutputCallback(generateUI(name, logoUrl, backgroundColorHex, primaryColorHex));
            }
        }
    } catch(e) {
        logger.error(e);
        outcome = NodeOutcome.ERROR;
    }

    action.goTo(outcome);
}());
```

Drag and drop this Scripted Decision Node into your Page Node, and then wire `Success` to your Data Store Decision and `Failure` to your Failure Node. You Journey should look something like this:

![A screenshot of the 'Set Theme Styling' Scripted Decision Node within the Page Node of the example Journey](/img/2-set-styling-journey.png)

_Changing the Styling in a Page Node_

Finally, we are going to add in and wire up the [`Get Org ID by Query Parameter` Scripted Decision Node]({{< ref "1-dynamic-theming-aic#changing-hosted-page-theming" >}}) that we made in the last section to the front of this Journey. Just like before, we’re using this approach as an example of how you could inform the Journey that there is an Organization associated with this session so that we can change the theme styling accordingly. Your completed Journey should look something like this:

![A screenshot of the example Journey including the "Get Org ID by Query Parameter" Script](/img/2-set-styling-journey-complete.png)

_Setting the Styling from the Org Metadata_

## How it Works

This Script uses the [scriptTextOutputCallback](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-backchannel-callbacks.html#scripttextoutputcallback) to execute JavaScript on the Hosted Page itself. The script we are executing takes the data we’ve stored on the Organization and then pushes the logoURL into the container we set within the Global Theme and changes the primary color on the buttons as well as the background color behind the card. If this data isn’t provided, or the container for the logo doesn’t exist, the theming does not change.

Within our `Set Theme Styling` script we do the following:

1. Take the organization ID stored in Shared state and query to see if that organization exists and if it has theme styling set.  
2. If the theme styling has been set, and there is a specified logo or any colors have been set, add the logo and colors as css an image tag onto the page.

## Testing

Modify the URL to contain the query parameter orgId where the value of that parameter is the GUID for your Organization (you can find this in the URL when you are managing the org in your tenant, i.e. `https://{your-domain}/platform/?realm=alpha#/managed-identities/managed/alpha_organization/{the-org-id}`)

Your full journey URL will look something like this:

```
https://{your-domain}/am/XUI/?realm=/alpha&authIndexType=service&authIndexValue=ChangeThemeCustomizationByOrg&orgId={the-org-id}
```

When you go to this URL route, nothing should have changed. You haven’t set any theme customizations on the Organization yet.

Now, in your Managed Organization, change the Logo URL to a valid image hosted on the internet (I’m using a wikimedia link to a .png), and some hex colors for the background and the buttons.

![A screenshot of example values being added to the "Example" Organization](/img/2-setting-theme-org.png)

_Setting the Styling_

Save the Organization, and then refresh the page with your Journey. The background color and button colors have changed, and your partner logo has appeared next to your main logo!

![A screenshot of the Journey's Hosted Page taking the customizations set on the Organization](/img/2-setting-theme-display.png)

_The Styling Carrying Through_

Now let’s carry over these customizations into our Email Templates.

# Changing the Styling in Emails

Back inside your Journey, connect the `True` output of your Data Store Decision Node to an Identify Existing User Node using the following values:

| Input | Value |
| ----- | ----- |
| Identifier | mail |
| Identity Attribute | userName |

This will ensure that we have the email address of the user stored in state, which we will use to send the email. Wire the `False` outcome to the Failure Node.

Next, drag and drop in another Scripted Decision Node, creating a new (next-gen) script entitled `Set Email Styling by Organization Metadata` with the outcomes `Success` and `Error`. If you went through the previous article, [Dynamically Changing Theme]({{< ref "1-dynamic-theming-aic" >}}), this process will be very familiar to you.

*Set Email Styling by Organization Metadata*

```javascript
/*
Utilizing the organization metadata provided, set the logo and colors of the brand inside emails.
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
    outcome = NodeOutcome.SUCCESS;

    try {
        var logoUrl = "";
        var name = "";
        var backgroundColorHex = "";
        var primaryColorHex = "";

        var orgId = nodeState.get('orgId');
        var objectAttributes = nodeState.get("objectAttributes");

        // If there's no object attributes we should create an object to fill
        if (null == objectAttributes) {
            objectAttributes = {};
        }

        if (null != orgId) {
            var queryFilter = `_id eq "${orgId}"`;
            var fields = 'themeCustomization';
            
            var orgMetadata = openidm.query(`managed/${REALM}_organization`, { 
              "_queryFilter": queryFilter,
              "_fields": fields
            }).result;
  
            if (null != orgMetadata && orgMetadata.length === 1) {
                var orgMetadataJson = JSON.parse(orgMetadata)[0];
                name = orgMetadata.name;
                var customization = orgMetadataJson.themeCustomization;
                if (customization != null) {
                    logoUrl = customization.logoUrl != null ? customization.logoUrl : '';
                    backgroundColorHex = customization.backgroundColorHex ? customization.backgroundColorHex : "";
                    primaryColorHex = customization.primaryColorHex ? customization.primaryColorHex : "";
                    objectAttributes["orgLogoUrl"] = logoUrl;
                    objectAttributes["backgroundColorHex"] = backgroundColorHex;
                    objectAttributes["primaryColorHex"] = primaryColorHex;
                }
            }

            objectAttributes["orgName"] = name;
            nodeState.mergeTransient({"objectAttributes": objectAttributes});
        }
    } catch(e) {
        logger.error(e);
        outcome = NodeOutcome.ERROR;
    }

    action.goTo(outcome);
}());
```

Wire up the `True` outcome of your Identify Existing User Node to this New Node and the `Success` and `Error` outcomes of this script to an Email Suspend Node. Configure the Email Suspend Node with the following:

| Input | Value |
| ----- | ----- |
| Name | Registration Email |
| Email Template | globalRegistration |
| Email Attribute | mail |
| Email Suspend Message | *Leave as default* |
| Object Lookup | False (Unchecked) |
| Identity Attribute | userName |

Wire up the Registration Email Suspend Node to the Success Node. Your completed Journey should look like the following:

![A screenshot of the complete set theme styling by hosted page and email journey](/img/2-set-styling-email-journey-complete.png)

_The Complete Journey, Styling Hosted Pages and Emails_

## How it Works

Within our `Set Email Styling` script we do the following:

1. Take the organization ID stored in Shared state and query to see if that organization exists and if it has theme styling set.  
2. If the theme styling has been set, and there is a specified logo or any colors have been set, add the logo and colors into the user’s `Object Attributes` which are passed into the Email Template.  
3. Inside the Email Template, if the values exist display the customized email.

## Testing

Go back to the base Journey Preview with no Organization ID passed into the query parameter. You’ll get the default branding as defined by your Global Hosted Page and Email Template.

![A screenshot of the default global hosted page](/img/2-global-theme.png)  
![A screenshot of the default global hosted page displaying a suspend node](/img/2-global-theme-suspend.png) 
![A screenshot of the default global email template sent](/img/2-global-theme-email.png)

_The Default Global Branding_

Now, let’s add the `orgId` query parameter like we did when we changed the Hosted Page theme. Not only will the theme styling change in the Journey, but it’ll show up in the email!

![A screenshot of the stylized email template sent](/img/2-custom-theme-email.png)

_The Custom-Styled Email_

# Summary

The completed Journey can be found in the link below. It additionally contains the Global Theme needed to render the styling.

[ChangeThemeCustomizationByOrg.json](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/dynamically-branding-journeys-in-aic/ChangeThemeCustomizationByOrg.json)

There are some “gotchas” you may want to consider when expanding upon this idea:

1. When the user returns to your Journey from their email, they may not have the same theming or javascript stored in their session. Consider re-setting the theme styling when the user returns to the page.  
2. In order to customize the email we are updating the user’s Object Attributes with custom attribute values. If you intend to update or create a user after resuming your Journey, make sure to remove the extra values to prevent a malformed object exception (basically, pass an empty object and then merge in the cleaned object - look at how we remove the `password` attribute from our Object Attributes before passing to Shared State as an example)

---

# Further Reading

- [Introduction]({{< ref "introduction" >}})
- [Part 1: Dynamically Changing the Theming]({{< ref "1-dynamic-theming-aic" >}})
- **Part 2: Selectively Changing the Theme Styling**
- [Conclusion & Recap]({{< ref "conclusion" >}})