---
draft: false
date: '2024-12-12'
title: 'Setting Organization IDM Delegated Administrative Privileges in PingOne Advanced Identity Cloud'
description: 'Define Fine-Grained Self-Service for your Delegated Administration, their Organizations, and their Users'
summary: Learn how what Delegated Administration for Organizations means in AIC and how to create your own Organization IDM Delegated Administrative Privileges
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

**Delegated Administration** of Identity and Identity Metadata is a key feature of PingOne Advanced Identity Cloud (AIC)’s Organization structure. It allows end-users to self-manage key aspects of themselves, their users, and the metadata within the group that they’re managing. As a Tenant Administrator, you have control over what capabilities are templatized on every identity type (i.e. Users/Orgs) as well as what level of control each type of delegated user has to those capabilities.

This How-To will teach you what Delegated Administration means in AIC and how to create your own Administrative permissions. It expects an existing understanding of [REST APIs](https://docs.pingidentity.com/pingoneaic/latest/developer-docs/crest/about-crest.html), AIC’s [Organization Model](https://docs.pingidentity.com/pingoneaic/latest/identities/organizations.html), and (optionally) how to use [FroDo CLI](https://gwizkid.com/posts/a-love-letter-to-frodo-cli/).

# Understanding the Delegated Administration Model

By default, an Organization will have 3 delegated relationship types: **Members, Admins**, and **Owners**. Members have the least amount of privilege while Owners have the most. As a Tenant Administrator, you have the highest level of permissions over and above the 3 delegated relationships.

Users within your tenant can be assigned more than one delegated relationship across more than one Organization or Organization hierarchy (tree). Users can also be assigned more than one delegated relationship type. Some good examples of why you may want to do this include:

* A **Medical Provider** (Admin) who also is a **Patient** (Member)  
* A **Customer Success Manager** (Admin) who supports multiple business units or business partners (Multiple Trees)  
* A **Fan** (Member) of multiple Teams in a Franchise 

**Organization membership is** **inherited**, meaning that if you assign a user as an Owner, Admin, or Member of a parent Organization they will also have that permission to the Organization’s children.

> Note that delegated administration in this case is specifically Identity Management (IDM)-focused: meaning that we are managing metadata on Users and Organizations, not delegating Tenant Admin functionality like IGA, Applications, Journeys, etc.

## Delegated Administration Defaults

The default Organization model provides a foundation to build additional templated functionality into your IAM strategy. As such, the default permissions for each delegated administration type are focused on creating the Organization and then managing the individuals associated with it.

{{< md-table >}}
{{< md-tr >}}
{{< md-td colspan="2" >}}

**🔴 Tenant Administrator**

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td colspan="2" >}}

Able to manage all aspects of AIC.

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td colspan="2" >}}

**🟢 Owner**

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td colspan="2" >}}

Able to manipulate all organizations, members, and admins in their “ownership area” (i.e. any part of the tree in or beneath the organization they own).

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td >}}

**Can Do**

{{< /md-td >}}
{{< md-td >}}

**Can't Do**

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td >}}

* Add and update members
* Add and update sub-organizations
* Make an org member an admin for the parent org or any sub-org

{{< /md-td >}}
{{< md-td >}}

* Create additional owners

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td colspan="2" >}}

**🔵 Administrator**

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td colspan="2" >}}

Able to manipulate all organizations, members, and admins in their “administrative area” (i.e. any part of the tree in or beneath the organization they administer).

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td >}}

**Can Do**

{{< /md-td >}}
{{< md-td >}}

**Can't Do**

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td >}}

* Add and update members
* Add and update sub-organizations

{{< /md-td >}}
{{< md-td >}}

* Create admins
* Create, view, or edit owners

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td colspan="2" >}}

**⚪️ Member**

{{< /md-td >}}
{{< /md-tr >}}
{{< md-tr >}}
{{< md-td colspan="2" >}}

No special privileges.

{{< /md-td >}}
{{< /md-tr >}}
{{< /md-table >}}

Think of the default permissions like a series of nested circles with Members in the center. As you traverse wider out in the circle, your permissions increase. An example diagram of what this might look like with additional features on your Organization is below.

