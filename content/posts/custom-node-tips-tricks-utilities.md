---
draft: false
date: '2026-01-06'
title: 'Custom Node Tips, Tricks, and Utilities'
description: 'Learn common Custom Node reusable patterns that make development more consistent, easier to manage, and faster to implement.'
summary: Learn when and how to build Custom Nodes in PingAM and PingOne AIC along with some kickstart utilities to make development faster and easier.
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

[Custom Nodes](https://docs.pingidentity.com/pingoneaic/latest/journeys/node-designer.html) in PingOne Advanced Identity Cloud and PingAM enable the creation and management of flexible, reusable Journey Nodes. Unlike [Scripted Decision Nodes](https://docs.pingidentity.com/auth-node-ref/latest/scripted-decision.html), custom nodes let us define input parameters and output values so that we can **templatize** and **genericize** common actions we find ourselves scripting (and re-scripting) over and over.

As I’ve been building out Custom Nodes I’ve collected some common reusable patterns that make development more consistent, easier to manage, and faster to implement.

This Guide will walk you through some tips and tricks when building Custom Nodes as well as provide you functionality to help kickstart your next Node design. It’s broken down into the following sections:

1. [When to Use a Custom Node](#when-to-use-a-custom-node)  
2. [When **Not** to Use a Custom Node](#when-not-to-use-a-custom-node)  
3. [How to Structure a Custom Node](#how-to-structure-a-custom-node)  
4. [Useful Functionality](#useful-functionality)  
5. [Gotchas](#gotchas)

This Guide expects a level of familiarity with Scripting in PingAM/PingOne Advanced Identity Cloud as well as an understanding of Journeys.

# When to Use a Custom Node

## Reusability and Traceability

If you find yourself using the same Scripted Decision Node across multiple Journeys, Realms, or Tenants, it may be time for you to build a Custom Node.

Custom Nodes make reusing custom code significantly easier, for a few reasons:

* Custom Nodes are available **cross-realm** \- meaning you can design and maintain the node in one place and use it across all of your PingAM realms.  
* Custom Nodes **detect and show usage and cannot be deleted when in use**. Ever accidentally deleted a script that was used in a Journey? Custom Nodes prevent that issue as well as give you broader visibility into how it’s being implemented.

## Sharability

If you find yourself creating custom capabilities that help your broader team, you’ll likely save a ton of time down the line if you convert your script into a Custom Node.

* Custom Nodes **show up in the Node selector in the Journey Canvas** \- meaning your designers don’t have to know what script to pick or configure and can organically find your node as they build their Journeys.  
* Custom Nodes have **preset outcomes** \- saving your Journey designers the step of reviewing your Scripted Decision Node code to find the right outcomes and hopefully spell them correctly in the Node configuration.  
* Custom Nodes have **built-in import and export utilities** \- meaning you have a preconfigured bundle that can be stored (locally, remotely, in git, etc), tracked, and imported using both API and UI.

## Parametrization

If you find yourself creating copies of the same Scripted Decision Node but with different starting values (e.g. gathered from state, ESVs, set in constants), you should probably consider building a Custom Node.

Custom Nodes support **parametrization** of inputs, including **ESV support**, meaning that you can build a genericized structure that supports dynamic data retrieved from ESVs, state values, and/or user input to determine its outcomes. Why create a scripted node that returns an access code from one service, for example, when you can create a custom node that returns an access code from *any* service given the provided client credentials and secret?

## Atomic Actions

Custom Nodes (and Journey Nodes in general) are meant to be atomic actions. This means that an individual node should do one thing and do that one thing well. As an example, take a look at the [Platform Username Node](https://docs.pingidentity.com/auth-node-ref/latest/platform-username.html) \- it collects a username and puts that value into Shared State. For a more complex version, there’s the [Attribute Collector](https://docs.pingidentity.com/auth-node-ref/latest/attribute-collector.html) \- it collects attributes found in the IDM schema and stores those in objectAttributes.

If you find yourself making a Scripted Decision Node or Custom Node that performs a wide range of diverse actions, consider breaking it down into smaller Custom Nodes. You’ll likely find that each component you’ve designed has its own utility outside of the larger node and, while more nodes are on your canvas, your Journey may become easier to follow since each action is explicitly shown on-screen.

## Tl;dr

Custom Nodes provide you the following **benefits**:

* Cross-Realm availability (primarily useful for PingAM)  
* Display of all usage of the node within the UI and prevention of deletion while in use  
* Node availability within the Journey Canvas Node Selector  
* Preset, static outcomes  
* Built-in import and export utilities  
* Parametrization, including passing of ESVs  
* Atomic, generic, templated patterning

# When **Not** to Use a Custom Node

## Hyper-Specificity

If your created node performs an extremely specific, likely one-off, action it is probably best to keep it a Scripted Decision Node. Creating many one-off nodes has the unintended consequence of cluttering your Custom Nodes viewer and your Node selection list within your Journey canvas and may also result in designers unfamiliar with your node using it in unexpected (likely incorrect) ways in their Journeys.

## Hyper-Generalization

There’s a reason we want our nodes to be self-contained with well-defined boundaries of use: it means that our users who will be building with these nodes will understand exactly what the node does and how to manipulate the inputs in that node to perform the actions they want to take. A super-duper do-it-all node will likely become too complicated to set up (or maintain\!) for anyone other than the designer of the node.

## Libraries

Since Nodes are meant to be atomic, their functionality must be self-contained to what is defined in the Node itself. As such, Custom Nodes do not have access to Library Scripts within their scripts.

If you are looking to create a Custom Node whose functionality would normally leverage a Library Script, see if you can place the necessary functions within the Node’s script editor without bloating the scope of that node too much (see [Hyper-Generalization](#hyper-generalization)). Otherwise, a Scripted Decision Node may be more effective in this use case.

## Tl;dr

Custom Node come with the potential **watch-out-fors**:

* Explosion of one-off nodes within the Journey Canvas  
* Creation of overly vague “super nodes” that are difficult to use and maintain  
* Lack of Library Script support

# How to Structure a Custom Node

As more and more Custom Nodes are designed, it’s important to adopt a common design practice to ensure that maintainers can easily learn from, adopt, modify, and improve each other’s designs. The provided structure was designed with portability, clarity, and consistency in mind for Custom Nodes as well as Scripted Decision Nodes.

Below is an example structure for Custom Nodes, including comments detailing what each section does.

```javascript
/**
 * Title: 
 * Provide the name of the node here. Makes searching for the node easier in Scripts
 * 
 * Description:
 * Explain what your node does. How does it work, what does it interact with?
 * Make this multi-line as needed.
 *  
 * Outcomes:
 * - Success   Description of this outcome
 * - Error     Description of this outcome
 *
 * Author(s):
 * - Author Name
 */

//// CONSTANTS
/**
 * Use this section to define constant values that will be used throughout your script.
 * These values should not change during execution of the script.
 * Placing constants at the top of the script makes it easy to find and modify them if needed.
 */
var CONSTANTS = {
    ERROR_TAG: "Example Node Name"
};

var NODE_OUTCOME = {
    SUCCESS: "Success",
    ERROR: "Error"
};

//// UTILITIES
/**
 * Use this section to define utility functions that will be used throughout your script.
 * These functions should be generic and reusable, helping to keep the main logic of your script clean and focused.
 * They can include tasks like data manipulation, callbacks definitions, APIs, etc.
 * Ideally, define these functions in a way that they can be reused in other scripts and nodes as well.
 */


//// MAIN
(function() {
    // Initial (default) outcome. This may be updated based on logic in the try/catch block.
    outcome = NODE_OUTCOME.SUCCESS;

    try {
        /**
         * Main logic of your script goes here.
         * Ensure to handle any potential errors gracefully and update the outcome accordingly.
         * Minimize the amount of code in this section by leveraging utilities defined above.
         */
        
        //// PARAMETERS
        /**
         * Get parameters defined for this node
         * Default values should be set in the node configuration and in the CONSTANTS section above.
         */
        // Example parameters
        var exampleBoolean = !!properties.exampleBoolean;
        var exampleString = properties.exampleString ? properties.exampleString : "defaultValue";
        var exampleNumber = properties.exampleNumber ? Number(properties.exampleNumber) : 42;
        var exampleArray = properties.exampleArray ? properties.exampleArray.toArray() : [];
        var exampleObject = properties.exampleObject ? properties.exampleObject : {};
        
        //// LOGIC
    } catch(e) {
        //// ERROR HANDLING
        logger.error(`${CONSTANTS.ERROR_TAG} ERROR: ${e}`);
        outcome = NODE_OUTCOME.ERROR;
    }
    action.goTo(outcome);
}());
```

A template not including the comments and examples is found below:

```javascript
/**
 * Title: 
 * 
 * Description:
 *  
 * Outcomes:
 * - Success   The node completed successfully
 * - Error     The node encountered an error. Check the logs for more details.
 *
 * Author(s):
 * - 
 */

//// CONSTANTS
var CONSTANTS = {
    ERROR_TAG: "Example Node Name ERROR"
};

var NODE_OUTCOME = {
    SUCCESS: "Success",
    ERROR: "Error"
};

//// UTILITIES

//// MAIN
(function() {
    outcome = NODE_OUTCOME.SUCCESS;

    try {      
        //// PARAMETERS
        
        //// LOGIC
    } catch(e) {
        //// ERROR HANDLING
        logger.error(`${CONSTANTS.ERROR_TAG}: ${e}`);
        outcome = NODE_OUTCOME.ERROR;
    }
    action.goTo(outcome);
}());
```

# Useful Functionality

## Retrieving, Concatenating, and Traversing Values from State

### Use Case

Very commonly we interact with data retrieved from Node State. In many cases, we may need to retrieve nested data stored in an object and/or array, and may want to concatenate that data with other static (or dynamically-retrieved) data.

The utility function `resolveHandlebars(element, replaceUnresolvedHandlebarsWithString)` provides you a way to do that. It supports nested objects and arrays using [JSON Pointer Notation](https://rapidjson.org/md_doc_pointer.html) as well as string interpolation using handlebars `{{}}` syntax.

To use the function within your Custom Node:

1. Paste the below [Utility Functions](#utility-functions) into your node.  
2. Reference the `resolveHandlebars` function against one of your parameters. I like to also set `replaceUnresolvedHandlebarsWithString` as a parameter so that the user can decide if the node should do something different if any of the state values are missing.

```javascript
// Within your MAIN, in the PARAMETERS section

var replaceUnresolvedHandlebarsWithEmptyString = !!properties.replaceUnresolvedHandlebarsWithEmptyString;
var message = properties.message ? resolveHandlebars(properties.message, replaceUnresolvedHandlebarsWithEmptyString) : null; // use whatever property here
```

### Examples

#### Simple State Values

Use Handlebars to reference state keys directly. This will return the value as stored in state.

* `{{errorMessage}}` \- retrieves the value from `nodeState.get("errorMessage")`  
* `{{username}}` \- retrieves the value from `nodeState.get("username")`  
* `{{objectAttributes}}` \- retrieves the value (**object**) from `nodeState.get("objectAttributes")`

#### Nested Objects with JSON Pointer Notation

For nested objects and arrays, use JSON Pointer Notation within the handlebars. JSON Pointer starts the value with a forward slash (`/`) and uses that slash as a delimiter to traverse objects (using the keys) or arrays (using the indices).

* `{{/objectAttributes/mail}}` \- retrieves `mail` from the `objectAttributes` object  
* `{{/results/0/name}}` \- retrieves the `name` from the first element of the `results` array

You may be wondering why period or bracket notation isn’t used here. There are three main reasons:

1. Existing nodes, such as the [Query Filter Decision Node](https://docs.pingidentity.com/auth-node-ref/latest/query-filter-decision.html), as well as IDM use this approach natively.  
2. Unlike period notation, JSON Pointer supports Objects and Arrays and requires less characters than bracket notation (supporting readability and easier debugging).  
3. Some Node State keys use periods *inside their key* (I’m looking at you, `PingOneProtectEvaluationNode.RISK` \- [Protect Evaluation Node](https://docs.pingidentity.com/auth-node-ref/latest/pingone/pingone-protect-evaluation.html)).

#### String Interpolation

Combine state values with other values and/or plaintext.

```javascript
"Welcome {{/objectAttributes/givenName}} {{/objectAttributes/sn}}, your risk score is {{/risk/0/score}}"
```

### Utility Functions

Add the following constants into the `CONSTANTS` section of your Custom Node:

```javascript
var CONSTANTS = {
    OBJECT_DELIMITER: "/",
    HANDLEBAR_REGEX: /\{\{(.*?)}}/g,
    HANDLEBAR_OPENER: "{{",
    HANDLEBAR_CLOSER: "}}"
};
```

Add this code into the `UTILITIES` section of your Custom Node:

```javascript
/**
 * Traverse a nested object path within a base object
 * @param {object} baseObject The base object to traverse
 * @param {Array<string>} pathArray An array of strings representing the path to traverse
 * @returns {any} The value at the end of the path, or null if not found
 */
function getNestedValue(baseObject, pathArray) {
    if (pathArray.length === 0) {
        return baseObject;
    } else {
        // Check if the baseObject is a valid object and if not try to parse it as JSON
        if (typeof baseObject !== "object" && baseObject !== null && baseObject !== undefined) {
            try {
                baseObject = JSON.parse(baseObject);
            } catch (e) {
                return null;
            }
        }
        var currentKey = pathArray[0];
        if (baseObject && baseObject[currentKey] !== undefined) {
            return getNestedValue(baseObject[currentKey], pathArray.slice(1));
        } else {
            return null;
        }
    }
}

/**
 * Stringifies the element only if necessary
 * @param {any} element The element to stringify
 * @returns {string} The stringified element
 */
function stringifyIfNeeded(element) {
    var type = typeof element;

    if (type === 'string') {
        // Already a string, return as is
        return element;
    } else if (type === 'undefined' || type === 'function') {
        // Return null or an empty string as a consistent fallback for undefined or functions
        return null;
    } else {
        // 3. For everything else (objects, arrays, numbers, booleans, null), stringify.
        return JSON.stringify(element);
    }
}

/**
 * Given an element containing handlebar values, retrieve those values from state and replace them in the element.
 * If the entire element is a single handlebars value, return the value as-is without stringifying.
 * If the value contains JSON pointer notation, traverse the nested object to retrieve the value.
 * If the value cannot be found in state, either replace with an empty string or throw an error based on the flag.
 * @param {string} element The element containing handlebars. This will always be a string
 * @param {boolean} replaceUnresolvedHandlebarsWithEmptyString Flag to replace unresolved handlebars with an empty string. If false, an error will be thrown for unresolved handlebars.
 * @returns {any} The resolved element.
 */
function resolveHandlebars(element, replaceUnresolvedHandlebarsWithEmptyString) {
    if (element) {
        var newElement = element;
        var matches = element.match(CONSTANTS.HANDLEBAR_REGEX);

        if (matches && matches.length > 0) {
            // Check if the entire element is a single handlebars value
            var isSingleHandlebar = matches.length === 1 && element === matches[0];
            
            matches.forEach(function(match) {
                // Extract the key inside the handlebars
                var handlebarsKey = match.replace(CONSTANTS.HANDLEBAR_OPENER, "").replace(CONSTANTS.HANDLEBAR_CLOSER, "").trim();
                
                // First, check if the element exists as-is in state
                // This accounts for non-nested values with dots inside the key like "PingOneProtectEvaluationNode.RISK"
                var resolvedValue = nodeState.get(handlebarsKey);
                
                // If not, check if it's a nested object using JSON pointer notation
                if ((resolvedValue === null || resolvedValue === undefined) && handlebarsKey.indexOf(CONSTANTS.OBJECT_DELIMITER) === 0) {
                    var stateValuePath = handlebarsKey.split(CONSTANTS.OBJECT_DELIMITER);
                    if (stateValuePath.length > 1) {
                        var baseObject = nodeState.get(stateValuePath[1]);
                        if (stateValuePath.length > 2) {
                            // Traverse the nested object path
                            resolvedValue = getNestedValue(baseObject, stateValuePath.slice(2));
                        } else {
                            resolvedValue = baseObject;
                        }
                    }
                }

                if (resolvedValue !== null && resolvedValue !== undefined) {
                    if (isSingleHandlebar) {
                        // If the entire element is handlebars, return the value as-is without stringifying
                        newElement = resolvedValue;
                    } else {
                        newElement = newElement.replace(match, stringifyIfNeeded(resolvedValue));
                    }
                } else if (replaceUnresolvedHandlebarsWithEmptyString) {
                    newElement = newElement.replace(match, "");
                } else {
                    throw(`Unable to resolve value for key: ${handlebarsKey} within element: ${element}`);
                }
            });
        }
        return newElement;
    } else {
        throw('Element is null or undefined');
    }
}
```

## Storing Results and Errors in State

### Use Case

It’s incredibly common for a node to return a result of an operation into state \- either a successful response of data (e.g. the [PingOne Verify Completion Decision Node](https://docs.pingidentity.com/auth-node-ref/latest/pingone/pingone-verify-completion-decision.html) returning the transaction in `pingOneVerifyTransactionId`) or a failure response with reasoning (e.g. the [PingOne Verify Evaluation Node](https://docs.pingidentity.com/auth-node-ref/latest/pingone/pingone-verify-evaluation.html) capturing failures in `pingOneVerifyEvaluationFailureReason`).

The great part about a Custom Node is that we not only can perform the same utility but we can enable the user to **specify** where that data should be stored. That way,

* If this node is being used in multiple places for different reasons we can store and reference each result separately without fear of overriding an existing value.  
* We enable users to standardize where all of their nodes return values (for example, specifying that all error messages get stored in the `errorMessage` Shared State Key for easy referencing later).

To enable storage of results and errors in state within your Custom Node:

1. Paste the below [Utility Functions](#utility-functions-1) into your node.  
2. Add the following two properties to the end of your property list (I like to keep them at the end since they aren’t directly tied to functionality and are placed in a consistent way for users to find):

| Name | Label | Description | Type | Default Value |
| :---- | :---- | :---- | :---- | :---- |
| resultSharedStateKey | Result Shared State Key | The location in which the result will be stored in Shared State. | String | result |
| errorMessageStateKey | Error Message State Key | If an error occurs, the location in Shared State where the error message will be stored. | String | errorMessage |

**Tip:** If you *always* want a result or error message to be pushed to state, enforce the parameter to be `required` from within the parameter configuration.

### Examples

Let’s say you have a Configuration Provider that displays a message containing the results of a Custom Node’s evaluation. 

You could set up your Custom Node to push the results **and** error messages to `output`. Then, the config provider would display the resulting information regardless of whether it’s a result or failure without any additional logic.

You could set up your Custom Node to push the results to `myResults` and error message to `myErrors`. That way, the config provider could display results or errors with different text depending on what it receives.

You could set up your Custom Node to push results to `myResults` and leave error messages empty \- resulting in only results and no error messages being passed to the config provider. The same could be said the other direction \- you could leave results empty and then the config provider could only get error messages.

### Utility Functions

Paste this within the `PARAMETERS` section of your `MAIN` within your Custom Node:

```javascript
var resultSharedStateKey = properties.resultSharedStateKey ? properties.resultSharedStateKey : null;
var errorMessageStateKey = properties.errorMessageStateKey ? properties.errorMessageStateKey : null;
```

After your node has performed its action successfully, add this line to your `MAIN`:

```javascript
// change `result` to the result you want to store
if (!!resultSharedStateKey) {
    nodeState.putShared(resultSharedStateKey, result);
}
```

Replace the `ERROR HANDLING` section of your `MAIN` within your Custom Node:

```javascript
catch(e) {
    //// ERROR HANDLING
    logger.error(`Error during object creation: ${e}`);
    if (!!errorMessageStateKey) {
        nodeState.putShared(errorMessageStateKey, e);
    }
    outcome = NodeOutcome.ERROR;
}
```

# Gotchas

Below are some common gotchas encountered when working with Custom Nodes.

## Property Gotchas

### Multi-Value Properties

If your property is multivalued, you’ll need to convert that property to an array before using it in your function (e.g. `properties.exampleArray.toArray()`)

### Enumerated Properties

Enumerated Values are defined in the Node property configuration as the **Key** being the *value* that returns to the script and the **Value** as the *label* that is used in the selector within the node. 

Additionally:

* The Key has restrictions on what characters it can support \- consider using `CONSTANT_CASE`.  
* The Value supports most characters \- consider using human-readable, `Title Case` with spaces.  
* The Default Value matches the Keys in the Enumerated Values.

As an example, if you wanted to create a dropdown list in which a user selected the “Message Type” when displaying a message to the user, your property configuration might look something like this:

```json
{
...
"messageType": {
    "title": "Message Type",
    "description": "The type of Message to display to the user.",
    "type": "STRING",
    "required": true,
    "defaultValue": "INFORMATION",
    "options": {
        "INFORMATION": "Information",
        "ERROR": "Error",
        "WARNING": "Warning"
    },
    "multivalued": false
}
}
```

In this setup, the user will see and be able to select the options “Information”, “Error”, and “Warning”.

When selected, the value “INFORMATION”, “ERROR”, or “WARNING” is passed into the script.

The default value is “INFORMATION”, meaning that the user will see “Information” by default and if unchanged the value “INFORMATION” will be passed into the script.

It is to your advantage to define the dropdown values as a dictionary (JSON object) in your script in which your passed value is mapped to a static value or state.

## Outcome Gotchas

* Outcomes are static and case-sensitive. If your code returns an outcome of `Success` that outcome should be defined as `Success` (**not** `success` or `SUCCESS`).  
* On a similar topic, you should make your outcomes `Title Case` with spaces to match the outcomes in a standard Journey Node. It’ll make your node look and feel like the rest on the canvas.  
* You can configure a Custom Node to return an error outcome automatically. If you do, the outcome is set as `Script Error`. Any error outcomes (`Error`, `Script Error`, etc) defined in code must still be defined in the configuration separately as the `Script Error` outcome is only returned on an error that isn’t handled. With this in mind, I suggest choosing either the built-in error or defining your own: not using both.

## Node Management Gotchas

* If you attempt to import a Custom Node with the same **`name`** or **`id`** into your tenant the import will fail. To fix, you must change the `name` and `id` within the JSON export prior to importing.  
* If you change a Custom Node, the new version of that custom node will be applied to all Journeys using it **immediately**.  
* Descriptions support the same basic HTML formatting as the content in [Page Nodes](https://docs.pingidentity.com/auth-node-ref/latest/page.html). Use that functionality to add greater detail like `<code>` blocks, links, and paragraph/list formatting. Markdown is not supported.

## Gotchas As of This Article

* Object Parameter types do not support HTML formatting in their descriptions.  
* Custom Nodes do not support versioning. Consider storing exports in an external git repository.  
* The UI forces Multi-Valued and Enumerated values to be `required` even though they technically don’t need to be. To get around this, use the API to update the node config with `required` set to `false` and the UI will work as normal.

# Conclusion

Through this Guide we have:

1. Learned when and when not to use Custom Nodes  
2. Worked through a standard structure to designing Custom Nodes and Scripted Decision Nodes  
3. Learned of some common utility functions to kickstart Custom Node development  
4. Walked through common “gotchas” when working with Custom Nodes

Custom Nodes are an incredibly powerful tool that massively improves collaborative Journey development. Knowing when and how to leverage them for your team will make you all the more effective in your IAM journey. 