---
draft: false
date: '2025-01-02'
title: 'Managing MFA in Journeys in PingOne Advanced Identity Cloud or PingAM'
description: 'List, rename (including during registration), and remove MFA devices within a Journey or Tree'
summary: Learn how to extend MFA functionality for Users and Delegated Administrators without requiring REST calls or custom UIs in PingOne AIC and PingAM
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

PingOne Advanced Identity Cloud (AIC) provides a plethora of out-of-the-box authentication methods, including OTP (both TOTP and HOTP), Push Notification, FIDO2/WebAuthN (Biometric/Passkeys), and Email “Magic Link” to name a few.

It’s likely that your users will onboard more than one factor of authentication, and with people replacing electronics every few years (or sometimes multiple times a year!)  it’s even more likely that they’ll need to manage those MFA devices.

Out of the Box, AIC provides device management screens within the Hosted Page UI where you can edit the name of or remove MFA devices associated with your account. However, there are some caveats to what you can do with it:

1. By default, certain types of MFA devices are always stored with a static name (for example, all WebAuthN devices are stored as “New Security Key”) meaning that if a user stores more than one of the same type of MFA it may be difficult to determine which device is which.  
2. A user can only remove an MFA device they used to log in for the current session.  
3. Delegated administrators, such as Managers or those defined in the Organization Model, do not have access to managing the user’s devices.

These features are available [via the API](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-mfa-devices.html), but rather than build an entirely separate UI let’s use the designs we’ve already created within the platform. Since MFA devices are invoked using AIC’s Orchestration layer (Journeys), it makes sense to expand our capabilities there.

## Managing MFA in Journeys

This How-To will teach you how to enable custom self-service of MFA devices from within a Journey, including:

