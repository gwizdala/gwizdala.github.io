---
draft: false
date: '2026-04-01'
title: 'PingOne DaVinci Primer: Creating Dynamic Experiences with PingOne Populations'
description: 'Learn how to build dynamic experiences using PingOne context in PingOne DaVinci'
summary: Learn the basics of building a PingOne DaVinci flow while creating a dynamically branching experience based on PingOne User and Population context.
categories: ["Ping Identity"]
tags: ["PingOne", "PingOne DaVinci"]
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

As part of creating a User in PingOne that User will need to be assigned to a [Population](https://docs.pingidentity.com/pingone/directory/p1_populations.html). Populations in PingOne create logical segments of user identities, which not only define administrative boundaries for Users and Groups with elevated permissions (Admin Roles) but also provide additional metadata that can be used to cater the User’s experience to fit their unique use cases.

This How-To will teach you how to create dynamic user experiences using PingOne’s built-in Populations coupled with PingOne DaVinci. Along the way you’ll learn common strategies and design decisions when building out PingOne DaVinci flows.

The level of detail here is meant for beginners to PingOne and PingOne DaVinci. If you are just looking for, or want to follow along with, the pre-built example, download the DaVinci flow using the link [here](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/davinci-primer-dynamic-populations/dynamic_experiences_with_populations.json).

There are DaVinci tips and tricks interspersed throughout this document. Search for the ⭐  emoji to jump between them.

This How-To is broken down into the following sections 

1. [Initial Setup](#initial-setup)
2. Retrieving the Population
   1. [Based on the User](#retrieving-the-population-based-on-the-user)
   2. [Based on the Alternative Identifier](#retrieving-the-population-based-on-the-alternative-identifier)
3. [Changing the Theme based on the Population](#changing-the-theme-based-on-the-population)
4. [Enforcing password policies based on the population](#enforcing-password-policies-based-on-the-population)
5. [Enforcing login with an external Identity Provider (IdP) based on the Population](#enforcing-login-with-an-external-idp-based-on-the-population)
6. [Connecting the Flow to Your Application(s)](#connecting-the-flow-to-your-applications)

By the end of this How-To you’ll have a DaVinci flow that gathers Population metadata based on the User and Population context and applies that metadata during the User’s login and registration experience. Along the way, you’ll learn a wide variety of DaVinci capabilities such as (but not limited to):

* Navigating the DaVinci canvas
* Building and using Forms within the page
* Interacting with Connectors, Triggers, and Action Decision Nodes
* Passing data between nodes
* Branching and teleporting during the user’s journey
* Finding and using nodes and flows from the Marketplace

This How-To expects a beginner level of familiarity with DaVinci and the PingOne console. Since [Managing Populations](https://docs.pingidentity.com/pingone/directory/p1_manage_populations.html) is well-defined in the documentation, we won’t be discussing that here.

# Initial Setup 

This How-To requires an Environment with PingOne SSO and PingOne DaVinci. Make sure you have the appropriate Environment, licenses, and permissions before continuing.

You can use the Default Population for this How-To, or you can create a new one: just make sure you have at least one User in that Population so that you can test your flows. The name of my Population is “Example” and my user is “Example User” with the username “example@example.com”.

Fortunately for us, importing and exporting experiences like the one in this document is trivial in PingOne. If you’d like to start with the pre-built example, download the DaVinci flow using the link [here](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/davinci-primer-dynamic-populations/dynamic_experiences_with_populations.json) and then import into your Environment to get going.

That being said, if you’re looking to get a better understanding of *how* to construct flows like this, it’s worth the time to follow along with this How-To.

## Creating the Forms

Since we are going to be capturing some user information to log in and are going to be branding them based on our Population, we should create some [Forms](https://docs.pingidentity.com/pingone/user_experience/p1_forms.html). Forms provide a drag-and-drop editor for building user interfaces that we can then reuse throughout our DaVinci Flows.

Go to “User Experience” - “Forms” in your PingOne navigation bar. You’ll see a series of example forms that can be used in your flows - feel free to click each of them to see what capabilities they provide in the preview.

Our experience is going to collect the user’s username and then use that information to figure out what Population they are associated with either for registration or authentication. We can quickly create that form by duplicating the **Example - Sign On** form already made for us.

Select the form entitled **Example - Sign On**, click on the three-dot menu (fun fact: those are called [Kebab Menus](https://van-ons.nl/en/blog/functioneel-ontwerp/wat-hamburgers-kebab-en-ux-met-elkaar-te-maken-hebben/)) and select **Duplicate**. You’ll see a new version created called something like **Example - SignOn1** - select it, hit the three dots and this time click **Edit**.

![A screenshot of the duplicated form and the highlighted "Edit" menu item](/img/davinci-primer-dynamic-populations/form-edit.png)
*Creating Your First Form*

The Editor view gives us access to a bunch of different capabilities, but we’re not going to need to make a whole lot of changes for our use case.

First, under Properties (in the left bar) change the Form Name to **Email-First Authentication**. Then, select the Password field and click on the trash can icon in the left window to delete it. Optionally, select the Username field, change the validation type to “Custom”, the Regex to `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$` and the Error Message to the translation key `forms.fields.user.email.errorMessage` (which is the phrase “Invalid Email Address”).

![A screenshot of the form editor for the Email-First Authetication flow. Just the username field is displayed.](/img/davinci-primer-dynamic-populations/form-editor.png)

*The Email-First Authentication Form*

When you’re done, make sure to save the form by hitting the “Save” button in the top-right of the page.

## Creating Your DaVinci Flow

We’re going to be retrieving and responding to our Users’ inputs within a DaVinci flow. 

On your left navbar, click on the DaVinci button. If it’s not there, make sure you’re in an Environment with DaVinci enabled.

A new tab will open in your browser to DaVinci. Click on the “Flows” button on the left navigation bar and the “Add Flow” in the top right corner of the page. If you’re using the example flow, feel free to select “Import Flow” otherwise click the “Blank Flow” option from the dropdown, giving it the name **Dynamic Experiences with Populations**. Once you hit “Create”, you’ll be taken into the DaVinci editor, which is where we’ll spend the majority of this How-To.

# Retrieving the Population

Our Population contains important information about our user - things like their password policy, their branding, and if they log in using an external identity provider. To use that information accordingly we need a way to figure out what population our user (or new user) is related to.

## Retrieving the Population based on the User 

### Collecting the Username

First off, let’s pull Population data based on the User that we’ve identified in our DaVinci flow. Think of this use case as the initial login page for your customers into your business: based on who we see, we’ll dynamically change their experience accordingly.

> **⭐ DaVinci Learning:** Connectors, Capabilities, and Triggers

To start, click the “+” button in the bottom-left corner of your Canvas (that’s the editor window), click “User Interface”, and click on your “Email-First Authentication” form. A little node will snap to your mouse which you can then click on the canvas to add.

{{< column-container wrap="true" >}}
{{< column >}}

![A screenshot of the add (+) button that, when clicked, shows a submenu where "User Interface" is selected](/img/davinci-primer-dynamic-populations/add-interface.png)

{{< /column >}}
{{< column >}}

![A view of the Add User Interface menu which includes the created forms](/img/davinci-primer-dynamic-populations/add-interface-menu.png)

{{< /column >}}
{{< /column-container >}}


*Adding a New Form*

The form is a Connector and is using the “Show Form” capability - you can check this out by clicking the back error at the top of the form editor screen. You’ll also see that you can select the forms you created previously from the “Form” dropdown that appears.

![A screenshot of the Form connector in which the "Show Form" trigger has been selected](/img/davinci-primer-dynamic-populations/trigger-show-form.png)
![A screenshot of the details within the Show Form Trigger in which the Email-First Authentication form has been selected](/img/davinci-primer-dynamic-populations/trigger-show-form-details.png)
*Selecting the “Show Form” Capability*

At this point you have a single node that collects the username and stores it under the key `user.username`. To test, hit the “Deploy” button in the top right corner and then the “Try Flow” button to open up the form in a new tab.

![A screenshot of the rendered flow in which the username is presented as an input](/img/davinci-primer-dynamic-populations/flow-username.png)
*A rendered Form*

Nice! In one node you have a log in form. Now, let’s figure out what Population this user is from based on their username.

### Finding the User

Click and drag from the black dot on the right side of the Form connector - a little line should show up. When releasing that line, you’ll be prompted to add a new connector. This time, select the PingOne connector.

![A screenshot of the DaVinci canvas where a line has been dragged from the Form node and a selection box has appeared to add a connector. PingOne has been searched and its connector is highlighted](/img/davinci-primer-dynamic-populations/adding-a-connector.png)
*Connecting Connectors*

You’ll see that your Form is now connected to a new node with a little grey bubble and a blue line with the word “True” on it. Those bubbles are called **Action Decision Nodes** and they allow us to branch decisioning logic based on the results of the prior connectors. Right now, that blue “True” is saying that if **All Triggers are True**, then the Form will continue to the PingOne connector. You can have multiple responses come from the same action decision node, which we’ll use later when handling actions like a user not existing in our Environment. If you ever need to change a trigger, you can right-click the word (in that case, “True”) or click on the Action Decision Node directly.

![A screenshot of the Action Decision Node editor in which the All Triggers True condition points to the PingOne node](/img/davinci-primer-dynamic-populations/action-decision.png)
*Action Decision Nodes*

Now that these two connectors are connected, let’s configure the PingOne node to lookup the user. Click on the PingOne connector and select the “Find User” capability. 

> **⭐ DaVinci Learning:** Passing Data with Handlebars

We are going to be looking up our user based on their username, using the username inputted by the form. To do so, under the **PingOne Attributes** field type the word `username` and hit enter. Then under the identifier, select the “{}” icon in the right side of the input and click on your Form. Any time you see the “{}” icon means that you have access to data collected along the course of your DaVinci flow.

![A screenshot of the variables selection screen in which Form is highlighted](/img/davinci-primer-dynamic-populations/handlebars-find.png)
*Collecting Input Data*

Once you’ve clicked on your Form connector, you’ll see that the output values include the username keyed based on the defined key we set in the Form Connector. Go ahead and select `user.username` as the Identifier.

![A screenshot of the user.username variable value being added into the Identifier field via handlebars](/img/davinci-primer-dynamic-populations/handlebars-add.png)
*Selecting the specific value*

If you’re curious why the icons for these values are curly braces, it’s because the data is actually being referenced using [handlebars notation](https://handlebarsjs.com/) - if you hover on the value you selected you can see what the reference looks like. For the most part, you’ll probably not have to work with the handlebars directly, but it’s useful to know in cases that you want to ensure a specific value is being used in connectors downstream.

![A screenshot of the hovered-over handlebars value on the username field just added](/img/davinci-primer-dynamic-populations/handlebars-view.png)
*Seeing the Handlebars representation of the data*

> **⭐ DaVinci Tip:** Flows can get quite large quite fast. Consider putting unique titles and descriptions on each connector (you can find that in the “Settings” tab within the action’s editor view) as well as annotations (right-click on the canvas and select “Annotation”) to help you reference the correct values in inputs as well as better understand what a flow is doing when maintaining/editing.

Hit “Apply” to save your changes.

### Retrieving the Population Details

If a User is found, we know their Population.

From the “Find User” connector, add an “All Triggers True” action connection to an Http connector with the capability “Custom HTML Message”. The message should say “Your population ID is” with the added variable of the matchedUser.population.id.

![A HTTP message node that contains the Population ID gathered from the PingOne node](/img/davinci-primer-dynamic-populations/debug-popid.png)
*Displaying the Population ID*

Apply, Deploy, and then Try your new flow. If you use a pre-existing user, you should see their Population ID.

![The resulting Flow display message of the collected Population ID. It is an information card with the text "Your Population ID is" along with a UUID](/img/davinci-primer-dynamic-populations/debug-popid-output.png)
*The matched user’s Population ID*

A Population ID isn’t too useful by itself. Let’s pull data on the **Population** directly using the ID we’ve retrieved.

> **⭐ DaVinci Learning:** Disabling Nodes

Move the Display Population ID connector out of the way, right click, and **Disable** the connector. This lets us keep the connector in our editor if we want to use it later without it being considered by DaVinci as a valid path. The node will turn semi-transparent to indicate that it’s disabled.

![The right-click menu that appears when interacting with a node. The Option to "Disable" is highlighted](/img/davinci-primer-dynamic-populations/disable-node.png)
*Disabling a Node*

From the same Action Decision Node, connect a PingOne Connector with the capability “Read Population”. In your input, add the same population ID from your matched user that you used in the Display Population step.

![A connected PingOne node in which the Population ID gathered from the prior node is being used to read full population details](/img/davinci-primer-dynamic-populations/read-pop.png)
*Reading a Population*

Now, if the Population can be found, we can pull more metadata from that Population. Connect a new Http connector with the “Custom HTML Message” capability, toggle on “Show Continue Button”, and add the Message “Your Population data:” with the additional value of the `population` output that came from the Read Population node. If you want to add an object or array, rather than a single piece of data, into an input click the “+” button next to the specific object value.

![A message node in which the entire population object is being added for display. A red arrow points to the "+" button next to the "population (object)" element in the variables selector](/img/davinci-primer-dynamic-populations/debug-popdata.png)
*Adding an Object as an Input Value*

Apply, Deploy, and Try your flow. This time, you should see all of the metadata that came back from your user’s population.
![An infomration message display that shows the returned population data object that was collected by the read population node](/img/davinci-primer-dynamic-populations/debug-popdata-output.png)
*The User’s Population*

## Retrieving the Population based on the Alternative Identifier 

Next, let’s get the Population based on an Alternative Identifier we’ve been provided during the flow. Alternative Identifiers are custom values that you can set on a Population and are used to help you identify what Population should be selected during a flow. Think of this as a domain (possibly in a redirect URL or in the user’s email address), an ID (in a query parameter or body of a REST call), or any other value you’d like to use to help you recognize what Population matches. This is a great approach when you want specific users to be registered into particular populations as you can identify them even before they have created an account in PingOne.

In fact, registering by context is exactly what we want to add into our flow. Currently, if you enter in a user that doesn’t exist in PingOne, the flow returns an error. Instead, let’s try to find the right population for this user: falling back to our Default population if nothing matches.

### Parsing the Email Domain

We are going to attempt to match the email domain extracted from our user’s username to an **Alternate Identifier** within our Population.

> **⭐ DaVinci Learning:** Ping Marketplace

You probably noticed that when you got the user information, the domain isn’t extracted by default. If you wanted, you could always write a custom function in a Functions Connector to do this for you: but rather than build from scratch, Ping provides a [Marketplace](https://marketplace.pingone.com/home) of Nodes and Flows pre-configured to handle the heavy-lifting for you. In this case, let’s use the [Parse Domain from Email Address node](https://marketplace.pingone.com/item/parse-domain-from-email-address).

Log into the Marketplace with your PingOne admin credentials, hit the “Copy to clipboard” button in the top right corner of the Marketplace listing, and then in your DaVinci canvas right-click an empty area and select “Paste Nodes”. You’ll now have the connector ready to go in your flow.

![A screenshot of the canvas in which the user has right-clicked an empty space to display a menue that includes "add annotation" and "paste nodes"](/img/davinci-primer-dynamic-populations/paste-node.png)
*Pasting Nodes*

As a rule of thumb: if there’s functionality you want but you can’t find a trigger that does it, a custom page in HTML that you want designed, or a more complicated flow that you don’t know how to start on, go to the Marketplace. It’s likely there’s something there that will either do or help you get to the solution you’re looking for.

Some notable Marketplace listings include:

* Nodes:
  * [User Registration Node Group](https://marketplace.pingone.com/item/user-registration-node-block): A set of nodes that complete a registration process. Useful when learning how to design HTML in DaVinci, action on button presses with the A Equals Multiple B trigger, and interacting with teleports.
* Styling:
  * [Ping UX - DaVinci CSS](https://marketplace.pingone.com/item/ping-ux-css): A set of CSS files to help you quickly stylize and brand your custom HTML pages, compatible with Bootstrap formatting.
  * [DaVinci Design Studio](https://marketplace.pingone.com/item/davinci-design-studio): A Chromium web plugin that lets you quickly create custom themes in CSS that you can add as a custom style to your DaVinci flows. If you’re using custom HTML rather than a Form, and don’t have your own CSS to use, this is a great starting point.
* Flows:
  * [DaVinci DNA Workforce Solution](https://marketplace.pingone.com/item/davinci-dna-workforce-identity-solution) and [DaVinci DNA CIAM Solution](https://marketplace.pingone.com/item/davinci-dna-ciam-solution): A comprehensive set of modular functionality built out in reusable subflows, including PingOne MFA, PingID, PingOne Protect, and PingOne Verify. This is the gold standard for how to design flows and should get you 90% of the way there when building out your own designs
  * [Verified Trust for Workforce](https://marketplace.pingone.com/item/verified-trust-for-workforce-helpdesk-solution): A flow that enables a helpdesk user to confirm someone’s identity before performing sensitive actions. These flows are also built on the DNA methodology.

Alright: enough about the Marketplace. Let’s wire up this new node.

Connect the Action Decision Node off of your Find User node to the Parse Email Domain node, right-click the “True” action connection and switch it to “Any Trigger False”. In other words, we’re telling DaVinci to parse the email domain when the user does not exist in PingOne. Finally, within the Parse Email Domain node set the value of the variable **email** to the username you collected within the form.

![The Parse Domain from Email editor in which the username collected is being passed into the email variable](/img/davinci-primer-dynamic-populations/marketplace-email.png)
*Parsing the Email Domain*

Hit “Apply”, and then drag and create a new PingOne Connector off of your Parse Domain node. From the new node select the “Read Population” trigger again but this time change the Population search to “Use Alternative Identifier” and pass in the domain that was parsed in the prior node.

![The parsed email domain being passed as a variable into the read population query, used as an alternative identifier](/img/davinci-primer-dynamic-populations/domain-search.png)
*Reading the Population from the Alternate Identifier*

Now that we have queried Populations based on the identifier, let’s display the resulting selection. Unlike reading the Population based on ID, though, the same alternative identifier could exist in multiple populations - meaning our response could return more than one population! 

> **⭐ DaVinci Learning:** Cloning, Copying, and Pasting Nodes

To see this in action, we’ll display that result in a message like we did before. But rather than build a brand new node, right-click and **Clone** the existing HTTP node with our population's message. Cloning a node, or copy/pasting a selection of nodes (highlighted by holding and dragging your mouse), makes building larger DaVinci flows faster and easier.

![A zoomed-in screenshot of the Http node in which the "Clone" command is clearly visible - displayed after right-clicking the element](/img/davinci-primer-dynamic-populations/clone-node.png)
*Cloning a Node*

Wire the response of Read Population to the cloned node, and then update your cloned node to look at the `populations` response from the query rather than a single population. **An important note - the population value in your cloned node is still pointing to the original connector it was connected to (i.e. Read Population by ID). When cloning, make sure your references are matching the correct nodes.**

![A message node displaying all populations tied to the alternative identifier](/img/davinci-primer-dynamic-populations/debug-popsdata.png)
*Displaying all Populations*

With this all connected, applied, and deployed, add an alternative identifier to a Population that matches an email domain you can test. In this example, I’m using “example.com”. 

![The Population admin view in which a population, Example, has been created with "example.com" as an alternative identifier](/img/davinci-primer-dynamic-populations/pops-altid.png)
*Setting the Alternative Identifier on the Population*

Now, when I try the flow and enter a user who doesn’t exist using that email domain, the population (or populations) with that alternative identifier show up.

![The rendered DaVinci flow in which a new user using the matched email address is being entered](/img/davinci-primer-dynamic-populations/debug-popsdata-entry.png)
![The resulting population(s) data displaying to the user in the DaVinci flow](/img/davinci-primer-dynamic-populations/debug-popsdata-output.png)
*Retrieving Populations on Email Domain*

### Selecting a Population from a List

When the User does not exist and we query Populations based on the email domain, there are a series of outcomes we need to account for:

1. The email domain is associated to one Population
2. The email domain is associated to more than one Population - in which we let the user select their Population
3. The email domain isn’t associated with any Population - in which we fall back to the Default Population

To do so, we will implement some **Function Connectors** to branch our path. Function Connectors let us compare values and make decisions based on what we find, or run JavaScript to do things like modify or create values to pass into downstream connectors.

Right-click and **Delete** your HTTP node that displays populations. We won’t be needing it anymore.

> **⭐ DaVinci Learning:** Functions Connectors**

In its place connect a “Functions” Connector and select the “A > B” Trigger, adding the title “Any Pops Found?”. Set the **Type** to “Number”. 

In **A**, add the “rawResponse” field from your Read Populations node, hover over the field, and copy the handlebars reference that appears. It’ll look something like `{{local.68dcy1mtem.payload.output.rawResponse}}`.

![A hover-over view of the raw response variable for the PingOne connector with the handlebars value highlighted](/img/davinci-primer-dynamic-populations/variable-copy.png)
*Getting the Raw Response*

The triggers within the PingOne connector are making API calls behind the scenes, and if you know the [Read Populations](https://developer.pingidentity.com/pingone-api/platform/populations/read-all-populations.html) request you’re aware that there’s an additional field that’s hidden in this result set: the **size** - which informs you how many results have been returned from the request.

> **⭐ DaVinci Learning:** Accessing Hidden Fields

Just because the field doesn’t show up in the schema selector doesn’t mean that the field is missing - it just means that the connector didn’t define that value in its output schema. To get access to size, modify your copied handlebars to include size as a key, something like `{{local.68dcy1mtem.payload.output.rawResponse.size}}` and then copy/paste that change as Value A in your A > B node. Value B, then, should be set to `0`.

![The A > B connector in which the hidden size variable has been added into the A field to be compared against B](/img/davinci-primer-dynamic-populations/variable-hidden.png)
*Accessing Hidden Fields*

> **⭐ DaVinci Tip:** That seemingly arbitrary string of values after `local` is your Node ID - and it’s how each node is being referenced throughout your flow. You can find the Node ID under the header of the Node editor or by toggling “Show Node IDs” in your flow settings (top-right menu).

We now have a True/False branch we can use. Connect an “All Triggers True” response to the Function Connector with the Trigger “A > B”, the **Title** “More than One Pop Found?”, **A** set to the handlebars you used in the prior node, **B** set to 1, and the **Type** set to Number.

![The A > B node in which the size is being compared against the value 1](/img/davinci-primer-dynamic-populations/check-many-pops.png)
*Checking for Many Populations*

From that same Action Decision, wire an “Any Trigger False” to a PingOne Node with the Trigger “Read Population” and the selector set to “Default”.

![The Any Trigger False Branch of the Any Pops Found? node conencting to a Read Population trigger (PingOne Connector)](/img/davinci-primer-dynamic-populations/get-default.png)
*Getting the Default Pop When None are Found*

> **⭐ DaVinci Learning:** HTML Templates and Custom CSS/JavaScript

If more than one Population is found, we want the User to select their preferred Population. To do this, we are going to go back to the marketplace to insert the [Dynamic Dropdown node](https://marketplace.pingone.com/item/format-select-options-for-custom-html-node-group). This node pack is a bit different than our forms as it uses the HTTP Connector’s Custom HTML Template to render content, which gives us a great opportunity to review a more complex node.

First off, connect an “All Triggers True” trigger from the “More than One Pop Found?” node to the “Form Options Array” node that you just imported. Then, in that node you just connected, add the “populations” array gathered from your “Read Pop(s) by Alt ID” Node, setting the `optionLabelKey` to `name` and the `optionValueKey` to `id`.

![The Form Options Array function editor in which the populations data has been added with the mapped keys](/img/davinci-primer-dynamic-populations/form-options-array.png)
*Connecting the Populations List*

Create a new Population (I named mine “Example 2”) with the same alternative identifier, and then Apply, Save, Deploy, and Try the Flow. When you enter in a new User with that domain, you should see a dropdown with the associated Populations. But what gives - the page looks terrible!

![A header of "select an option" with a dropdown and a continue button. It is raw HTML with no formatting](/img/davinci-primer-dynamic-populations/display-dropdown-nocss.png)
*The Unformatted HTML Page*

The HTTP Node you imported lets you serve custom HTML, CSS, and JavaScript - it’s great in cases when you require heavier customization, interactivity, or interaction with an external CMS. In this case, our template that we imported relies on [Ping UX - DaVinci CSS](https://marketplace.pingone.com/item/ping-ux-css): so let’s import that into the flow.

To do so, click on the menu (three dots) in the top right corner and select “Flow Settings”, and then in the “Customizations” tab under “Page Customization” enable “Use Custom CSS”. There you can add in the Custom CSS Rules and the Custom CSS files provided in the Marketplace (or, for your own use cases, the custom assets you have from your own system).

![The settings window in which the custom css rules and custom css files have been added. Use Custom CSS is turned on](/img/davinci-primer-dynamic-populations/add-css.png)
*Adding the Custom Ping CSS*

> **⭐ DaVinci Tip:** Want a starting off point? Add the [DaVinci Design Studio](https://marketplace.pingone.com/item/davinci-design-studio) Chrome extension to get custom css that can be used in tandem with the Ping CSS provided.

Apply, Save, Deploy, and Try the Flow again. You’ll see the form shows up with the formatting matching the baseline branding in your form editor, with one exception: the Logo isn’t showing up.

![The same select page but in a nicely-formatted card with the appropriate font and input styling](/img/davinci-primer-dynamic-populations/display-dropdown-formatted.png)
*The Properly-Formatted Dropdown*

If you read through the Dynamic Dropdown node, you’ll see that it also (optionally) expects the [DaVinci Branding Variables Node](https://marketplace.pingone.com/item/davinci-branding-variables-node). This node lets you quickly and easily set the name of your company and the URL of your logo to be used across all HTTP nodes. Copy that node and place it at the beginning of your flow, connected to the first form in which you show the username.

![The first node connected is the Set Company & Logo node connected to the Username form](/img/davinci-primer-dynamic-populations/set-logo.png)
*Set Company & Logo Node*

You know the drill. Apply, Save, Deploy, Try. Now your logo shows up on the page too.

![The formatted dropdown with the Ping Identity Logo displayed](/img/davinci-primer-dynamic-populations/display-dropdown-withlogo.png)

> **⭐ DaVinci Learning:** Input and Output Schema

Both the Functions Connector and the HTTP Connector we just added have an **Input** and an **Output schema.** These schemas exist so that you can add specific data requirements on what data is needed for this node to function and what data will be returned for use in downstream nodes.

Let’s take a look at how the schemas are defined and what they look like in practice, starting with the “Form Options Array” Custom Function Node.

Clicking into this node, you’ll see that some JavaScript is being executed to parse an array of objects and format that array into something that the HTML node can use when building the dropdown. At the top of the node you’ll see a **Variable Input List** - this is where the variables that are being passed in the `params` object you can see referenced in line 3 of the script are coming from. These variables are given a name, a value, and a data type, which helps inform the script how to parse and interact with that params value.

Underneath the hood, these variables are being defined with an **Input Schema** using [JSON Schema](https://json-schema.org/) formatting. In the case of the Custom Function Node, how those parameters are passed into the JavaScript is handled for you.

At the other end of the JavaScript function, you’ll see that you’re returning an object with the key `options`). This contains the key/value pairs that are being referenced in the HTML node. But while this data is being returned, the JavaScript by itself doesn’t inform the downstream nodes what the data looks like. The **Output Schema** provides that definition.

Scrolling to the bottom of the node definition, you’ll see an Output Schema section with the following JSON:

```json
{
  "output": {
    "type": "object",
    "properties": {
      "options": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "label": {
              "type": "string"
            },
            "value": {
              "type": "string"
            }
          },
          "required": [
            "label",
            "value"
          ]
        }
      }
    }
  }
}
```

You’ll see that the **output** of this node is expected to return a property of name **options**, which is an **array that** contains **objects** whose properties are **label** and **value**: both strings.

When you click on the “Dynamic Dropdown” HTTP Node, you’ll see that the **options** array is passed and that you can reference it within the output of the Form Options Array node. Since the Label and Value were also defined, you can see that you can select that option directly from that object, to do things like target a specific element in the array.

![The "options" result appearing in a variables selector as defined from the Form Options Array node](/img/davinci-primer-dynamic-populations/output-variables.png)
*The Resulting Output*

Since we are in the HTTP node, let’s take a look at how the inputs are formatted - since all of the input elements up to the HTML template are special to just this created node. Instead of an input editor like we saw in the Function Node, the Inputs are defined in an Input Schema halfway down the page - right next to the Output schema and just below the Form validation rules.

> **⭐ DaVinci Tip:** If you are making your own HTML template, toggle the “Switch View” button to view the template using a classic code editor. When you’re adding variables, toggle back and use the “{}” button like you’ve done in other inputs.

The Input Schema here defines not only what values are expected to be available, but also how the inputs should be formatted for the user to fill in (`preferredControlType`) and whether or not the user can dynamically pass parameters from other nodes (`enableParameters`). The type and displayName indicate how the input is used and how it shows up for the user in this node. When editing the Input Schema, it’s important to know:

* Your inputs should be put within the `properties` object.
* The only `preferredControlType` available is `textField`.
* Your inputs **must use unique keys**.
* If you define a propertyName, **the propertyName must match the key.**
* It’s best to edit the Input Schema in a separate text editor. It makes editing and validating the schema easier.

Try adding another input - let’s call it `example`. It should look something like this:

```json
"example": {
"type": "string",
"displayName": "My New Example Input",
"preferredControlType": "textField",
"enableParameters": true
}
```

Apply your changes. You’ll see the new input ready to go with the key `example` available for use in your HTML, CSS, and JavaScript via handlebars.

![The new "My New Example Input" input created from the input schema update](/img/davinci-primer-dynamic-populations/input-schema.png)
*The New Input*

There’s no need to keep that input. You can remove or leave it in, your call.

The Input and Output Schema shows up throughout DaVinci - you’ll see it in Teleports (we’ll get to that later) and the overall Input Schema for your flow. These definitions allow you to strongly define what is being passed in and out of each node, from one flow to another, or from another application into the DaVinci flow.

> **⭐ DaVinci Learning:** Output Fields

While looking at the HTTP Node you probably noticed that there was nothing defined in the Output Schema but that node needs to return some data to the next node - in this case, what was selected and what button was pressed. Custom HTML Templates reference output data defined directly in the HTML itself and use the Output Fields list to link that data back to the node’s output.

In the Output Fields List section you should see two properties: one called `select` and one called `buttonValue`. Now, take a look in the HTML Template: you’ll see that our select has the `id` of `select` and our submit button has the `data-skbuttontype` of `form-submit` and the `data-skbuttonvalue` of `CONTINUE`.

For standard inputs, the `id` of that input is referenced in the Output Fields List. For buttons, that definition is handled by the `sk` attributes that live on it - in particular, that `skbuttonValue` is holding the value that you can reference later, and that “buttonValue” output field is referring to it.

For a list of SK Attributes, head to the documentation [here](https://docs.pingidentity.com/davinci/flows/davinci_sk_attributes.html). For a list of standard components that can be added to HTML (called SK-Components), such as polling, camera, recaptcha, and filepickers, go [here](https://docs.pingidentity.com/davinci/flows/davinci_sk_components.html).

You’ve also probably noticed that both the button and the select have a Data Type of String and a Control Type of Text Field. For all intents and purposes, those two fields should be left as-is: the conversion from your HTML to the other options may not work as expected.

> **⭐ DaVinci Tip:** When working with forms, make sure that the `id` of your form matches the `data-skform` html tag on your submission buttons - otherwise, your DaVinci flow won’t receive the results of your form data.

# Changing the Theme based on the Population 

You now have a means to identify the Population based on an existing User or a new User with the added capability for the User to select their Population if their username matches more than one. Let’s cater the experience based on the context that their Population provides.

Each Population can be assigned its own Theme. This Theme can then be referenced within DaVinci to stylize your user’s experiences.

> **⭐ DaVinci Learning:** Teleports

We have two different experiences that we will want to present to our user, a **Login** experience and a **Registration** experience, and more than one place in which we need to surface it. In the case of Registration, for instance, we both will need to perform the registration process if a population was selected by the user, if there was only one population found, or if the default population was selected instead. You could duplicate functionality, but if you want to change something later you’re going to have to find and change each implementation one at a time, which could lead to some pretty hard to troubleshoot errors. You could wire up each node to the same path, but not only does that get visually confusing you’ll have to handle all of the different outputs per node. This is where **Teleports** come in.

From a coding sense, think of a DaVinci flow as a **module**. It likely performs a series of things in combination with each other to enact a larger action - like registration and login. Within that module are **functions** - little bits of smaller atomic actions that are invoked for the module to do its thing. **Teleports** are these functions. Teleports allow you to break out smaller pieces of functionality into reusable segments that can be called on, looped from, and reused without any copy/pasting or complex wiring.

As a rule of thumb, if you are reusing the same nodes over and over again, consider putting them in a Teleport.

To start, scroll below the flow you’ve built so far and add an annotation to indicate what this teleport is for (I’m using “Login”). It’s best practice to write flows horizontally and individual teleports vertically, ideally with the teleports in execution order (for example, if you have a teleport that returns a session or an error, it should probably be at the bottom) - that way, you can read through the content of a flow similar to how you would read a flowchart or diagram.

Next, add in a Teleport connector with the trigger “Define a Start Node”. Once you do, you’ll see something pretty familiar - an Input schema! Teleport Start Nodes tell the flow where a new section begins and what data can be passed into that section: think of it like a function definition.

First off, change the name of your Teleport in the Settings tab - you’ll use that name to identify the Teleport later. I’m naming mine “Login”.

The Input Schema should look something like this:

```json
{
    "type": "object",
    "properties": {
        "p1UserId": {
            "type": "string",
            "displayName": "PingOne User ID",
            "preferredControlType": "textField",
            "enableParameters": true,
            "propertyName": "p1UserId"
        },
        "p1UserName": {
            "type": "string",
            "displayName": "PingOne Username",
            "preferredControlType": "textField",
            "enableParameters": true,
            "propertyName": "p1UserName"
        },
        "populationId": {
            "type": "string",
            "displayName": "Population ID",
            "preferredControlType": "textField",
            "enableParameters": true,
            "propertyName": "populationId"
        },
        "populationThemeId": {
            "type": "string",
            "propertyName": "populationThemeId",
            "displayName": "Population Theme ID",
            "enableParameters": true,
            "preferredControlType": "textField"
        },
		"populationIdPId": {
            "type": "string",
            "propertyName": "populationIdPId",
            "displayName": "Population IdP ID",
            "enableParameters": true,
            "preferredControlType": "textField"
        }
    }
}
```

This allows us to pass our PingOne User’s ID and Population Details into the Teleport.

> **⭐ DaVinci Tip:** Teleports stringify the data that comes into it and as a result, won’t be able to handle nested object definitions. Either define your required fields individually in the input schema, set them in a Flow Instance Variable, or write a custom function to parse the values from the stringified object.

In our case, since we only need two values from our Population it’s easiest to define them at the top level.

Now connect a Form Connector with the Show Form Trigger to your Teleport Start Node, selecting the “Password Authentication” form. Change the Form Theme to “Use Theme ID” and pass in the Theme ID from your parsed Population.

![The Show Form Editor window in which the Population Theme ID collected from the teleport has been added as a variable](/img/davinci-primer-dynamic-populations/teleport-inputs.png)
*Passing Parsed Object Variables into a Form*

You probably noticed that your variables list looked different inside this new section. That’s because the Teleport only has access to the data that was passed into it or variables set at the Global or Flow Instance level. Just like a function, the teleport can only see its parameters, top level constants, or run-time variables.

Back up to your Read Population by ID node (the one called after you discover an existing user), replace your HTTP node with a Teleport but this time use the **Go to Start Node** trigger and select your Login teleport as the start node.

Your variables for a PingOne User ID and Population Details will appear. Add in your PingOne User from your Find User node and the PingOne Population Details from the Find Population node.

![The Teleport Go To Start Node in which the inputs from gathering the population have been provided](/img/davinci-primer-dynamic-populations/teleport-inputs-start.png)
*Go to Teleport*

You’ve now set up a way to pass data into a teleport - basically, execute a function in your module. Apply, Save, Deploy, and Try the flow with a user that does exist. You should now hit the Password page.

![The resulting show password screen, without any themes on the population](/img/davinci-primer-dynamic-populations/flow-password.png)
*The Password Page*

But the theme looks just like the last page! Let’s change that.

Back in PingOne, under User Experience → Branding and Themes, go ahead and create a new theme. I’m using the base “Slate” theme for this example.

When you’re feeling good about the theme, under Directory → Populations select your Population and under Configuration assign your new theme.

The next time you go through the flow, you’ll see the theme change to your selected theme.

![The resulting show password theme with the "Slate" theme](/img/davinci-primer-dynamic-populations/show-password-slate.png)
*The Updated Theme*

# Enforcing Password Policies based on the Population 

You can set a unique Password Policy per Population so that each User segment can meet the security requirements set by their organization.

Since a User can only be a part of a single Population, the default node already knows their password policy during login. From the Password Authentication Node, connect a PingOne Connector with a Check Password Trigger, adding the teleported p1UserId and the password you collected from the form.

![The Check Password node pulling in the user ID and collected password](/img/davinci-primer-dynamic-populations/check-password.png)
*The Check Password Node*

A password can be in more than one state - it’s valid, invalid, or it needs to be updated in some way (e.g. it’s expired, doesn’t meet the new password policy requirements). If you wanted to just check if the password is/isn’t valid, creating responses using the “All Triggers True” and “Any Trigger False” responses works just fine. But, if you’d like to perform additional branches on validation you can use an A \== B (Multiple Conditions) Function Node to compare the `status` of the Check Password response to the response options shown in the [Check Password API reference](https://developer.pingidentity.com/pingone-api/platform/users/user-passwords/password-check.html).

For the sake of this example, we’ll create an outcome only on success - the error response will be handled for us.

Wire up the Check Password Node to an HTTP connector with the Custom HTML Message trigger and give it the message “Password Status:” including the `status` variable from your Check Password node. If you want to see all status information, set the path to be “Any Trigger Completes”.

Now, when you log in with the correct password you’ll see the status information that you’d likely branch from in a more robust flow.

![The resulting check password status message node. In this case, the message indicates that the password must be changed.](/img/davinci-primer-dynamic-populations/check-password-status.png)
*Returned Password Check Status*

# Enforcing Login with an External IdP based on the Population 

Some of your business groups and partnerships may have their own established Identity Provider that they’ll want to bring rather than having their users create and manage a separate account. Populations let you assign a default provider that, when a user federates with it, will associate them with the right Population (and as such the appropriate theming and managed administration).

Your teleport has already been configured to pull in the external IdP ID tied to your population - we’ll just need to add the branching path that decides whether to log in with that IdP instead of locally.

Connect your Teleport Start Node to a Function Connector with the “A is Empty” trigger and enable “Check undefined/null”. Connect the “All Triggers True” outcome to the Password Authentication node you made in the last section.

![An A is Empty function checking if the populationIdpId passed is undefined or null](/img/davinci-primer-dynamic-populations/local-login-check.png)
*Check for Local Login*

Next, connect the “Any Trigger False” outcome to a PingOne Authentication Connector with the trigger “Sign On with External Identity Provider”, using the `populationIdPId`, the `populationId`, and the `p1UserName` (in the Login Hint) you collected in the teleport.

> **⭐ DaVinci Tip:** If you don’t want users to be able to link to an existing account or create a new account from an external IdP, uncheck the “Link with PingOne User” checkbox.

![The External IdP Login node providing the population ID and external IDP ID](/img/davinci-primer-dynamic-populations/external-idp.png)
*External IdP Login*

To test, add an HTTP Connector with a custom HTML Message of your choosing - I’ve added the `statusCode` from the external IdP Login. When you Log in with a Population that doesn’t have an External IdP configured, you should see the password page as you did before. But in the case you have an External IdP configured, you’ll redirect to their login page and then come back with your login status.

![A message with the statuscode 200 indicating a successful login with an external IDP](/img/davinci-primer-dynamic-populations/statuscode.png)
*A Successful External IdP Login*

# Dynamic Registration

We’ve created the Teleport for Logging in, but we haven’t handled the users who don’t exist yet. Hold and drag with your mouse to highlight all of your login teleport nodes and Duplicate the selection. Next:

1. Rename the Annotation and the Teleport Start node to “Registration”.
2. In the Teleport Start Node, remove `p1UserId` since a user doesn’t exist yet.
3. Change the Password Authentication form to “Example - Registration” and pass in the `p1UserName` into the username and email field values.
4. Change the Check Password Trigger to the Create User Trigger and pass in the Username, Email, Password, and Population ID.

Finally, wire up the following connections in your New User branch at the top of your flow to the Go To Teleport for Registration:

* Using an “All Triggers True” after the “Get Default Pop” node, using the population details from the Get Default Pop and the Username from the Username form.
* Using an “Any Triggers False” after the “More than One Pop Found?” node, using the top Population result from the “Read Pop(s) by Alt Id” node and the Username from the Username form.
* Connecting from the Dynamic Dropdown node (replacing the HTTP node), connect a PingOne Connector with a Find Population trigger using the selected population ID, and then wire the true outcome to the Registration teleport using the newfound population details. Alternatively, replace the Get Population node with a Custom Function that returns the appropriate population details from the previous call.

Now, when you enter a new user you are taken to the appropriate registration page depending on your Population preferences, automatically using the appropriate password policy and external IdP.

![The registration page in which the user has to add their username, email, and password. The username and email address are auto-populated](/img/davinci-primer-dynamic-populations/flow-registration.png)
*New User Registration*

# Connecting the Flow to Your Application(s) 

Up to this point we have used the “Try Flow” button to test out our DaVinci flow. While this approach is great for testing or for flows that execute actions without the need for returning user context, in the real world it’s highly likely that your flows will complete with a user logging into an Application with an active session from PingOne.

To start, let’s make two more Teleports: one to return an authentication success and the other to return a failure.

Duplicate just the annotation and the Start Teleport Node from your Login section and then change the Input Schema for your Login Success Teleport to only have the `p1UserId` - that’s all we need to authenticate the user. Name the new section and teleport “Login Success”.

From that node connect a PingOne Authentication Connector with the trigger “Return Success Response (Redirect Flows)”. Configure this node to use the User ID from the Teleport and leave the Authentication Method as “pwd” since we aren’t performing any additional form of risk or multifactor-based authentication. You’ll see in this node you can optionally decorate the ID and Access Tokens with custom claims as needed by your downstream application.

![The return success response which is providing the p1userid collected in the teleport](/img/davinci-primer-dynamic-populations/return-success.png)
*The Success Route*

Duplicate the nodes and annotations, renaming the new section and teleport “Login Failure”. Inside the teleport, change the input schema to capture the Error Message:

```json
{
	"type": "object",
	"properties": {
		"errorMessage": {
			"type": "string",
			"displayName": "Error Message",
			"preferredControlType": "textField",
			"enableParameters": true,
			"propertyName": "errorMessage"
		},
		"errorDescription": {
			"type": "string",
			"displayName": "Error Description",
			"preferredControlType": "textField",
			"enableParameters": true,
			"propertyName": "errorDescription"
		},
		"errorReason": {
			"type": "string",
			"displayName": "Error Reason",
			"preferredControlType": "textField",
			"enableParameters": true,
			"propertyName": "errorReason"
		}
	}
}
```

Then, change the PingOne Authentication Connector’s trigger to “Return Error Response” with a custom error message that uses the inputs you just provided in the Teleport’s Input Schema (I set the Error Message to `invalid_request`).

![The login failure node which includes the error message responses from the teleport](/img/davinci-primer-dynamic-populations/return-failure.png)
*The Success and Failure Teleports*

Now that you have a means to inform the downstream applications of a success or failure, let’s wire it up to the appropriate paths in the flow.

| From Node | Connect Go To Teleport | With Variables |
| ----- | ----- | ----- |
| Check Password, Inside Login | Login Success (All True) | `p1UserId` from the teleport, or the `id` returned from Check Password |
| Check Password, Inside Login | Login Failure (Any False) | The error details from Check Password |
| External IdP Node, Inside Login | Login Success (All True) | `p1UserId` from the teleport, or the `id` returned from External IdP Login |
| External IdP Node, Inside Login | Login Failure (Any False) | The error details from External IdP Node |
| Create User, Inside Registration | Login Success (All True) | The `id` returned from Create User |
| Create User, Inside Registration | Login Failure (Any False) | The error details from Create User |
| External IdP Node, Inside Registration | Login Success (All True) | The `id` returned from the External IdP Login |
| External IdP Node, Inside Registration | Login Failure (Any False) | The error details from External IdP Node |

![The connected teleports to the nodes described in the above table](/img/davinci-primer-dynamic-populations/teleports.png)
*The Connected Teleports*

> **⭐ DaVinci Learning:** PingOne Flows

In order for our DaVinci flow to be connected to an application in PingOne, we will need to make it a PingOne Flow.

PingOne operates with DaVinci via an OIDC redirect. When a user attempts to log in to an application in PingOne, and DaVinci is protecting that application (via a **Flow Policy**), PingOne will redirect the user to DaVinci to perform its actions upon which it redirects back to PingOne to generate a user session and log the user in. The handoff looks something like this:

![A diagram which explains the flow between pingone, an application, and DaVinci](/img/davinci-primer-dynamic-populations/pingone-to-dv-diagram.png)
*DaVinci with PingOne*

Underneath your Flow Settings (click the three dots in the top-right, select Flow Settings and under the General tab) enable PingOne Flow. This converts the flow to run for either OIDC or SAML authentication with PingOne. 

![The settings switch that enables the flow to be a PingOne Flow. It is turned on](/img/davinci-primer-dynamic-populations/pingone-flow.png)
*Enabling a PingOne Flow*

Once you do this (and deploy!) you’ll notice that a little “P1” icon appears next to your title and if you try to use the “Try Flow” button you’ll be redirected to an error page that indicates that this flow requires PingOne to run.

![The default error message that appears when attempting to use the "Try Now" button on a PingOne flow. It says that the flow cannot be executed because it is configured as a PingOne flow](/img/davinci-primer-dynamic-populations/pingone-flow-error.png)
*Hitting “Try Flow” with a PingOne Flow*

DaVinci connects flows to PingOne applications using Flow Policies. Flow Policies let us define what flows and flow versions run, when they run, and what statistics we’d like to gather when users are driven through the flows (say, a successful outcome or a failure).

On the DaVinci sidebar, click on “Applications”. You should see a page that includes a PingOne SSO Connection. You can add more Flow Policies to the existing PingOne Connection or create a new one - this lets you create bundles of Policy categories, making it easier to identify when selecting the Policy in PingOne.

![The DaVinci Applications view in which the default PingOne SSO connection app is displayed](/img/davinci-primer-dynamic-populations/flow-app.png)
*DaVinci Applications*

Create “Add Application” and name it “Dynamic Populations”, then click on the created Application. You’ll see that this application is its own OIDC client with a unique client ID and client secret, and that the “Company ID” is actually the ID of the PingOne environment that this instance of DaVinci is tied to. This lets you take that same application and use it with non-PingOne systems you want DaVinci to interact with.

Select the “Flow Policy” tab and click on “Add Flow Policy”. **Make sure you add a name in the first step** (in my case I’m doing “Population Login and Registration”) and select “PingOne Flow Policy” since we will be interacting with a PingOne application. An option to Bypass flows will appear - skip that section and click Next.

![The first page in adding a flow policy. PingOne Flow Policy is selected and a title has been written](/img/davinci-primer-dynamic-populations/flow-policy-1.png)
*Adding a Flow Policy - Step 1*

You’ll now see a picker of PingOne-configured flows that you have created. Select the flow you’ve been working on (Dynamic Experiences with Populations) and click on “Latest Version”. 

> **⭐ DaVinci Tip:** DaVinci holds up to 100 past versions of your deployed flows - meaning that you can “lock” an application down to a specific version of a flow while you develop a new one without impacting your users’ experience.

![The second page in adding a flow policy. The specific flows and flow versions are selected](/img/davinci-primer-dynamic-populations/flow-policy-2.png)
*Adding a Flow Policy - Step 2*

Next, set your selected flow to 100% distribution and set the PingOne Authentication (Login Success) node as your Success Node.

> **⭐ DaVinci Tip:** DaVinci Flow Policies let you select multiple flows and flow versions in the same policy and then set a distribution to each instance within that policy. That, coupled with the success node definitions, allows you to A/B test and gather feedback on different approaches with your users simultaneously.

![The final step in adding a flow policy, in which the weight of each selected flow as well as their success criteria is defined](/img/davinci-primer-dynamic-populations/flow-policy-3.png)
*Adding a Flow Policy - Step 3*

At this point you have defined an Application and a Flow Policy within that Application. Let’s assign that policy to PingOne to be evaluated when a user attempts to login.

Back in PingOne, click on the “Applications” tab and submenu. Feel free to use a custom SAML or OIDC application, but for the sake of simplicity I’m going to use the “PingOne Self-Service - MyAccount” application.

Clicking on your application, select the “Policies” tab and click “Add Policies”. You’ll see a DaVinci tab in the subsequent editor - click on that tab and then check the new policy you just made.

![The application editing window in PingOne in which policies can be edited. The DaVinci Flow policy has been checked](/img/davinci-primer-dynamic-populations/flow-policy-assign.png)
*Assigning a Flow Policy to a PingOne Application*

By assigning this policy you have informed PingOne that for a user to successfully authenticate into this application they are required to first go through the DaVinci flow you created.

On the Overview tab, click on the Home Page URL of the Self Service Application (or in the case of your OIDC/SAML App, the appropriate calling URL) and open in an incognito tab. You’ll be greeted with a familiar sight - your DaVinci flow! Successfully complete login and you’ll be redirected to your My Account page (or wherever your app completes).

![The My Account Page for the example](/img/davinci-primer-dynamic-populations/myaccount.png)
*A Logged-In User*

# Conclusion

You now have a way to dynamically branch login and registration experiences based on contextual data supplied by the User and Population. You’ve also built a toolbox of relevant skills in DaVinci to build more complex, interesting, and incredible flows confidently.

DaVinci is an incredibly powerful tool, and coupling it with the strong capabilities and services within PingOne you can design the most secure and seamless experiences for your users.

For a copy of the completed flow, go [here](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/davinci-primer-dynamic-populations/dynamic_experiences_with_populations.json).

# Other Helpful Tricks

There are some other capabilities in PingOne that may prove useful as you work your way through more complicated flows.

## Variables

Variables allow us to set data that can either be stored and referenced at a particular instance of a running flow (that’s a Flow Instance Variable, like a module-level constant) or globally across all executions (that’s a Global Variable or Company Context, and it’s exactly what you think).

You can set variables within nodes in Flows (you actually did that in the “Set Company & Logo” node) as well as via the “Variables” tab on the left menu. It’s likely that you’ll set Flow Instance Variables dynamically within a Flow and Global Variables from within the menu (as those are normally static for all flow executions). A great example use case of Global Variables is the DNA Flows mentioned before - check out their [configuration guide](https://github.com/pingone-davinci/davinci-dna-flows/blob/main/DV-VariableObject-ConfigurationGuide.md) for an example of what you can store.

## Parsing Objects from a Teleport

If you had a complex object that you wanted to pass directly into a teleport, it’s useful to parse that object in a subsequent Custom Function Connector.

The function within that connector should be incredibly simple - return the name of the object you want to return and parse the passed object into JSON.

As an example, if you wanted to pass the Population object into a teleport directly you could create a script that looks something like this:

```javascript
module.exports = async ({params}) => {
	return {'population': JSON.parse(params.populationString)}
}
```

In a production use case you may want to add some parametrization and checking to validate that your object has all of the values you’re looking for.

Next, set your Output Schema to map to the object you want to reference.

Using the same use case, your output schema may look something like this:

```json
{
	"output": {
		"type": "object",
		"properties": {
 			"population": {
				"type": "object",
				"propertyName": "population",
				"displayName": "Population",
				"properties": {
					"id": {
						"type": "string",
						"displayName": "Population ID",
						"propertyName": "id"
					},
					"name": {
						"type": "string",
						"displayName": "Population Name",
						"propertyName": "name"
					},
					"preferredLanguage": {
						"type": "string",
						"displayName": "Preferred Language",
						"propertyName": "preferredLanguage"
					},
					"passwordPolicy": {
						"type": "object",
						"displayName": "Password Policy",
						"propertyName": "passwordPolicy",
						"properties": {
							"id": {
								"type": "string",
								"displayName": "Password Policy ID",
								"propertyName": "id"
							}
						}
					},
					"theme": {
						"type": "object",
						"displayName": "Theme",
						"propertyName": "theme",
						"properties": {
							"id": {
								"type": "string",
								"displayName": "Theme ID",
								"propertyName": "id"
							}
						}
					},
					"defaultIdentityProvider": {
						"type": "object",
						"displayName": "Default Identity Provider",
						"propertyName": "defaultIdentityProvider",
						"properties": {
							"id": {
								"type": "string",
								"displayName": "Default Identity Provider ID",
								"propertyName": "id"
							},
							"type": {
								"type": "string",
								"displayName": "Default Identity Provider Type",
								"propertyName": "type"
							}
						}
					}
				}
			}
		}
	}
}
```

This approach gives us a parsed object we can reference in later nodes.