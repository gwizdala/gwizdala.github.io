---
draft: false
date: '2025-05-01T02:00:00+00:00'
title: 'Secure Applications with Kerberos and AD using AIC and PingGateway'
description: 'Centralize your Application Management while continuing to use your Company Standards'
summary: Learn how to configure PingGateway and PingOne AIC to perform Kerberos authentication in the context of a user Journey
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "PingGateway", "Microsoft"]
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

[Kerberos](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) is a highly-prevalent network authentication protocol that (for one) enables Windows users to authenticate to client-server applications using the credentials from their Active Directory (AD) domain. This allows for a truly singular “Single Sign On” experience: allowing the user to seamlessly authenticate into applications with the same credentials they used to log into their computer without having to retype them in.

PingOne Advanced Identity Cloud (AIC), along with the deployed proxy PingGateway, can utilize this protocol to enforce a user’s domain credentials while managing application authorization and access via AIC’s centralized SaaS solution.

# Securing Applications with Kerberos and AD using AIC and PingGateway

This Guide will teach you how to configure PingGateway to communicate with Advanced Identity Cloud to perform Kerberos Authentication in the context of a user Journey. By the end of this Guide you’ll have a configured gateway and sample application that interacts with your AIC tenant.

The Guide is broken down into the following parts:

1. [Configuring your Domain Controller](#configuring-your-domain-controller)  
2. [Configuring PingGateway](#configuring-pinggateway)

Some important callouts before we begin:

* This Guide is tested up to **PingGateway version 2024.11**. While the concepts should be applicable to future releases, there may be slight changes to underlying configuration and/or routes - as such, ensure that you are following the documentation that matches the version you’ve downloaded (you can do so by selecting the version on the same page links provided in this Guide).  
* This Guide uses two **Windows Server 2019** for the example VMs. The same principles should still apply to other versions of Windows but the interfaces shown in the screenshots may have changed.  
* This Guide expects that you use the Sample Application provided in the [PingGateway QuickStart](https://docs.pingidentity.com/pinggateway/2024.11/getting-started/start-sampleapp.html). You may use a separate application but will need to alter the routes defined in the gateway and in your `hosts` file.  
* This Guide covers how to configure PingGateway and AIC. It is not a guide on setting up Kerberos on Windows. It is expected that you already have a running instance of Active Directory with its own Domain Service and a Domain-Joined computer in which you’ll be running PingGateway.  
* PingGateway doesn’t require any special permissions to run. That being said, you’ll need administrative access to your environments so that you can 1) create an Active Directory Service Account and 2) edit your `hosts` file to include your PingGateway domain.

There’ll be some common names I’ll use in this Guide:

* The Domain Controller is named **dgwiz-vm** and I’ll be referring to it as **VM 1**  
* The Gateway Host is named **dgwiz-vm2** and I’ll be referring to it as **VM 2**  
* My Domain is **example.com** to match the PingGateway tutorials

In the real world, you will likely have a separate domain-joined Client that the user will be logging in with. Since domain-joining your computers does not impact how you configure PingGateway, I'm performing Kerberos authentication on the Gateway Host directly.

# Configuring your Domain Controller

At this point you should already have an instance of Active Directory with a Domain Service installed (on **VM 1**). All we’ll need here is to set up a means for the Domain Controller and PingGateway to securely communicate with one another.

The steps in this section are on your Domain Controller (**VM 1**).

## Create a Service Account

An Active Directory service account will let us securely communicate with PingGateway and the domain when performing Kerberos authentication. As a Service Account is normally required to domain-join you might have one already that you can use. Under “Member of” permissions, this Service Account will need to be a part of the **Domain Admins** and **Domain Users** AD Group. My example is also part of the Administrators AD Group but this isn’t necessary.

After you’ve created this account, you’ll need to do two things:

1. Copy the user’s username and password to be used later (the DN can be found under Attribute Editor in the user details). I set my password to never expire but that’ll be up to your own security rules.  
2. Register a Service Principal Name with this account using the command `setspn -s <service-principal-name> <username>`. In my case, my command looks like this:  
   `setspn -s HTTP/dgwiz-vm2.example.com fidcsvc`

![The Service Account User Details](/img/secure-apps-kerberos-ad-pinggateway/service-acct-details.png)
![The Service Account AD Groups](/img/secure-apps-kerberos-ad-pinggateway/service-acct-ad-groups.png)

_The Service Account Details_

## Setting the Hosts File

We will need our Domain Controller to know that it can communicate with PingGateway. To do so, update your Hosts File found at `%SystemRoot%\system32\drivers\etc\hosts` to include an entry pointing to the IP address (found with the `ipconfig` command) and name of the machine (found under Server Manager → Local Server → Computer Name) holding PingGateway. You’ll need to edit with admin permissions and may need to toggle your file explorer filter to see this file (“Show all Files”). The name of my VM 2 is `dgwiz-vm2.example.com` which is why you see it listed here.

**`hosts`**

```
# PingGateway Hosts, where xxx.x.x.x is the IPv4 address on your Gateway
    xxx.x.x.x  dgwiz-vm2.example.com
```

# Configuring PingGateway

The steps below will walk you through how to install and set up PingGateway for Kerberos Authentication for the first time. Since Kerberos Authentication normally implies downstream connections to external systems, this walkthrough will teach you how to apply Kerberos Authentication to an authentication Journey in Advanced Identity Cloud and perform Cross-Domain Single-Sign-On (CDSSO) to a sample application. Along the way, it will include checkpoints to ensure that PingGateway is operating correctly.

The steps in this section are on your Gateway Host (**VM 2**). 

For the sake of the example, it’s also useful to [set up the Sample Application](https://docs.pingidentity.com/pinggateway/2024.11/getting-started/start-sampleapp.html) so that we have an example application to redirect to. Note that this application runs on Java and in my example I’m using the JDK version 17 installed from Adoptium Temurin: [https://adoptium.net/temurin/releases/](https://adoptium.net/temurin/releases/) with the `JAVA_HOME` System Variable pointing to this JDK.

![A terminal window showing the hava version. This example is using openjdk 17.0.14](/img/secure-apps-kerberos-ad-pinggateway/java.png)

_The Java Environment_

The rest of this document will operate under the assumption that you’ve installed the example application.

## Create A Secrets Folder

We’ll be creating some secrets to interact with services like AIC and AD. This example helps us easily store and re-reference those secrets later.

For ease of use with our example, I’d suggest creating a `secrets` folder on the Desktop. In a production instance, you’ll likely want this to live elsewhere.

## Install and Start Up PingGateway

PingGateway can be run in the foreground or in the background as a service. In our case we will be running the gateway from our command prompt following the instructions provided in the [Gateway Quick Start Guide](https://docs.pingidentity.com/pinggateway/2024.11/getting-started/preface.html).

[Navigate to Identity Cloud Downloadable Components](https://backstage.forgerock.com/downloads/browse/identity-cloud/featured) and download and unzip PingGateway to a location of your choosing. I’m using Desktop.

Update your Hosts File, found at `%SystemRoot%\system32\drivers\etc\hosts`, to include an entry pointing to PingGateway and the example app hosted locally on this system. You’ll need to edit with admin permissions and may need to toggle your file explorer filter to see this file (“Show all Files”). The name of my VM 2 is `dgwiz-vm2.example.com` and the path of my example app is `app.example.com` which is why you see them listed here.

**`hosts`**

```
# PingGateway Hosts
    127.0.0.1  dgwiz-vm2.example.com app.example.com
```

Start your gateway by running the `start.bat` file located under the `bin` directory of your IG download. The easiest way is to drag and drop the file into your terminal.

![A resulting command prompt window where the start.bat file has been dragged and dropped](/img/secure-apps-kerberos-ad-pinggateway/start.png)

_Starting PingGateway_

## ☑️ Checkpoint - Validate that the Gateway is Running

Once your gateway is up and running, you should be able to ping it at the location you defined in your host file and the port/path of `8080/openig/ping`. In my case, it’s `http://dgwiz-vm2.example.com:8080/openig/ping`. 

> **Note:** For PingGateway 2025.3 and above it’s a different port and path: `8085/ping` or, in my example, `http://dgwiz-vm3.example.com:8085/ping`

Stop your gateway. If PingGateway closed unexpectedly (e.g. you closed your command window) you’ll likely want to delete the `ig.pid` file located in your `tmp` folder (`%appdata%\OpenIG\tmp`).  

## Configure PingGateway for Server-Side TLS

So that we can communicate with AIC securely we’ll be configuring PingGateway for Server-Side TLS as described in the [Installation Instructions](https://docs.pingidentity.com/pinggateway/2024.9/installation-guide/securing-connections.html#server-side-tls). To do this we will need to create and save a certificate.

For the sake of simplicity, follow **step 2 only** in [Serve one certificate for TLS connections to all server names](https://docs.pingidentity.com/pinggateway/2024.9/installation-guide/securing-connections.html#server-side-tls-keyManager). I set up a self-signed certificate in a (PKCS#12) keystore and saved the keystore to the `secrets` folder I defined earlier. I’m using `keytool` as I have Java installed on my machine.

**In your “Secrets” folder**

```
# Change -storepass and -keypass to your own password
# Change your -dname to your own name
 keytool -genkey -alias https-connector-key -keyalg RSA -keystore keystore.pkcs12 -storepass password -keypass password -dname "CN=dgwiz-vm2.example.com,O=Example,C=Com"
```

We then store the password in a secure file. In the example we’re putting it in the same `secrets` folder we generated the keystore file in.

**In your “Secrets” folder**

```
# Change password to your own password. When this file is generated, ensure that you don't have any trailing whitespaces
(echo password) > keystore.pass
```

Note the following - I’m defining my gateway as the Common Name (that's the `CN=` part of the first command) and am using the password `password` for the key (the word `password` in both of the commands). You’ll want to change these values in your implementation and ensure that your `keystore.pass` file doesn’t contain any trailing spaces/newlines.

## Configure PingGateway Settings

Now that you’ve started your gateway for the first time you’ll have a configuration folder that you can update with additional configuration regarding what and how it protects. The files and routes you can manipulate are listed here: [Configure PingGateway](https://docs.pingidentity.com/pinggateway/2024.11/configure/configure.html).

To start, we’ll want to update our `admin.json` file to define our ports, identify the location in which we are holding our secrets and keystores (for things like connecting to AIC and to AD), and enable Gateway Studio which allows us to visualize our routes we’re creating in a nice editor. This file should be under `%appdata%\OpenIG\config\admin.json` - if it doesn’t exist you can create it instead.

**[`admin.json`](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/secure-apps-kerberos-ad-pinggateway/admin.json)**

```javascript
{
  "mode": "DEVELOPMENT", // Enables a PUBLIC route to Gateway Studio on http://<gateway-url>:8080/openig/studio/
  "connectors": [
    {
      "port": 8080 // Your default Gateway hosting
    },
    {
      "port": 8443, // Required for Identity Assertion
      "tls": "ServerTlsOptions-1"
    }
  ],
  "session": {
    "cookie": {
      "sameSite":  "none",
      "secure": true // ensure the browser passes the session cookie in form-POST for CDSSO
    }
  },
  "heap": [
    {
      "name": "ServerTlsOptions-1",
      "type": "ServerTlsOptions",
      "config": {
        "keyManager": {
          "type": "SecretsKeyManager",
          "config": {
            "signingSecretId": "key.manager.secret.id",
            "secretsProvider": "ServerIdentityStore"
          }
        }
      }
    },
    {
      "type": "FileSystemSecretStore",
      "name": "SecretsPasswords",
      "config": {
        "directory": "<your-file-path>/Desktop/secrets", // Where the secrets are located
        "format": "PLAIN"
      }
    },
    {
      "name": "ServerIdentityStore",
      "type": "KeyStoreSecretStore",
      "config": {
        "file": "<your-file-path>/Desktop/secrets/keystore.pkcs12", // Where the keystore is located
        "storePasswordSecretId": "keystore.pass", // The file containing the password for your keystore
        "secretsProvider": "SecretsPasswords",
        "mappings": [
          {
            "secretId": "key.manager.secret.id",
            "aliases": ["https-connector-key"] // The name of your key for your cert
          }
        ]
      }
    }
  ]
}
```

### (Optional) Increase the PingGateway Logging Level

If you’d like to see more logging within your terminal and within the **`%appdata%\OpenIG\logs`** path, [add the environment system variable](#using-system-variables) `ROOT_LOG_LEVEL` with the value of `DEBUG`.

![A Screenshot of the System Variables where ROOT_LOG_LEVEL has been set to DEBUG](/img/secure-apps-kerberos-ad-pinggateway/debug.png)

_Increasing the Log Level_

**If you did add this variable, you’ll need to restart PingGateway in a new Command Prompt for the variable to take effect.**


## ☑️ Checkpoint - Validate the certificate is loaded

We will follow **steps 4 and 5** in [Serve one certificate for TLS connections to all server names](https://docs.pingidentity.com/pinggateway/2024.9/installation-guide/securing-connections.html#server-side-tls-keyManager). Start your gateway and then head to the ping route you went to earlier (in my case, `https://dgwiz-vm2.example.com:8443/openig/ping`). If you’re running a new version of PingGateway you may need to test your certificate on a route directly (more on that later).

From there you can inspect your certificate to see that it was loaded successfully.  
![A Screenshot of the browser url bar where the certificate details are being inspected](/img/secure-apps-kerberos-ad-pinggateway/cert-inspect.png)
![A Screenshot of the browser where the certificate icon is being selected](/img/secure-apps-kerberos-ad-pinggateway/cert-select.png)
![A Screenshot of the browser inspecting the general information about the certificate](/img/secure-apps-kerberos-ad-pinggateway/cert-general.png)

_Viewing the Certificate_

## Connect PingGateway to AIC

We first want to create a secure connection between PingGateway and AIC. To do so, we will be using the setup information defined in the [AIC Setup Documentation](https://docs.pingidentity.com/pinggateway/2024.9/identity-cloud-guide/preface.html).

First, [Create the Agent Journey](https://docs.pingidentity.com/pinggateway/2024.9/identity-cloud-guide/preface.html#authenticate-agent-idc). You can follow the instructions provided or import a precreated Journey here: [Agent-Journey.json](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/secure-apps-kerberos-ad-pinggateway/Agent-Journey.json).

![A Screenshot of the Import Journey button, which you'll use to import this Journey](/img/secure-apps-kerberos-ad-pinggateway/import.png)

_Importing the Journey_

Next, [Register Your PingGateway Agent in AIC](https://docs.pingidentity.com/pinggateway/2024.9/identity-cloud-guide/preface.html#register-agent-idc) and save the password. You don’t need to "use the secret store for the password" for the examples to work.

![A Screenshot of the "New Gateway" button](/img/secure-apps-kerberos-ad-pinggateway/gateway-new.png)
![A Screenshot of the gateway selection screen, where "Identity Gateway" is selected](/img/secure-apps-kerberos-ad-pinggateway/gateway-select.png)
![A Screenshot of entering in the gateway name and password for the connection](/img/secure-apps-kerberos-ad-pinggateway/gateway-config.png) 
![A Screenshot of the generated gateway connection](/img/secure-apps-kerberos-ad-pinggateway/gateway-created.png)

_Registering the Gateway_

Back in VM 2 you’ll want to save the password so that you can reference it when connecting to your tenant. For the sake of the example, I’ve put mine into a [base64-encoded](#base64-encoding-variables-in-windows) [system variable](#using-system-variables). **Note that the password must be Base-64 encoded when being referenced by a route in the gateway.**

![A Screenshot of the AGENT_SECRET_ID system variable with the base64-encoded password](/img/secure-apps-kerberos-ad-pinggateway/gateway-password.png)

_Storing the Gateway Password in a System Variable_

Later we will be [defining a route in our Gateway for Cross-Domain Single-Sign-On](#set-up-cross-domain-single-sign-on). So that AIC knows where to redirect from a CDSSO request, we’ll add a redirect URL on our gateway configuration that points to `https://<your-gateway-url>:8443/home/cdsso/redirect`. I’ve added the route `https://dgwiz-vm2.example.com:8443/home/cdsso/redirect` in my example.

![A Screenshot of the CDSSO Redirect URL added to the Gateway config](/img/secure-apps-kerberos-ad-pinggateway/gateway-redirect.png)

_Setting the Redirect URL_

So that AIC knows that it can safely redirect to the URLs requested by our gateway, including that `cdsso` one we just added to our Gateway config, we’ll need to add it to the Validation Service. 

Go to **Native Consoles → Access Management → Services → Validation Service** and add the URLs that PingGateway will be redirecting the user to post-authentication. The two routes I’ve added are `https://dgwiz-vm2.example.com:8443/*` and `https://dgwiz-vm2.example.com:8443/*?*` to encompass any routes underneath my Gateway - however you can specify paths here if you’d prefer to lock down what redirects are allowed. 

![A Screenshot of the the validation service where the aforementioned routes have been added](/img/secure-apps-kerberos-ad-pinggateway/validation-service.png)

_Updating the Validation Service_

## Set Up Identity Assertion

Identity Assertion allows us to perform actions in PingGateway from an AIC Journey and vice versa. As such, we can use Identity Assertion to direct unauthenticated requests from PingGateway to a Journey in AIC (think Cross-Domain Single-Sign-On) or from AIC to a local authentication service (such as Kerberos) with PingGateway. Identity Assertion communicates this information with a JWT and will use a shared secret to encrypt and decrypt the responses between the two systems. We will be using the setup information defined in the [Identity Assertion Node Example](https://docs.pingidentity.com/auth-node-ref/latest/auth-node-identity-assertion-node.html#auth-node-gateway-comm-example).

Follow the steps in [Create and import a secret encryption key](https://docs.pingidentity.com/auth-node-ref/latest/auth-node-identity-assertion-node.html#auth-node-identity-assertion-key). In AIC, you should see your secret in “Tenant Settings”. If you have a prompt to update your ESVs select the “View Updates” button to complete loading the value in your tenant. You’ll see something like what’s shown below if you’ve succeeded.

![A Screenshot of the ESV page in which one update needs to be applied](/img/secure-apps-kerberos-ad-pinggateway/idassert-esv-platform.png)

_Loading the ESV Update from the Platform UI_

Additionally, store the `idassert.pem` file in the directory **`%appdata%\OpenIG\secrets\igfs`** on your Gateway Host to match the example route we’ll use later. You may have to create these folders if they don’t exist.

![A Screenshot of the pem file stored under the aforementioned path](/img/secure-apps-kerberos-ad-pinggateway/idassert-pem.png)

_Storing the Pem File_

Follow the steps to [Configure the Identity Assertion Service](https://docs.pingidentity.com/auth-node-ref/latest/auth-node-identity-assertion-node.html#auth-node-identity-assertion-service). These steps will show you how to create the service and configure your gateway. Screenshots for how to do so are below.

Go to **Native Consoles → Access Management → Services → Add a Service → Identity Assertion Service**:

![A Screenshot of selecting the Access Management tab from the Native Consoles Dropdown](/img/secure-apps-kerberos-ad-pinggateway/idassert-service-native.png) 
![A Screenshot of steps taken to enable the Identity Assertion Service, pointing to Services and the New button](/img/secure-apps-kerberos-ad-pinggateway/idassert-service-add.png)  
![A Screenshot of the enabled Identity Assertion Service](/img/secure-apps-kerberos-ad-pinggateway/idassert-service-enable.png)

_Adding the Identity Assertion Service_ 

Go to **Secondary Configurations → Add a Secondary Configuration →** *Your Gateway URL, using port 8443 in your URL and the name of your ESV for the Shared Encryption Secret*:  

![A Screenshot of the Secondary Configuration being added for https://dgwiz-vm2.example.com:8443 with the idassert encryption secret](/img/secure-apps-kerberos-ad-pinggateway/idassert-service-config.png)

_Creating the Secondary Configuration_

Next, follow the steps to [map the secret label to the encryption key](https://docs.pingidentity.com/auth-node-ref/latest/auth-node-identity-assertion-node.html#auth-node-identity-assertion-secret-store). These steps will show you how to assign the ESV you’ve set to your assertion service. Screenshots for how to do so are below.

*Still in the Native Access Management Console,*  
Go to **Secret Stores → ESV → Mappings → Add Mapping**:

![A screenshot of the native console where Secret Store and ESV are selected](/img/secure-apps-kerberos-ad-pinggateway/esv-mapping.png) 
![A screenshot of the new esv mapping button](/img/secure-apps-kerberos-ad-pinggateway/esv-mapping-add.png)

_Getting to the ESV Mapping_

**Select the secret label you created for your Assertion Service** *(my example is `am.services.identityassertion.service.idassert.shared.secret`) **and use the name of your ESV secret encryption key you defined earlier** (mine was **`esv-idassert`**)* **as the alias. Click Add, then Create.**  

![A Screenshot of the ESV mapping tied to the ESV we set by API earlier](/img/secure-apps-kerberos-ad-pinggateway/esv-mapping-config.png)

_Adding the ESV Mapping_

Next, we’ll create a route in PingGateway that tests if Identity Assertion is working correctly. This route will be targeted by a Journey in AIC.

**Underneath the path `%appdata%\OpenIG\config\routes\`, create the file `identity-assertion-idc.json`.** This route will reach out to and authenticate PingGateway to Advanced Identity Cloud and then return a “test” user to validate that the assertion succeeded. You can copy the route directly from the [documentation](https://docs.pingidentity.com/pinggateway/2024.9/reference/IdentityAssertionHandler.html#IdentityAssertionHandler-examples) - the same code is below but with comments explaining what each handler is doing and what values to configure.

**[identity-assertion-idc.json](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/secure-apps-kerberos-ad-pinggateway/identity-assertion-idc.json)**

```javascript
{
  "name": "identity-assertion-idc",
  "condition": "${find(request.uri.path, '^/idassert')}", // The URL path that triggers this flow. AIC will be hitting Gateway with this request.
  "properties": {
    "amIdcPeer": "<your-tenant-url>" // The AIC tenant we are targeting, e.g. openam-example.forgeblocks.com
  },
  "handler": "IdentityAssertionHandler-1",
  "heap": [
    {
      "name": "IdentityAssertionHandler-1",
      "type": "IdentityAssertionHandler", // https://docs.pingidentity.com/pinggateway/2024.9/reference/IdentityAssertionHandler.html
      "config": {
        "identityAssertionPlugin": "BasicAuthScriptablePlugin",
        "selfIdentifier": "https://<gateway-url>:8443", // The gateway url
        "peerIdentifier": "&{amIdcPeer}",
        "secretsProvider": [
          "secrets-pem"
        ],
        "encryptionSecretId": "idassert" // The pem file we generated and stored in aic as an esv as well as locally for our gateway
      }
    },
    {
      "name": "BasicAuthScriptablePlugin",
      "type": "ScriptableIdentityAssertionPlugin",
      "config": {
        "type": "application/x-groovy",
        "source": [
          "import org.forgerock.openig.assertion.IdentityAssertionClaims",
          "import org.forgerock.openig.assertion.plugin.IdentityAssertionPluginException",
          "import org.forgerock.openig.assertion.IdentityRequestJwtContext",
          "logger.info('Running ScriptableIdentityAssertionPlugin')",
          "logger.info('IdentityRequestJwtContext redirect: {}', context.asContext(IdentityRequestJwtContext.class).redirect())",
          "return new IdentityAssertionClaims('test')" // The user information we are returning in this example
        ]
      }
    },
    {
      "name": "pemPropertyFormat",
      "type": "PemPropertyFormat"
    },
    {
      "name": "secrets-pem",
      "type": "FileSystemSecretStore",
      "config": {
        "directory": "&{ig.instance.dir}/secrets/igfs", // The location where this secret is stored. You'll need to create this folder
        "suffix": ".pem",
        "format": "pemPropertyFormat",
        "mappings": [
          {
            "secretId": "idassert",
            "format": "pemPropertyFormat"
          }
        ]
      }
    }
  ]
}
```

Now that we have our route, let’s create a new Journey that uses it. This Journey will match the example provided in [Configure the example authentication journey](https://docs.pingidentity.com/auth-node-ref/latest/auth-node-identity-assertion-node.html#configure_the_example_authentication_journey).

Go to **Journeys → New Journey**, and create a Journey with the following details:

| Input | Value |
| ----- | ----- |
| Name | IdAssert |
| Identity Object | Alpha realm - Users `managed/alpha_user` |
| Description (optional) | An example journey showcasing how you can assert an identity from a local source using PingGateway |
| Tags (optional) | Gateway |

This Journey really only needs one node - the Identity Assertion Node. In it you’ll want to select the Identity Assertion Service you configured during this step and the route that matches the route we just made in our gateway (`/idassert`). Additionally, we will be mapping the claim passed back from PingGateway into the `username` field so that we can authenticate a user. You can add a Failure URL Node if you’d like but it’s not necessary.

![A screenshot of the IdAssert Journey in editor highlighting the Identity Assertion Node](/img/secure-apps-kerberos-ad-pinggateway/idassert-journey.png)

_The IdAssert Journey_

Our `/idassert` route as we configured it returns a user with the username `test`. To make this Journey more realistic let’s add an Identify Existing User Node to generate a session and take our user into their end-user dashboard.  You’ll note that I set my “Identity Attribute” in my Identify Existing User node to “userName” since that’s the property returned by our gateway in this example.

![A screenshot of the IdAssert Journey in editor where the Identify Existing User Node has been added](/img/secure-apps-kerberos-ad-pinggateway/idassert-journey-session.png)

_The IdAssert Journey with User Session_

## ☑️ Checkpoint - Testing Identity Assertion 

Let’s test that Identity Assertion is working. If you haven’t already, [start PingGateway](https://docs.pingidentity.com/pinggateway/2024.11/getting-started/start-stop.html) **in a new command prompt (you need to do this to load your system variables)**.

Since we’ve enabled Development mode we should have access to Gateway Studio, a visual editor that allows us to view and manage the routes we’ve created.

Go to your Gateway Studio route, defined as `http://<gateway-url>:8080/openig/studio/` (mine is [`http://dgwiz-vm2.example.com:8080/openig/studio/`](http://dgwiz-vm2.example.com:8080/openig/studio/), newer versions may be `8085/studio` without the `/openig` route). You should load Gateway Studio and see the route you just defined.

![A screenshot of IGStudio in which the IDAssert Route has appeared](/img/secure-apps-kerberos-ad-pinggateway/studio-idassert.png)

_Gateway Studio Showing the IdAssert Route_

Next, we’ll test if Identity Assertion is working correctly. To do so, copy the Preview URL from your Journey and open it in an incognito tab in a browser on VM 2 (where PingGateway is hosted).

> **Note:** this route is returning the username `test`. Make sure you have a user with that username or your Journey will return No User Found.

Going to the Journey URL will result in an active session for your `test` user.  

![A screenshot of the Test User logged in based on the Journey provided](/img/secure-apps-kerberos-ad-pinggateway/idassert-journey-success.png)

_The Test User's Dashboard_ 

## Set Up Cross-Domain Single-Sign-On {#set-up-cross-domain-single-sign-on}

CDSSO allows us to authenticate a User based on a session generated from AIC. This also gives our gateway access to invoke and interact with Journeys. Later, we will be using CDSSO to get the user’s session in AIC based on the Kerberos Authentication performed by the gateway.

To perform CDSSO, we will be following the tutorial provided at [Cross-domain single sign-on](https://docs.pingidentity.com/pinggateway/2024.9/identity-cloud-guide/cdsso.html). Since we’ve already configured our gateway and set up Identity Assertion, we only have to do **steps 2d and 2e.**

**Underneath the path `%appdata%\OpenIG\config\routes\`, create the file `00-static-resources.json`.** This will be our first path we are going to protect - in this case, to serve static resources to our sample application. You can copy the route directly from the documentation.

**Underneath the path `%appdata%\OpenIG\config\routes\`, create the file `cdsso-idc.json`.** This route will perform CDSSO with AIC, retrieving a session based on an AIC user’s account. You can copy the route directly from the documentation - the same code is below but with comments explaining what each handler is doing and what values to configure.

**[cdsso-idc.json](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/secure-apps-kerberos-ad-pinggateway/cdsso-idc.json)**

```javascript
{
  "name": "cdsso-idc",
  "baseURI": "http://app.example.com:8081", // The application we redirect to with the session
  "condition": "${find(request.uri.path, '^/home/cdsso')}", // The URL path that triggers this flow. Gateway will be hit first with this request.
  "properties": {
    "amInstanceUrl": "https://<your-tenant-url>/am" // The AIC tenant we are targeting, e.g. openam-example.forgeblocks.com
  },
  "heap": [
    {
      "name": "SystemAndEnvSecretStore-1",
      "type": "SystemAndEnvSecretStore" // How we retrieve the secret. In this case, from the system's ESV
    },
    {
      "name": "AmService-1",
      "type": "AmService",
      "config": {
        "url": "&{amInstanceUrl}",
        "realm": "/alpha",
        "agent": {
          "username": "<your-gateway-name>", // The gateway name in AIC. In this case, dgwiz-vm2
          "passwordSecretId": "agent.secret.id" // The ESV on this machine that holds the gateway's password - stored in base64 with the naming convention AGENT_SECRET_ID
        },
        "secretsProvider": "SystemAndEnvSecretStore-1",
        "sessionCache": {
          "enabled": false
        }
      }
    }
  ],
  "handler": {
    "type": "Chain",
    "config": {
      "filters": [
        {
          "name": "CrossDomainSingleSignOnFilter-1",
          "type": "CrossDomainSingleSignOnFilter",
          "config": {
            "redirectEndpoint": "/home/cdsso/redirect",
            "authCookie": {
              "path": "/home",
              "name": "ig-token-cookie"
            },
            "amService": "AmService-1", // The default login journey
            "_authenticationService": "IdAssertKerberos" // Your own login Journey - this is how we can assert the identity later. To use, remove the "_" from the beginning of the key name
          }
        }
      ],
      "handler": "ReverseProxyHandler"
    }
  }
}
```

## ☑️ Checkpoint - Testing Cross-Domain Single-Sign-On

Let’s test that CDSSO is working. If you haven’t already, [start PingGateway](https://docs.pingidentity.com/pinggateway/2024.11/getting-started/start-stop.html) and [start the Sample Application](https://docs.pingidentity.com/pinggateway/2024.11/getting-started/start-sampleapp.html#start-sampleapp-start). **If you’ve [modified or added any system variables](#using-system-variables), restart Gateway in a new Command Prompt for the changes to take effect.**

Since we’ve enabled Development mode we should have access to Gateway Studio, a visual editor that allows us to view and manage the routes we’ve created.

Go to your Gateway Studio route, defined as `http://<gateway-url>:8080/openig/studio/` (mine is [`http://dgwiz-vm2.example.com:8080/openig/studio/`](http://dgwiz-vm2.example.com:8080/openig/studio/) but it could also be `8085/studio` depending on your version). You should load Gateway Studio and see the routes you just defined.

![A screenshot of IGStudio in which the CDSSO and Static Resources Routes have appeared](/img/secure-apps-kerberos-ad-pinggateway/studio-cdsso.png)

_Gateway Studio Showing the CDSSO Routes_

Next, we’ll test if CDSSO is working correctly. To do so, in an incognito tab in VM 2 we’ll target the new CDSSO route we made at  `https://<gateway-url>:8443/home/cdsso/` (mine is [`https://dgwiz-vm2.example.com:8443/home/cdsso/`](https://dgwiz-vm2.example.com:8443/home/cdsso/)). You should be redirected to your AIC login page. Enter in your details and you’ll be redirected to the Sample Application with a valid session.

![A screenshot of the redirect to AIC's login screen](/img/secure-apps-kerberos-ad-pinggateway/cdsso-login.png)
![A screenshot of the sample app, redirected from AIC with a valid session](/img/secure-apps-kerberos-ad-pinggateway/cdsso-session.png)

_Cross-Domain Single Sign On Performed to the Sample App_

## Set Up Kerberos Authentication

Now, let’s combine the two concepts we just learned to perform Kerberos Authentication from within a Journey and then CDSSO into an application.

Firstly, in VM 2 you’ll want to save the Windows AD Domain service account password so that you can reference it when performing Kerberos Authentication. For the sake of the example, I’ve put mine into a [base64-encoded](#base64-encoding-variables-in-windows) [system variable](#using-system-variables). **Note that the password must be Base-64 encoded when being referenced by a route in the gateway.**

![A screenshot of the service account password stored as a system variable](/img/secure-apps-kerberos-ad-pinggateway/kerb-esv.png)

_Storing the Kerberos Service Account Password_

Next, we’re going to create a new route that utilizes the [KerberosIdentityAssertionPlugin](https://docs.pingidentity.com/pinggateway/2024.11/reference/KerberosIdentityAssertionPlugin.html#KerberosIdentityAssertionPlugin-example).

**Underneath the path `%appdata%\OpenIG\config\routes\`, create the file `identity-assertion-kerberos.json`.** This route will allow a Journey to reach out to PingGateway to perform Kerberos authentication.

**[identity-assertion-idc-kerberos.json](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/secure-apps-kerberos-ad-pinggateway/identity-assertion-idc-kerberos.json)**

```javascript
{
  "name": "identity-assertion-idc-kerberos",
  "condition": "${find(request.uri.path, '^/kerb')}", // The URL path that triggers this flow. AIC will be hitting Gateway with this request.
  "properties": {
    "amIdcPeer": "<your-tenant-url>" // The AIC tenant we are targeting, e.g. openam-example.forgeblocks.com
  },
  "handler": "IdentityAssertionHandler-1",
  "heap": [
    {
      "name": "SystemAndEnvSecretStore-1",
      "type": "SystemAndEnvSecretStore" // How we retrieve the secret. In this case, from the system's ESV
    },
    {
      "name": "IdentityAssertionHandler-1",
      "type": "IdentityAssertionHandler", // https://docs.pingidentity.com/pinggateway/2024.9/reference/IdentityAssertionHandler.html
      "config": {
        "identityAssertionPlugin": "KerberosIdentityAssertionPlugin-1",
        "selfIdentifier": "https://<gateway-url>:8443", // The gateway url
        "peerIdentifier": "&{amIdcPeer}",
        "secretsProvider": [
          "secrets-pem"
        ],
        "encryptionSecretId": "idassert" // The pem file we generated and stored in aic as an esv as well as locally for our gateway
      }
    },
    {
      "name": "KerberosIdentityAssertionPlugin-1",
      "type": "KerberosIdentityAssertionPlugin", // https://docs.pingidentity.com/pinggateway/2024.11/reference/KerberosIdentityAssertionPlugin.html
      "config": {
        "serviceLogin": "UsernamePasswordServiceLogin-1",
        "trustedRealms": ["EXAMPLE.COM"] // The (trusted) Realm matching the user's principal name
      }
    },
    {
      "name": "UsernamePasswordServiceLogin-1",
      "type": "UsernamePasswordServiceLogin",
      "config": {
        "username": "<service-account-username>", // Your service account username (run as admin on AD host): setspn -s HTTP/<gateway-host> <service-account-username>
        "passwordSecretId": "kerberos.secret.id", // Your user account password, stored as a base64-encoded secret called KERBEROS_SECRET_ID
        "secretsProvider": "SystemAndEnvSecretStore-1"
      }
    },
    {
      "name": "pemPropertyFormat",
      "type": "PemPropertyFormat"
    },
    {
      "name": "secrets-pem",
      "type": "FileSystemSecretStore",
      "config": {
        "directory": "&{ig.instance.dir}/secrets/igfs", // The location where this secret is stored. You'll need to create this folder
        "suffix": ".pem",
        "format": "pemPropertyFormat",
        "mappings": [
          {
            "secretId": "idassert",
            "format": "pemPropertyFormat"
          }
        ]
      }
    }
  ]
}
```

Back inside AIC, duplicate your IdAssert Journey and give it the name `IdAssertKerberos`. Inside that Journey the only change you’ll need to make is to point the route in your Identity Assertion Node to the new route we created (`/kerb`).

![A screenshot of the idassert journey that is directed to the Kerberos route](/img/secure-apps-kerberos-ad-pinggateway/idassert-journey-kerb.png)

_The Identity Assertion Journey Pointing to the Kerberos Route_

Finally, let’s update our cdsso-idc route to point to this new Journey we created. Remove the `_` from `_authenticationService` (line 44) in your `cdsso-idc.json` file so that the authentication service used is the IdAssertKerberos Journey.

![A screenshot of the "_" removed from line 44 in the cdsso-idc route](/img/secure-apps-kerberos-ad-pinggateway/kerb-cdsso.png)

_Pointing to the Kerberos Journey_

### (Optional) Create the krb5.ini file

If you’re testing Kerberos on a local machine or with an older version of AD/Kerberos, there’s a chance that you may need to provide additional information for authentication to work properly. Adding the following file allows us to specify configuration properties when performing Kerberos with PingGateway. My version is below but you can use a generic one in the link provided.

**[krb5.ini](https://github.com/gwizdala/lib-ping/blob/main/How-Tos/secure-apps-kerberos-ad-pinggateway/krb5.ini)**

```
[libdefaults]
        default_realm = EXAMPLE.COM
        allow_weak_crypto = true
[realms]
        EXAMPLE.COM = {
            kdc = dgwiz-vm.example.com
            master_kdc = dgwiz-vm.example.com
            admin_server = dgwiz-vm.example.com
            default_domain = EXAMPLE.COM
        }
[domain_realm]
        example.com = EXAMPLE.COM
[login]
        krb4_convert = true
```

> **Note:** If you support older encryption types, in your [libdefaults] section you may need to add the following (underneath the `allow_weak_crypto` line):
> 
> ```
> default_tkt_enctypes = des-cbc-crc rc4-hmac
> default_tgs_enctypes = des-cbc-crc rc4-hmac
> ```
> 
> You’ll know you need this if you see the error message Identity assertion response error `java.security.PrivilegedActionException: GSSException: Failure unspecified at GSS-API level (Mechanism level: Encryption type RC4 with HMAC is not supported/enabled)`


This file is helping us with the following items:

* (`allow_weak_crypto = true` and `krb4_convert = true`) The KerberosIdentityAssertion node expects a stronger level of encryption that comes default with older deployments of Kerberos so we’ll need to let PingGateway know that this older method is valid in our deployment. In a real deployment it may be wise to increase your security stance by updating your encryption method.  
* (`default_realm = EXAMPLE.COM` and the `[realms]` section) We’re mapping the domain to our Domain Controller (in my case, `dgwiz-vm.example.com`). This helps the handler fall back to a default if none is specified.

I’ve saved this file to my `secrets` folder and then added it to the `IG_OPTS` [system variable](#using-system-variables) using the following value:

```
"-Djava.security.krb5.conf=C:\Users\david_gwizdala\Desktop\secrets\krb5.ini"
```

![A screenshot of the IG_OPTS system variable](/img/secure-apps-kerberos-ad-pinggateway/kerb-config-esv.png)

_The System Variable_

Depending on where you’ve saved this file you’ll need to change the filepath to match.

**If you did add this file, you’ll need to restart PingGateway in a new Command Prompt for the variable to take effect.**

## ☑️ Checkpoint - Testing Kerberos Authentication  

To test Kerberos we are going to use the same cdsso route we built before. If you haven’t already, [start PingGateway](https://docs.pingidentity.com/pinggateway/2024.11/getting-started/start-stop.html) **in a new command prompt** and start the Sample Application.

In an incognito tab in VM 2 go to `https://<gateway-url>:8443/home/cdsso/` (mine is `https://dgwiz-vm2.example.com:8443/home/cdsso/`). Instead of being directed to AIC’s login page, you should either log in immediately (or if you’re logged in with a user that’s not part of the domain, or have not configured no-prompt browser settings for Kerberos you should be prompted with a Windows login).  

![A screenshot of the Windows login prompt](/img/secure-apps-kerberos-ad-pinggateway/kerb-login.png)

_Kerberos Authentication_  

>**Note:** since AIC will be issuing the token for our user, we will need a user in AIC whose username matches the user we are logging in with via Kerberos. Make sure you have created a user in AIC with the username you’re testing with.

Upon completing login, you’ll be taken through to your sample application.

# Conclusion

PingGateway enables us to connect local authentication systems such as Kerberos with cloud identity solutions such as PingOne Advanced Identity Cloud. In this guide we:

1. Installed PingGateway  
2. Connected PingGateway to PingOneAIC  
3. Sent authenticated users **to** AIC (Identity Assertion) and retrieved sessions **from** AIC (Cross-Domain Single-Sign-On)  
4. Utilized PingGateway’s built-in Kerberos handler to authenticate a local user with Kerberos and then generate a session within AIC

These building blocks should give you the ability to secure applications with PingGateway and AIC using your existing Windows Domain structure.

# Appendix

## Using System Variables {#using-system-variables}

We use System Variables to store secure information like Secrets and Service Account passwords. Where you want to store this information is ultimately up to you, however if you’d like to do the same, hit the Windows key, type “System Variables”, click the “Edit the system environment variables” result that shows up and then hit the Environment Variables button. You’ll see a section called “System Variables” where you can add new values.

![A screenshot of the search results when searching "System Variables" in Windows](/img/secure-apps-kerberos-ad-pinggateway/windows-esv-search.png)
![A screenshot of the System Properties window with the "Environment Variables" button highlighted](/img/secure-apps-kerberos-ad-pinggateway/windows-esv-select.png)
![A screenshot of the System Variables section with the "New" button highlighted](/img/secure-apps-kerberos-ad-pinggateway/windows-esv-new.png)

_Adding System Variables_

## Base64-Encoding Variables in Windows {#base64-encoding-variables-in-windows}

There are a couple ways to base-64 encode a value using utilities in Windows.

Firstly, you can use the following PowerShell command, subbing in `"my-variable"` with the variable that you’d like to encode.

```
[Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes("my-variable"))
```

Alternatively, if you Bing “Base64 encode” it’ll pull up a utility in-browser.

![A screenshot of the built-in decoder rendered in the Bing Search Page](/img/secure-apps-kerberos-ad-pinggateway/bing-decoder.png)

_Decoding in your Browser in Bing_