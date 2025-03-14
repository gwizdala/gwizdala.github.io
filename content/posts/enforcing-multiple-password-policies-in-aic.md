---
draft: false
date: '2025-03-10T02:00:00+00:00'
title: 'Enforcing Multiple Password Policies per Realm in PingOne Advanced Identity Cloud'
description: 'Create, Evaluate, and Enforce Dynamic Password Policies from External and Internal PDPs'
summary: Learn how to dynamically evaluate custom password policies during a Journey in PingOne Advanced Identity Cloud
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

# Introduction

PingOne Advanced Identity Cloud (AIC) enables [simple administrative configuration of password policies](https://docs.pingidentity.com/pingoneaic/latest/realms/password-policy.html), which is enforced across the Realm any time a user creates, updates, or uses their password.

While a single password policy works in many Identity and Access Management (IAM) use cases, it may not be enough in instances where:

* Different groups/subdivisions of identity (think departments, teams, franchises, brands) require different levels of enforcement  
* Different SSO Applications require stronger password policy requirements based on the sensitivity of data  
* Different Provisioning Applications enforce different password policies on their identities downstream  
* External Identities (think partnerships, consultancies, Bring-Your-Own-Identity (BYOI)) enforce different password policies for the identities you’re consuming

# Enforcing Multiple Password Policies per Realm

This How-To will teach you how you can create, manage, and enforce multiple password policies per Realm by using Journeys. It is broken down into the following parts:

1. [Custom Password Policy Evaluation with Library Scripts](#custom-password-policy-evaluation-with-library-scripts)  
2. [Enforcement of Custom Password Policy within Journeys](#enforcement-of-custom-password-policy-within-journeys)  
3. [Management of Custom Password Policies within Organizations](#management-of-custom-password-policies-within-organizations)

By the end of this How-To you’ll have a Library Script, a Journey, and a configured Organization object that allows you to dynamically set Password Policy and enforce it per Journey, Application, and Organization. You’ll also have a genericized password policy approach that can be managed, updated, and enforced anywhere else you may need it in AIC, such as on a User, Role, or Group.

> **Disclaimer: The custom password policy we create is validated and enforced at a JOURNEY LEVEL. The password policy is not enforced at an API or IDM level - meaning that a user’s password is still enforced by the default policy defined at the Realm.**

This How-To expects a level of familiarity with Journeys, Journey Scripting, and Identity Management.

Let’s get started!

# Custom Password Policy Evaluation with Library Scripts {#custom-password-policy-evaluation-with-library-scripts}

First off, let’s create a standardized way to enforce our custom password policy - and what better way to do it than in a [Library Script](https://docs.pingidentity.com/pingoneaic/latest/am-scripting/library-scripts.html)?

Library Scripts are reusable blocks of code that can be used across AIC. They allow us to build generic, centralized, function sets that we can reference throughout actions like Journeys without having to copy/paste/manage the code in a bunch of different places.

To create a Library Script, go to Scripts → Auth Scripts, Select “New Script”, and then select “Library” as the Script Type (under the “Other” category).

{{< column-container >}}
{{< column >}}

![A screenshot of the Auth Scripts tab in the Admin UI](../images/enforcing-multiple-password-policies-in-aic/select-auth-script.png)

{{< /column >}}
{{< column >}}

![A screenshot of the "New Script" button being pressed](../images/enforcing-multiple-password-policies-in-aic/new-script.png)

{{< /column >}}
{{< /column-container >}}

![A screenshot of the Library Script Button from the New Script Dialog](../images/enforcing-multiple-password-policies-in-aic/select-library-script.png)

_Creating the Library Script_

Let’s call this new script `library_passwordPolicy`. I like starting all of the library scripts with the `library_` prefix because it is easy to search and quickly tells me what the script is for.

In the code section, copy/paste the library script found at this [GitHub link](https://github.com/gwizdala/lib-ping/blob/main/Library%20Scripts/library_passwordPolicy.js). Once saved, the library script is now available throughout Journeys using Scripted Decision Nodes. 

## Using the Library Script

This section gives you an overview of how to use the Password Policy Library Script in your own custom scripts. If you’d like to use the example provided in this documentation, jump straight to [Enforcement of Custom Password Policy within Journeys](#enforcement-of-custom-password-policy-within-journeys).

### Importing the Library Script

To use your new password policy library script in a Journey, drag and drop in a [Scripted Decision Node](https://docs.pingidentity.com/auth-node-ref/latest/auth-node-scripted-decision.html) with Next Gen Scripting selected. In the code editor provided, you can select the “Libraries” tab (it’s the little document icon in the top-left corner), open up the `library_passwordPolicy` dropdown, and click the functions you’d like to be auto-added into your editor.

![A screenshot of the Script Editor in which the Password Policy Library has been selected and inputted](../images/enforcing-multiple-password-policies-in-aic/use-library-script.png)

_Using the Library Script in an Editor_

You’ll need two function calls in total: the first to add the library to your script, and the second to evaluate the password based on your custom policy.

### Evaluating the Password

Since the library script import is fairly self-explanatory, let’s look at the evaluation function in more detail. Note that documentation and examples are also in the library script itself that you added to your tenant.

`evaluatePassword()` takes four arguments:

1. `caller` The Context that the function is being called in, `this`  
2. `passwordPolicy` The Password Policy that is being evaluated  
3. `password` The Password that is being evaluated against the Password Policy  
4. (Optionally) `userContext` the User and Organization Information that is being evaluated along with the Password and Password Policy

From these arguments, it’ll return an evaluation including whether each of the policy requirements succeeded or failed.

The Password Policy itself is an object with the following options available:

| Password Policy Option | Type | Description |
| ----- | ----- | ----- |
| `minPasswordLength` | Number | The minimum password length allowed. Must be greater than 0 |
| `maxPasswordLength` | Number | The maximum password length allowed. Must be greater than or equal to `minPasswordLength` |
| `requireNoCommonlyUsedPasswords` | Boolean | No commonly used passwords. Uses the [EXTERNAL HaveIBeenPwned API](https://haveibeenpwned.com/API/v2#PwnedPasswords) - use at your own risk |
| `requireUpperCase` | Boolean | > 1 Upper Case Letter (English) |
| `requireLowerCase` | Boolean | > 1 Lower Case Letter (English) |
| `requireNumber` | Boolean | > 1 Number (0-9) |
| `requireSpecialChar` | Boolean | > 1 Special Character (e.g. !@#$%) |
| `requireNoRepeatChars` | Boolean | No Repetitive Characters (e.g. aaa) |
| `disallowedUserAttributes` | Array[String] | A list of user attributes not allowed within the password - requires a `uid` or `objectAttributes` stored in `userContext` |
| `disallowedOrgAttributes` | Array[String] | A list of organization attributes not allowed within the password - requires an `orgId` stored in the `userContext` |

These options should look pretty familiar to you - they mimic much of the functionality from the [Realm’s Password Policy](https://docs.pingidentity.com/pingoneaic/latest/realms/password-policy.html).

The User Context is used in evaluation when the Password Policy contains `disallowedUserAttributes` and/or `disallowedOrgAttributes`. It can contain three values:

1. `uid` The User’s `_id`  
2. `orgId` The Organization’s `_id`  
3. `objectAttributes` The identity information currently gathered about the user. This is most likely used during Registering or Updating a user’s account.

The returned evaluation is an object with the following structure:

* `valid` a boolean that indicates if the policy as a whole passed (`true`) or failed (`false`). The policy will fail if any option within that policy has failed  
* `evaluation` an array of results evaluating each individual password policy option. The `evaluation` objects have the following structure:  
  * `policy` The policy description, localized (by default English)  
  * `passed` Whether or not this policy passed (`true`) or failed (`false`)

### Testing Examples

Let’s take a look at some examples of how this password policy could be used.

To start, let’s try testing a password policy that requires an Upper Case Character and No Repeating Characters.

{{<details title="Example Password Policy that tests only the password (Click to View)">}}
```javascript
const passwordPolicy = require("library_passwordPolicy");
const passwordPolicy = {
   requireUpperCase: true,
   requireNoRepeatChars: true
};

logger.info(passwordPolicy.evaluatePassword(this, passwordPolicy, 'Example!123ee')); 
/*
Result:
{ 
valid: false, 
evaluation: [
{ 
policy: "Must not contain any repeated characters (e.g. aaa)",
passed: false 
}, 
{ 
policy: "Must contain at least one uppercase character",
passed: true
}
]
}
*/
logger.info(passwordPolicy.evaluatePassword(this, passwordPolicy, 'example!')); 
/*
Result:
{ 
valid: false,
evaluation: [
{ 
policy: "Must not contain any repeated characters (e.g. aaa)",
passed: true 
},
{ 
policy: "Must contain at least one uppercase character",
passed: false
}
]
}
*/
```
{{</ details >}}

Next, let’s evaluate a password policy that evaluates if the password contains any values that are a part of the User or the Organization.

{{<details title="Example Password Policy that tests data on the User and Org (Click to View)">}}
```javascript
const passwordPolicy = require("library_passwordPolicy");  
const userConfig = {
uid: '123-456-789',
orgId: '098-765-432',
objectAttributes: {
givenName: 'MyNew',
sn: 'Name',
userName: 'example'
    }
};
  
const userAttributePasswordPolicy = {
disallowedUserAttributes: ['userName', 'givenName', 'sn', 'mail']
};
// User's username is "example"
logger.info(passwordPolicy.evaluatePassword(this, userAttributePasswordPolicy, 'Example!123ee', userConfig)); 
/*
Result:
{
valid: false,
evaluation: [
{ 
policy: "Must not contain values that are part of your account (e.g. name, email, username)",
passed: false 
}
]
}
*/
  
const orgAttributePasswordPolicy = {
disallowedOrgAttributes: ['name', 'description']
};
// Org's name is 'Org'
logger.info(passwordPolicy.evaluatePassword(this, orgAttributePasswordPolicy, 'Org!123ee', userConfig)); 
/*
Result:
{
valid: false, 
evaluation: [
{ 
policy: "Must not contain easy-to-guess values that relate back to the platform (e.g. the Company Name)",
passed: false 
}
]
}
*/
```
{{</ details >}}

Looking good! Now we can pass custom password policies to be evaluated within our scripts.

# Enforcement of Custom Password Policy within Journeys {#enforcement-of-custom-password-policy-within-journeys}

Now that we have a way to validate our own password policies, let’s enforce them within the context of a Journey.

To start, let’s configure our Realm’s password policy. We need to make sure that the global policy doesn’t conflict with the custom policy that we’ll be enforcing.

Under Security → Password Policy, update your policy to what you’ll consider the least restrictive that your organization allows. This will be the baseline that our policies will follow, and will continue to be enforced on IDM actions such as saving or updating a user’s password.

For my example, I’m going to remove the “Part of Alpha realm - User attributes”, remove that it must contain any of the character type requirements, change the minimum character length to 4, and prevent reuse of the last 3 passwords. This is likely too lenient in a real-world use case but ensures during this demonstration that we won’t have any policy conflicts. If you have your own policy set you can skip this: later in this document we’ll walk through [how to set your global policy to be the default established for all custom policies moving forward](#ensuring-the-default-policy).

![A screenshot of the password policy set for the realm using the parameters provided](../images/enforcing-multiple-password-policies-in-aic/set-default-policy.png)

_Setting the Realm Default Password Policy_

Next, navigate to “Journeys” inside of your tenant and duplicate the Registration Journey, setting the following defaults:

| Input | Value |
| ----- | ----- |
| Name | DynamicPasswordPolicy_Registration |
| Identity Object | Alpha realm - Users `managed/alpha_user` |
| Description (optional) | An example journey showcasing how you can enforce dynamic password policies at registration |

Since this How-To is focused on dynamic password policy enforcement, we’ll remove the Accept Terms and Conditions node, the KBA Definition node, and the Email Suspend Node from our copy. Your Journey should look something like this:

![A screenshot of the Journey editor where a pared-down copy of the Registration Journey is shown](../images/enforcing-multiple-password-policies-in-aic/initial-journey.png)

_The Starter Journey_

Click on the Platform Password node and deselect the “Validate Password” option. Since we are evaluating off of our own custom policy we won’t want to confuse the user by listing a separate one.

![A screenshot of the "Validate Password" check disabled](../images/enforcing-multiple-password-policies-in-aic/validate-password-unchecked.png)

_Unchecking Password Validation_

Next, drag and drop in a Scripted Decision node and give it the outcomes `Valid`, `Invalid`, `Undefined`, and `Error`. Create a new Next Gen Script for this node called “Evaluate Password Policy from State” and copy/paste into it the following code:

{{<details title="`Evaluate Password Policy From State.js` (Click to View)">}}
```javascript
/*
  Evaluate inputted password based on a dynamic policy configuration

  This scripted decision node expects the following library scripts defined:
  - library_passwordPolicy

  This scripted decision node expects a user, their password, and a password policy stored in state.
 
 The scripted decision node needs the following outcomes defined:
 - Valid       // The user's password meets the requirements in the password policy
 - Invalid     // The user's password does not meet the requirements in the password policy
 - Undefined   // Either the user's uid, password, or password policy was not provided
 - Error
 
 Author: @gwizdala

 */

//// IMPORTS
var libraryPasswordPolicy = require('library_passwordPolicy');

//// CONSTANTS
var SCRIPT_NAME = "EvaluatePasswordPolicyFromState";
var NodeOutcome = {
  VALID_PASSWORD: "Valid",
  INVALID_PASSWORD: "Invalid",
  UNDEFINED: "Undefined",
  ERROR: "Error"
};

//// MAIN
(function () {
    // Default Outcome
    outcome = NodeOutcome.INVALID_PASSWORD;
    
    try {
        var password = nodeState.get('password');
        var uid = nodeState.get('_id');
        var orgId = nodeState.get('orgId');
        var objectAttributesRaw = nodeState.getObject("objectAttributes");
        var objectAttributes = objectAttributesRaw ? JSON.parse(objectAttributesRaw) : null;
        var userContext = {
            uid: uid,
            orgId: orgId,
            objectAttributes: objectAttributes
        };
        
        var passwordPolicy = nodeState.get("passwordPolicy");
        // Example config - useful for testing
        // var passwordPolicy = {
        //     minPasswordLength: 8,
        //     maxPasswordLength: 10,
        //     disallowedUserAttributes: ['userName', 'givenName', 'sn', 'mail'],
        //     disallowedOrgAttributes: ['name'],
        //     requireNoCommonlyUsedPasswords: true,
        //     requireUpperCase: true,
        //     requireLowerCase: true,
        //     requireNumber: true,
        //     requireSpecialChar: true,
        //     requireNoRepeatChars: true
        // };
        
        if (password && passwordPolicy) {
            var result = libraryPasswordPolicy.evaluatePassword(this, passwordPolicy, password, userContext);

            if (result.valid) {
                outcome = NodeOutcome.VALID_PASSWORD;
            } else {
                nodeState.putShared("passwordEvaluationResult", JSON.stringify(result.evaluation));
                outcome = NodeOutcome.INVALID_PASSWORD
            }
        } else {
            outcome = NodeOutcome.UNDEFINED;
        }
      
    } catch(e) {
      logger.error(`Password policy evaluation failed: "${e}"`);
      outcome = NodeOutcome.ERROR;
    }

    action.goTo(outcome);
}());
```
{{</ details >}}

This script will evaluate the password and branch the result according to if it passed the provided password policy. If it does not pass, the branching path will include the result of the evaluation stored in Shared State. Let’s next make a page that shows the user how to fix their password to match policy.

First, drag in a Page Node and give it the header “Password Policy Evaluation Failed”, the message “Your password doesn't meet the policy requirements set by your organization. Please update your password to continue.”, and the footer “Password Policy Requirements:”. Connect this to the `Invalid` outcome of your Password Policy evaluation node.

Next, drag a Scripted Decision Node into the Page Node you just created. This node will have the same outcomes of `Valid`, `Invalid`, `Undefined`, and `Error`. Create a new Next Gen Script with the name “Enforce Password Policy on Input” and copy/paste into it the following code:

{{<details title="`Enforce Password Policy on Input.js` (Click to View)">}}
```javascript
/*
  Evaluate inputted password based on a dynamic policy configuration

  This scripted decision node expects the following library scripts defined:
  - library_passwordPolicy

  This scripted decision node expects a password policy stored in state.
 
 The scripted decision node needs the following outcomes defined:
 - Valid       // The user's password meets the requirements in the password policy
 - Invalid     // The user's password does not meet the requirements in the password policy
 - Undefined   // Either the user's uid, password, or password policy was not provided
 - Error
 
 Author: @gwizdala

 */

//// IMPORTS
var libraryPasswordPolicy = require('library_passwordPolicy');

//// CONSTANTS
var SCRIPT_NAME = "EnforcePasswordPolicyOnInput";
var NodeOutcome = {
  VALID_PASSWORD: "Valid",
  INVALID_PASSWORD: "Invalid",
  UNDEFINED: "Undefined",
  ERROR: "Error"
};

//// HELPERS
// Displays the results of an evaluation within the UI, including what succeeded and what failed.
// Since we are evaluating on the back-end with a library script, these actions aren't happening in real-time.
// Rather, we are displaying an evaluation result on submit.
function displayPasswordEvaluationResults(evaluation) {
    var css = `\
    .policy {\
        text-align: left;\
        list-style-type: none;\
        color: #ec4949;\
    }\
    .policy li::before {\
        content: "\\u274C";\
        margin-right: 0.5em;\
    }\
    .policy--passed {\
        color: #3fa13f;\
    }\
    .policy--passed::before {\
        content: "\\u2713"!important;\
    }\
    `;
    var passwordPolicyListID = 'passwordPolicyEvaluationRequirements';
    var evaluationScript = `\
        document.head.appendChild(document.createElement("style")).innerHTML = '${css}';\
        var passwordList = document.getElementById('${passwordPolicyListID}');\
        var footer = document.getElementsByClassName('card-footer')[0];\
        var header = document.getElementsByClassName('card-header')[0];\
        var body = document.getElementsByClassName('card-body')[0];\

        var policies = ${evaluation};\
        var policyList = document.createElement('ul');\
        policyList.setAttribute('class', 'policy');\
        policyList.setAttribute('id', '${passwordPolicyListID}');\
        policies.forEach(function(policy) {\
            var policyListElement = document.createElement('li');\
            if (policy.passed) {\
                policyListElement.setAttribute('class', 'policy--passed');\
            }\
            policyListElement.innerHTML = policy.policy;\
            policyList.appendChild(policyListElement);\
        });\
        
        // Append the policy \n\
        if (!!passwordList) {\
            passwordList.parentNode.removeChild(passwordList);\
        }\
        
        if (!!footer) {\
            footer.appendChild(policyList);\
        } else if (!!header) {\
            header.appendChild(policyList);\
        } else if (!!body) {\
            body.appendChild(policyList);\
        }
    `;

    return evaluationScript;
}

//// MAIN
(function () {
    // Default Outcome
    outcome = NodeOutcome.INVALID_PASSWORD;
    
    try {     
        var uid = nodeState.get('_id');
        var orgId = nodeState.get('orgId');
        var objectAttributesRaw = nodeState.getObject("objectAttributes");
        var objectAttributes = objectAttributesRaw ? JSON.parse(objectAttributesRaw) : null;
        var userContext = {
            uid: uid,
            orgId: orgId,
            objectAttributes: objectAttributes
        };

        // Check if an evaluation already occurred. If so, present the results of the evaluation.
        var evaluation = nodeState.get("passwordEvaluationResult");
        
        var passwordPolicy = nodeState.get("passwordPolicy");
        // Example config - useful for testing
        // var passwordPolicy = {
        //     minPasswordLength: 8,
        //     maxPasswordLength: 10,
        //     disallowedUserAttributes: ['userName', 'givenName', 'sn', 'mail'],
        //     disallowedOrgAttributes: ['name'],
        //     requireNoCommonlyUsedPasswords: true,
        //     requireUpperCase: true,
        //     requireLowerCase: true,
        //     requireNumber: true,
        //     requireSpecialChar: true,
        //     requireNoRepeatChars: true
        // };

        if (passwordPolicy) {
            if (!evaluation) {
                // gather the evaluation information if no pre-existing evaluation was provided
                evaluation = libraryPasswordPolicy.evaluatePassword(this, passwordPolicy, "", userContext).evaluation; // an empty password will always fail unless the password policy is empty.
            }
                
            if (callbacks.isEmpty()) {
                callbacksBuilder.passwordCallback("Password", false);
                callbacksBuilder.scriptTextOutputCallback(displayPasswordEvaluationResults(evaluation));
            } else {
                var password = callbacks.getPasswordCallbacks().get(0);
                var result = libraryPasswordPolicy.evaluatePassword(this, passwordPolicy, password, userContext);
                
                nodeState.putTransient("password", password);
                nodeState.putShared("passwordEvaluationResult", JSON.stringify(result.evaluation));

                if (result.valid) {
                    nodeState.mergeTransient({'objectAttributes': { 'password': password }});
                    outcome = NodeOutcome.VALID_PASSWORD;
                } else {
                    // Since we can't loop back to the same node, it's suggested to loop back to a previous node and continue from there
                    // or to an error message.
                    outcome = NodeOutcome.INVALID_PASSWORD
                }
            }
        } else {
            outcome = NodeOutcome.UNDEFINED;
        }
      
    } catch(e) {
      logger.error(`Password policy enforcement failed: "${e}"`);
      outcome = NodeOutcome.ERROR;
    }

    action.goTo(outcome);
}());
```
{{</ details >}}

Now we have a means for the user to see what policies they have successfully passed and which they have failed so that they can update their password.

Connect the `Valid` outcomes of both nodes to the Create Object Node, the `Undefined` and `Error` outcomes to the Failure node, and the `Invalid` outcome of the Enforce Password Node back to the Evaluate Password node. Your Journey should look something like this:

![A screenshot of the Journey editor that includes the password validation and enforcement](../images/enforcing-multiple-password-policies-in-aic/journey-evaluate-enforce.png)

_Connecting the Evaluation and Enforcement Nodes_

## Testing Custom Enforcement

In the real world, we’d be getting the password policy from somewhere, be it a service like PingOne Authorize or a managed object like an Organization. So that we can test this example, however, let’s drag in an additional Scripted Decision Node with the outcome of `true`, name of “Load Password Policy” and paste in the following code block.

{{<details title="`Load Password Policy.js` (Click to View)">}}
```javascript
/*
  Loads in a password policy to be evaluated within the Journey.

  This example uses a testing password policy, but you could just as easily pass this policy as the result of:
  
  - An HTTP request to an external PDP (like the Policy Engine or PingOneAuthorize)
  - Retrieved Data on the User, Group, Organization, or other Managed Object
  - Information from an external IdP (such as inside an Assertion)
  - Results of a branching Journey path (e.g. requiring stricter policies to access particular resources)
 
 The scripted decision node needs the following outcomes defined:
 - true
 
 Author: @gwizdala

 */
var passwordPolicy = {
    minPasswordLength: 8,
    maxPasswordLength: 10,
    disallowedUserAttributes: ['userName', 'givenName', 'sn', 'mail'],
    disallowedOrgAttributes: ['name'],
    requireNoCommonlyUsedPasswords: true,
    requireUpperCase: true,
    requireLowerCase: true,
    requireNumber: true,
    requireSpecialChar: true,
    requireNoRepeatChars: true
};

nodeState.putShared("passwordPolicy", passwordPolicy);

outcome = "true";
```
{{</ details >}}

Wire up the Page node where you are collecting the user’s attributes and password to this new node, and then the outcome of that node to your Evaluate Password Policy node. Your Journey will ultimately look like this:

![A screenshot of the Journey editor that includes the testing policy](../images/enforcing-multiple-password-policies-in-aic/journey-test.png)

_Connecting the Test Policy_

Copy the Journey Preview URL and open it up in a Guest, Incognito, or separate browser window. You’ll be prompted with a signup screen.

Let’s test our password policy. I’ll be entering the following details, but feel free to use your own.

| Key | Value |
| ----- | ----- |
| Username | example |
| First Name | Test |
| Last Name | User |
| Email Address | badUser@example.com |
| Password | example1234 |

![A screenshot of the end-user registration UI in which the above values have been entered](../images/enforcing-multiple-password-policies-in-aic/ui-entry-baduser.png)

_The User with the Bad Password_

Hitting next, you’ll find that you need to update your password to meet the policy requirements. In my case, my example password is greater than 10 characters long, contains my username, has been found in the list of breached passwords, doesn’t have an uppercase character, and doesn’t have a special character.  

![A screenshot of the user's password being flagged as invalid along with the list of reasons why](../images/enforcing-multiple-password-policies-in-aic/ui-enforce-baduser.png)

_Enforcing the Password Policy_

Let’s update our password to meet the requirements. I’m using the password `G3#m8+hv4`. 

Inputting the new password, your user has been registered and is in the platform. Nice!

![A screenshot of the user successfully registering an account with the updated password](../images/enforcing-multiple-password-policies-in-aic/ui-success-baduser.png)

_Registering the User_

The same scripted decision nodes can be used to enforce policies during authentication - there’ll be some example Journeys for you at the end of this How-To.

# Management of Custom Password Policies within Organizations {#management-of-custom-password-policies-within-organizations}

Now that we have a way to enforce a custom password policy, let’s create a means for our administrators (and delegated administrators) to enforce different policies based on their different business units.

To do so, we’re going to use the Organization Object. This gives us a built-in way to templatize functionality and delegate it to groups of owners and users.

## Configuring the Managed Object

To start, head over to your `Alpha_Organization` managed object config in your IDM native console (under Native Consoles → Identity Management → Configure → Managed Objects) and create a new attribute with the following properties:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `passwordPolicy` | Password Policy | Object | False |

Select that attribute and add the following properties to it, all with “Required” set to false:

| Password Policy Option | Label (Optional) | Type |
| ----- | ----- | ----- |
| `minPasswordLength` | Minimum Password Length | Number |
| `maxPasswordLength` | Maximum Password Length | Number |
| `disallowedUserAttributes` | Forbidden User Attributes | Array[String] |
| `disallowedOrgAttributes` | Forbidden Organization Attributes | Array[String] |
| `requireNoCommonlyUsedPasswords` | Require No Passwords Found on HaveIBeenPwned (External) | Boolean |
| `requireUpperCase` | > 1 Upper Case Letter (English) | Boolean |
| `requireLowerCase` | > 1 Lower Case Letter (English) | Boolean |
| `requireNumber` | > 1 Number (0-9) | Boolean |
| `requireSpecialChar` | > 1 Special Character (e.g. !@#$%) | Boolean |
| `requireNoRepeatChars` | No Repetitive Characters (e.g. aaa) | Boolean |

Your `passwordPolicy` object should look like this:  

![A screenshot of the Native IDM Console showing the passwordPolicy object created in the Organization](../images/enforcing-multiple-password-policies-in-aic/idm-policy.png)

_The Managed Password Policy Object_

With this configuration in place, when we head over to an Organization we’ve created we’ll see a new tab entitled “Password Policy” with the options we’ve configured.  

![A screenshot of the Admin console where the Password Policy is available to be edited on an Organization](../images/enforcing-multiple-password-policies-in-aic/idm-policy-object.png)

_Viewing the Policy_

## Ensuring the Default Policy {#ensuring-the-default-policy}

To ensure that each new password policy created meets the requirements of the global policy, you’ll likely want to do the following:

1. Set [Validation Policies](https://docs.pingidentity.com/pingoneaic/latest/idm-objects/configuring-default-policy.html) on each individual attribute that needs to be enforced to a particular standard, such as `minimumNumber` or `maximumNumber`  
2. Set Default Values (Under Details → Default) so that when a new organization is created it starts with the defaults needed  
3. Set [Script Triggers](https://docs.pingidentity.com/pingoneaic/latest/idm-scripting/script-triggers.html) that enforce particular values cannot be removed or updated

Regarding the script trigger example, here’s a code snippet that would ensure that the `userName` property is always set within the Forbidden User Attributes array:

```javascript
// onUpdate, onCreate
var newProperty = !!property ? property : [];

if (!newProperty.includes('userName')) {
    newProperty.append('userName');
}

property = newProperty;
```

## Enforcing the Organization’s password policy within a Journey

Let’s update our Journey to validate the user’s password against the Organization they’re registering to.

Heading back into our `DynamicPasswordPolicy_Registration` Journey, let’s add two new Scripted Decision Nodes: one which will find what Organization we’re looking for by a query parameter we’ll pass in and another that retrieves the Password Policy from that Organization. If you’ve read the How-To on [Dynamic Branding](https://gwizkid.com/posts/dynamically-branding-journeys-in-aic/introduction/) this will be very familiar to you.

The first scripted decision node, entitled “Get Org ID by Query Parameter”, will have the outcomes `Found`, `Missing`, and `Error`. We’ll attach it to the Start Node.

{{<details title="`Get Org ID by Query Parameter.js` (Click to View)">}}
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
{{</ details >}}

The next scripted decision node, entitled “Get Password Policy From Organization Metadata”, will have the outcomes `Success` and `Error`. We’ll attach it to the `Found` outcome of the previous scripted decision node.

{{<details title="`Get Password Policy From Organization Metadata.js` (Click to View)">}}
```javascript
/*
Utilizing the organization ID provided gather the password policy information.

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
            var fields = 'name,passwordPolicy';
            
            var orgMetadata = openidm.query(`managed/${REALM}_organization`, { 
              "_queryFilter": queryFilter,
              "_fields": fields
            }).result;
  
            if (null != orgMetadata && orgMetadata.length === 1) {
                var orgMetadataJson = JSON.parse(orgMetadata[0]);
                var passwordPolicy = orgMetadataJson.passwordPolicy;
                nodeState.putShared('passwordPolicy', passwordPolicy);
                outcome = NodeOutcome.SUCCESS;
            } else {
                throw('No Organization metadata found.')
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
{{</ details >}}

Connect the `Success` outcome of gathering the password policy to the Page Node and both the `Error` outcomes to the Failure Node. For the sake of testing, move the Test Password Policy Node so that it is connected to the `Missing` outcome of the Get Org ID from Query Param node and its outcome is connected to the Page Node, with the Page Node connected to the Evaluate Password Policy Node. As mentioned before, it’s likely you’d be setting a default or reaching out to a PDP here.

That’s a lot of connections! Let’s see what our Journey should look like now:

![A screenshot of the Registration Journey including the Organization metadata and policy enforcement scripts](../images/enforcing-multiple-password-policies-in-aic/journey-complete.png)

_The Completed Journey_

## Testing Organizational Password Policy Enforcement

Open the preview URL in a Guest, Incognito, or separate browser window. By default, you’ll be able to create a User following the default password policy we created in the last section.

Now let’s enforce a password policy set by an Organization. Head back to the Example Org that you created earlier or create a brand new one. In the “Password Policy” tab, let’s set some values to enforce.

| Key | Value |
| ----- | ----- |
| Minimum Password Length | 5 |
| Forbidden User Attributes | userName |
| Forbidden Organization Attributes | name, description |
| No Repetitive Characters (e.g. aaa) | Checked (true) |

Your Organization password policy should look like this:

![A screenshot of the Admin console where the Password Policy has been set on the organization according to the provided details](../images/enforcing-multiple-password-policies-in-aic/test-org-policy.png)

_The Test Policy_

Select the “Raw JSON” tab. You should see the Password Policy set on your organization.

![A screenshot of the updated organization's raw JSON values, which include the inputted policy](../images/enforcing-multiple-password-policies-in-aic/test-org-json.png)

_Viewing the Org's Test Policy_

With our password policy set, let’s apply it to the Journey.

On the window with your Journey, modify the Journey URL to contain the query parameter `orgId` where the value of that parameter is the GUID for your Organization (you can find this in the URL when you are managing the org in your tenant, i.e. `https://{your-domain}/platform/?realm=alpha#/managed-identities/managed/alpha_organization/{the-org-id}`)

Your full journey URL will look something like this:

```
https://{your-domain}/am/XUI/?realm=/alpha&authIndexType=service&authIndexValue=DynamicPasswordPolicy_Registration&orgId={the-org-id}
```

Just like when we tested before, let’s add in a user that will flag the password policy we created. In this case, we’ll use the username `aaaa` and the password `aaaa`. 

![A screenshot of the end-user UI in which a test user has been provided as part of this organization](../images/enforcing-multiple-password-policies-in-aic/test-org-user.png)

_The Test User_

As expected, our password was flagged that it did not meet most criteria in our policy. Now, let’s change our password to the name of the Organization - in my case it was `Example Org`. 

![A screenshot of the end-user UI in which the password `aaaa` has been flagged and the password `Example Org` is being entered](../images/enforcing-multiple-password-policies-in-aic/test-org-failure.png)

_Evaluation Results for `aaaa`_

We’re now flagged that we can’t match the name of the organization, either. Go ahead and change your password to something that passes the policy - I’m using `b$5j0sW`.

![A screenshot of the end-user UI in which the password `Example Org` has been flagged and the password `b$5j0sW` is being entered](../images/enforcing-multiple-password-policies-in-aic/test-org-name-failure.png)

_Evaluation Results for `Example Org`_

And just like that we’ve created a new user registered under their Organization’s password policy.

# Conclusion

We now have a way to dynamically evaluate a user’s password against more than one Password Policy. This policy could be provided through an external service, an internal policy, or stored as metadata on managed objects like Organizations.

While we walked through a Registration Journey, the same concepts can be applied during authentication. 

The Library Script we used, the Registration journey we built, and an Authentication Journey using the same concepts from Registration, can be found here:

* [library_passwordPolicy.js](https://github.com/gwizdala/lib-ping/blob/main/Library%20Scripts/library_passwordPolicy.js)  
* [DynamicPasswordPolicy_Registration.json](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/enforcing-multiple-password-policies-in-aic/DynamicPasswordPolicy_Registration.json)  
* [DynamicPasswordPolicy_Login.json](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/enforcing-multiple-password-policies-in-aic/DynamicPasswordPolicy_Login.json)

With this How-To you have created an extensible approach to handling the diverse password policies for your unique directories, external systems, business groups, and partnerships.

# Enhancements

The password policy library is built to be extended so that you can fit it to match your specific use cases. Use the examples below as a guide on implementing some common enhancements.

## Localizing the Password Policy Messages

The policy messages returned by the evaluator can be localized into any language needed. To do so, you’ll need to add the localizations into the `policyRequirements` object within the library script. The suggested method is to wrap the policyRequirements messages within an object keyed to the [ISO 639 language code](https://en.wikipedia.org/wiki/List_of_ISO_639_language_codes) to match the approach used across AIC (such as in message nodes). Once added, the localized message will return within the evaluation for use within your scripting.

An example of localizing the policy requirements to have English and Spanish translations is below.

```javascript
this.policyRequirements = {
  minPasswordLength: {
    en: `Must be at least ${this.minPasswordLength} characters long`,
    es: `Debe tener al menos ${this.minPasswordLength} caracteres de longitud`
  },
  maxPasswordLength: {
    en: `Must be at most ${this.maxPasswordLength} characters long`,
    es: `Debe tener como máximo ${this.maxPasswordLength} caracteres de longitud`
  },
  disallowedUserAttributes: {
    en: `Must not contain values that are part of your account (e.g. name, email, username)`,
    es: `No debe contener valores que formen parte de su cuenta (p. ej., nombre, correo electrónico, nombre de usuario)`
  },
  disallowedOrgAttributes: {
    en: `Must not contain easy-to-guess values that relate back to the platform (e.g. the Company Name)`,
    es: `No debe contener valores fáciles de adivinar que se relacionen con la plataforma (p. ej., el nombre de la empresa)`
  },
  requireNoCommonlyUsedPasswords: {
    en: `Must not be a commonly used password (e.g. 'Password123')`,
    es: `No debe ser una contraseña de uso común (p. ej., 'Contraseña123')`
  },
  requireUpperCase: {
    en: `Must contain at least one uppercase character`,
    es: `Debe contener al menos una letra mayúscula`
  },
  requireLowerCase: {
    en: `Must contain at least one lowercase character`,
    es: `Debe contener al menos una letra minúscula`
  },
  requireNumber: {
    en: `Must contain at least one number`,
    es: `Debe contener al menos un número`
  },
  requireSpecialChar: {
    en: `Must contain at least one special character (e.g. !@#$%^&)`,
    es: `Debe contener al menos un carácter especial (p. ej., !@#$%^&)`
  },
  requireNoRepeatChars: {
    en: `Must not contain any repeated characters (e.g. aaa)`,
    es: `No debe contener caracteres repetidos (p. ej., aaa)`
  }
};
```

## Adding More Password Policy Options

To add a new password policy option, you’ll need to:

1. Provide its default value underneath the function declaration  
2. Add the message returned after this policy has been evaluated  
3. Add a conditional statement that checks if this password policy option should be evaluated and then runs the evaluation if so  
4. Provide an evaluation function that returns whether the option passed (`true`) or failed (`false`)

Let’s look at one of the policies, `requireNumber` as an example.

Underneath the function declaration for `evaluatePassword`, we set `requireNumber` to have a default of `false` in case it wasn’t provided in the password policy object:

```javascript
this.requireNumber = !!passwordPolicy.requireNumber || false;
```

Inside of the `policyRequirements` object, we see that there’s a policy message for `requireNumber`:

```javascript
requireNumber: `Must contain at least one number`,
```

We then check if this option is required from this policy, and if so, we evaluate it:

```javascript
  if(this.requireNumber) {
    policyEval = this.evaluateNumber(password);
    
    policiesEvaluated.push({
       policy: this.policyRequirements.requireNumber,
       passed: policyEval
    });

    totalPolicyEval = totalPolicyEval & policyEval;
  }
```

And the evaluation for this option, `evaluateNumber`, is defined as such:

```javascript
/**
     * Determines if the password contains any numbers
     * @param password the password to evaluate
     * @return {boolean} whether or not the password passes evaluation
     */
function evaluateNumber(password) {
  const isContainsNumber = /^(?=.*[0-9]).*$/;
  return isContainsNumber.test(password);
}
```

Using the same paradigm, you can evaluate passwords however your business requires it. And just like before, you can add the new evaluation options to your Organization object to be managed on an Organization-by-Organization basis.