![A diagram showing cocentric circles of increasing permissions. The permissions match the descriptions in the provided table.](../images/setting-delegated-admin-privileges-in-aic/example-relationship-mapping.png)

*Example Relationship Mapping for Default Organization Modeling User Types*

## Where the Privileges are Managed

Delegated administration for organizations is represented by a relationship between the Organization and the User. The permissions on these relationships are in two JSON configuration files stored within the Identity Manager (IDM) of AIC, called `alphaOrgPrivileges` and `privilegeAssigments`. `alphaOrgPrivileges` and `privilegeAssigments` are the files we are going to be looking at and modifying within this How-To.

# Managing Delegated Admin Privileges

As your business grows your Organization model and the ways your users interact with the Organization will grow alongside it. We need to update our delegated administrator’s permissions to match the capabilities we need each relationship type to be able to do. 

Example calls will be provided both as a cURL and FroDo command. If you are using cURL, you will need to generate an Access Token using a service account with the `fr:idm:*` scope.

## Retrieving the Privileges

### Retrieving the Organization Privileges

Retrieve your `alphaOrgPrivileges` using one of the following options. Since we will be updating your Organization privileges, **save a copy of the output in case you’d like to revert your changes later**.

**Option 1: REST**

```bash
curl --location 'https://{tenant-domain}/openidm/config/alphaOrgPrivileges' \
--header 'Authorization: Bearer ****' -o original_alphaOrgPrivileges.json
```

**Option 2: FroDo**

```bash
frodo idm export -f original_alphaOrgPrivileges.json -i alphaOrgPrivileges <tenant>
```

### (Optional) Retrieving the Privilege Assignments

If you are looking to create additional permissions you’ll need to pull down the `privilegeAssignments.json` configuration, which maps the permissions in the privilege file to the appropriate delegated administration type. As before, **save a copy of the output in case you’d like to revert your changes later**.

**Option 1: REST**

```bash
curl --location 'https://{tenant-domain}/openidm/config/privilegeAssignments' \
--header 'Authorization: Bearer ****' -o original_privilegeAssignments.json
```

**Option 2: FroDo**

```bash
frodo idm export -f original_privilegeAssignments.json -i privilegeAssignments <tenant>
```

## Understanding the Privileges

Open the `original_alphaOrgPrivileges.json` file in a text editor of your choosing. You’ll find a JSON object with two main keys: the `_id` telling you what config file you’re looking at (should be `alphaOrgPrivileges` if you pulled the right file) and a `privileges` array that holds all of the capabilities for each delegation type.

Let’s take a look at a single privilege to understand how this works. I’ve stripped out the details so that we can look at the keys specifically.

```json
{
    "accessFlags": [
        {
            "attribute": "",
            "readOnly": true
        }
    ],
    "actions": [],
    "filter": "queryfilter",
    "name": "descriptive-permission-name",
    "path": "managed/path",
    "permissions": [
        "CREATE",
        "VIEW",
        "UPDATE",
        "DELETE",
        "ACTION"
    ]
}
```

