---
draft: false
date: '2024-10-29'
title: 'Modeling Inheritance in PingOne Advanced Identity Cloud or PingIDM'
description: 'Deploy object-oriented concepts within your IAM strategy'
summary: Learn how to enable inheritance and inheritance overrides for managed identities within PingOne AIC and PingIDM
categories: ["Ping Identity"]
tags: ["PingOne Advanced Identity Cloud", "PingIDM", "JavaScript"]
types: ["How-To"]
cover:
  image:
  alt:
  caption:
  relative: false
ShowToc: true
author: David Gwizdala
---

# What is Inheritance?

PingOne Advanced Identity Cloud and PingIDM enable you to create complex relationships between identities - be it users, groups, organizations, things, or any other custom grouping of metadata you need to represent accounts and permissions across your systems. While relationships on their own are an incredibly powerful way to represent and link data to each other (e.g. a Manager/Employee, Parent/Child, Provider/Patient), **Inheritance** enables us to update and enforce data downstream from these relationships dynamically and automatically as enforced by the parent. This significantly reduces overhead as it enables an upstream entity to control aspects of its children without the manual intervention of that child and allows a child to derive information from its parent.

Some examples of why we might need an inheritance structure includes:

* A department managing password policy for its sub-departments  
* A parent controlling the family phone plan (and its associated permissions) for their children  
* A sub-brand opting to use the general theming/branding from its parent

In short: **Inheritance** allows us to pass information from a parent to a child, enabling faster implementation and enforcement of capabilities as the number of relationships expand within your solution.

# Modeling Inheritance

