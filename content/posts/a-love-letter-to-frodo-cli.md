---
draft: false
date: '2024-10-28'
title: 'A Love Letter to Frodo-CLI'
description: 'The one CLI to rule them all'
summary: Learn how to leverage frodo-cli to easily manage your PingOne AIC Tenant
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "Bash", "JavaScript"]
types: ["How-To"]
cover:
  image:
  alt:
  caption:
  relative: false
ShowToc: true
---

# What is Frodo-Cli, and Why Do I Love it So Much?

{{<details title="And no, I'm _not_ talking about the Baggins variety.">}}
![An Image of Frodo Baggins Looking Sad](../images/a-love-letter-to-frodo-cli/sad-frodo.avif)

_Sorry, friend_
{{</details>}}

[FroDo (ForgeRock Do)](https://github.com/rockcarver/frodo-cli) is a Command-Line Interface that enables you to quickly and easily connect to and interact with your PingOne Advanced Identity Cloud Tenant, ForgeOps deployment, and Classic PingIDM/PingAM deployments. It wraps a variety of functions and features that would otherwise require a deep knowledge of your tenant plus an encyclopedia's worth of API calls.

As a Sales Engineer at Ping, **I use FroDo daily**. When I create a new tenant, setting up FroDo is the first thing that I do. When someone needs help in their tenant, I ask them if they've connected FroDo yet.

But enough about me - let's talk why I firmly believe that **if you administer a PingOne AIC tenant you should use FroDo too**.

## What FroDo Does to Help You

I'm not going to give you all of the ins and outs because FroDo is very well-documented: just play around with `help` alongside any command and you'll see that it provides full usage, options, arguments and command details.

That being said, let's walk through a basic FroDo connection to start.

The first thing you should do with FroDo is connect it to your tenant (see [Connection Profiles](https://github.com/rockcarver/frodo-cli?tab=readme-ov-file#connection-profiles)). You do this with the following command:

```bash
frodo conn add https://<environment>/am <admin-email> '<admin-password>'
```

When you do this, FroDo performs some magic for you.

1. FroDo creates a new Service Account with all of the fixin's
2. FroDo creates a new log API key and secret
3. FroDo takes these new clients, plus details about your account and your tenant, and stores it encrypted on your device at `~/.frodo/.frodorc`

![A Screenshot of the generated FroDo Service Account](../images/a-love-letter-to-frodo-cli/frodo-service-account.png)
![A Screenshot of the generated FroDo Logger](../images/a-love-letter-to-frodo-cli/frodo-logger.png)

_One Command, Infinite Possibilities_

With one command, you have all of the capabilities you need to manage your tenant and interact with logs. This is all keyed to a shortcode, too - so the next time you want to, say, export Journeys from your tenant, rather than typing out the entire url you just need a unique substring, like:

```bash
frodo journey export -A my-unique-substring
```

No need to have a folder of jwks and client secrets, no need to re-generate an access token before all of your API calls, no need to have a post-it-note of all of the environments you're an administrator of - FroDo handles all of the dirty work for you.

## Exporting and Importing Tenant Details

PingOne AIC [takes frequent snapshots of data](https://docs.pingidentity.com/pingoneaic/latest/product-information/high-availability-disaster-recovery.html) and will backup to a secondary location dependent on your [region](https://docs.pingidentity.com/pingoneaic/latest/tenants/data-residency.html#regions). This coupled with the capability to [self-service promote](https://docs.pingidentity.com/pingoneaic/latest/tenants/self-service-promotions.html) covers most requirements for how a tenant administrator wants to securely backup and restore their tenant.

However, there are reasons as to why an administrator may want to take this a step further, for example:

1. The IAM team wants to manage configuration as code within a git repository
2. The IAM team has created something in a Sandbox environment that they want in a Promotable environment
3. The IAM administrator wants more control to the frequency and specificity of what is being backed up

FroDo gives you the tools to do this. Within FroDo, you can export any and every configuration and customization in your tenant into JSON, and import that JSON back into a tenant of your choosing.

For my job I'm usually working in multiple tenants on a daily basis, many of them being brand new ones for customer demos, feature development, and Proof of Concepts (POCs). FroDo gives me the ability to save down all of the reusable components I've designed to a standardized format and **within seconds** have it up an running in another tenant. 

As an additional benefit, I use FroDo to snapshot my entire tenant configuration before I try making a complex change - that way, if anything happens I can revert immediately to a previously-working state - and can even pick and choose which configuration I want to revert to. To do this, I've written a tiny bash script that runs a series of these export commands. When running a POC with a customer, I'll have this run on a schedule - usually saving a backup on a daily or twice daily basis.

{{<details title="`frodo_save.sh` (Click to View)">}}
```bash
#!/bin/bash

# Gather User info for loading
echo "What is the tenant name you're looking to import? (e.g. example)"
read TENANT_NAME
echo "Have you loaded this profile before? Y/N"
read IS_LOADED

# Global Variables
FULL_TENANT_NAME="openam-${TENANT_NAME}"
BASE_URL="https://${FULL_TENANT_NAME}.forgeblocks.com/am"
OUTPUT_DIR=`date +%s`

if [ "$IS_LOADED" != "y" ] && [ "$IS_LOADED" != "Y" ]; then
echo "What is your admin email address?"
read ADMIN_EMAIL
echo "What is your admin password?"
read -s ADMIN_PWD

echo "Adding account connection for $ADMIN_EMAIL"
frodo connections add $BASE_URL $ADMIN_EMAIL $ADMIN_PWD
fi

if [ ! -d "$TENANT_NAME" ]; then
	echo "Creating directory $TENANT_NAME"
	mkdir "$TENANT_NAME"
fi

echo "Navigating into directory $TENANT_NAME"
cd "$TENANT_NAME"
# Create the appropriate directories to store the data
echo "Creating directory $OUTPUT_DIR"
mkdir "$OUTPUT_DIR"
echo "Navigating into directory $OUTPUT_DIR"
cd "$OUTPUT_DIR"

echo "Creating export directories"
mkdir idm
mkdir journeys
mkdir applications
mkdir email
mkdir idp
mkdir esv
mkdir services
mkdir saml
mkdir script
mkdir theme

echo "Exporting idm configurations"
frodo idm export -A -D idm $BASE_URL
echo "Exporting Journeys ..."
cd journeys
frodo journey export -A $BASE_URL
frodo journey export -a $BASE_URL
echo "Exporting applications ..."
cd ../applications
frodo app export -A $BASE_URL
frodo app export -a $BASE_URL
echo "Exporting email templates ..."
cd ../email
frodo email template export -A $BASE_URL
frodo email template export -a $BASE_URL
echo "Exporting idp ..."
cd ../idp
frodo idp export -A $BASE_URL
echo "Exporting saml config ..."
cd ../saml
frodo saml export -A $BASE_URL
frodo saml export -a $BASE_URL
echo "Exporting scripts ..."
cd ../script
frodo script export -A $BASE_URL
frodo script export -a $BASE_URL
echo "Exporting themes ..."
cd ../theme
frodo theme export -A $BASE_URL
frodo theme export -a $BASE_URL
echo "Exporting esvs ..."
cd ../esv
frodo esv variable list $FULL_TENANT_NAME
frodo esv secret list $FULL_TENANT_NAME
echo "Exporting services ..."
cd ../services
frodo service export -A $FULL_TENANT_NAME

echo "Process Complete. Your tenant is exported. Returning to home directory"
cd ../..
```
{{</details>}}

Sure, you could do all of this with the API. You could write your own automations and wrappers and build out a really fancy approach that does everything and more - but FroDo already does it for you. If you really wanted to roll your own, check out its [internal JS library](https://github.com/rockcarver/frodo-lib).

## Tailing Logs

Another one of FroDo's killer features is the ability to live-tail logs. Rather than [standing up an ELK stack](https://github.com/atomicsamurai/elk-docker) or fumbling around with [REST calls](https://docs.pingidentity.com/pingoneaic/latest/tenants/audit-debug-logs.html#tail-logs), in a single command you can get running access to any and all data inside your tenant.

Running the following command returns pre-formatted live logs into the file `/tmp/logs.txt`. It expects that you have `jq` already on your device (if not, you can find instructions on how to download jq here: [Download and Install jq](https://jqlang.github.io/jq/download/)).

```bash
frodo logs tail -l 4 -c {{Log Option}} {{Subdomain}} |jq |sed 's/\\n/\n/g' |sed 's/\\t/\t/g' > /tmp/logs.txt
```

The available Log Options can be found here: [Get Audit and Debug Logs](https://docs.pingidentity.com/pingoneaic/latest/tenants/audit-debug-logs.html). That being said, I'd suggest **starting broad and focusing down**, more specifically:

- If it's _Access Related_, i.e. you're debugging a _Journey_, _OAuth or SAML Application_, _Policy_, or _Token_: start with `am-everything`
- If it's _Identity Related_, i.e. you're debugging a _Managed Config_, _Connector_ (but NOT the underlying Remote Connector Server - those have their own logs), or _Identity-Related Event_ (such as a `PostUpdate`): start with `idm-everything`

Once you have those logs, you can open them in your favorite text editor. I prefer using `less` since it lets me run the logs live with value highlighting.

To view the logs using `less`, run the following commands:

```bash
# Open in less
less /tmp/logs.txt

# Start live navigation
F
# Ctrl + C is Pause, while paused you can start a new search term
? {Search Term}
? {Search Term 1}|{Search Term 2}|{etc...}
```

If you don't want to remember the above commands, you can add them to your `bashrc`/`zshrc` file as a code function, but if you're like me you just have to hit the up arrow a couple times before they show up.

That's right - **I love FroDo so much, it's on my speed-dial**.

### Quickly Debugging Scripted Decision Nodes

Now let's take live tailing one step further.

I mentioned eariler that the `am-everything` log option gives us a broad view of _Access Related_ events, including Journeys and scripts within Journeys. That being said, with more than one person targeting your tenant it can be difficult to find and parse the error message that you're looking to debug in particular - especially if you're unsure where/when this error is happening.

That's where [library_logger](https://gist.github.com/gwizdala/9ae0ea4a847bf742a1d95009f0fcd504#file-library_logger-js) comes in. It's a library script that spits out a formatted log with an easy-to-search identifier in the sea of data coming in from your tail.

To import with FroDo, it's one command (download the file first!):

```bash
frodo script import -f library_logger.js -n library_logger
```

Then, in a Scripted Decision Node, I like to wrap my main function in a try-catch with the error message piping into my formatted log, something like this:

```javascript
/*
  An example of using the error logs to output data to the logger, using a library script
  Requires the library_logger library script

  DO NOT USE LIBRARY_LOGGER IN PRODUCTION

  Outcomes:
  - true
 */
//// CONSTANTS
// Import Library Script
var libraryLogger = require('library_logger');
var UNIQUE_LOGGING_IDENTIFIER = "GwizBlogs";

var NodeOutcome = {
  SUCCESS: "Success",
  ERROR: "Error"
};

//// MAIN
(function () {
  var FUNCTION_NAME = "Main";
  try {
    // (...) Your main code here
    outcome = NodeOutcome.SUCCESS;
  } catch(e) {
    libraryLogger.logFormatted(this,
    {
        callingFunction: FUNCTION_NAME,
        identifier: UNIQUE_LOGGING_IDENTIFIER,
        level: "error",
        message: e,
        scriptName: scriptName
    });
    outcome = NodeOutcome.ERROR;
  }
  action.goTo(outcome);
}());
```

If an error occurs, I'll see it in my logs like this:

```bash
**** Log ID: GwizBlogs ****
  ==== START LOG ====
  Logged At	: timestamp
  Log Level	: error
  Script	: myScriptName
  Function	: Main
  Message	: My error Message
  ==== END LOG ====
```

Is my logger standards-compliant? **Nope**. Is it really helpful for quickly troubleshooting? **You betcha**. Rather than searching for a Transaction ID (which changes every session) or a Node name (which the script could be used in more than one place), I can search for `Zephyr` (my default search term - because how many things have a Z, a P, and a Y in the same word?) and quickly get to the root of the issue. Also, by ensuring every error has an outcome, my Journey doesn't explode when a failure happens - rather, I can route it safely to an error state that I can handle however I'd like. It's a win-win.

# Summary

FroDo is an incredible tool, and I've barely scratched the surface on what it is truly capable of. It makes my job far easier and has saved my bacon time and time again. In all reality, I think of FroDo more as Samwise, because [while it can't carry the burden of a proper IAM deployment, it can carry you](https://youtube.com/shorts/hVkuicTqpCE).

Thanks FroDo team! You all (Forge)Rock!