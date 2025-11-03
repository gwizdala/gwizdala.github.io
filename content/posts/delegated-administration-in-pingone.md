---
draft: false
date: '2025-01-30'
title: 'Delegated Administration in PingOne'
description: 'Enable your administrators, customers, and partners to manage their data, config, and users within PingOne'
summary: Learn how Ping's Multi-tenant SaaS solution enables Delegated Administration of its platform, services, and users.
categories: ["Ping Identity"]
tags: ["PingOne"]
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

Your business is likely not just one set of users. In today’s world, identity stretches beyond the borders of siloed Internal (Workforce Employee) or External (Customer) user engagement and relationships. And as that line blurs, so does the process of user administration.

PingOne, Ping’s Multi-Tenant SaaS solution, provides a robust means to delegate and manage administrative capabilities across the platform.

This Guide will teach you how you can create and manage delegated administration of Policies and Users within the PingOne Platform. It is broken down into the following parts:

1. [Defining Key PingOne concepts](#definitions)  
2. [Delegating Administration](#delegated-administration)  
3. [Extending Delegated Administration](#extending-delegated-administration)

By the end of this Guide you’ll have configured multiple Environments with their own Applications, Populations, Groups, and Users. Additionally, you’ll have assigned Delegated Administrators to oversee capabilities within these Environments.

This Guide expects a beginner level of familiarity with the PingOne administrative console. If you haven’t set one up yet, be sure to get access to a PingOne instance (register for a trial [here](https://www.pingidentity.com/en/try-ping.html)) so that you can follow along.

But before we get into the details, let’s go over some quick platform definitions.

# Definitions {#definitions}

Delegated Administration in PingOne will primarily interact with **Environments**, **Populations**, **Groups**, and **Admin Roles**.

An [**Environment**](https://docs.pingidentity.com/pingone/settings/p1_environments.html) is a subdivision of your instance of PingOne. It contains the core resources you’ll use to build your IAM services, including your Users, the Applications they’ll single-sign-on (SSO) into, and the Services and Policies you’ll use to secure the access activities of those Users.

A [**Population**](https://docs.pingidentity.com/pingone/directory/p1_populations.html) is a unique administrative set of Users and Groups. Populations contain their own configuration such as their own Password Policies, Branding, default Identity Provider, and/or “alternative identifiers” used to easily look up a Population in things like REST APIs and DaVinci Flows.

A [**Group**](https://docs.pingidentity.com/pingone/directory/p1_groups.html) is a collection of Users. Groups can both be [assigned dynamically and statically](https://docs.pingidentity.com/pingone/directory/p1_groups_vs_populations.html#static-and-dynamic-groups)  and can be [nested inside one another](https://docs.pingidentity.com/pingone/directory/p1_groups_vs_populations.html#nested-groups). Groups are used as a way to assign permissions and membership for Users, including Application access and Admin Roles. 

Groups can be created one of two ways:

1. **External Groups** come from an external connection, either by an Identity Provider (IdP) or LDAP Gateway. They are provisioned either via just-in-time or through synchronization and cannot be modified manually in PingOne.  
2. **Internal Groups** are created manually within the PingOne portal like a Population and are a construct represented entirely in PingOne.

A [**Admin Role**](https://docs.pingidentity.com/pingone/directory/p1_roles.html#built-in-roles-tab) is a collection of permissions that you can assign to Users and Groups. Admin Roles allow Users (and Users within the Groups) to interact with the PingOne platform at a higher level of privilege, such as managing the Users within a particular Population. **Only Users and static Groups can be assigned Admin Roles.**

## When to Create Multiple Environments

A popular use case of Environments is for building a promotion pipeline (e.g. development/staging/production/etc.), but additionally **Environments are subdivided from one another in data and configuration** - meaning that while they share the same hosting resources they do not share the same PingOne Service Selection (e.g. one Environment can have PingOne MFA while another can have PingID for multi-factor), Policy Configuration (e.g. password policies, risk policies, mfa policies, DaVinci policies, etc.), Branding (e.g. theming, forms, localization), and user data (e.g. Populations/Groups/Users and their user information). 

**Environments are great when you have requirements for Users, Applications, and Policies that strongly differ from one another.** Let’s look at an example of managing Employees vs Customers:

| Requirement | Employee | Customer |
| ----- | ----- | ----- |
| **Multi-Factor Authentication** | Desktop and Mobile Authenticator | Passkeys and Biometric |
| **Risk Profiles** | High Risk for New Device | Low Risk for New Device |
| **Localization** | English-Only (US Office) | English, Spanish, and French Support |
| **User Onboarding** | Provisioning from an on-premise Directory | Social Login with Google and Facebook with progressive profiling |

The same can be said with business relationships. Perhaps you have a grouping of Suppliers who interact with your internal services - if so, it’s likely you’ll be securing and provisioning them in a way more similar to your internal (employee) population. On the other hand, if you are working with Distributors that are then interacting with end-customers, those individuals may be purchasing your products more similarly to an external (customer) population. These B2B2E and B2B2C relationships require drastically different interactions, and as such may benefit from separated Environments.

## Populations Vs. Groups

At a first glance, Populations and Groups look kind of similar. However, they differ in some meaningful ways, namely:

1. Users must be part of a Population. Groups do not.  
2. Users and Groups can be a part of ONLY ONE Population. Users and Groups inside a Population can only be a part of Groups in that Population.  
3. Users and Groups can be a part of MORE THAN ONE Group.  
4. Groups can have Admin Roles assigned to them, Populations cannot.  
5. Admin Roles can be scoped to Populations, but not to Groups.

Think of **Populations as a representative set of Users** and **Groups as a categorization of Users**. In the real world:

* A Population could be a Business Partner, while a Group inside that population could be one of their Departments.   
* If you had a shared Application (let’s call it *Partner Hub*) that all of your Business Partners use, a Group that spans multiple Populations could dynamically contain the Users that need access to that app.  
* If you had Support personnel that aided your Business Partners, they could be in a group that has Admin Roles tied to the different Populations.

## Universal vs Segmented Identity

Depending on your business case your users may require a **Universal Identity** or a **Segmented Identity**.

A **Universal Identity** is where **one Identity has one User account for all Services**. This approach is advantageous in tightly-coupled partnerships such as a family of brands, a sports conference/league, a dealership, or a franchise. Universal Identities create seamless login experiences and greatly reduce your identity management footprint.

A **Segmented Identity** is where **one Identity has multiple User accounts for different Services**.This approach is advantageous in distinct relationships such as a supplier, branded enterprise application, an affiliate, or a distributor. Segmented Identities allow finer-grained control and administrative delegation into different levels of secure Applications and Users.

Using Environments, Populations, and Groups, PingOne enables us to create both Universal and Segmented Identities. Consider the following table when setting up PingOne:

| Identity Type / Service Type | Separated Services | Interconnected Services |
| ----- | ----- | ----- |
| **Segmented Identity** | **Different** Environments, Populations, and Groups | **Same** Environment and Group, **Different** Populations |
| **Universal Identity** | **Same** Environment and Population, **Different** Groups | **Same** Environment, Population, and Group |

Given the top-level “unique-set” structure of Environments and Populations, PingOne is best suited for Segmented Identity at scale. Please review the [Standard Platform Limits](https://docs.pingidentity.com/pingone/getting_started_with_pingone/p1_platform_limits.html) to understand what approach fits your implementation and size.

# Delegated Administration {#delegated-administration}

Now that we have a grasp on the different segmentations of identity and configurations within PingOne, let’s set up our instance to take advantage of it.

Our example in this guide will start with an out-of-the-box instance of PingOne. Feel free to work from an existing instance - our work here won’t override any of what you’ve done already.

Your PingOne instance comes pre-loaded with a dedicated **Administrators Environment**. This environment is where your “super-user” account lives and where you’ll add in most of your Administrators. Let’s leave this Environment alone for now while we set up some basic use-cases.

![A screenshot of the PingOne Environments list](/img/delegated-administration-in-pingone/p1-env-list.png)

*Your PingOne Instance and Administrators Environment*

## Example Setup

To start, we are going to configure an example for each of the 4 different Service and Identity scenarios we defined in the prior section. To do so, let’s set up two Environments with some Populations, Groups, and Users in them.

From your Environments View, select the plus “+” icon at the top of the page (or the “Create Environment” button in the “Manage Environments” modal if it’s still on your page).

![A screenshot with arrows pointing to the "Create Environment" buttons](/img/delegated-administration-in-pingone/p1-create-env.png)

*Creating an Environment*

### Creating The Customer Environment

First off, let’s create the Customer Environment. On the next page, select the “Customer solution” option. This will come pre-bundled with a variety of services that are common for CIAM deployments, such as Single-Sign-On, Multi-Factor Authentication (PingOne MFA), Risk Profiling (PingOne Protect), Identity Verification (PingOne Verify), and Orchestration (PingOne DaVinci).

![A screenshot of step 1 in creating an Environment in which the Customer solution has been selected](/img/delegated-administration-in-pingone/p1-env-customer.png)
![A screenshot of step 2 in creating an Environment in which the listed solutions for a Customer prebuild is listed](/img/delegated-administration-in-pingone/p1-env-customer-services.png)  
*The Environment Type and Services*

In Step 3, where you set the deployment options, input the following values and select “Finish”:

| Name | Value |
| ----- | ----- |
| **Environment Name** | Customer Example |
| **Description** | A testing Environment containing sample flows, policies, and users. |
| **Environment Type** | Sandbox |
| **Generate sample populations and users in this environment** | True (checked) |
| **Region** | The closest region to you. I’m using North America (US) |
| **License** | The default license that comes with your instance. |
| **Include a solution designer to easily design and test experiences** | True (checked) |
| **Choose your Industry** | Default |

![A screenshot of step 3 in creating a Customer Environment in which the details provided by the above table are entered](/img/delegated-administration-in-pingone/p1-env-customer-details.png)
*The Environment Deployment Options*

While this Environment is deploying, let’s create the Workforce Environment.

### Creating the Workforce Environment

Head back to the Environments view by clicking the Home icon or your Organization name in the top tab on your page just to the right of the Ping Identity logo. Mine is that `internal_davidgwizdala` text you saw earlier.

![A screenshot of the home icon and organization listing](/img/delegated-administration-in-pingone/home.png)
*Heading Home*

Back at the Environments view, create another new Environment like you did last time but rather than selecting “Customer solution” we’ll be selecting “Workforce solution” instead. You’ll notice that the services deployed will be slightly different, most notably that PingOne MFA has been replaced by PingID and PingOne Verify has been removed.

![A screenshot of step 1 in creating an Environment in which the Workforce solution has been selected](/img/delegated-administration-in-pingone/p1-env-workforce.png)
![A screenshot of step 2 in creating an Environment in which the listed solutions for a Workforce prebuild is listed](/img/delegated-administration-in-pingone/p1-env-workforce-services.png)  
*The Workforce Environment Type and Services*

In Step 3, where you set the deployment options, input the following values and select “Finish”:

| Name | Value |
| ----- | ----- |
| **Environment Name** | Workforce Example |
| **Description** | A testing Workforce Environment containing sample flows, policies, and users. |
| **Environment Type** | Sandbox |
| **Generate sample populations and users in this environment** | True (checked) |
| **Region** | The closest region to you. I’m using North America (US) |
| **License** | The default license that comes with your instance. |
| **Include a solution designer to easily design and test experiences** | True (checked) |
| **Choose your Industry** | Default |

![A screenshot of step 3 in creating a Workforce Environment in which the details provided by the above table are entered](/img/delegated-administration-in-pingone/p1-env-workforce-details.png) 
*Workforce Environment Deployment Options*

Now that we’ve created some example Environments to play with, let’s see what samples we are starting with.

### Reviewing the Setup

After setup is complete for both your Customer and Workforce Environments, you’ll see that the steps you’ve taken have set up a few things for you. Click into each of the Environments to check for the following samples:

1. Under Directory → Users, a series of Users have been added to your Environment.  
2. Under Directory → Groups, two Groups have been added: **Sample Group** and **Another Sample Group**. If you click into the Group Details (by clicking the Group) you’ll see that Sample Group is not tied to any Population while Another Sample Group has been tied to the More Sample Users Population.  
3. Under Directory → Populations, you’ll have been given two Populations: **Sample Users** and **More Sample Users**. These Populations come pre-loaded with their Users and, in the case of the More Sample Users Population, their own Group.  
4. Under Applications → Applications, you’ll see that you have some Applications already created:  
   1. The **Getting Started Application**, which is the example application you saw created in the “Getting Started” tab of your Environment and is what is attached to the Solutions Designer you enabled earlier.  
   2. The **PingOne Admin Console**, which is where your administrators **created in this Environment** will go to manage configuration and identity data for this Environment.  
   3. The **PingOne Application Portal**, which is where your Users can go to access their applications (think of this as an Application Dock)  
   4. The **PingOne DaVinci Connection**, which links the Environment to orchestration in PingOne  
   5. The **PingOne Self-Service - MyAccount**, which is where your Users can go to manage their own profile details, passwords, and MFA devices.  
   6. Under the Workforce Environment, you’ll also have the **PingID Desktop** and **PingID Mobile** applications, which allow your workforce to use the PingID authenticator for Multi-factor authentication. We’ll leave these be for the sake of this example.

With this setup, we have a working demonstration of both how Customers and Employees can interact with PingOne. Now let’s delegate some permissions to it.

Since this guide is focused on Delegated Administration, we’re going to specifically use the **PingOne Admin Console** Application. Copy and save the Admin Console URL from the **Administrators Environment**, either from the Applications page, the Environment Properties (Settings → Environment Properties), or Environment Summary tab (on the Environments List page).

![A screenshot of the home page url found for the Environment's admin console](/img/delegated-administration-in-pingone/homepage-url.png)   
![A screenshot of the console url found under the Environment properties](/img/delegated-administration-in-pingone/console-url.png)   
![A screenshot of the console login url found on the Environment list page](/img/delegated-administration-in-pingone/consolelogin-url.png)   
*Finding the Admin Console URL*

### Creating the Delegated Administrators

Remember the Administrators Environment we mentioned earlier? This Environment is the hub where we’ll be assigning permissions for Users to Environments, Populations, and Applications.

Select the Administrators Environment from your Environments list, copying the Self Service URL on the details tab before entering, and then click on Directory → Users.

![A screenshot of the admin environment self-service url found for the Environment's admin console](/img/delegated-administration-in-pingone/self-service-url.png)   
*Saving the Admin Environment Self-Service URL for Later*

We are going to be creating a couple different users to administer the Environments we created. Click the Plus “+” icon next to the Users header, enter in each of the following User details, and hit “Save” when done. You’ll be creating two users in total.

| Name | User 1 | User 2 |
| ----- | ----- | ----- |
| **Given Name** | Customer | Workforce |
| **Family Name** | Admin | Admin |
| **Username** | customerAdmin | workforceAdmin |
| **Email** | An email you can access | An email you can access |
| **Require Email to be Verified** | True (checked) | True (checked) |
| **Population** | Administrators Population | Administrators Population |
| **Authoritative Identity Provider** | PingOne (default) | PingOne (default) |
| **Password** | Ch4ngeIt! | Ch4ngeIt! |

> **NOTE:** Email Addresses for all admin accounts (i.e. accounts with delegated roles) **must be verified**.

Next, let’s verify these accounts so that we can perform delegated administration. In an incognito, guest, or separate browser window, go to the Self-Service URL for the Admin Environment that you copied earlier. Enter in the username and password of your administrators - you’ll then be presented with a change password prompt. 

![A screenshot of the user flow in which the admin is required to change their password](/img/delegated-administration-in-pingone/admin-change-password.png)   
*The Change Password Prompt*

Enter in the current password and then a new password of your choice. You’ll then be requested to enter in the verification code that was sent to your User when you created their account. 

![A screenshot of the user flow in which the admin is required to enter in their verification code they received via email](/img/delegated-administration-in-pingone/admin-email-verification.png) 
*Entering in the Verification Code*

Repeat the above steps for the Workforce Administrator. When you return to your Administrator Environment and check the Users, you’ll see that the status of their email address is “Verified”.

![A screenshot of the admin page indicating the user's email has been verified](/img/delegated-administration-in-pingone/admin-verified.png) 
*The Verified Admin Email Address*

## Delegating Administration by User

Right now, these Administrators are a part of the Administrators Environment but don’t have any specific permissions. Remember the Console URL we copied earlier?  Paste it into its own separate browser (or guest account) window and try to log in as either of the admins you just created. You’ll encounter an “Incorrect username or password. Please try again.” message.

![A screenshot of the user flow in which the admin fails a login attempt to an environment they don't have permission to](/img/delegated-administration-in-pingone/admin-failed-login.png)   
*Trying to Log In without Permissions*

Now, let’s give these users some permissions. Starting with the Customer Admin, select the “Roles” tab in their user profile and then “Grant Roles”.

![A screenshot of the admin portal where the grant roles option has been highlighted for the customerAdmin user](/img/delegated-administration-in-pingone/grant-user-role.png) 
*Granting Admin Roles to a User*

From here, you’ll see a wide variety of permissions available to assign. Let’s start at the highest level and work our way down.

First, set your Customer Admin to be an Environment Admin of the Customer Example Environment and hit Save. This gives your admin permission to View, Create, Update, and Delete configuration within the Environment. More details can be found in the info “i” icon next to the title, but for all intents and purposes **think of the Environment Admin the manager of Environment Configuration.**

![A screenshot of the admin portal where the Environment Admin permission to the Customer Environment is being granted to the customerAdmin](/img/delegated-administration-in-pingone/grant-user-role-env.png) 
*Adding Environment Permissions*

If you try logging in again, you’ll see that you’re now prompted with MFA enrollment (a default setting for Admin Users). Go ahead and enroll in a device of your choice.

![A screenshot of the user flow where the admin has to enroll an mfa device](/img/delegated-administration-in-pingone/admin-enroll-mfa.png)
*Enrolling MFA as an Admin*

You’ll see that your Customer Administrator has access to configuration details such as Monitoring, Applications, Policies, Roles, and Experience but doesn’t have the capability to see or manage Users directly.

![A screenshot of the admin portal for the Customer environment with the granted roles to the customerAdmin. In this page, the admin can see a Group but can't edit it nor can they see the Users in the Group](/img/delegated-administration-in-pingone/admin-env-view.png) 
*The Environment Admin’s Capabilities*

Most Admin Roles give us the capability to assign by Organization or Environment (see DaVinci Admin, Application Admin), but let’s get a bit more granular. Back in the Administrators Environment view, update the Customer Admin so that they **don’t** have the Environment Administrator Admin Role and **do** have the Identity Data Admin Role specifically for the “Sample Users” Population. To do that, select the Identity Data Admin Role, 1) click the Filter icon (next to the checkbox) and 2) select the Sample Users Population. When you’re done, you should see “Limited Access” next to the Population and then the specific population after saving in the User summary.

![A screenshot of the admin portal where the identity data admin granted permission is filtered to a specific Population. The steps taken to the filter are highlighted.](/img/delegated-administration-in-pingone/grant-user-role-filter.png)   
![A screenshot of the admin portal where the filter has been applied](/img/delegated-administration-in-pingone/grant-user-role-filter-applied.png) 
![A screenshot of the admin portal where the grant has been updated and the summary is shown for the customerAdmin](/img/delegated-administration-in-pingone/grant-user-role-filter-summary.png)   
*Limiting Access to the Sample Users Population*

Now go back to your customerAdmin’s dashboard. After refreshing the page, you’ll see that most of your access has been revoked to Read-Only but now you can manage the Users within the Sample Population.

![A screenshot of the customerAdmin's new permissions to access the Sample Users Population](/img/delegated-administration-in-pingone/admin-view-pop.png) 
![A screenshot of the customerAdmin's view into the users under the Sample Users Population](/img/delegated-administration-in-pingone/admin-view-users.png) 
*Identity Data Admin View*

Using what you just learned, assign the same Admin Role to your Workforce Admin but with their permission pointed to the Sample Users Population in the Workforce Environment. 

## Delegating Administration by Group

So right now we have a User that can manage the Identity Data of one Population in one Environment. Likely, as your organization expands, it’ll become unwieldy (and error-prone) to assign these permissions one at a time. Rather than assigning on a User level, let’s assign to a Group instead.

Inside the Administrators Environment, under Directory → Groups, select the Plus “+” icon and give it the Group Name “Helpdesk”. This group is going to be able to manage all of the identities for both the customer and workforce environments.

![A screenshot of creating the helpdesk Group within the admin portal](/img/delegated-administration-in-pingone/create-group.png) 
*Creating the Group*

This Group is assigned outside of a Population so we can add any Users that are in this Environment to it. With that in mind, go to the Users tab in the Helpdesk group and select “Add Individually”. We mentioned this before, but to protect from accidental overpermissioning **Administrative Groups Permissions CANNOT be assigned via a filter.**

![A screenshot of the admin portal where the "Add Individually" button for adding Users to a Group is being selected](/img/delegated-administration-in-pingone/assign-to-group.png) 
*Adding Individual Users to the Helpdesk Group*

On the next screen, select the customerAdmin and the workforceAdmin Users to this Group.

Next, go to the Roles Tab and select "Grant Roles". This screen will look very familiar to you: it’s the same screen we used when assigning Admin Roles to Users!

This time, let’s give this Group the Identity Data Admin Role across both the Customer Example and Workforce Example Environments.

![A screenshot of the admin portal where the Group is being assigned Identity Data Admin roles](/img/delegated-administration-in-pingone/group-roles.png)  
*Assigning the Identity Data Admin Roles*

When you hit Save, and go back to your customerAdmin or workforceAdmin User, you’ll see that their permissions have been “Granted By Group” to the two Environments.

![A screenshot of the admin portal where the admins assigned to the Group have had updated permissions](/img/delegated-administration-in-pingone/updated-permissions.png) 
*The User’s Assigned Admin Roles, Direct and Group*

> You probably also saw an error message pop up - we’ve just tried to assign the same Admin Role twice to our Admin Users. The highest amount of permission will win here: in this case, the Helpdesk Identity Data Admin Role that grants access to all Users in both Environments.

Go ahead and refresh the admin portal for either your customerAdmin or your workforceAdmin. Not only will you see all 40 users in your current Environment, you’ll additionally be able to navigate into and see the other Environment too.

![A screenshot of the customerAdmin being able to see all 40 users from all populations](/img/delegated-administration-in-pingone/admin-view-all-users.png) 
![A screenshot of the customerAdmin being able to see multiple environments](/img/delegated-administration-in-pingone/admin-view-environments.png)  
*The Assigned Group Permissions to the End User*

## Putting it to Practice

So now you know how to assign Admin Roles to a User and to a Group - let’s try some common combinations that will help you and your Users best interact with the platform. If you want a challenge, try to assign the right roles before reading which options you need.

{{<details title="Managing Data about Users">}}
- Identity Data Admin
{{</details>}}
{{<details title="Creating New Applications and Managing Specific Applications">}}
- Client App Developer
- Application Admin
{{</details>}}
{{<details title="Defining Authentication Policies and Experiences">}}
- Environment Admin
- DaVinci Admin
{{</details>}}
{{<details title="User Experience Tester ">}}
- DaVinci Admin
- Identity Data Admin (you need a test user to test a flow!)
{{</details>}}

# Extending Delegated Administration {#extending-delegated-administration}

We’ve barely scratched the surface for what Delegated Administration can do within PingOne. To give you some jumping off points, take a look at common ways the Delegated Administration model is extended within the platform.

## In-Environment Admins (Partner Admins)

Remember when we first looked at the Administrators Environment and said that **most** delegated administrators would be in this environment? Let’s talk through the scenarios in which you may want to **assign delegated administration in a non-admin Environment** instead. Think of this section as the **Universal Identity** vs. Segmented Identity (which we did in the last section) scenario.

1. The delegated administrator utilizes the same policies (e.g. password, MFA, DaVinci, Risk) as the environment or identities that they manage (e.g. service user)  
2. The delegated administrator requires a distinct login experience (e.g. UI/Branding/Flow) indicative of the partnership or brand they are coming from (e.g. branding director)  
3. The delegated administrator accesses the same Applications that their users are managing, perhaps with a different authorization policy (e.g. Application manager)  
4. The delegated administrator Federates from the same IdP as their users (e.g. external helpdesk, business owner)

In these cases (and more!) ensure that your delegated administrators are Grouped in meaningful ways - the more that the Admin Roles are defined at a Group level the easier it will be to track what permissions each of your Users have.

## Custom Admin Roles

While PingOne provides a series of Admin Roles out of the box for you and your administrators to utilize, you can create more catered Admin Role permissions through the use of [Custom Admin Roles](https://docs.pingidentity.com/pingone/directory/p1_custom_role_add.html). These allow you to set CRUD-based permissions on the same set of Administrative Roles we were working with in the other section.

Since the Documentation contains [Custom Admin Role Scenarios](https://docs.pingidentity.com/pingone/directory/p1_custom_roles_scenarios_intro.html) for you to look at, I’m not going to provide a walkthrough here. That being said, combining Custom Admin Roles with your Groups and Users gives you the ultimate level of control across your Environments.

# Conclusion

Through this Guide we have:

1. Learned about the key definitions and differences between structures within PingOne  
2. Created our own unique example Environments containing these structures  
3. Delegated administration to external users to manage the different structures.

Additionally, we have looked at ways that Ping extends this capability through the user of assigning Admin Roles within non-Admin Environments (“Partner Admins”) and through the creation of Custom Admin Roles.

With what you have learned today you should be able to confidently create and assign complex Admin Role relationships for your internal and external administration.