For the examples below, we’ll be using a pre-built model in Advanced Identity Cloud (AIC) with a Parent-Child relationship: [Organizations](https://docs.pingidentity.com/pingoneaic/latest/identities/organizations.html). Organizations are a great way to structure groups of users and their associated metadata and their relationship structure follows a logical tree that plays nicely with the idea of inheritance.

That being said, whether in AIC or PingIDM, any relationship defined between any managed object follows the same principles described below (for example: manager/employee, parent/child, department/team). Feel free to follow along with an Organization or with your own custom implementation.

> **NOTE:** Do not attempt inheriting Relationships, as those are defined as references between objects. Instead, consider defining a Relationship that links your managed objects together through associations with other objects.

This walkthrough expects a basic understanding of Relationships and RDVPs. If you need a refresher, check out the links below:

[Relationships](https://docs.pingidentity.com/pingoneaic/latest/planning/plan-object-modeling-relationships.html)  
[Relationship-Derived Virtual Properties (RDVPS)](https://docs.pingidentity.com/pingoneaic/latest/idm-objects/managed-object-virtual-properties.html#relationship-derived-virtual-properties)

---

## _Step 1_: Representing Inherited Attributes

Let’s start with a simple example: we’ll create a single inherited attribute whose value is inherited by its parent if a parent exists. Since Organizations already have a `description` attribute built-in, we’ll start here.

To do so, in the Identity Management Console’s (in AIC, Native Consoles → Identity Management) top toolbar select Configure → Managed Objects and then click on Alpha_organization. You’ll be taken to the Object Management page where you can manage attributes and their behavior.

![Screenshot of selecting "Alpha_organization" under "Managed Objects" in PingIDM](../images/modeling-inheritance-in-aic-or-idm/managed-objects.png)  
![Screenshot of a default "Alpha_organization"](../images/modeling-inheritance-in-aic-or-idm/alpha-org.png)

_Managing the Alpha Organization Object_

We’ll create a new attribute that we want to have inherited. Select “Add a Property”, and at the bottom of the screen put in the following:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `inherited_description` | Inherited Description | String | False |

Select the newly-created attribute, and under Details → Advanced Options, **disable** *Viewable* and *User Editable* and **enable** *Virtual* and *Return by Default*. You know you’ve done this right when after saving the *Query Configuration* tab appears on the page.

![Screenshot of the inherited_description with query configuration enabled](../images/modeling-inheritance-in-aic-or-idm/inherited-description.png)

_The `inherited_descripton` with Query Configuration Displayed_

We’ll use an RDVP to store the value of `description` from the Parent of an Organization. To do so, we will reference the `"parent"` relationship and then reference the `description` attribute within the parent.

| Field | Value |
| ----- | ----- |
| Referenced Relationship Fields | `["parent"]` |
| Referenced Object Fields | `description` |

![Screenshot of the Inherited Description Query](../images/modeling-inheritance-in-aic-or-idm/inherited-description-query.png)

_The `inherited_description` Query_

Backing out of this object and back into the Organization managed object, select `description`. We’ll need to make sure that every time the description changes it notifies the children of the change to update the RDVP we just created. Under Default → Advanced Options, add `children` to the Notify Relationships array.

![Screenshot of the modified "description" attribute with notifying children relationships](../images/modeling-inheritance-in-aic-or-idm/description-details.png)

_Notifying the Children_

## _Workshop_: Testing Inherited Values

First, create an Organization named “Parent” and put a unique description on it. I’m using “Parent Description” for this example.

![Screenshot of the Parent Organization with "Parent Description" set](../images/modeling-inheritance-in-aic-or-idm/parent-description.png)

_The Parent Organization_

Next, create an Organization named “Child” and with its own unique description. For the sake of consistency, this one has the name “Child Description”.

![Screenshot of the Child Organization with "Child Description" set](../images/modeling-inheritance-in-aic-or-idm/child-description.png)

_The Child Organization_

Finally, update the child to have “Parent” as its Parent Organization. If you take a look at the “Raw JSON” for this Organization, you’ll see that it contains the description of the parent under `inherited_description`.

![Screenshot of the Child Organization's Raw JSON with the inherited_description and description attributes highlighted](../images/modeling-inheritance-in-aic-or-idm/child-json.png)

_The Child's JSON_

So we now have an attribute that updates with the inherited value of its parent every time it is changed - however, some functionality is still missing:

1. If a child changes its `description`, there’s nothing to enforce that it should match the parent. That means that the grandchild will inherit the value from the child, *not* from the parent, even if inheritance should be carried all the way down the line.  
2. If a system/application is looking to understand the description for this Organization, which value do they choose - the inherited description or the description that’s been set by the Child? Currently, due to having multiple options, the “correct” value would be determined on an application-by-application basis.

## _Step 2_: Enforcing Inheritance on an Attribute

Let’s go one step further and enforce inheritance on the base attribute, `description`, using the information that we have stored in `inherited_description`.

Back in the base Organization object, under the Scripts tab add the following **postCreate** and **postUpdate** scripts:

{{<details title="`postCreate` (Click to View)">}}
```javascript
// POST CREATE: Enforcing Inheritance, No Overrides
var INHERITED_PROPERTY_SUFFIX = 'inherited_';
// rdvp = inherited_{option name}

var newObjectAttributes = Object.keys(newObject);

for (var i = 0; i < newObjectAttributes.length; i++) {
    var attribute = newObjectAttributes[i];
    // Check if this is an RDVP
    if (attribute.indexOf(INHERITED_PROPERTY_SUFFIX) === 0) {
        // Gather the attribute to update
        var inheritedProperty = attribute.substring(INHERITED_PROPERTY_SUFFIX.length);
        // Gather the RDVP
        var inheritedPropertyAttributes = newObject[attribute];
        // Determine if the RDVP contains a value to inherit
        if (inheritedPropertyAttributes && inheritedPropertyAttributes[0] && inheritedPropertyAttributes[0][inheritedProperty] != undefined) {
            // Update the resource with the inherited value
            openidm.patch(
                resourceName+"", 
                null,
                [
                    {
                        "operation": "replace",
                        "field": `/${inheritedProperty}`,
                        "value": inheritedPropertyAttributes[0][inheritedProperty]
                    }
                ]
            );
        }
    }
}
```
{{</details>}}

{{<details title="`postUpdate` (Click to View)">}}
```javascript
// POST UPDATE: Enforcing Inheritance, No Overrides
var INHERITED_PROPERTY_SUFFIX = 'inherited_';
// rdvp = inherited_{option name}

var areEqual = (objA, objB) => {
    if ((!objA && !!objB) || (!!objA && !objB)) {
        return false;
    }

    if (typeof objA != typeof objB) {
        return false;
    }

    if (typeof objA == 'object') {
        // Array, Object
        var aKeys = Object.keys(objA);
        var bKeys = Object.keys(objB);

        if (aKeys.length != bKeys.length) {
            return false;
        }

        for (var i = 0; i < aKeys.length; i++) {
            var aKey = aKeys[i];
            if (!bKeys.includes(aKey)) {
                return false;
            } else {
                return areEqual(aKeys[aKey], bKeys[aKey]);
            }
        }
    } else {
        // String, Boolean, Number
        return objA == objB;
    }

    return true;
}

var newObjectAttributes = Object.keys(object);

for (var i = 0; i < newObjectAttributes.length; i++) {
    var attribute = newObjectAttributes[i];
    // Check if this is an RDVP
    if (attribute.indexOf(INHERITED_PROPERTY_SUFFIX) === 0) {
        // Gather the attribute to update
        var inheritedProperty = attribute.substring(INHERITED_PROPERTY_SUFFIX.length);
        // Determine if the RDVP contains a value to inherit and if there is a mismatch between the current value and the RDVP
        var inheritedPropertyAttributes = object[attribute];
        var propertyAttributes = object[inheritedProperty];
        if (inheritedPropertyAttributes && inheritedPropertyAttributes[0] && inheritedPropertyAttributes[0][inheritedProperty] != undefined) {
            if (!(propertyAttributes && areEqual(propertyAttributes, inheritedPropertyAttributes[0][inheritedProperty]))) {
                openidm.patch(
                    resourceName+"", 
                    null,
                    [
                        {
                            "operation": "replace",
                            "field": `/${inheritedProperty}`,
                            "value": inheritedPropertyAttributes[0][inheritedProperty]
                        }
                    ]
                );
            }
        }
    }
}  
```
{{</details>}}

![Screenshot of the alpha_org postCreate and postUpdate scripts](../images/modeling-inheritance-in-aic-or-idm/alpha-org-scripts.png)

_Setting the `postCreate` and `postUpdate` Scripts_

These scripts do the following:

1. If you add an RDVP with the prefix `inherited_`, it will look for a corresponding attribute with the same name (e.g. `inherited_description` and `description`)  
2. If the RDVP is populated, and there’s a mismatch between the RDVP’s value and the value stored on the main attribute, update the main attribute to match the value in the RDVP.

The nice part about this approach is that it’s **infinitely extensible** - as long as you add an attribute and a matching inherited_attribute, of any type (other than a relationship) it can be recognized and inherited. Some examples include:

* A list (array) of scopes  
* A grouping (single-layer object) of functionality/capabilities  
* A switch (boolean) that enables/disables functionality in a Journey or is referenced in an Role’s conditional filter

But enough exposition - let’s try it out!

## _Workshop_: Testing Attribute Inheritance

Since our inheritance is enforced on a Create or Update, we’ll need to update a value to see inheritance enforced in action. 

Go back into the Parent Organization and update its description - we’ll use “Parent Description v2”. Once you hit save, you’ll see that the description has been updated in both the parent and the child!

![Screenshot of the Child Description updating to match the Parent](../images/modeling-inheritance-in-aic-or-idm/child-description-updated-v2.png)

_The Child Updating to Match the Parent_

To show down-the-line inheritance, let’s add one more Organization named “Grandchild” and set the parent to “Child”. You should see the description update to match that of the Child AND of the Parent - showing inheritance flowing all the way down the tree.

![Screenshot of the Grandchild Description updating to match the Child (its Parent)](../images/modeling-inheritance-in-aic-or-idm/grandchild-description-updated-v2.png)

_The Grandchild Updating to Match the Child_

In this current approach, if you try updating the description you’ll find that as long as there’s a Parent tied to the Organization it’ll take the value that the Parent has instead. But what about cases in which the Child should be able to override their inherited attributes?

## _Step 3_: Enabling Inheritance Overrides

There are plenty of situations where a child should inherit values from their parent but have the ability to override this value with their own. This enables you to set up templated objects that can be configured to match unique use cases without starting from scratch every single time. Some examples include:

* The child department inherits the parent’s Password Policy but needs to override enforcing a stronger MFA method  
* The child brand inherits branded emails but changes the logo to match their own  
* The child employee inherits their manager’s scopes but adds more scopes based on their position  
* The child user inherits their parent’s surname but updates it later after legally changing it

To do this, we’ll place a boolean flag on the attribute to indicate whether or not it should be overridable. This way, we can scope which attributes can be overridden as well as who can enable/disable the capability to override.

For this example, all of our overrides will live inside of an `inheritance` object attribute inside the Organization - that way, we can more easily scope by individual flag or by the group of flags for our delegated administration.

Back inside the IDM Native Console underneath the managed alpha_organization, add another property with the following values:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `inheritance` | Inheritance | Object | False |

Once created, select that attribute and define the following property:

| Name | Label (Optional) | Type | Required |
| ----- | ----- | ----- | ----- |
| `description` | Inherit Description from Parent | Boolean | False |

Your `inheritance` object attribute should look like this:

![Screenshot of the Alpha Organization inheritance object attribute](../images/modeling-inheritance-in-aic-or-idm/alpha-org-inheritance-attribute.png)

_The Inheritance Object Attribute_

To ensure that inheritance is enabled by default when creating new objects, under the “Details” tab set the default value for your boolean flag to be `true`.

![Screenshot of the inheritance attribute details page where the description subattribute is set to "true"](../images/modeling-inheritance-in-aic-or-idm/enforce-inheritance-default.png)

_The Inheritance Object Attribute's Default Behavior_

Once this is done, let’s update the **postUpdate** and **postCreate** scripts that we created earlier on the base Organization object to recognize and enforce inheritance based on the override switch.

{{<details title="`postCreate` (Click to View)">}}
```javascript
// POST CREATE
var INHERITED_PROPERTY_SUFFIX = 'inherited_';
// rdvp = inherited_{option name}
var INHERITANCE_OVERRIDES = 'inheritance';
// inheritance overrides (boolean) = inheritance[option_name]
// - true = inherit, false = override

var shouldBeInherited = (attribute, objectToReview) => {
    var overrides = objectToReview[INHERITANCE_OVERRIDES];
    
    if (overrides && overrides[attribute] != undefined && overrides[attribute] != null) {
        // An override has been found. Inherit only if not overriding
        return overrides[attribute];
    } else {
        // No overrides for this attribute. Inherit by default
        return true;
    }
}

var newObjectAttributes = Object.keys(newObject);

for (var i = 0; i < newObjectAttributes.length; i++) {
    var attribute = newObjectAttributes[i];
    // Check if this is an RDVP
    if (attribute.indexOf(INHERITED_PROPERTY_SUFFIX) === 0) {
        // Gather the attribute to update
        var inheritedProperty = attribute.substring(INHERITED_PROPERTY_SUFFIX.length);
        // Check if the property should be inherited or if there's a flag indicating overrides
        if (shouldBeInherited(inheritedProperty, newObject)) {
            // Gather the RDVP
            var inheritedPropertyAttributes = newObject[attribute];
            // Determine if the RDVP contains a value to inherit
            if (inheritedPropertyAttributes && inheritedPropertyAttributes[0] && inheritedPropertyAttributes[0][inheritedProperty] != undefined) {
                // Update the resource with the inherited value
                openidm.patch(
                    resourceName+"", 
                    null,
                    [
                        {
                            "operation": "replace",
                            "field": `/${inheritedProperty}`,
                            "value": inheritedPropertyAttributes[0][inheritedProperty]
                        }
                    ]
                );
            }
        }
    }
}
```
{{</details>}}

{{<details title="`postUpdate` (Click to View)">}}
```javascript
// POST UPDATE
var INHERITED_PROPERTY_SUFFIX = 'inherited_';
// rdvp = inherited_{option name}
var INHERITANCE_OVERRIDES = 'inheritance';
// inheritance overrides (boolean) = inheritance[option_name]
// - true = inherit, false = override

var areEqual = (objA, objB) => {
    if ((!objA && !!objB) || (!!objA && !objB)) {
        return false;
    }

    if (typeof objA != typeof objB) {
        return false;
    }

    if (typeof objA == 'object') {
        // Array, Object
        var aKeys = Object.keys(objA);
        var bKeys = Object.keys(objB);

        if (aKeys.length != bKeys.length) {
            return false;
        }

        for (var i = 0; i < aKeys.length; i++) {
            var aKey = aKeys[i];
            if (!bKeys.includes(aKey)) {
                return false;
            } else {
                return areEqual(aKeys[aKey], bKeys[aKey]);
            }
        }
    } else {
        // String, Boolean, Number
        return objA == objB;
    }

    return true;
}

var shouldBeInherited = (attribute, objectToReview) => {
    var overrides = objectToReview[INHERITANCE_OVERRIDES];
    
    if (!!overrides && overrides[attribute] != undefined && overrides[attribute] != null) {
        // An override has been found. Inherit only if not overriding
        return !!overrides[attribute];
    } else {
        // No overrides for this attribute. Inherit by default
        return true;
    }
}

var newObjectAttributes = Object.keys(object);

for (var i = 0; i < newObjectAttributes.length; i++) {
    var attribute = newObjectAttributes[i];
    // Check if this is an RDVP
    if (attribute.indexOf(INHERITED_PROPERTY_SUFFIX) === 0) {
        // Gather the attribute to update
        var inheritedProperty = attribute.substring(INHERITED_PROPERTY_SUFFIX.length);
        // Check if the property should be inherited or if there's a flag indicating overrides
        if (shouldBeInherited(inheritedProperty, object)) {
            // Determine if the RDVP contains a value to inherit and if there is a mismatch between the current value and the RDVP
            var inheritedPropertyAttributes = object[attribute];
            var propertyAttributes = object[inheritedProperty];
            if (inheritedPropertyAttributes && inheritedPropertyAttributes[0] && inheritedPropertyAttributes[0][inheritedProperty] != undefined) {
                if (!(propertyAttributes && areEqual(propertyAttributes, inheritedPropertyAttributes[0][inheritedProperty]))) {
                    openidm.patch(
                        resourceName+"", 
                        null,
                        [
                            {
                                "operation": "replace",
                                "field": `/${inheritedProperty}`,
                                "value": inheritedPropertyAttributes[0][inheritedProperty]
                            }
                        ]
                    );
                }
            }
        }
    }
}  
```
{{</details>}}

## _Workshop_: Testing Inheritance Overrides

With the already-created objects, inheriting the description is set to `false`. That means we should be able to change our Child or Grandchild’s description without it being overridden by the description of the Parent.

Go back into the Child Organization. You’ll see that there’s now an “Inheritance” tab and inside it the “Inherit Description from Parent” is disabled.

![Screenshot of the Child Organization with the Inheritance options set to false](../images/modeling-inheritance-in-aic-or-idm/child-inherit-false.png)

_The Child Organization with Inheritance set to False_

Back on the “Details” tab, change the description to something different. I’m going to use “Child Description v2”. After hitting Save, you’ll see that the new description is saved. You’ve successfully overridden the inherited Parent Attribute!

![Screenshot of the Child description overriding the parent](../images/modeling-inheritance-in-aic-or-idm/child-overriding-parent.png)

_The Child Overriding the Parent_

Now, head over the Grandchild Organization. In this example, we are going to inherit from our parent (in this case, the Child Organization). Once you enable “Inherit Description from Parent”, you’ll see that its description has been updated to reflect that of the Child Organization.

![Screenshot of the Grandchild Organization with the Inheritance options set to true](../images/modeling-inheritance-in-aic-or-idm/grandchild-inherit-true.png)
![Screenshot of the Grandchild Organization inheriting the description of the Child](../images/modeling-inheritance-in-aic-or-idm/grandchild-inheriting-child.png)

_The Grandchild Inheriting from the Child_

Finally, enable the Child’s description inheritance. Both the Child and the Grandchild now are inheriting upstream to the Parent.

![Screenshot of the Child Organization with the Inheritance options set to true](../images/modeling-inheritance-in-aic-or-idm/child-inherit-true.png)
![Screenshot of both the Child and Granchild inheriting the description from the Parent](../images/modeling-inheritance-in-aic-or-idm/all-inheriting.png)

_All Layers of Children Inheriting from the Root_

# Conclusion

Through this workshop we have shown how to inherit attributes from related identities within PingOne Advanced Identity Cloud and PingIDM. We’ve also learned how to enable inheritance overrides to enable flexible down-the-line control.

As your organization grows, the capabilities needed for your users will continue to expand in complexity. Rather than explode the number of controls that each individual has to manage, simplify through inherited properties.