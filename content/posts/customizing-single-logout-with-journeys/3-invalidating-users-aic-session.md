---
draft: false
date: '2024-03-15T02:00:00+00:00'
title: 'Customizing Single Logout Using Journeys, Pt. 3: Invalidating the User’s PingOne AIC Session'
description: 'Part 3 in 4 of the series Customizing Single Logout Using Journeys'
summary: Learn how to terminate an internal PingOne AIC user session using REST and inside a Journey in PingOne Advanced Identity Cloud
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "JavaScript", "REST"]
types: ["How-To"]
cover:
  image:
  alt:
  caption:
  relative: false
ShowToc: true
---

# Invalidating the User’s PingOne AIC Session

Once you’ve captured the user’s SSO token, and have performed any necessary logout activities with external services, you can use that token to force session termination for that user. In this example, we’ll be terminating all of a user’s active sessions - but you can always view and terminate specific sessions instead. To accomplish this, we’ll be using the [Session Management API](https://docs.pingidentity.com/pingoneaic/latest/am-sessions/managing-sessions-REST.html) within a Scripted Decision Node.

## Capturing the Cookie and Existing Session Data

Create a new Journey entitled _Terminate Session_. In your Journey, connect your Start Node to an Inner Tree Node and select the name of your Journey you created in [Capturing the User’s Existing Browser Session]({{< ref "1-capture-browser-session" >}}). If you haven’t done this step yet, jump back there first - it’ll show you how to reference the user’s cookie (which you’ll use as an SSO token to log them out).

Now that we’ve captured the session, we can optionally call the logout actions we made in [Terminating an External Session via REST]({{< ref "2-terminating-external-session-rest" >}}) with another Inner Tree Node. Note that if you don’t need to call any external endpoints, you’ll need to add an Identify Existing User Node with the Identifier and Identity Attribute both set to “userName” (as mentioned in [Ensuring the User Has an Active Session]({{< ref "2-terminating-external-session-rest#ensuring-the-user-has-an-active-session" >}})) as we’ll be referencing the user’s `_id` later.

> **Note**: if you’d prefer not use inner trees, you can add the Scripted Decision Nodes you created in previous sections directly into this Journey instead or combine the scripts together (code example [here](https://gist.github.com/gwizdala/b2ab2b41949933545f6fff7ba97a723e#file-getandendusersession-js))

Your journey will either look like this:

![Screenshot of No External Sessions Logout Journey](../images/3-no-external-sessions-journey.png)

_Logging Out the User in PingOne AIC - No External Sessions_

Or if you added the REST call Journey, like this:

![Screenshot of External Sessions Logout Journey](../images/3-external-sessions-journey.png)

_Logging Out the User in PingOne AIC - External Sessions_

## Sending the Request

Next, drag and drop in a Scripted Decision Node and create a Script named `Terminate Sessions`. This script will have four outcomes defined: `Success`, `Failure`, `No Session`, and `Error`.

Copy/Paste the following Javascript into your Node:

```javascript
/*
Checks if the user has an existing session, and if so, logs them out.
 
 This script expects the user's cookie and _id to be stored in shared state.

 This script expects the following ESV to be set:
 - esv.cookie - the cookie of the tenant (found in "Tenant Settings")
 
 The scripted decision node needs the following outcomes defined:
 - Success
 - Failure
 - No Session
 - Error
 
 Author: se@pingidentity.com
 */

// Request Params
var HOST = requestHeaders.get("host").get(0);
var AM_BASE_URL = "https://" + HOST + "/am/";
// Long-Lived API Token
var TENANT_COOKIE = systemEnv.getProperty("esv.cookie");

var OUTCOMES = {
  SUCCESS: "Success",
  FAILURE: "Failure",
  NO_SESSION: "No Session",
  ERROR: "Error"
};

//// HELPERS

/**
 * Kills all of a user's sessions based on their identifier
 * https://backstage.forgerock.com/docs/idcloud/latest/am-sessions/managing-sessions-REST.html#invalidate-sessions-user
 * @param {string} sessionCookie the user's cookie containing their session authorization
 * @param {string} uid the universal identifier of the user
 * @returns {boolean} whether or not the invalidation was successful
 */
function logoutByUser(sessionCookie, uid) {
  var wasLogoutSuccessful = false;
  
  var options = {
    method: 'POST',
    headers: {
      "Content-Type": "application/json; charset=UTF-8",
      "Accept-API-Version": "resource=5.1, protocol=1.0"
    },
    body: {
      username: uid
    }
  };
  options.headers[TENANT_COOKIE] = sessionCookie;
  
  var requestURL = `${AM_BASE_URL}json/realms/root/realms/alpha/sessions/?_action=logoutByUser`;

  var response = httpClient.send(requestURL, options).get();

  if (response.status === 200) {
    var payload = response.text();
    var jsonResult = JSON.parse(payload);
    wasLogoutSuccessful = jsonResult.result;
  }
  
  return wasLogoutSuccessful;
}

//// MAIN
(function () {
  try {
    var sessionCookie = nodeState.get("sessionCookie");
    var uid = nodeState.get("_id");

    if (!sessionCookie || !uid) {
      outcome = OUTCOMES.NO_SESSION; 
    } else if (logoutByUser(sessionCookie, uid)) {
      outcome = OUTCOMES.SUCCESS;
    } else {
      outcome = OUTCOMES.FAILURE;
    }
  } catch(e) {
   logger.error(e);
   outcome = OUTCOMES.ERROR;
  }
}());
```

### How It Works

This script is really doing only one action - it’s calling `logoutByUser` from the [Session Management API](https://docs.pingidentity.com/pingoneaic/latest/am-sessions/managing-sessions-REST.html#rest-api-session-logout) using the user’s personal `sessionCookie` and their `_id` we captured earlier. Once the user’s been logged out, it returns success. That’s it!

### Testing

To test that our Journey is successful, we’ll log the user out and then attempt to enter into their user portal. We’ll also use our admin portal to check for any active sessions.

To set this up, at the end of your Journey put an Inner Tree Node that points to your default Login Journey and connect to the `False` response of your Message Node. The `True` outcome of the Login Journey should point to the Success Node and the `False` outcome should point to the Failure node. That way when the user is invalidated they are placed back on a page to come back in if they need to.

Finally, we’ll want to make sure that any errors from our Session Invalidation Script are caught, so put a Message Node on the `Error` output of your Scripted Decision Node that informs you if an error occurred. In a production system, you’ll likely want to handle this error by returning to another Journey, page, or failure outcome but in this case we’ll display the message “An Error Occurred. Please view the logs.” Wire the output of your Message Node to the Failure Node.

Your completed Journey should look something like this:

![Screenshot of External Sessions Logout Journey with Testing](../images/3-external-sessions-journey-test.png)

_Invalidating PingOne AIC Sessions Journey_

Open an incognito (or separate browser) window and log in as your test user. You should see their active session within your AM Native Console under the “Sessions” tab by searching for their `_id` (this attribute is in your managed users user page under “Raw JSON”).

![Screenshot of the Logged-In User](../images/3-logged-in-user.png)
![Screenshot of the Logged-In User's Active Session](../images/3-logged-in-user-session.png)

_The User and Their Active Session_

Once you’ve validated that the user’s session exists and is active, paste in the URL of your new Journey in the browser where you’re logged in as that user. You should be taken to your default login screen. When you check the AM Native console, all sessions related to that user should be gone!

![Screenshot of the login screen](../images/3-login-screen.png)
![Screenshot of no sessions](../images/3-no-session.png)

_Login Screen for User and No Sessions in the Admin Console_

# Summary

With this How-To you have invalidated a user’s sessions both from external applications and all sessions within PingOne AIC.

The full Journey, including the testing output, can be downloaded here:

[Invalid PingOne AIC Sessions Journey](https://gist.github.com/gwizdala/b2ab2b41949933545f6fff7ba97a723e#file-pt-3_terminate-forgerock-session-json)

This is part of a 4-part series on Creating Custom Single Logout Using Journeys. To continue this series, head to [Part 4: Redirecting the User to a Custom URL]({{< ref "4-redirecting-custom-url" >}}) to learn how to specify where the user goes when they logout.

---

# Further Reading

- [Introduction]({{< ref "introduction" >}})
- [Part 1: Capturing the user’s existing browser session]({{< ref "1-capture-browser-session" >}})
- [Part 2: Sending out an external API request to terminate an external session]({{< ref "2-terminating-external-session-rest" >}})
- **Part 3: Invalidating the user’s PingOne AIC session**
- [Part 4: Redirecting the user to a custom URL]({{< ref "4-redirecting-custom-url" >}})
- [Conclusion & Recap]({{< ref "conclusion" >}})