| Key | Value |
| :---- | :---- |
| `accessFlags` | The list of attributes covered by this privilege. If an attribute is not listed, it is not a part of the privilege. If `readOnly` is set to `true`, the attribute can be viewed but not edited. |
| `actions` | An empty array (`[]`). This is a standard object applicable to [Inner Role IDM privileges](https://backstage.forgerock.com/docs/idm/7.5/auth-guide/delegated-admin.html#creating-privileges). |
| `filter` | The [Query Filter](https://docs.pingidentity.com/pingoneaic/latest/developer-docs/crest/query.html#crest-query-queryFilter) used to specify what subset of identities this permission applies to. Dynamic filters can take any value from the authenticated user using handlebars or the special `__org_id_placeholder__` placeholder value which substitutes in the ID of the specific Organization. |
| `name` | The name of the privilege, mapped to the `privilegeAssignments.json` configuration file in your tenant. The defaults for these privilege assignments can be found in the [Default Privilege Assignments](#default-privilege-assignments) section.  |
| `path` | The managed identity this permission applies to. This is either `managed/alpha_organization` or `managed/alpha_user`. |
| `permissions` | A list of specific operations that can be performed by the user with this permission. Can be `CREATE`, `VIEW`, `UPDATE`, and/or `DELETE`. |

### Default Privilege Assignments {#default-privilege-assignments}

The `privilegeAssignments.json` configuration file comes with a series of default privilege assignments which map to the `name` attribute in the privileges of your `alphaOrgPrivileges` file.

Default Owner Privileges (an Owner being defined by the `ownerOfOrg` relationship field):

* `owner-view-update-delete-orgs`  
* `owner-create-orgs`  
* `owner-view-update-delete-admins-and-members`  
* `owner-create-admins`  
* `admin-view-update-delete-members`  
* `admin-create-members`

Default Admin Privileges (an Admin being defined by the `adminOfOrg` relationship field):

* `admin-view-update-delete-orgs`  
* `admin-create-orgs`  
* `admin-view-update-delete-members`  
* `admin-create-members`

> **Note:** `owner-view-update-delete-admins-and-members` applies to both Admins and Members through the `memberOfOrgIDs` Relationship-Derived Virtual Property (RDVP), meaning that if an Administrator is assigned to an Organization and *is not* a member, an Owner will not be able to see or manage them (and additionally, if an Admin *is* a Member, and Admin can see and manage them). I’ll show you later in this document [how you can break down permissions by Owner, Admin, and Member](#splitting-owner-admin-and-member-privileges).

### Reading an Existing Privilege

So let’s see what this means in practice. Taking a look at the permission `owner-view-update-delete-orgs`, we can learn the following:

| Key | Value | What it Means |
| :---- | :---- | :---- |
| `accessFlags` | `Object` | In this object, we see that the Owner can **view and manage** an organization’s `name`, `description`, `admins`, `members`, `parent`, and `children`. They can only **view**  `owners`, `parentIDs`, `adminIDs`, `parentAdminIDs`, `ownerIDs`, and `parentOwnerIDs`. |
| `actions` | `[]` | No actions here since we are using a filter. |
| `filter` | `"/ownerIDs eq \"{{_id}}\" or /parentOwnerIDs eq \"{{_id}}\""` | The user only has this permission if they are an Owner of this Organization or its parent. |
| `name` | `owner-view-update-delete-orgs` | The name of the privilege, mapped to the `privilegeAssignments.json` configuration file in your tenant. This privilege is mapped to users with the `"relationshipField": "ownerOfOrg"`. |
| `path` | `"managed/alpha_organization"` | This privilege applies to the `alpha_organization` managed object. |
| `permissions` | `"VIEW","UPDATE","DELETE"` | The permission allows reading, updating, and deleting the attributes listed (update and delete specifically for the ones set to `”readOnly”: false`. |

## Modifying Privileges

Make a copy of your `alphaOrgPrivileges` file and name it something like `updated_alphaOrgPrivileges`. When you’ve made your changes to the file, `PUT` the file back into your tenant to auto-update the privileges for your delegated administration.

**Copy Command**

```bash
cp original_alphaOrgPrivileges.json updated_alphaOrgPrivileges.json
```

**Option 1: REST**

```bash
curl --location --request PUT 'https://{tenant-domain}/openidm/config/alphaOrgPrivileges' \
--header 'Accept: application/json, text/javascript, */*; q=0.01' \
--header 'Content-Type: application/json' \
--header 'Bearer ****' \
--data updated_alphaOrgPrivileges.json
```

**Option 2: FroDo**

```bash
frodo idm import -f updated_alphaOrgPrivileges.json -i alphaOrgPrivileges <tenant>
```

The same can be said for your `privilegeAssignments` configuration file. Copy, and then `PUT`.

**Copy Command**

```bash
cp original_privilegeAssignments.json updated_privilegeAssignments.json
```

**Option 1: REST**

```bash
curl --location --request PUT 'https://{tenant-domain}/openidm/config/privilegeAssignments' \
--header 'Accept: application/json, text/javascript, */*; q=0.01' \
--header 'Content-Type: application/json' \
--header 'Bearer ****' \
--data updated_privilegeAssignments.json
```

**Option 2: FroDo**

```
frodo idm import -f updated_privilegeAssignments.json -i privilegeAssignments <tenant>
```

When you’re done modifying permissions, it’s possible that your privileges will look more like a Venn Diagram than a series of nested circles.  

![An example diagram illustrating how delegated permissions can overlap depending on how you customize privileges](../images/setting-delegated-admin-privileges-in-aic/example-relationship-mapping-altered.png)


Let’s take a look at some ways we may want to modify our privileges.

### Updating Existing Privileges

In most cases, the existing privilege set defined in your `alphaOrgPrivileges` will already cover what actions you want your delegated administrator to take \- you’ll just need to update what attributes they have access to.

As an example, let’s say that we want our Administrators to be able to view the first and last name of their pre-existing Members but not be able to update them. 

![A screenshot of the end-user portal where an admin can manage the member's first and last name](../images/setting-delegated-admin-privileges-in-aic/admin-managing-member-default.png)

*Default Admin Behavior - Managing Member First Name and Last Name*

Make a copy of your `alphaOrgPrivileges` file and search for the `admin-view-update-delete-members` privilege. Under its `accessFlags`, you can see that the `readOnly` values for both `givenName` and `sn` are set to `false`. 

```json
{
    "accessFlags": [
        {
            "attribute": "userName",
            "readOnly": false
        },
        {
            "attribute": "password",
            "readOnly": false
        },
        {
            "attribute": "givenName",
            "readOnly": false // <-- this one
        },
        {
            "attribute": "sn",
            "readOnly": false // <-- and this one
        },
        ...
    ],
    "actions": [],
    "filter": "/memberOfOrgIDs eq \"__org_id_placeholder__\"",
    "name": "admin-view-update-delete-members",
    "path": "managed/alpha_user",
    "permissions": [
        "VIEW",
        "DELETE",
        "UPDATE"
    ]
}
```

Let’s set both of those fields to `true`. 

```json
// Inside the "accessFlags" section of "admin-view-update-delete-members":
{
    "attribute": "givenName",
    "readOnly": true
},
{
    "attribute": "sn",
    "readOnly": true
},
```

Save and push the updated config. When you refresh the page as an admin, you’ll see that you have read-only permissions to those attributes.

![A screenshot of the end-user portal where an admin cannot manage the member's first and last name](../images/setting-delegated-admin-privileges-in-aic/admin-managing-member-readonly.png)

_Updated Admin Behavior - Read Only_

Some other reasons why you may modify an existing privilege include:

* **Adding Attributes** on Organizations or Members for Owners and/or Admins to manage or view. This is your most likely use case as you add custom attributes to your Organization or leverage existing indexed attributes on your Users.  
* **Removing Attributes** that you don’t want Owners and/or Admins to manage at all. As you specify what permissions each role is capable of managing, it’s likely that some features and functionality should be stripped from the delegated administrator.  
* **Modifying Attribute Permissions** so that an Owner and Admin has distinctly different permissions when interacting with an Organization.

> **Note: Modify the Filters and Permissions at your own risk**. Updating these values without careful consideration could result in over-permissioning your delegated administrators to an unexpected subset of Users and Organizations.

### Creating New Privileges

There may be rare cases in which you’d like to create an additional delegated administrative permission. To do so, you’ll need to update the `privilegeAssignments` file to handle that permission.

In this example, let’s create a privilege that enables administrators to view, edit, and delete the first and last name of a subset of their Members. This subset will be defined by a “Enable Customer Support” attribute that we’ll define on the User object \- that way, Members can self-service if they want their name to be updated by an Administrator.

#### Creating an Attribute to Filter by

To create this attribute, head to the Native Consoles → Identity Management and then to your Managed Alpha User (Configure → Managed Objects → alpha\_User).

![A screenshot of editing the alpha user](../images/setting-delegated-admin-privileges-in-aic/alpha-user-edit.png)

_The Managed Alpha User_

Once there, select “Add a Property” and create a new attribute with the following details:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `custom_supportable` | Enable Customer Support | Boolean | False |

Once created, select the attribute and under Show advanced options set “Searchable” to `true` (enabled).

![A screenshot of setting the new attribute to searchable](../images/setting-delegated-admin-privileges-in-aic/alpha-user-searchable-true.png)

_Setting Searchable to True_

After saving our updates, when we look at a User we’ll see the “Enable Customer Support” option on their profile.

![A screenshot of the searchable field on the user's management screen](../images/setting-delegated-admin-privileges-in-aic/alpha-user-searchable-management.png)

_The User's View_

Now that we have the attribute we want to filter by, let’s create a new privilege which will look for this filter.

#### Creating the New Privilege

Make a copy of your `privilegeAssignments` file. Within that file, underneath the `adminPrivileges` `privilegeAssignment`, add the privilege `"admin-view-update-delete-supportable-members"`. Your `adminPrivileges` should look something like this:

```json
{
    "name": "adminPrivileges",
    "privileges": [
        "admin-view-update-delete-orgs",
        "admin-create-orgs",
        "admin-view-update-delete-members",
        "admin-view-update-delete-supportable-members", // <-- This one is new!
        "admin-create-members"
    ],
    "relationshipField": "adminOfOrg"
}
```

Save and push the updated privileges config.

Next, inside the `alphaOrgPrivileges`, let’s make a new privilege. I’ll be copying the functionality from within `admin-view-update-delete-members` to start.

In our last example, we prevented Admins from being able to update an existing Member’s first name and last name. This privilege is going to let us update the names of Members who fulfill a specific criteria, in this case if the member has enabled the “supportable” permission.

In our new `admin-view-update-delete-supportable-members` privilege, let’s change the `readOnly` values of `givenName` and `sn` back to `false`. Additionally, let’s update the `filter` to only include Members who have the `custom_supportable` attribute set to `true`:

```json
"filter": "/memberOfOrgIDs eq \"__org_id_placeholder__\" and custom_supportable eq true"
```

Your new privilege should look something like this:

```json
{
    "accessFlags": [
        {
        "attribute": "userName",
        "readOnly": false
        },
        {
        "attribute": "password",
        "readOnly": false
        },
        {
        "attribute": "givenName",
        "readOnly": false // <-- Make sure to change this back
        },
        {
        "attribute": "sn",
        "readOnly": false // <-- Change this one too
        },
        ...
    ],
    "actions": [],
    "filter": "/memberOfOrgIDs eq \"__org_id_placeholder__\" and custom_supportable eq true", // <-- Your new filter
    "name": "admin-view-update-delete-supportable-members", // <-- Your new privilege
    "path": "managed/alpha_user",
    "permissions": [
        "VIEW",
        "DELETE",
        "UPDATE"
    ]
}
```

Save and push the updated alpha org privileges config.

#### Testing the Functionality

Log in as a Member and an Administrator on two separate browsers. By default, the Administrator won’t be able to manage the Member’s first and last name. As the Member, set “Enable Customer Support” to `true` (enabled) and then refresh the Administrator’s page. You now have access as an Administrator to manage the Member’s name\!

![A screenshot of the user enabling customer support](../images/setting-delegated-admin-privileges-in-aic/alpha-user-searchable-management-enabled.png)
 
*Setting Member’s Support to Enabled (under /enduser/?realm=/alpha#/profile)*

![A screenshot of the admin being able to support the user again](../images/setting-delegated-admin-privileges-in-aic/admin-managing-member-default.png)

*Supporting that Identity as an Administrator*

By creating a new privilege we can filter to specific subsets of users that we want different sets of delegated administration to manage. Some other reasons why you may create a new privilege include:

* **Custom Relationships** such as Employee/Manager, Department Leader/Team, Family Head/Members which have increased permissions than a standard Administrator  
* **Attribute-based flags (ABAC and RBAC)** where users can self-service or be dynamically added to specific support tiers or management rights

### Splitting Owner, Admin, and Member Privileges

As noted in [Default Privilege Assignments](#default-privilege-assignments), there are some privileges that apply to both Administrators and Members due to the default configuration of the Organization and User. There may be situations, however, where it’s advantageous to have a separate privilege applying to Owners, Admins, or Members only.

In this example, we’ll create a new privilege for Owners to be able to view, update, and delete attributes on Administrators only. To do that, we’ll need to define a way to reference Admins and their downstream Organizations that they are managing.

#### Creating the Admin of Orgs Reference {#creating-the-admin-of-orgs-reference}

First off, we’re going to need to create a new RDVP on our Managed Organization which tracks the IDs of the Children. 

In your Native IDM console (Native Consoles → Identity Management) and head to your Managed Organization (Configure → Managed Objects → alpha_organization). 

![A screenshot of the alpha_organization editor](../images/setting-delegated-admin-privileges-in-aic/alpha-org-edit.png)

*Editing the Alpha Organization*

Once there, select “Add a Property” and create a new attribute with the following details:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `childIDs` | child org ids | Array | False |

Select the newly-created attribute, and under Details → Advanced Options, **disable** *Viewable* and *User Editable* and **enable** *Virtual* and *Return by Default*. You know you’ve done this right when after saving the *Query Configuration* tab appears on the page.

![A screenshot of the childIDs attribute with virtual and return by default set](../images/setting-delegated-admin-privileges-in-aic/childids-edit.png)

*Editing the Child IDs Advanced Properties*

Select Query Configuration, and input the following fields:

| Field | Value |
| ----- | ----- |
| Referenced Relationship Fields | `["children"]` |
| Referenced Object Fields | `_id,childIDs` |
| Flatten Properties | `true` (checked) |

![A screenshot of the childIDs query editor screen](../images/setting-delegated-admin-privileges-in-aic/childids-query.png)

*Setting the ChildIDs Query*

Next, we will need to notify the appropriate relationships that changes have been made so that they update correctly. To do that, head back to the Organization Attribute management screen.

Once there, select the `parent` attribute. Under Relationship Configuration, select the Edit button next to the “parent” `alpha_organization` and set “Notify” to `true` (enabled). This notifies the child that a relationship change has occurred on the parent.

![A screenshot of where the edit icon is on the Child within the Parent relationship (aria label edit, under alpha_organization)](../images/setting-delegated-admin-privileges-in-aic/parent-relationship-child.png)

*Editing the Child Relationship*

![A screenshot of setting the "Notify" property on the Child to "true"](../images/setting-delegated-admin-privileges-in-aic/notify-child.png)

*Ensuring the Child is Notified on Relationship Change*

Back on the main Organization screen, Select the `children` attribute and under Details → Advanced Options → Notify Relationships select `parent` and `admins`. This notifies the parent and the admins that a change has occurred on the child.

![A screenshot of adding parent and admins to the child's notify relationships field](../images/setting-delegated-admin-privileges-in-aic/notify-parent-admin.png)

*Notifying the Parent and Admin of Changes on the Child*

Finally, we’re going to need to create a new RDVP on our Managed User which tracks the Organizations a User is an Administrator of. Head to your Managed Alpha User (Configure → Managed Objects → alpha_user) and select the first Generic Indexed Multivalue attribute you aren’t using (mine is `frIndexedMultivalued1`). Note that you can use a custom attribute, but if an attribute is already there let’s use it!

Just like with the `childIDs`, under advanced options deselect “Viewable”, “Searchable”, and “User Editable” and select “Virtual” and “Return by Default”. I’m also renaming my title to “Admin of Org IDs” so it’s easier for me to find later.

![A screenshot of setting the virtual property on the frIndexedMultivalue1](../images/setting-delegated-admin-privileges-in-aic/adminorgids-virtual.png)

*Enabling the RDVP*

Select Query Configuration, and input the following fields:

| Field | Value |
| ----- | ----- |
| Referenced Relationship Fields | `["adminOfOrg"]` |
| Referenced Object Fields | `_id,childIDs` |
| Flatten Properties | `true` (checked) |

![A screenshot of setting the query on the frIndexedMultivalue1](../images/setting-delegated-admin-privileges-in-aic/adminorgids-query.png)

*Setting the Query*

Now for an RDVP to take effect we will need to make a change in our Organization. Make a Child Organization and assign it to the Organization you’ve been testing with.

![A screenshot of setting a Child org to a Parent Org](../images/setting-delegated-admin-privileges-in-aic/child-parent-org.png)

*The Child to Parent Org Relationship*

If you’ve set up your new RDVP correctly, you’ll see back on your parent organization, inside of “Raw JSON”, that the child you just created has been assigned.

![A screenshot of the parent org containing the child org's id](../images/setting-delegated-admin-privileges-in-aic/parent-json.png)

*There's the Child Org ID!*

The same will be said for your Admin. If you look at their `frIndexedMultivalued1` attribute (the one we set in this example), both the IDs of the Parent they are directly assigned to AND the Child they now manage are present.

![A screenshot of frIndexedMultivalue1 containing the Parent and Child Org Ids, in raw JSON](../images/setting-delegated-admin-privileges-in-aic/adminorgids-json.png)

*AdminOrgIDs in Practice*

Now that we have the means to reference specifically organization IDs related to our Administrators, let’s create a privilege that targets Administrators only.

#### Creating the Owner-to-Admin Privilege

Make a copy of your `privilegeAssignments` file. Within that file, underneath the `ownerPrivileges` `privilegeAssignment`, add the privilege `"owner-view-update-delete-admins"`. Your `ownerPrivileges` should look something like this:

```json
{
    "name": "ownerPrivileges",
    "privileges": [
        "owner-view-update-delete-orgs",
        "owner-create-orgs",
        "owner-view-update-delete-admins-and-members",
        "owner-view-update-delete-admins", // <-- This is your new privilege
        "owner-create-admins",
        "admin-view-update-delete-members",
        "admin-create-members"
    ],
    "relationshipField": "ownerOfOrg"
}
```

Save and push the updated privileges config.

In our last example, we prevented Admins from being able to update an existing Member’s first name and last name. By default, since owners have their own privilege to manage Admins and Members (`owner-view-update-delete-admins-and-members`), an Owner is still able to manage these attributes. Let’s change it so an Owner can still change the first and last name on a Member but not on an Administrator.

Going back to the `alphaOrgPrivileges`, let’s duplicate the `owner-view-update-delete-admins-and-members` permission.

In our new `owner-view-update-delete-admins` privilege, let’s change the `readOnly` values of `givenName` and `sn` to `true` just like we did for `admin-view-update-delete-members`. Additionally, we are going to change our `filter` so that it targets the new “Admin of Org IDs” (`frIndexedMultivalued1`) attribute we created on our users.

Your new privilege should look something like this:

```json
{
    "accessFlags": [
        {
            "attribute": "userName",
            "readOnly": false
        },
        {
            "attribute": "password",
            "readOnly": false
        },
        {
            "attribute": "givenName",
            "readOnly": true // <-- This should be set
        },
        {
            "attribute": "sn",
            "readOnly": true // <-- And this
        },
        ...
    ],
    "actions": [],
    "filter": "/frIndexedMultivalued1 eq \"__org_id_placeholder__\"", // <-- Your new query
    "name": "owner-view-update-delete-admins", // <-- Your new privilege
    "path": "managed/alpha_user",
    "permissions": [
        "VIEW",
        "DELETE",
        "UPDATE"
    ]
}
```

Save and push the updated alpha org privileges config.

#### Testing the Functionality

Now, let’s log in as an Owner. When you select a Member, you’ll see that you can update their first name and last name.

![A screenshot of an owner editing a member's first and last name](../images/setting-delegated-admin-privileges-in-aic/owner-member.png)

*The Owner's Member Management Permissions*

When you select the Administrator, you’ll see that you can’t update their first and last name!

![A screenshot of an owner not being able to edit an admin's first and last name](../images/setting-delegated-admin-privileges-in-aic/owner-admin.png)

*The Owner's Admin Management Permissions*

We’ve now seen how to manage privileges for each delegated administration type. Note that you can extend this further, by:

* **Tracking OwnerOfOrgIDs** so that you can delegate permission to Owner users from Members or Admins  
* **Creating new Delegated Administration types** where you can granularize the management of users further

Note that if you add any new RDVPs to match what you did for admins you’d have to use a new indexed attribute and add that relationship (for example, `owners`) to the “Notify Relationships” on the Children relationship [just like you did for admins](#creating-the-admin-of-orgs-reference).

# Conclusion

This How-To should give you the tools you need to read, manage, and update delegated administrative permissions for your Organization users within your AIC tenant. With it you should be able to build access policies and controls that define fine-grained permissions for each user type that can self-serve themselves, their Organizations, and the people they are related to.