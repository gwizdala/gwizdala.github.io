---
draft: false
date: '2024-03-15T01:00:00+00:00'
title: 'Customizing Single Logout Using Journeys, Pt. 4: Redirecting the User to a Custom URL'
description: 'Part 4 in 4 of the series Customizing Single Logout Using Journeys'
summary: Learn how to redirect the user to a custom URL both statically and dynamically using a Journey in PingOne Advanced Identity Cloud
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "JavaScript"]
types: ["How-To"]
cover:
  image:
  alt:
  caption:
  relative: false
ShowToc: true
author: David Gwizdala
---

# Redirecting the User to a Custom URL

Instead of taking the user back to a Login screen, you may want to redirect them to a specific location (such as a company portal or ecommerce page). If every user should redirect to the same page you can utilize a [Failure URL Node](https://docs.pingidentity.com/auth-node-ref/latest/auth-node-failure-url.html) wired up to the Failure node, looking something like this:

![Screenshot of the Failure URL Node](../images/4-failure-url-node.png)

_Failure URL Node_

However, there may be cases where the route that each user takes will be unique to that user - for example, if you are redirecting to the user’s shopping cart or personalized logout screen, or triggering an additional external logout by redirect URI. In these situations you can use a Scripted Decision Node to pull information like the Organization they’re a part of, the Role or Attribute they have, or the initial path that they came from to initiate logout, and then pass them into a custom-built URL to redirect the user.

## Constructing the Redirect

Create a new Journey entitled _Failure Redirect By Attribute_. In your Journey, connect your Start Node to an Identify Existing User Node with the Identifier and Identity Attribute both set to “userName”.

Next, drag and drop in a Scripted Decision Node and create a Script named `Get Metadata`. This script will have two outcomes defined: `Found` and `Not Found`.

Copy/Paste the following Javascript into your Node:

```javascript
/*
Pulls metadata from a user identity and sets in state.
This example uses constants to determine what metadata is pulled and how it is set,
but these type of actions lend well to Library Scripts where said values can be passed as function params.
In a production instance, consider Library Scripts for greatest reusability.
Doc: https://docs.pingidentity.com/pingoneaic/latest/am-scripting/library-scripts.html

This script expects the following ESV to be set:
- esv.admin.token - a long-lived access token.  You can use frodo to generate an OAuth2 client/secret and
  set the associated esv-admin-token, via the following command:
    frodo admin create-oauth2-client-with-admin-privileges --llt <your-tenant-name>
    
The scripted decision node needs the following outcomes defined:
 - Found
 - Not Found
 
 Author: @gwizdala
 */

//// CONSTANTS
// Request Params
var HOST = requestHeaders.get("host").get(0);
var IDM_BASE_URL = "https://" + HOST + "/openidm/";
// Long-Lived API Token
var IDM_API_TOKEN = systemEnv.getProperty("esv.admin.token");

var OUTCOMES = {
  FOUND: "Found",
  NOT_FOUND: "Not Found"
};

var IDENTITY_OPTIONS = {
  USER: "user",
  ORGANIZATION: "organization",
  ROLE: "role",
  GROUP: "group"
}

/**
CONSTANT PARAMS - consider using these as parameters if converting into a Library Script
*/
// Where the value is stored
var REALM = "alpha";
var IDENTITY = "user"; // could be "user", "organization", "role", "group", etc.
var ATTRIBUTE = "fr-attr-str3"; // could be an attribute on the user, an object reference in an org, etc.
// How to set the value in state
var STATE_KEY = "failureUrl";

//// HELPERS
/**
	Returns metadata from an organization, searching off of the user's uid
    This example will return the first instance of that attribute found.
    
    @param {string} uid the user id in which to find the organization
    @param {string} attribute the attribute to retrieve
    @return {string} the organization metadata, stringified
*/
function getOrgMetadataByUID(uid, attribute) {
  var orgsMetadata = [{}];
   
  var userOrgMembership = openidm.query(`managed/${REALM}_user/${uid}/memberOfOrg`, { "_queryFilter": "true" });
  
  if (userOrgMembership && userOrgMembership.result && userOrgMembership.result.length > 0) {
    // Build the query filter to pull every organization result
    var queryFilter = "";
    var userOrgMembershipResult = userOrgMembership.result;
    for (var i = 0; i < userOrgMembershipResult.length; i++) {
      var org = userOrgMembershipResult[i];
      if (i > 0) {
       queryFilter += " or "; 
      }
      queryFilter += `(_id eq "${org._refResourceId}")`;
    }
    
    orgsMetadata = openidm.query(`managed/${REALM}_organization`, { 
      "_queryFilter": queryFilter,
      "_fields": attribute
    }).result;
  }
  
  for (var j = 0; j < orgsMetadata.length; j++) {
    var orgMetadata = orgsMetadata[j];
    // Check to see if there's an attribute value. If so, return that value.
    if (null != orgMetadata[attribute]) {
     return orgMetadata[attribute];
    }
  }
}


//// MAIN
(function () {
  var uid = nodeState.get("_id");
  var stateValue = "";
  
  // Determine how to pull the defining attribute based on the passed identity
  switch(IDENTITY.toLowerCase()) {
   case(IDENTITY_OPTIONS.USER):
      var identity = idRepository.getIdentity(uid);
      var attribute = identity.getAttributeValues(ATTRIBUTE);
  	  if (attribute != '[]') {
      	stateValue = attribute[0]; // pulling the first value in the array
      }
      break;
   case(IDENTITY_OPTIONS.ORGANIZATION):
      stateValue = getOrgMetadataByUID(uid, ATTRIBUTE);
      break;
    default:
      stateValue = "";
  }
  
  if (stateValue) {
    nodeState.putShared(STATE_KEY, stateValue);
    outcome = OUTCOMES.FOUND;
  } else {
    outcome = OUTCOMES.NOT_FOUND;
  }

  action.goTo(outcome);
}());
```

Once you’ve pasted this in, wire `Found` to the Failure Node and `Not Found` to a Failure URL node redirecting to a location of your choosing (this will work as your fallback).

Your completed Journey should look like this:

![Screenshot of Dynamic Redirect Journey](../images/4-dynamic-redirect-journey.png)

_Dynamic Redirect Journey_

### How It Works

This script provides a generic way to retrieve metadata from different identity types, and has examples for both Organization and User Identity Types. In a production environment, it is suggested to make this functionality a [Library Script](https://docs.pingidentity.com/pingoneaic/latest/am-scripting/library-scripts.html) to be more widely usable across your Tenant. When making the Library Script, consider moving constants such as `REALM`, `IDENTITY`, `ATTRIBUTE`, and `STATE_KEY` into function parameters for even greater reusability. In this example, we’re using a Scripted Decision Node to make the demonstration easy to follow and understand.

Let’s break down the two different types of metadata retrieval:

#### User Attribute Retrieval

```javascript
case(IDENTITY_OPTIONS.USER):
    var identity = idRepository.getIdentity(uid);
    var attribute = identity.getAttributeValues(ATTRIBUTE);
    if (attribute != '[]') {
        stateValue = attribute[0]; // pulling the first value in the array
    }
    break;
```

You’ve seen this strategy before. Just like in [Terminating an External Session](), We gather the desired attribute from the user using the [idRepository script binding](https://docs.pingidentity.com/pingoneaic/latest/am-scripting/scripting-api-node.html#scripting-api-node-id-repo) which takes in the `_id` we collected with the Identify Existing User Node.

#### Organization Metadata Retrieval

The [openidm query function set](https://docs.pingidentity.com/pingoneaic/latest/idm-scripting/scripting-func-engine.html) allows us to query and manage all Identity types within Identity Cloud, accessible via the route matching the identity (e.g. `managed/organizations`). The same paradigm you see in this example could be extended to other Identity types such as Group, Role, or a Custom Object.

If you’d like a primer on Organizations, take a look here:

[Organizations](https://docs.pingidentity.com/pingoneaic/latest/identities/organizations.html)

Let’s take a deeper look at the `getOrgMetadataByUID` function, which is how we’re pulling our organization attribute. In this function, we’re returning the first instance of the attribute we can find on an Organization of which our User is a Member.

```javascript
var userOrgMembership = openidm.query(`managed/${REALM}_user/${uid}/memberOfOrg`, { "_queryFilter": "true" });
```

The first query we encounter returns all Organizations in which our User is a Member.

```javascript
if (userOrgMembership && userOrgMembership.result && userOrgMembership.result.length > 0) {
    // Build the query filter to pull every organization result
    var queryFilter = "";
    var userOrgMembershipResult = userOrgMembership.result;
    for (var i = 0; i < userOrgMembershipResult.length; i++) {
      var org = userOrgMembershipResult[i];
      if (i > 0) {
       queryFilter += " or "; 
      }
      queryFilter += `(_id eq "${org._refResourceId}")`;
    }
```

Then, if the user does have membership, we construct a query filter that will return all of the matching organization’s metadata.

```javascript
orgsMetadata = openidm.query(`managed/${REALM}_organization`, { 
    "_queryFilter": queryFilter,
    "_fields": attribute
}).result;
```

Our second query makes that search, requesting only the attribute we’re looking for to be returned. Note that if your Organization has nested attributes, you’ll need your requested field to match (e.g. `parent/child`).

```javascript
for (var j = 0; j < orgsMetadata.length; j++) {
    var orgMetadata = orgsMetadata[j];
    // Check to see if there's an attribute value. If so, return that value.
    if (null != orgMetadata[attribute]) {
        return orgMetadata[attribute];
    }
}
```

Finally, we parse the response and search for a populated instance of that attribute.

### Testing

To test that our Journey is successful, we’ll add some test routes to our user as well as an Organization and show that the Journey redirects to the correct location.

#### Routing by User Attribute

In the User example, we’re expecting that the user has some redirect information stored on their user profile (for example, a session pointing back to a customized cart). Create a test user in PingOne Advanced Identity Cloud (we’re using the username `test`) and add a recognizable string to the `frUnindexedString3` attribute (by default, this attribute is labeled as `Generic Unindexed String 3` in the Identity Cloud Console). The user’s profile should look something like this:

![Screenshot of Test User](../images/2-user-details-page.png)
![Screenshot of url added to Generic Unindexed String 3](../images/4-test-user-url.png)

_Test User Data_

Open an incognito (or separate browser) window, log in as your test user, and then paste in the URL of your new Journey. You should be redirected to the URL stored on their profile.

![Screenshot of a redirect to ForgeRock.com](../images/4-redirect-result.png)

_Routing to a URL based on a User Attribute_

#### Routing by Organization Attribute

To test routing by an Organizational Attribute, we’ll need to update the `Get Metadata` script to look for the right values on the right Identity.

Update your constant params to the following:

```javascript
/**
CONSTANT PARAMS - consider using these as parameters if converting into a Library Script
*/
// Where the value is stored
var REALM = "alpha";
var IDENTITY = "organization"; // could be "user", "organization", "role", "group", etc.
var ATTRIBUTE = "description"; // could be an attribute on the user, an object reference in an org, etc.
// How to set the value in state
var STATE_KEY = "failureUrl";
```

Next, create a test Organization and add the routing URL you’d like into the `Description` field. In the future you can always add or change fields on the Organization - we’re using `Description` here as it’s included by default. Finally, add the test user you created as a Member of the Organization.

![Screenshot of Organization Details](../images/4-test-org-details.png)
![Screenshot of test user added to organization](../images/4-test-user-org-membership.png)

_Test Organization Data_

Now, if you open an incognito (or separate browser) window, log in as your test user, and then paste in the URL of your new Journey you should be redirected to the URL stored on the Organization.

![Screenshot of a redirect to pingidentity.com](../images/4-redirect-result-org.png)

_Routing to a URL based on an Organization Attribute_

# Summary

With this How-To you have redirected a user to a different failure URL based on an attribute stored on an Identity.

The full Journey, including the testing output, can be downloaded here:

[Redirect by Custom Attribute Journey](https://gist.github.com/gwizdala/b2ab2b41949933545f6fff7ba97a723e#file-pt-4_failure-redirect-by-attribute-json)

This is part of a 4-part series on Creating Custom Single Logout Using Journeys. To see the completed Journey, combining all aspects of the series, continue on to the [Recap]({{< ref "conclusion" >}}).

---

# Further Reading

- [Introduction]({{< ref "introduction" >}})
- [Part 1: Capturing the user’s existing browser session]({{< ref "1-capture-browser-session" >}})
- [Part 2: Sending out an external API request to terminate an external session]({{< ref "2-terminating-external-session-rest" >}})
- [Part 3: Invalidating the user’s PingOne AIC session]({{< ref "3-invalidating-users-aic-session" >}})
- **Part 4: Redirecting the user to a custom URL**
- [Conclusion & Recap]({{< ref "conclusion" >}})