1. [Listing all existing MFA Devices registered to a User](#listing-existing-mfa-devices)  
2. [Informing the User what Device they are Currently Interacting with](#interacting-with-an-existing-mfa-device)  
3. [Naming/Renaming an MFA Device during a Journey](#renaming-an-existing-mfa-device) ([including at Device Registration](#setting-the-name-of-a-new-mfa-device))  
4. [Removing an existing MFA Device](#removing-mfa-devices)

By the end of this How-To you’ll have a Journey that contains each of the above capabilities. The Journey is split into independent inner Journeys and nodes so that you can pick and choose which options apply to you - feel free to use as much or as little as you’d like in Journeys of your own.

This How-To expects a level of familiarity with Journeys and Journey Scripting. You’ll be extensively using the [idRepository Wrapper](https://docs.pingidentity.com/pingoneaic/latest/am-scripting/scripting-api-node.html#scripting-api-node-id-repo) but you don’t need to have much familiarity with them to follow along.

**An important note:** Enabling the management of MFA devices incurs risk as it potentially opens up a path for a threat actor to modify or remove an additional factor they otherwise couldn’t get around. It’s incredibly important to lock down these actions with strong assurance that your User is who they say they are. Some suggested options include:

* Risk/Fraud Detection ([PingOne Protect](https://docs.pingidentity.com/pingoneaic/latest/release-notes/rapid-channel/pingone-protect-nodes.html))  
* Identity Verification ([PingOne Verify](https://docs.pingidentity.com/pingoneaic/latest/pingone/auth-node-ping-verify-service.html))  
* Certified Wallet Credentials ([PingOne Credentials](https://docs.pingidentity.com/pingoneaic/latest/pingone/auth-node-p1-cred-overview.html))  
* OAuth2 Backchannel Request Grant ([CIBA](https://docs.pingidentity.com/pingoneaic/latest/am-oidc1/openid-connect-backchannel-request-flow.html))

# Listing Existing MFA Devices {#listing-existing-mfa-devices}

To start, let’s create a base Journey that will list the user’s devices and then allow the user to select an action on those devices.

Navigate to “Journeys” inside of your tenant and create a new Journey using the following defaults:

| Input | Value |
| ----- | ----- |
| Name | ManageMFADevices |
| Identity Object | Alpha realm - Users `managed/alpha_user` |
| Description (optional) | An example journey showcasing how you can enable a user to List, Add, Rename, and Remove MFA devices. |
| Tags (optional) | MFA |

To make this example simple to test, we aren’t going to ask the user to authenticate. That way we’ll just need to type the username to access the features we’ll be adding. As noted in the Introduction, this is for our example only. **Strongly prove your user’s identity before allowing them to manage their MFA devices.**

Connect your Start node to a Platform Username node and then connect the Platform Username node to an Identify Existing User node, with the `False` outcome headed to the Failure node.

Next, we’ll be rendering a dynamic list of MFA devices that are registered to the user. To do this, connect the `True` outcome of your Identity Existing User node to a Scripted Decision Node with the outcomes `No Devices`, `Selected`, and `Error` with the following Next-Gen code:

{{<details title="`SelectMFADevice.js` (Click to View)">}}
```javascript
/*
Renders a multiselect where the user can select from a list of MFA devices (e.g. WebAuthN, Push, OATH).
Once selected, add the selected MFA into the shared state.

This script does not need to be parametrized. It will work properly as is.
This script expects a user to be loaded in state.
 
 The scripted decision node needs the following outcomes defined:
	- No Devices    // The user doesn't have any stored MFA devices
  - Selected      // The user has seleced a device
  - Error         // An error has occurred. Please consult the logs
 
 Author: @gwizdala
 */

//// CONSTANTS
var MFA_DEVICE_TYPES = ["webauthn", "push", "oath"];
var MFA_DEVICE_PROFILE = 'DeviceProfiles';

var NodeOutcome = {
  NO_DEVICES: "No Devices",
  SELECTED: "Selected",
  ERROR: "Error"
};

//// HELPERS
/**
	Returns a list of MFA metadata, keyed by the username.
    
    @param {string} uid the _id of the user
    @return {object[]} the mfa metadata, keyed to type
*/
function getMFADevices(uid) {
  var out = [];
  var identity = idRepository.getIdentity(uid);

  MFA_DEVICE_TYPES.forEach(function(deviceType) {
    var deviceProfiles = identity.getAttributeValues(`${deviceType}${MFA_DEVICE_PROFILE}`);
    deviceProfiles.forEach(function(deviceProfile) {
      // e.g. { deviceType: webauthn, deviceProfile: {...} }
      out.push({
        deviceType: deviceType,
        deviceProfile: JSON.parse(deviceProfile)
      });
    });
  });

  return out;
}

//// MAIN
(function () {
  try {
    outcome = NodeOutcome.NO_DEVICES; // default
    var uid = nodeState.get("_id");
    var mfaMethods = getMFADevices(uid);

    if (mfaMethods.length > 0) {
      // Construct the Choice options for the dropdown selector
      var choices = [];
      mfaMethods.forEach(function(mfaMethod) {
        // e.g. "push - My Push Authenticator"
        choices.push(mfaMethod.deviceType + " - " + mfaMethod.deviceProfile.deviceName);
      });

      // Render the Callback
      if (callbacks.isEmpty()) {
        // Interactive callbacks: https://backstage.forgerock.com/docs/idcloud/latest/am-authentication/authn-interactive-callbacks.html
        callbacksBuilder.choiceCallback(
          "Select MFA method",
          choices,
          0,
          false
        );
      } else {
        var choiceIndex = callbacks.getChoiceCallbacks().get(0)[0];

        // Device Selected - put the info in state
        var mfaMethod = mfaMethods[choiceIndex];
        nodeState.putShared("mfaDeviceType", mfaMethod.deviceType);
        nodeState.putShared("mfaDeviceName", mfaMethod.deviceProfile.deviceName);
        nodeState.putShared("mfaDeviceProfile", JSON.stringify(mfaMethod.deviceProfile));

        outcome = NodeOutcome.SELECTED;
      }
    }
  } catch(e) {
    logger.error(e);
    outcome = NodeOutcome.ERROR;
  }

  action.goTo(outcome);
}());
```
{{</ details >}}

This code is doing the following:

1. Based on the identity stored in state (the one we retrieved from the Identify Existing User node), pull the device profiles stored on that user’s identity object (see [API Docs](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-mfa-list-devices.html) and `getMFADevices` function in this code).  
2. If the user has devices, render a dropdown (that’s the `callbacksBuilder.choiceCallback`) and store the selected mfa device in shared state under the values `mfaDeviceType` (either `webauthn`, `push`, or `oath`), `mfaDeviceName`, and `mfaDeviceProfile`.  
3. If the user doesn’t have any devices, branch to a separate path.

![A screenshot of the Journey editor highlighting the Select MFA Device Script](/img/managing-mfa-in-journeys/select-mfa-device-script.png)
*The Selecting MFA Devices Script*

## Testing

Let’s test what we have so far.

Drag in a message node connected to the `No Devices` outcome of your Scripted Decision node with the following values:

| Input | Value |
| ----- | ----- |
| Name | MFA Action Selection |
| Message | en: No MFA devices found. Would you like to register a new one? |

![A screenshot of the Journey editor highlighting the Register MFA Prompt](/img/managing-mfa-in-journeys/register-device-prompt.png)
*Register MFA Prompt*

Next, using an example user without any MFA devices registered - my user is named `example`, copy the Preview URL in your Journey editor and open it in an Incognito Window, guest profile, or separate browser and type in the username.

![A screenshot of the rendered Journey in which the username "example" has been inputted](/img/managing-mfa-in-journeys/enter-username.png)
*Entering Your Username*

Upon hitting “Next”, you’ll be prompted to register a new MFA device.

![A screenshot of the rendered Journey where the user inputted has no devices. They are being prompted to register a new one](/img/managing-mfa-in-journeys/no-devices-found.png)
*No MFA Devices Found*

Now, register an MFA device or multiple devices to that user (if you haven’t made a Journey that does this already, check out the [WebAuthN](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-mfa-webauthn.html), [Push](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-mfa-trees-push.html), [OATH](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-mfa-about-oath.html) documentation). I’ll register a WebAuthN, Push, and OATH device.

Re-entering the Journey and typing in the username will reveal a list of MFA options for your user to select from.

![A screenshot of the rendered Journey in which the user inputted has 3 devices to choose from, one of each category](/img/managing-mfa-in-journeys/select-device.png)
*Select an MFA Device* 

Now that we have a way to retrieve and select a user’s MFA devices, let’s interact with those devices in meaningful ways.

# Interacting with an Existing MFA Device {#interacting-with-an-existing-mfa-device}

Since this Journey is all about Managing our MFA devices, let´s give the user some choices as to what actions they can take on their devices.

First thing’s first: let’s put something on our page that helps the user identify what MFA device they’ve selected. That way, when they pick an option they are certain that they are interacting with the correct device.

To do this, connect a Page Node to the `Selected` outcome of your Select MFA Devices node with the following options:

| Input | Value |
| ----- | ----- |
| Name | MFA Actions |
| Page Header | en: MFA Actions |
| Page Description | en: Select what action you’d like to perform on your MFA device. |

Then drag a Scripted Decision node inside your Page Node, entitle it “Display Device” and give it the outcome of `Success` with the following script:

{{<details title="`DisplayMFADeviceName.js` (Click to View)">}}
```javascript
/*
Displays an Information message to the user indicating what MFA device they have selected.
If no data has been stored in shared state, or an error has occurred, the script will not display a message.

This script does not need to be parametrized. It will work properly as is.
This script expects to be placed inside of a Page Node.
This script expects the following to be loaded in shared state:
  - mfaDeviceType
  - mfaDeviceName
 
 The scripted decision node needs the following outcomes defined:
	- Success 
 
 Author: @gwizdala
 */

//// CONSTANTS
var MFA_DEVICE_TYPE = "mfaDeviceType";
var MFA_DEVICE_NAME = "mfaDeviceName";
var MESSAGE_LEVEL = 0; // 0: Info, 1: Warning, 2: Error

var NodeOutcome = {
  SUCCESS: "Success"
};

//// MAIN
(function () {
  outcome = NodeOutcome.SUCCESS;

  try {
    var mfaDeviceType = nodeState.get(MFA_DEVICE_TYPE);
    var mfaDeviceName = nodeState.get(MFA_DEVICE_NAME);

    if (!!mfaDeviceType && !!mfaDeviceName) {
      // Render the Callback
      if (callbacks.isEmpty()) {
        // Read-Only callbacks: https://docs.pingidentity.com/pingoneaic/latest/am-authentication/callbacks-read-only.html#textoutputcallback
        callbacksBuilder.textOutputCallback(
          MESSAGE_LEVEL,
          `MFA Device Selected: ${mfaDeviceType} - ${mfaDeviceName}`
        );
      }
    }
  } catch(e) {
    logger.error(e);
  }

  action.goTo(outcome);
}());

```
{{</ details >}}

This script is pushing an Info message onto the page that informs the user what device they’ve selected. If the values of the selected device aren't in shared state, no message will be rendered.

## Testing

Save the Journey and select an MFA device for your example user. You’ll see the page node along with the info message.

![A screenshot of the rendered Journey where the device named "webauthn - New Security Key" has appeared](/img/managing-mfa-in-journeys/info-message.png)
*The Page with the Info Message*

Now let’s add some actions for the user to pick from. Drag a Choice Collector Node into the Page node and give it the following options:

| Input | Value |
| ----- | ----- |
| Name | Select MFA Action |
| Choices* | Rename, Remove, Select Another Device |
| Default Choice | Rename |
| Prompt | Select MFA Action |

*Option List. Input each item one at a time, without commas.

Your Journey should look something like this:

![A screenshot of the Journey editor in which the display device and select mfa action have been placed inside of a page node](/img/managing-mfa-in-journeys/select-mfa-action.png)
*Selecting MFA Action*

Reloading the Journey, you’ll see that you now have some choices alongside your message.

![A screenshot of the rendered Journey in which the different actions from the choice node have appeared alongside the info message](/img/managing-mfa-in-journeys/select-mfa-action-display.png)
*MFA Actions*

You now have a way to quickly inform the user of what MFA device they have in the current context alongside any of the out of the box nodes (like the Choice Collector). Since we’ve made some choices, let’s put them to use.

# Renaming an Existing MFA Device {#renaming-an-existing-mfa-device}

First up: let’s rename our MFA devices to something more memorable than “New Security Key”.

Connect a Scripted Decision Node to the `Rename` outcome of your Choice Collector/Page Node with the outcomes of `Success` and `Error` and with the following code:

{{<details title="`UpdateMFADeviceName.js` (Click to View)">}}
```javascript
/*
Based on the device type and name stored in state, prompts the user for a user-friendly name of the device,
and then saves that device using that name.

This script does not need to be parametrized. It will work properly as is.
This script expects a user to be loaded in state.
This script expects the following to be stored in shared state:
    - mfaDeviceType  // The type of device, e.g. webauth, push, oath
    - mfaDeviceName  // The current name of the device, e.g. "New Security Key"
    - [optional] mfaDeviceProfile // The full profile of the device.

If the mfaDeviceProfile is stored in state, the uuid will be used to match the device
If the mfaDeviceProfile is not stored in state, the mfaDeviceName will be used to match the device.
This means that if the mfaDeviceName is used, the FIRST instance (newest) of a device with a name found is updated
    (Consider using the mfaDeviceName ONLY during registration to ensure a new name each time)
 
 The scripted decision node needs the following outcomes defined:
    - Success      // An input has been provided and stored on the deviceKey object
    - Error        // An error has occured. Please consult the logs.
 
 Author: @gwizdala
 */

//// CONSTANTS
var MFA_DEVICE_TYPE = 'mfaDeviceType';
var MFA_DEVICE_NAME = 'mfaDeviceName';
var MFA_DEVICE_PROFILE = 'mfaDeviceProfile';
var DEVICE_KEY = 'DeviceProfiles';
var NodeOutcome = {
    SUCCESS: "Success",
    ERROR: "Error"
};

var config = {
    INPUTS: [
        {
            name: 'Device Name',
            id: 'deviceName',
            type: 'text',
            required: true,
            deviceKey: DEVICE_KEY
        }
    ],
    BUTTONS: ["Continue"],
    CONTINUE_ACTION_PRESSED: 0
};

//// HELPERS
/**
 * Formats the provided input type given the values provided
 * @param name The name of the NameCallback, used to target the element
 * @param id The ID to assign to the input
 * @param type The HTML input type (e.g. text, tel, email, number)
 * @param required The HTML tag indicating the input is required
 * @returns A formatted JS string to be used in a ScriptTextOutputCallback
 */
function formatInput(name, id, type, required) {
    return `\
      var input = document.querySelector('*[data-vv-as="${name}"]');\
        input.id = "${id}";\
        input.type = "${type}";\
        input.required = ${!!required};\
    `;
  }

//// MAIN
(function() {
    try {
        var uid = nodeState.get('_id');
        var mfaDeviceType = nodeState.get(MFA_DEVICE_TYPE);
        var mfaDeviceName = nodeState.get(MFA_DEVICE_NAME);
        var mfaDeviceProfile = JSON.parse(nodeState.get(MFA_DEVICE_PROFILE));
        outcome = NodeOutcome.SUCCESS;

        if (!uid) {
            throw('Missing User context in shared state');
        }
        
        if (!mfaDeviceType) {
            throw('Missing mfaDeviceType in Shared State');
        }

        if (!mfaDeviceName && !mfaDeviceProfile) {
            throw('Missing mfaDeviceName AND mfaDeviceProfile in Shared State - you need one to successfully update the MFA device name.');
        }

        if (callbacks.isEmpty()) {
            // Interactive callbacks: https://backstage.forgerock.com/docs/idcloud/latest/am-authentication/authn-interactive-callbacks.html
            var inputScript = '';
            config.INPUTS.forEach(function(input) {
                callbacksBuilder.nameCallback(input.name);
                inputScript += formatInput(input.name, input.id, input.type, input.required); // Create Input(s)
            });
            callbacksBuilder.scriptTextOutputCallback(String(inputScript)); // Invoke JavaScript
            callbacksBuilder.confirmationCallback(0, config.BUTTONS, 0); // Create Confirmation Button(s)
        } else {
            var userSelection = callbacks.getConfirmationCallbacks().get(0);
            if (userSelection == config.CONTINUE_ACTION_PRESSED) {
                // Gather input(s)
                var nameCallbacks = callbacks.getNameCallbacks();
                for (var i = 0; i < nameCallbacks.length; i++) {
                    if (config.INPUTS[i].deviceKey) {
                        // Collect the Input
                        var newDeviceName = nameCallbacks.get(i) ? nameCallbacks.get(i) : `My ${mfaDeviceType.toUpperCase()} Device`;
                        var deviceKey = `${mfaDeviceType.toLowerCase()}${config.INPUTS[i].deviceKey}`;

                        // check if this device is already set in this profile type.
                        var identity = idRepository.getIdentity(uid);
                        var deviceProfiles = identity.getAttributeValues(deviceKey);
                        var updatedDeviceProfile = {};
                        var foundProfile = false;
                        var profileIndex = 0;

                        var comparator = { 
                            key: mfaDeviceProfile ? 'uuid' : 'deviceName', 
                            value: mfaDeviceProfile ? mfaDeviceProfile.uuid : mfaDeviceName 
                        };

                        while (!foundProfile && profileIndex < deviceProfiles.length) {
                            var deviceProfile = JSON.parse(deviceProfiles[profileIndex]);
                            
                            if (deviceProfile[comparator.key] == comparator.value) {
                                // Index found. Update existing device
                                updatedDeviceProfile = deviceProfile;
                                updatedDeviceProfile.deviceName = newDeviceName;
                                deviceProfiles[profileIndex] = JSON.stringify(updatedDeviceProfile);
                                foundProfile = true;
                            }

                            profileIndex += 1;
                        }

                        if (!foundProfile) {
                            // Index not found. Throw error
                            throw(`Device not found.`);
                        } else {
                            // Save the changes on the Identity
                            identity.setAttribute(deviceKey, deviceProfiles);
                            identity.store();

                            // Update shared state to reflect the new name
                            nodeState.putShared(MFA_DEVICE_NAME, newDeviceName);
                            nodeState.putShared(MFA_DEVICE_PROFILE, updatedDeviceProfile);
                        }
                    }
                    // If you have extra inputs, process them here.
                }
            }
        }
    } catch(e) {
        logger.error(e);
        outcome = NodeOutcome.ERROR;
    }

    action.goTo(outcome);
})();
```
{{</ details >}}

Let’s break this script down:

1. We pull the information about the MFA device and User that we stored in Shared State.  
2. If the information is there, we render and format an input for the User to enter in their new MFA device name.  
3. Once the user has inputted the name, we update the MFA device by searching for its `uuid` (if we provided a device profile) or `deviceName` (in cases when we don’t have the profile saved).

Note that this script is flexible in that you can add more inputs in the `config` inside the `CONSTANTS` section and it’ll render them as you need them - just process the inputs in the section labeled `// If you have extra inputs, process them here`.

## Testing

To test this, let’s wire up some actions to our Success and Error outcomes.

First, connect a Message node to the `Success` outcome of your new Scripted Decision node with the following values:

| Input | Value |
| ----- | ----- |
| Name | MFA Action Successful |
| Message | en: MFA Action Successful |
| Positive answer | en: Perform Another Action |
| Negative answer | en: Select Another Device |

Next, connect a Message node to the `Error` outcome of your new Scripted Decision node with the following values:

| Input | Value |
| ----- | ----- |
| Name | MFA Action Unsuccessful |
| Message | en: MFA Action Unsuccessful |
| Positive answer | en: Perform Another Action |
| Negative answer | en: Select Another Device |

Our connecting lines are about to get a little squiggly. Connect the `True` outcomes of both Message nodes to the “MFA Actions” Page node and the `False` outcomes to the “Select MFA Device” node. Your Journey will look something like this:

![A screenshot of the Journey editor where the success/failure messages are connected back to the described nodes](/img/managing-mfa-in-journeys/connecting-lines.png)
*It’s Getting Squiggly*

> A quick callout here: In the real world, you’d probably take the user through a relatively linear path instead of hopping back and forth between these dropdowns. That being said, this path is much easier for us as admins to learn and test a bunch of devices rapidly. In summary, these are _self-inflicted learning squiggles_ that you may not see much in the wild.

With everything hooked up, reload your Journey, enter in your user, and select an MFA device to be renamed.

![A screenshot of the end Journey in which the Rename action has been selected for the webauthn device](/img/managing-mfa-in-journeys/renaming-action.png)  
*Selecting an MFA Device to be Renamed*

You’ll next be prompted with an input where you can put in a new name.

![A screenshot of the end Journey where the user has entered "Desktop Browser" for the new name of their WebAuthN device](/img/managing-mfa-in-journeys/entering-name.png)   
*Renaming the Device*

After hitting “Continue”, you’ll be presented with the Success screen.

![A screenshot of the end Journey in which renaming the device has been successful and a message has been presented to the user](/img/managing-mfa-in-journeys/renaming-action-successful.png)    
*MFA Action Successful*

And then, if you choose “Select Another Device”, you’ll see that your device name has changed and is updated in your MFA device list.

![A screenshot of the end Journey where in the list view the user now sees the updated name - Desktop Browser - in their list of devices](/img/managing-mfa-in-journeys/updated-name-list.png)  
*The Updated Name, Shown in the Device List*

Another neat part of this approach to renaming is that we can use the same script for setting the name of a brand-new device, no changes needed.

# Setting the Name of a New MFA Device {#setting-the-name-of-a-new-mfa-device}

Let’s create an inner Journey that enables a user to register and name a new MFA. This Journey will show you how to use the script defined in the previous section alongside an abridged MFA registration flow.

Inside the base ManageMFADevices Journey, attach the `True` outcome of your “Register MFA” Message Node to an Inner Tree Evaluator node. Inside the node configuration, click the “+” button in the “Tree Name” dropdown to create and enter a new Journey with the following details:

| Input | Value |
| ----- | ----- |
| Name | RegisterMFADevices |
| Identity Object | Alpha realm - Users `managed/alpha_user` |
| Description (optional) | An example journey showcasing how you can create an MFA device with an assigned name. |
| Tags (optional) | MFA |
| Inner Journey | `true` (Checked) |

To start, we’ll let the user pick what MFA device they want to register. In the real world you’ll likely want to dynamically build this list by MFA policy, but here we’ll use a Choice Collector node to do so.

Connect your Start node to a Choice Collector node with the following details:

| Input | Value |
| ----- | ----- |
| Name | MFA Registration Selection |
| Choices* | WebAuthN, Push, OATH |
| Default Choice | WebAuthN |
| Prompt | Select MFA Type |

*Option List. Input each item one at a time, without commas.

In some cases, you’ll find that a User might cancel, use an unsupported device, or timeout in the middle of an MFA registration. To account for this, add a Message Node with the following details:

| Input | Value |
| ----- | ----- |
| Name | Reg Unsuccessful |
| Message | en: MFA Registration Failed. |
| Positive Answer | en: Select Another Device Type |
| Negative Answer | en: Select Another Action |

Wire the `True` outcome to the MFA Registration Selection node and the `False` outcome to the Success node.

![A screenshot of the Journey editor highlighting the reg unsuccessful node](/img/managing-mfa-in-journeys/reg-unsuccessful.png)
*The Reg Unsuccessful Node*

Conversely, we’ll want a way to indicate to the user that their registration has succeeded. In the real world, you’ll likely continue them into their account or login but in this example we’ll provide them the option to register another device before returning to the action list.

Add another Message node with the following details:

| Input | Value |
| ----- | ----- |
| Name | Reg Successful |
| Message | en: MFA Registration Succeeded. |
| Positive Answer | en: Register Another Device Type |
| Negative Answer | en: Select Another Action |

Wire the `True` outcome to the MFA Registration Selection node and the `False` outcome to the Success node.

![A screenshot of the Journey editor highlighting the reg successful node](/img/managing-mfa-in-journeys/reg-successful.png)
*The Reg Successful Node*

From here, we are going to use a series of nodes that are outlined in the Multi-Factor Authentication section of the documentation ([WebAuthN](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-mfa-webauthn.html), [Push](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-mfa-trees-push.html), [OATH](https://docs.pingidentity.com/pingoneaic/latest/am-authentication/authn-mfa-about-oath.html)). Since these Nodes and Journeys are well-documented we won’t be going into how they function in detail - just note that normally you should create a separate Inner Journey for each registration type not only for reuse but to test for existing MFA devices and validate proper registration.

Drag in a WebAuthN Registration node, a Push Registration node, and a OATH Registration node, each connected to their respective Choice Collector outcomes. Leave these nodes as default for now - if you do decide to edit, just make sure to keep recovery codes enabled (it’s not only good for your users, it’s the way we are going to retrieve and update the name of the device later).

![A screenshot of the Journey editor in which the different registration nodes have been connected](/img/managing-mfa-in-journeys/mfa-reg-nodes.png)
*The MFA Registration Nodes*

Next, we’ll need a way to inform our script what option the user has selected to register. To do this, drag in and connect a Set State node to each of the `Success` outcomes of your registration nodes. Each of these Set State nodes will have the same attribute, `mfaDeviceType`, with the following attribute mapping:

| Connection | Attribute Value for Key `mfaDeviceType` |
| ----- | ----- |
| WebAuthN Registration node | `webauthn` |
| Push Registration node | `push` |
| OATH Registration node | `oath` |

![A screenshot of the Journey editor in which the set state nodes have been added](/img/managing-mfa-in-journeys/set-state-nodes.png)
*Setting the State*

There’s one more thing we need to get from shared state: the current name that has been set on the MFA device. Fortunately, that name is mapped in Transient State to the Recovery Code Display Name - let’s store it in a place that we can use later.

Drag in a Scripted Decision node and select the “Legacy” scripting engine option to create the script entitled “Get Current Device Name”. It’ll have the outcomes of `Success` and `Error` with the following code:

{{<details title="`GetNewMFADeviceName.js` (Click to View)">}}
```javascript
/*
Retrieve the Current Device Name from Transient State and populate it in Shared State for use in a Next-Gen script.

This script does not need to be parametrized. It will work properly as is.
This script expects a recoveryCodeDeviceName stored in transient state.
 
 The scripted decision node needs the following outcomes defined:
    - Success      // name found and stored in state
    - Error        // An error has occured. Please consult the logs.
 
 Author: @gwizdala
 */

//// IMPORTS
var fr = JavaImporter(org.forgerock.openam.auth.node.api.Action);

//// CONSTANTS
var SHARED_STATE_KEY = 'mfaDeviceName';
var NodeOutcome = {
    SUCCESS: "Success",
    ERROR: "Error"
};

//// MAIN
(function() {
    try {
        var currentDeviceName = nodeState.get('recoveryCodeDeviceName');

        if (!currentDeviceName) {
            throw('No recovery device name found');
        } else {
            sharedState.put(SHARED_STATE_KEY, currentDeviceName);
        }
        
        outcome = NodeOutcome.SUCCESS;
    } catch(e) {
        logger.error(e);
        outcome = NodeOutcome.ERROR;
    }

    action = fr.Action.goTo(outcome).build();
})();
```
{{</ details >}}

This script is rather simple - it’s taking the recovery code device name, stored in the `recoveryCodeDeviceName` Transient State value, and storing it in a standard Shared State key entitled `mfaDeviceName`. There’s a couple reasons why this script exists:

1. We are standardizing our state value to `mfaDeviceName` so that we can stick with a common shared state value for our Update Name script without worry of conflicting or overriding system values.  
2. We are using the “Legacy” scripting engine because of its capability to access and interact with Transient State directly. As of the writing of this How-To, the “Next-Gen” scripting language does not have access to the `recoveryCodeDeviceName` Transient State value.

Wire up all of the Set State nodes to the Scripted Decision node you just created. Your Journey should look something like this:

![A screenshot of the Journey editor in which the get current device name node has been connected to the outputs of every set state node](/img/managing-mfa-in-journeys/get-device-name.png)  
*Getting the Current Device Name*

Now let’s finish this Journey up. Drag in a Scripted Decision Node and select the Update MFA Device Name script we created in the last section, connected to the `Success` outcome of “Get Current Device Name” node.

![A screenshot of the Journey editor in which the update device name node has been connected to the get current device name node](/img/managing-mfa-in-journeys/update-device-name.png)
*Updating the New Device’s Name*

Almost done - now to just connect all of the Success and Error outcomes together.

Wire the `Success` outcome of your “Update Device Name” node to the “Reg Successful” node and all other open outcomes (they should all be errors, failures, timeouts, or unsupported outcomes) to the “Reg Unsuccessful” node. Your completed Inner Journey should look something like this:

![A screenshot of the Journey editor of the entire device registration renaming journey](/img/managing-mfa-in-journeys/device-reg-rename.png)
*The Complete Device Registration Renaming Journey*

## Testing

Jump back to your parent Journey (ManageMFADevices). Since our only outcome from our Inner Tree is `True`, and currently you can only register a new device if you don’t have any to start with, wire the `True` outcome to the “MFA Actions” Page node.

![A screenshot of the Journey editor in the base level Journey in which mfa registration has been connected to the mfa actions node](/img/managing-mfa-in-journeys/mfa-reg-wired.png)
*Wiring MFA Registration*

Now, go to the Preview URL in an Incognito Window, guest account, or separate browser and enter in the username of a user that doesn’t have any MFA devices registered for their account - I’m using `example2` here.

![A screenshot of the end Journey in which the user "example2" has been inputted](/img/managing-mfa-in-journeys/input-empty-user.png)
*The `example2` User*

On the next screen, when you are asked if you want to register a new MFA device, click “Yes”.

![A screenshot of the rendered Journey where the user inputted has no devices. They are being prompted to register a new one](/img/managing-mfa-in-journeys/no-devices-found.png)
*Registration Prompt*

Next, select the MFA device you want to register. You’ll then be guided through registering either a WebAuthN/Passkey/Biometric, Push, or OATH device.

![A screenshot of the rendered Journey where the user selects an MFA device to register](/img/managing-mfa-in-journeys/select-mfa-reg.png)
*Registering the Device*

After successfully registering your device, you’ll be prompted with the same Device naming screen you saw when renaming a device. Enter in your name here.

![A screenshot of the rendered Journey where the user sets the name of their new device](/img/managing-mfa-in-journeys/name-new-device.png)
*Renaming the MFA Device*

After renaming and hitting “Continue”, you’ll be sent to the Success screen.

![A screenshot of the rendered Journey where the user is taken to a successful mfa registration screen](/img/managing-mfa-in-journeys/mfa-reg-success.png) 
*MFA Registration Succeeded*

If you click “Select Another Action”, you’ll be taken to the device management screen for the device you just created and named.

![A screenshot of the rendered Journey where the user can see their newly inputted device alongside the actions they can take](/img/managing-mfa-in-journeys/new-mfa-device-actions.png)
*MFA Actions on New Device*

# Removing MFA Devices {#removing-mfa-devices}

So now we can **create**, **list**, and **rename** the devices we have - but what if we need to remove them?

Back inside your ManageMFADevices Journey, connect a new Scripted Decision node to the `Remove` outcome of your MFA Actions node with the outcomes `Success` and `Error` and the following code:

{{<details title="`RemoveMFADevice.js` (Click to View)">}}
```javascript
/*
Given the Selected MFA Device, remove that device from the user's profile.

This script does not need to be parametrized. It will work properly as is.
This script expects a user to be loaded in state.
This script expects the following to be stored in shared state:
    - mfaDeviceType  // The type of device, e.g. webauth, push, oath
    - mfaDeviceProfile // The full profile of the device.
 
 The scripted decision node needs the following outcomes defined:
	- Success
  - Error
 
 Author: @gwizdala
 */
//// CONSTANTS
var MFA_DEVICE_TYPE = 'mfaDeviceType';
var MFA_DEVICE_PROFILE = 'mfaDeviceProfile';
var DEVICE_KEY = 'DeviceProfiles';
var NodeOutcome = {
    SUCCESS: "Success",
    ERROR: "Error"
};

//// MAIN
(function () {
  try {
    var uid = nodeState.get('_id');
    var mfaDeviceType = nodeState.get(MFA_DEVICE_TYPE);
    var mfaDeviceProfile = JSON.parse(nodeState.get(MFA_DEVICE_PROFILE)); 
    outcome = NodeOutcome.SUCCESS;

    if (!uid) {
      throw('Missing User context in shared state');
    }
    
    if (!mfaDeviceType) {
        throw('Missing mfaDeviceType in Shared State');
    }

    if (!mfaDeviceProfile) {
        throw('Missing mfaDeviceProfile in Shared State');
    }

    var deviceKey = `${mfaDeviceType.toLowerCase()}${DEVICE_KEY}`;

    var identity = idRepository.getIdentity(uid);
    var deviceProfiles = identity.getAttributeValues(deviceKey);
    var newDeviceProfiles = [];
    var foundProfile = false;

    deviceProfiles.forEach(function(deviceProfileString) {
      var deviceProfile = JSON.parse(deviceProfileString);

      if (deviceProfile['uuid'] == mfaDeviceProfile.uuid) {
        // Index found. Don't push this value
        foundProfile = true;
      } else {
        newDeviceProfiles.push(deviceProfileString);
      }
    });

    if (!foundProfile) {
        // Index not found. Throw error
        throw(`Device not found.`);
    } else {
        // Save the changes on the Identity
        identity.setAttribute(deviceKey, newDeviceProfiles);
        identity.store();

        // Wipe shared state - this device doesn't exist anymore
        nodeState.putShared(MFA_DEVICE_TYPE, null);
        nodeState.putShared(MFA_DEVICE_PROFILE, null);
    }


  } catch(e) {
    logger.error(e);
    outcome = NodeOutcome.ERROR;
  }  

  action.goTo(outcome);
}());
```
{{</ details >}}

This script works almost identically to the one we used to update the name. Rather than changing a value in the list of devices, however, we **remove** one before updating our identity. Short and sweet!

## Testing

To test this, let’s wire the `Error` outcome of our Remove script to the “MFA Action Unsuccessful” node and the `True` outcome to a new Message node with the following values:

| Input | Value |
| ----- | ----- |
| Name | Removal Successful |
| Message | en: MFA Device Removed |
| Positive answer | en: Register A New Device |
| Negative answer | en: Select Another Device |

We aren’t going to the MFA Action Successful node because one of the options there is to perform actions on the selected device - which would be the one we removed! This message lets us either select another device or register a brand new one (since in many cases after removing a device, a user may need to re-add). Connect the `True` outcome to the MFA Registration Inner Tree node and the `False` outcome to the Select MFA Device node.

Your Journey should look something like this:  

![A screenshot of the Journey editor that contains the list, registration, renaming, and removal nodes and messages](/img/managing-mfa-in-journeys/removal-message-node.png)
*Connecting it All Together*

Head back to the Preview URL and select the User and MFA device you created in the previous section. This time, though, select the “Remove” action.

![A screenshot of the rendered Journey where the user selects the "Remove" action on their new device](/img/managing-mfa-in-journeys/action-remove.png)
*Removing the Device*

After hitting “Next”, you should be taken to the Device Removed Screen.

![A screenshot of the rendered Journey where the user has successfully removed their MFA device and sees the resulting success screen](/img/managing-mfa-in-journeys/remove-successful.png)
*Device Successfully Removed*

If you go back to the “Select Another Device” screen, you’ll see that your device is gone!

![A screenshot of the rendered Journey where the user inputted has no devices. They are being prompted to register a new one](/img/managing-mfa-in-journeys/no-devices-found.png)
*No Devices Once More*

# Conclusion

Using Journeys, we were able to extend self-service management of MFA devices without having to make a single REST call or build any custom UI.

By interacting with the Identity of the User in a Journey, we were able to:

1. List all existing MFA Devices registered to a User  
2. Inform the User what Device they are Currently Interacting with  
3. Name/Rename an MFA Device during a Journey (including at Device Registration)  
4. Remove an existing MFA Device

The Combined Journey, using the Journeys developed in the previous parts, can be found here:

[ManageMFADevices.json](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/managing-mfa-in-journeys/ManageMFADevices.json)

Each action to **list**, **rename**, and **reset** MFA devices for a User is usable as a single node that can be dropped into any Journey that needs it - just make sure to strongly prove your user (be it delegated administrator, device owner, or otherwise) before making any changes.