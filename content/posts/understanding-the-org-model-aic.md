---
draft: false
date: '2025-02-07'
title: 'Understanding the Org Model'
description: A high-level overview of AIC and PingIDM's Organizations
summary: Learn how AIC and PingIDM's Organizations enable out-of-the-box extensible, inherited relationships between users and associated groups of users
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "PingIDM"]
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

PingOne Advanced Identity Cloud (AIC) and PingIDM (Ping’s self-managed Identity Management) provide a powerful model called Organizations which enables out-of-the-box extensible, inherited relationships between users and associated groups of users. In short, Organizations give you the ability to match your IAM strategy to your business strategy, and not the other way around - they are a means to unify, simplify, and standardize your process across your internal, external, and partner identities. 

Considering its flexibility and scalability, starting with an empty tenant and the base Organization model can feel a bit daunting.

This Guide intends to teach you the key information you need to know when interacting with AIC/PingIDM’s Organization model. It is split into the following sections:

1. [Object Modeling Definitions](#object-modeling-definitions)  
2. [What is the Organization Model](#what-is-the-organization-model)  
3. [Utilizing the Organization Model](#utilizing-the-organization-model)  
4. [Referencing the Organization Model in the Platform](#referencing-the-organization-model-in-the-platform)  
5. [Extending the Organization Model](#extending-the-organization-model)

By the end of this Guide you’ll have a medium-level understanding of Organizations and how they operate generically within AIC and PingIDM.

This Guide expects a beginner level of familiarity with AIC or PingIDM. Certain topics covered here will be specific to the AIC platform: when this happens, we’ll call it out. If you do have a tenant or deployment available, feel free to follow along however it is not necessary.

# Object Modeling Definitions

Before we define Organizations, we need to understand the underlying infrastructure that surrounds it. Let’s start broadly and then narrow down as we go.

The Organization model is a part of **Identity Management (IDM)**. PingIDM is the self-deployed instance and is a core feature set within AIC. Identity Management’s job is to represent, control, connect, and map identity data in and out of the Ping platform. Regarding Organizations, we care most about *representing* and *controlling* its data - what does the template look like, and ensuring that the data within it is managed appropriately (think CRUDPAQ - Create, Read, Update, Delete, Patch, Action, Query).

[**Managed Objects**](https://docs.pingidentity.com/pingoneaic/latest/idm-objects/appendix-managed-objects.html) are a templatized instance of data in IDM that contains **Properties**. In essence, they are the representation of identities within IDM. By default, your instance comes with Users, Roles, Assignments, Groups, and Organizations, but you can create as many as you’d like. Managed Objects store data within a Directory, with each Managed Object being categorized separately.

[**Properties**](https://docs.pingidentity.com/pingoneaic/latest/identities/user-identity-properties-attributes-reference.html) (sometimes called Attributes) are the individual data fields within a Managed Object that represent the data types stored within that object. They define the specific pieces of information each instance of that Managed Object will hold. As an example, a User is a Managed Object, and their Username is one of their Properties. Properties can be booleans, numbers, arrays (called *multivalues*), objects (including nested objects), strings, **Relationships**, and **Virtual Properties**. You can also [create custom types](https://docs.pingidentity.com/pingoneaic/latest/idm-objects/creating-modifying-managed-objects.html#) as needed. Properties have their own unique indexing[^1], validation, hashing or encryption, and triggers which can be performed on actions such as validating, retrieving, or storing an instance of this Property.

[**Relationships**](https://docs.pingidentity.com/pingoneaic/latest/idm-objects/relationships-custom.html) are a type of Property that creates a linkage between instances of Managed Objects. Relationships can be one-to-one, one-to-many, many-to-one, and many-to-many, and can be linked to different Managed Objects or the same Managed Object. Common examples of relationships include Manager to Employees, Parent to Children, or Department to Teams. Relationships can contain their own data within the link itself, can return their data alongside either end of the link in a query, and can be used as a means to **notify** the connected Managed Objects that changes have occurred on opposite ends of the link (**Relationship-Derived Virtual Property**).

[**Virtual Properties**](https://docs.pingidentity.com/pingoneaic/latest/idm-objects/managed-object-virtual-properties.html) are a powerful Property type that can dynamically calculate its own data on the fly, either on time of retrieval or based on notifications received from a Relationship tied to this Managed Object (**Relationship-Derived Virtual Property**). This significantly simplifies data and queries as well as improves performance because the complex calculations are handled for you dynamically at run time or statically at storage time - for example, you could have a virtual property that dynamically returns the “Full Name” of a User as a combination of their First and Last name without actually having to store and update that value separately.

[**Relationship-Derived Virtual Properties (RDVPs)**](https://docs.pingidentity.com/pingoneaic/latest/idm-objects/managed-object-virtual-properties.html#relationship-derived-virtual-properties) are a type of Virtual Property that update based on notifications from the Relationships linked to its Managed Object. An RDVP has access to content from the Object’s Relationships and can travel the chain of relationships to create a composite value of the chain. Relationships and RDVPs are the two components of IDM that make Organizations so flexible and capable. 

## What’s the Deal with RDVPs?

RDVPs are an insanely powerful tool for reducing complexity and increasing performance.

As an example, let’s say that you have a business structure of Managers to Employees like we mentioned in the Relationships section. In most companies, you’re not going to have one layer of Managers - you’ll probably have at least 3 and each of those layers may have tens to hundreds of Managers who then manage their own number of Employees.

If I wanted to get a list of all of the Managers above an Employee, how would I go about it? Normally, it would likely be a series of queries that would look something like this:

1. Get the Employee  
2. Find the Employee’s Manager  
3. Get the Manager  
4. Find the Manager’s Manager  
5. Get the Manager’s Manager  
6. Find the Manager’s Manager’s Manager  
7. …  
8. Manager X has no Manager, return

Think about this the other way, too. If I wanted to get all of the Employees *under* a Manager, and each Manager has more than one Employee, how many operations do I have to take? Now what if I had to perform this action every time I entered an application? This is starting to sound a lot like one of those [classic graph traversal interview questions](https://www.geeksforgeeks.org/top-50-graph-coding-problems-for-interviews/)!

The deeper or wider you go, with the more connections you have, the more operations you would normally have to take. This makes your code complex and significantly increases the number of queries you need to make to get the data you and your users need to be successful. **RDVPs solve this problem.**

**RDVPs travel the Relationships for you and update a static result on the object directly at time of change from anywhere in the Relationship.**

If we looked at the previous example, if I wanted to get a list of the Employee’s Managers, or a list of the Manager’s Employees, **the data is on the User directly**. If the Employee is promoted, or switches teams, and the managerial hierarchy changes, **the data on the User updates automatically**.

**RDVPs proactively updates relevant aggregate data so that you only have to look at one Object to understand what’s going on with everything it’s linked to.**

# What is the Organization Model?

[**Organizations**](https://docs.pingidentity.com/pingoneaic/latest/identities/organizations.html) are a **Managed Object** that contain multiple **Relationships** both to Users and to other Organizations. These Relationships store **RDVPs** that define the linkages that the Organization has with other Organizations and Users. Since Organizations are a Managed Object, they are adherent to the same scale as Users, meaning that AIC is rated for hundreds of millions of Organizations while PingIDM is hypothetically as scalable as your hardware will allow.

Organizations, at their core, build a **hierarchy** of templatized data. Due to the Relationships between Organizations and Users, this data can be **inherited** up and down the hierarchy.

Visualized, the Organization model often looks like a tree.

![A hierarchy diagram providing an example of the org model](/img/understanding-the-org-model-aic/org-model-basic.png)
*The Organization Model*

**Organizations enable hierarchical, inherited data templates**, allowing you to **standardize and represent complex structures** such as families, business partnerships, and supplier/distributor models.

But just having this data is only one part of what makes this model special. Organizations don’t just represent Parent/Child relationships between other Organizations: they also by default contain 3 different Relationship types with Users - and Users can retrieve and inherit information from the hierarchy as well. Due to the RDVPs already on the Organization, and the permission model in place on the Organization, elevated Relationship types (i.e. Owners and Admins) have the ability to manage aspects of both the Organization, its related Users, and any Organizations and Users that they’ve inherited - both via REST API and (in the case of AIC) hosted UI. 

![A hierarchy diagram providing an example of the org model with some users tied to it. Bubbles highlight the users to showcase their permissions to the organizations they're tied to](/img/understanding-the-org-model-aic/org-model-basic-users.png) 
*The Organization Model with Users*

**Organizations enable hierarchical, inherited delegated administration**, allowing you to **scale administration alongside your platform** by delegating operational management of the hierarchy and its Users to the Users within the Organization (or its Parent).

Additionally, these Relationship types are **many to many**, meaning that a single User can be a Member, Admin, and Owner of as many different Organizations and Hierarchies as they need.

![A hierarchy diagram providing an example of the org model where a user is linked to more than one organization hierarchy](/img/understanding-the-org-model-aic/org-model-universal-identity.png)  
*Universal Identity in Organizations*

**Organizations enable a Universal Identity**, where **one Identity has one User account for all Services**. 

So, why Organizations?

Organizations let you:

1. Templatize grouping of content so that you can build a standard that the platform adheres to  
2. Delegate the content so that the platform can be managed at scale without having to hire thousands of admins to keep up  
3. Link the content so that it and its users can inherit information and administration for quicker time-to-value

## A Quick Analogy

Think of the Organization Model as a Condominium.

As the Tenant Administrator, you’re the owner of the Building itself. You dictate what services are available, how the condos are laid out, and the keycard system for how the residents get into the building. You can also add floors and rooms to the Condo at any time.

The people in your Condo (Users) could be Residents (Admins) or Staff (Owners). Everyone must use their keycard to get into the building, and that same keycard may give them access to different rooms or Condos in the building. A Resident, for example, may have access to their Condo as well as to a friend’s Condo. Staff may have access to all Condos on a given floor, and may have access to systems like plumbing and electrical which the Residents don’t interact with.

Residents can modify their Condo, but how they can change it must adhere to the rules and regulations defined by the Building. To the benefit of the Resident, ongoing maintenance of the Condo and its services are managed by the Building and not by them.

If the Resident decides to rent out their Condo, their Tenants (Members) will have access to the Building and to the Condo but won’t be able to modify the Condo at all. They can only use the facilities given to them by the Resident.

As your Building grows, you may need more staff to run it. As such, you can appoint Floor Managers (Owners of a Parent Org) who also have the ability to create new Condos and manage the Floors,  Residents, and Tenants within. 

Additionally, some of your Maintenance Staff and Floor Managers live in your Building. That means they are also Residents, and have different capabilities and responsibilities depending on what Condo they’re currently in.

# Utilizing the Organization Model

So let’s put this all together and see how the Organization Model works in practice. Note that the platform screens shown below are taken from the built-in Hosted Pages within AIC, however each and every action is available in PingIDM via the Native UI or REST API.

Our goal today will be to create a basic business hierarchy. We’ll take the Supplier/Distributor model for this example.

![A hierarchy diagram providing an example of the org model related to a supplier-distributor model. Suppliers and distributors are located below BX Manufacturing, and Supplier A and B are underneath Suppliers](/img/understanding-the-org-model-aic/org-model-sd.png)  
*Supplier/Distributor Example*

In our example we have a Manufacturing vertical named **BX Manufacturing**. BX Manufacturing has partnerships with **Suppliers**, who provide goods and services, and **Distributors**, who purchase BX Manufacturing’s products to then be sold elsewhere. Both Suppliers and Distributors need to give their employees access to BX Manufacturing’s systems but what they ultimately do with those systems is very different.

To start, let’s create the Root Organization. As the Tenant Admin, we can select “New Organization”, enter in the Name, and then we’re ready to go.

![A screenshot of the managed identities page in which the new organization button is highlighted](/img/understanding-the-org-model-aic/create-org.png)
![A screenshot of the managed identities page in which the the name BX Manufacturing is entered](/img/understanding-the-org-model-aic/create-org-name.png)
![A screenshot of the managed identities page in which the new org BX Manufacturing is selected and created](/img/understanding-the-org-model-aic/create-org-page.png)  
*Creating the Root Organization*

Now, an Organization is only as useful as its Relationships. With that in mind, let’s assign an Owner to this Organization so that we can delegate management of the Manufacturing Vertical and its subdivisions.

Underneath the “Owner” section, we can assign a User to our Organization. In my example, the user “Owner” has already been registered. Note that while we are assigning manually here, you can always automate this assignment during provisioning, user registration (likely via a Journey/Tree in AIC/PingAM), or (if you have Governance features in your AIC tenant) during Access Request or with a Form submission that triggers a Workflow or Approval.

![A screenshot of the BX Manufacturing Org with the Add Owner button highlighted](/img/understanding-the-org-model-aic/add-owner.png)
![A screenshot of the BX Manufacturing Org with the owner "owner" being assigned](/img/understanding-the-org-model-aic/add-owner-assign.png)
![A screenshot of the BX Manufacturing Org with the "owner" user assigned as owner](/img/understanding-the-org-model-aic/add-owner-assigned.png) 
*Assigning the Owner*

Let’s log in as the Owner User. You’ll find that you have access to update and delete BX Manufacturing and its users and can create new Organizations underneath the Organization you’re an Owner of.

![A screenshot of the owner logged in and viewing the Org BX Manufacturing](/img/understanding-the-org-model-aic/owner-permission.png)
![A screenshot of the owner logged in and viewing the Org BX Manufacturing's details](/img/understanding-the-org-model-aic/owner-permission-details.png) 
*Default Owner Permissions*

While you have very similar capabilities to the Tenant Administrator by default, you won’t have access to manage other Owners - that’s the job of the Tenant Administrator.

![A screenshot of the owner logged in and viewing the Org BX Manufacturing in which they cannot see or edit other owners](/img/understanding-the-org-model-aic/owner-permission-owners.png)
*Default Owner Permissions - No Owner Management*

Since we’ve been delegated ownership of BX Manufacturing, let’s set up the Suppliers and Distributors suborganizations and assign some people to manage them. We’ll create a new Organization and assign that Organization to BX Manufacturing as a Parent. 

![A screenshot of the owner selecting the new organization button](/img/understanding-the-org-model-aic/owner-create-org.png)
![A screenshot of the owner entering the name "Suppliers" and associating it to the parent "BX Manufacturing"](/img/understanding-the-org-model-aic/owner-create-org-assign.png)
![A screenshot of the owner viewing the Suppliers org they created](/img/understanding-the-org-model-aic/owner-create-org-result.png) 
*Creating the Sub-Organizations (Children)*

We’ll repeat the same action for the Distributors Organization.

![A screenshot of the owner viewing the organizations they have access to - BX Manufacturing, Suppliers, and Distributors](/img/understanding-the-org-model-aic/owner-create-orgs.png) 
*Creating the First-Level Children*

Currently, our Organizations don’t have any Users in them. Let’s fix that. As an Owner, we can create a new User and assign them as a Member and/or an Administrator to any and all Organizations that we own and have inherited ownership to.

![A screenshot of the owner selecting the new user button](/img/understanding-the-org-model-aic/owner-create-member.png)
![A screenshot of the owner adding in the user details and assigning that user to an org as a member](/img/understanding-the-org-model-aic/owner-create-member-assign.png)
![A screenshot of the owner assigning the new user to be an admin to an organization by selecting the "Add Organizations I Administer" button](/img/understanding-the-org-model-aic/owner-create-admin.png)
![A screenshot of the owner assigning the user to be an admin of the Suppliers org](/img/understanding-the-org-model-aic/owner-create-admin-assign.png)  
*Creating the Suppliers Administrator*

If we log out as an Owner, and log in as the Suppliers Admin, we’ll see that we only have access to the Organization that we were assigned, and unlike the Owner can’t manage other Administrators of our Organization(s).

![A screenshot of the administrator's permissions to the Suppliers org only](/img/understanding-the-org-model-aic/admin-permission.png)
![A screenshot of the administrator's permissions to the Suppliers org where they cannot add or edit admins](/img/understanding-the-org-model-aic/admin-permission-admins.png) 
*Default Administrator Permissions*

We *can*, however, manage our Users and create and manage new Organizations. Let’s create Supplier A and Supplier B now as children of Suppliers.

![A screenshot of the administrator's portal in which they have created Supplier A and Supplier B and assigned them to the Suppliers Org as children](/img/understanding-the-org-model-aic/admin-create-orgs.png) 
![A screenshot of the administrator's view of Supplier B in which the org has been assigned](/img/understanding-the-org-model-aic/admin-create-orgs-assign.png)
*Creating the Suppliers*

Additionally, let’s create some Users who are Members of the different Suppliers. 

![A screenshot of the administrator's user view in which the Supplier A Member and Supplier B Member have been created](/img/understanding-the-org-model-aic/admin-create-users.png)
![A screenshot of the administrator's view into Supplier A where they have assigned Supplier A Member](/img/understanding-the-org-model-aic/admin-assign-member-a.png) 
*Assigning Members*

Log back in as the Owner. You’ll see that you have inherited permissions to the Suppliers and the Users made by your delegated Administrator.

![A screenshot of the owners view in which they now have visibility into the newly created orgs](/img/understanding-the-org-model-aic/owner-view-tree.png)
![A screenshot of the owners view in which they now have visibility into the newly created users underneath the orgs they now manage](/img/understanding-the-org-model-aic/owner-view-tree-members.png)  
*Inheritance at Work*

**Owners and Admins** **inherit permission** to all of their sub-Organizations and its Users. **Members inherit context** from all of their Parent Organizations. **Owners and Admins permission goes down the tree, Members context goes up.**

We mentioned before that Users can have more than one Relationship to one or more Organization. Let’s say, for example, that Supplier B interacts with Distributor A to sell OEM add-ons to a customer, like a spoiler on a car. Since BX Manufacturing is the creator and owner of the finished product, they must be intrinsically involved in the linkage between this Supplier and Distributor. Here we’re dealing with a **B2B2C** scenario, or Business-to-Business-to-Consumer.

![An example diagram in which one identity is being used across multiple organizations in different capacities](/img/understanding-the-org-model-aic/unified-identity.png)  
*Unified Identity*

In this situation, our User is an Admin to Supplier B and is also a Member to Distributor A. At the end of the day, this is the same person with the same account and the same login but with different relationships to each Organization. One person is tied to one identity, but the Relationships that they have to the Organizations give them different permissions to Organizations and Users and give others different permissions to them.

In practice, as the user Supplier B Admin I can see Supplier B and the Members associated with Supplier B. **As Distributor A Admin, I can see and manage Supplier B Admin because they’re a Member of my Organization.**

My Supplier received the context they needed to interact with Distributor while still being able to manage their own Users and Supplier information. My Distributor cannot receive the context of the Supplier but can manage the suppliers associated with their Organization.

![A screenshot of the supplier B admin's view in which they can access Supplier B's organization](/img/understanding-the-org-model-aic/supplier-admin-org.png)
![A screenshot of the supplier B admin's view in which they can access Supplier B's members](/img/understanding-the-org-model-aic/supplier-admin-members.png)
![A screenshot of the distributor A admin's view in which they can access Distributor A's organization](/img/understanding-the-org-model-aic/distributor-admin-org.png)
![A screenshot of the distributor A admin's view in which they can access Distributor A's members, which includes Supplier B Admin](/img/understanding-the-org-model-aic/distributor-admin-members.png)  
*B2B2C Using Organizations*

# Referencing the Organization Model in the Platform

Advanced Identity Cloud takes the Organization Model one step further and incorporates it throughout its platform. That way, actions and interactions can reference the context and relationships to make unique experiences and define specific permissions defined by each individual user.

To start, the [openidm binding](https://docs.pingidentity.com/pingoneaic/latest/idm-scripting/scripting-func-engine.html) allows you to access and manage any IDM data throughout AIC. This gives you the flexibility to interact with your Organization throughout the User journey, for example:

* An Event Hook that auto-assigns Organization membership based on a Property or Group they’re a member of (for example, a specific AD OU)  
* A Custom Endpoint that returns Relationship information for decentralized Policy Enforcement  
* User Journeys where things like branding, MFA, and risk policies are dictated by the configuration on the Organization  
* Access Token Modification where the token includes context of what Organizational privileges the User has

To give you a sense of what actions are useful in the context of Organizations, take a look at this [Library Script](https://github.com/gwizdala/lib-ping/blob/main/Library%20Scripts/library_org.js) that you can use wherever Next-Gen scripting is available.

If you have Governance enabled within your AIC Tenant, Organizations appear in the following places:

* Owners and Admins can request access to Applications, Entitlements, and Roles on behalf of their Members.  
* Identity Certification Campaigns can be filtered to Users in specific Organizations  
* Scopes can be filtered to specific Organizations and Relationships (e.g. Owner/Admin/Member)  
* Forms allow a dynamically enumerated list of Organizations as a Multi-select option

Just like with the rest of the platform, you can utilize the same bindings to interact with the Organization in actions such as Workflows.

# Extending the Organization Model

So where do we go from here? 

Organizations give us a template with pre-built relationships that can be referenced throughout Ping’s platform. And that’s just it: it’s a **template** upon which we can extend and expand to meet the needs of our organization.

As a start, add Properties to your Organization - things you see your business units and partnerships doing over and over again that could (and should) be standardized.

Once you’ve built that model the way you like it, connect the model to your platform. It could be [a way to control branding]({{< ref "dynamically-branding-journeys-in-aic" >}}), [extended support capabilities]({{< ref "managing-mfa-in-journeys" >}}), or something completely different - work with your stakeholders to decide what is important for them and their users to **have** and to **manage**.

When you’ve got the model in a place you like, [delegate administration]({{< ref "setting-delegated-admin-privileges-in-aic" >}}) to the capabilities you’d like to share with your partnerships. You own the model, and you delegate the controls.

If you see that the parent organizations need more control than the children, consider [applying inheritance rules]({{< ref "modeling-inheritance-in-aic-or-idm" >}}). 

And, as you expand, keep changing it! The beauty of this template is that it grows with you: you can enable the right experience, security, lifecycle, and overall IAM strategy for each and every use case without having to rebuild your entire platform every time a new business requirement comes along.

# Conclusion

Through this Guide we have:

1. Learned about the key elements of IDM that make up the Organization model  
2. Learned how the Organization model applies these concept  
3. Created, delegated, and managed Organizations and Users  
4. Learned how to connect Organizations throughout AIC  
5. Learned about common approaches in extending the Organization model to meet our business needs

The Organization model provides you the bricks to build dynamic, magical experiences for your Administrators and your Users. And with this foundation the only limit you have is your imagination.

[^1]:  Note that AIC provides [general purpose extension attributes](https://docs.pingidentity.com/pingoneaic/latest/identities/identity-cloud-identity-schema.html#use-general-purpose-extension-attributes) for cases in which you need to index additional values as a means to protect tenant response time. This is not a limitation in PingIDM.