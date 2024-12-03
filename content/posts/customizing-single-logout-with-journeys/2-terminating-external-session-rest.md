---
draft: false
date: '2024-03-15T03:00:00+00:00'
title: 'Customizing Single Logout Using Journeys, Pt. 2: Terminating an External Session via REST'
description: 'Part 2 in 4 of the series Customizing Single Logout Using Journeys'
summary: Learn how to terminate an external user session using REST inside a Journey in PingOne Advanced Identity Cloud
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "JavaScript", "REST"]
types: ["Coding"]
cover:
  image:
  alt:
  caption:
  relative: false
ShowToc: true
author: David Gwizdala
---

# Terminating an External Session via REST

There may be instances where your users maintain sessions with PingOne Advanced Identity Cloud (AIC) and an external system. If the external application/service follows the [SAML 2.0 Spec](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0-cd-02.html#5.3.Single%20Logout%20Profile%7Coutline) for Single Logout, you can configure AIC to [invalidate the external session alongside a Ping Logout](https://docs.pingidentity.com/pingoneaic/latest/am-saml2/saml2-sso-slo.html). Unfortunately, many systems have their own bespoke method to session termination.

In cases where the external application/service does not follow the SAML 2.0 spec for Single Logout, but does have a method of logout available via REST API or URL, you can use Scripted Nodes from within a Journey to log a user out of external applications and services.

## Ensuring the User Has an Active Session

Since we’ll be pulling attribute data from a user, we’ll want to ensure that this user has an active session.

Create a new Journey entitled _Terminate External API Session_. In your Journey, connect your Start Node to an Identify Existing User Node with the Identifier and Identity Attribute both set to “userName”. This will check for an existing userName in the session and will load additional attribute information such as the user’s `_id`, which we’ll use to access their attribute information.

![Identify Existing User Node Details](../images/2-id-existing-user-details.png)

_Identify Existing User Details_

## Sending the Request

In this example, we’ll be mocking out a logout request with https://jsonplaceholder.typicode.com/. The request we’ll need to make includes passing in the user’s external `username` and `sso_token`, which we have stored on the user’s identity in PingOne Advanced Identity Cloud, and will look something like this:

```bash
curl -X POST https://jsonplaceholder.typicode.com/posts \
-H "Content-type: application/json" \
-d '{ "username": "foo", "sso_token": "bar"}' 
```

The response will return a status of `201` if successful.

For the sake of this example, we’ll assume that the `username` in the external system is the same as the Ping `userName`, and that the `sso_token` is stored on `frUnindexedString2`.

In your Journey, drag and drop in a Scripted Decision Node and create a Script named _Send Logout Request_, connected to the `True` output of your Identify Existing User Node. This script will have three outcomes defined: `Success`, `Failure`, and `Error`.

Copy/Paste the following Javascript into your Node:

```javascript
/*
 Call an external API to terminate a session
 
 In a production instance, store credentials and routes in an ESV for security and reuse.
 
 The scripted decision node needs the following outcomes defined:
 - Success
 - Failure
 - Error
 
 Author: @gwizdala
 */

//// CONSTANTS
var BASE_URL = "https://jsonplaceholder.typicode.com";
var USER_SSO_TOKEN = "fr-attr-str2";

var OUTCOMES = {
  SUCCESS: "Success",
  FAILURE: "Failure",
  ERROR: "Error"
}

//// HELPERS
/**
	Calls an external endpoint to terminate a session 
    
    @param {String} username the user's username
    @param {String} ssoToken the user's external ssoToken
    @return {object} the response from the API, or null. Throws an error if not 201
*/
function terminateSession(username, ssoToken) {
  var expectedStatus = 201;
  var options = {
    method: "POST",
    headers: {
      "Content-Type": "application/json"
    },
    // If you need to add a bearer token, use something like this: `token: bearerToken,`
    body: {
      "username": username,
      "sso_token": ssoToken
    }
  };

  var requestURL = `${BASE_URL}/posts`;
  var response = httpClient.send(requestURL, options).get();

  if (response.status === expectedStatus) {
    var payload = response.text();
    var jsonResult = JSON.parse(payload);
    return jsonResult;
  } else {
  	throw(`Response is not ${expectedStatus}. Response: ${response.text}`); 
  }
}

//// MAIN
(function() {
  // We wrap our main in a try/catch for the following reasons:
  // - Since we are hitting exernal functions that could throw errors, like httpClient
  // - So that if a user isn't logged in, the sharedState retrieval fails gracefully.
  try {
    var username = nodeState.get("_id");
    var attribute = USER_SSO_TOKEN;
    
    var identity = idRepository.getIdentity(username);
    // If no attribute by this name is found, the result is an empty array: []
    var ssoToken = identity.getAttributeValues(attribute);
    
    // If we found a username and ssotoken
    if (username && ssoToken != '[]') {
      	var response = terminateSession(username, ssoToken);
      	// Store the response in shared state to review later
        // In this script, you could branch paths or perform actions based on the response data.
      	nodeState.putShared('terminationResponse', JSON.stringify(response));
    	outcome = OUTCOMES.SUCCESS;
    } else {
      	nodeState.putShared('terminationResponse', `username: ${username}, ssotoken: ${ssoToken}`);
    	outcome = OUTCOMES.FAILURE;
    }
  } catch(e) {
    logger.error(e);
    outcome = OUTCOMES.ERROR;
  }
}());
```

### How It Works

Let’s break this script down.

Starting at our Main function (the one wrapped in `(function(){...}());`):

```javascript
var username = nodeState.get("_id");
var attribute = USER_SSO_TOKEN;
    
var identity = idRepository.getIdentity(username);
// If no attribute by this name is found, the result is an empty array: []
var ssoToken = identity.getAttributeValues(attribute);
```

We first gather the `ssoToken` attribute from the user using the [idRepository script binding](https://docs.pingidentity.com/pingoneaic/latest/am-scripting/scripting-api-node.html#scripting-api-node-id-repo) which takes in the `_id` we collected with the Identify Existing User Node.

```javascript
// If we found a username and ssotoken
if (username && ssoToken != '[]') {
var response = terminateSession(username, ssoToken);
    // Store the response in shared state to review later
    // In this script, you could branch paths or perform actions based on the response data.
    nodeState.putShared('terminationResponse', JSON.stringify(response));
    outcome = OUTCOMES.SUCCESS;
} else {
    nodeState.putShared('terminationResponse', `username: ${username}, ssotoken: ${ssoToken}`);
    outcome = OUTCOMES.FAILURE;
}
```

We then check to see if the `username` and the `ssoToken` exist on the user, and if they do, we call the `terminateSession` function which calls the API. Let’s look at that next.

```javascript
/**
	Calls an external endpoint to terminate a session 
    
    @param {String} username the user's username
    @param {String} ssoToken the user's external ssoToken
    @return {object} the response from the API, or null. Throws an error if not 201
*/
function terminateSession(username, ssoToken) {
  var expectedStatus = 201;
  var options = {
    method: "POST",
    headers: {
      "Content-Type": "application/json"
    },
    // If you need to add a bearer token, use something like this: `token: bearerToken,`
    body: {
      "username": username,
      "sso_token": ssoToken
    }
  };

  var requestURL = `${BASE_URL}/posts`;
  var response = httpClient.send(requestURL, options).get();

  if (response.status === expectedStatus) {
    var payload = response.text();
    var jsonResult = JSON.parse(payload);
    return jsonResult;
  } else {
  	throw(`Response is not ${expectedStatus}. Response: ${response.text}`); 
  }
}
```

In `terminateSession`, we construct an `httpClient` request that `POST`s to our API endpoint. This request includes a request body and will return a JSON-formatted response. In this example, we return the response - but in your version you could do things like parse the response, check for a specific code, or branch into different actions (such as calling more requests, changing state, managing user data, etc) based on its outcome. With APIs, the possibilities are endless!

### Testing

To test this script, we’ll add a message on both success and failure that 1) tells us if the API was called successfully, and 2) shows us what data was received.

First, put a Message Node wired to the `False` output of the Identify Existing User Node and the `Failure` output of your Scripted Decision Node. This can be whatever message you’d like - the example uses the message “No User Found”. Wire the output of your Message Node to the Failure Node.

Secondly, put a Message Node on the `Error` output of your Scripted Decision Node that informs you if an error occurred. In a production system, you’ll likely want to handle this error by returning to another Journey, page, or failure outcome but in this case we’ll display the message “An Error Occurred. Please view the logs.” Wire the output of your Message Node to the Failure Node.

Finally, add a Configuration Provider Node to the `Success` output with its `True` output going to the Success Node. It will use the Node Type “Message Node” and the following transformation script:

```javascript
/*
	Displays the response from the external API request from "terminationResponse" stored in state.
*/

var terminationResponse = nodeState.get("terminationResponse");

config = {
  "message": {
    "en": `Termination Response: ${terminationResponse ? terminationResponse : 'No response data found'}`
  },
  "messageYes": {"en": "Continue"},
  "messageNo": {"en": "Cancel"},
}
```

Your Journey should look something like this:

![Screenshot of the Terminate Session Journey](../images/2-terminate-session-journey.png)

_Terminating a Session Journey_

In this example, we’re expecting that the user has some session information stored on their user profile. Create a test user in PingOne Advanced Identity Cloud (we’re using the username `test`) and add a recognizable string to the `frUnindexedString2` attribute (by default, this attribute is labeled as `Generic Unindexed String 2` in the Identity Cloud Console). The user’s profile should look something like this:

![Screenshot of the User Details Page](../images/2-user-details-page.png)
![Screenshot of the phrase "exampleSSOToken" added to the User's Generic Unindexed String 2](../images/2-updating-frUnindexedString.png)

_Test User Data_

Open an incognito (or separate browser) window, log in as your test user, and then paste in the URL of your new Journey. You should reach your Message Node displaying the API response from the request you made.

![Screenshot of the termination response in state](../images/2-termination-response.png)

_The Termination Response in Shared State_

Now, hit “Continue” to reach the user’s dashboard, log out the user, and then go back to your Journey. You should hit the Message Node indicating that no user was found.

![Screenshot of no user found in state](../images/2-no-user-found.png)

_No User Found_

# Summary

With this How-To you have:

1. Called an external API and retrieved its response
2. Stored response data in State
3. Branched your Journey based on an API call

The full Journey, including the testing output, can be downloaded here:

[Terminate Session with API Call Journey](https://gist.github.com/gwizdala/b2ab2b41949933545f6fff7ba97a723e#file-pt-2_terminate-session-with-api-call-json)

This is part of a 4-part series on Creating Custom Single Logout Using Journeys. To continue this series, head to [Part 3: Invalidating the User’s PingOne AIC Session]({{< ref "3-invalidating-users-aic-session" >}}) to learn how to terminate a user’s PingOne AIC session while still inside a Journey.

---

# Further Reading

- [Introduction]({{< ref "introduction" >}})
- [Part 1: Capturing the user’s existing browser session]({{< ref "1-capture-browser-session" >}})
- **Part 2: Sending out an external API request to terminate an external session**
- [Part 3: Invalidating the user’s PingOne AIC session]({{< ref "3-invalidating-users-aic-session" >}})
- [Part 4: Redirecting the user to a custom URL]({{< ref "4-redirecting-custom-url" >}})
- [Conclusion & Recap]({{< ref "conclusion" >}})