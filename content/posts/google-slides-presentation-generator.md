---
draft: false
date: '2025-07-09'
title: 'Google Slides Presentation Generator'
description: 'Create a centralized branded portfolio of content that can auto-generate to meet your team’s demo, poc, and presentation needs.'
summary: Learn how to make, manage, and deploy a dynamic generator for templated Google Slides.
categories: ["Google Apps Script"]
tags: ["Google Slides"]
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

Tell me if you’ve seen this before: you, a team member, a vendor, a consultant, et al, pulls up a slide deck to share and it’s all over the place. The dates on some slides are dated from years ago, the branding is inconsistent, the content is outdated and slides contradict themselves - all in all, it’s a mess. It’s clear that whoever is presenting pulled together a hodgepodge of slides that they found from old file shares, team folders, web searches, or AI prompts.

This isn’t an unknown problem - the larger a business gets, with more products and people working on those products, the more content is produced to cover what it is and how it works. Entire teams develop standard practices to maintain a company’s branding and aesthetic and are checked by other teams to ensure that their products are represented correctly.

However, it’s also common that this content hits a saturation point that’s hard to keep up with, inevitably ending up where people on the team don’t know where to go to gather specific information for the content they need for their end-audience. Do I use *MASTER-SLIDE-FINAL_A*, or *PRODUCT_FEATURE_ANNOUNCEMENT_NEW*?

With a bit of help from Google Apps Script, we can make this process simpler, both for our content makers and for our content consumers.

## Google Slides Presentation Generator

This How-To shows you how you can enable your team to **rapidly generate custom presentations** **from a centralized presentation template** maintained by your content team. It is broken down into the following parts:

1. [Building the Slide Template](#building-the-slide-template)  
2. [Building the Form](#building-the-form)  
3. [Adding The Sheet](#adding-the-sheet)  
4. [Adding the Script](#adding-the-script)  
5. [Generating Slides](#generating-slides)  
6. [Extending the Generator](#extending-the-generator)

When done, your content makers will only have to update one slide deck for all content and your team will have a single form to create whatever presentation they need in a matter of seconds. You’ll learn how to add custom variables that can be filled out automatically or by your presenters as well as how to dynamically reformat slides given the content provided by a Form. The generator itself follows class inheritance so that you can extend this approach to other Google Drive objects like Google Docs or Google Sheets.

# Building the Slide Template

To begin, we’ll need the base slide deck that we’ll be referencing when generating content. This is the deck that your content creators will be updating moving forward.

Since this How-To is about template generation, and not theming, the slides provided won’t be branded in any way. You can add whatever themes/brands/designs you’d like later - changing the design will not impact the generator.

Let’s start with a blank presentation. To make finding this generator and its associated content easier I’ve put everything into a single folder.

![A screenshot of the right-click menu in a Google Drive folder where Google Slides and Blank Presentation have been selected](../images/google-slides-presentation-generator/creating-the-template.png)  
*Creating the New Google Slide Template*

I’ve named this file “Slide Template” but you can name this whatever you’d like.

## Defining Dynamic Variables

Since your presentation is likely going to be presented *to* someone, let’s put in a variable that defines who we are talking to. Our generator uses Handlebars syntax, which basically means that when we want a custom-generated value we are going to type two curly-braces surrounding the name of that custom value. In this example, we’re going to use `CompanyName` as the variable that holds the name of the company, which will look like `{{CompanyName}}` on the slide.

![A screenshot of the title slide of your presentation in which the CompanyName variable has been added](../images/google-slides-presentation-generator/first-variable.png)  
*Adding Your First Variable*

Presentations often have the year marked on the bottom of the slide, which helps us identify the recency of data presented. Since this slide will be dynamically generated, we’ll want to auto-populate that with the current year - so let’s use another variable! This time we’re adding `{{Year}}` in a text box at the bottom of the page. **Don’t use the Slide Theme for this - we won’t be able to target it in our script.**

![A screenshot of the title slide in which a footer, including the year variable, has been added](../images/google-slides-presentation-generator/footer-variable.png)  
*Adding a Dynamic Footer*

If you expect to use the same footer on every page, make it a group so that you can copy/paste more easily.

You can repeat this process for any other variables you’d like to dynamically add - think content that your presenter should know but will be different per presentation (like date presented, presenters, start/end dates, unique URLs). Save these variable names - you’ll want a way for your team to reference what values are available to them when building the slides.

## Defining Dynamic Sections

Let’s create some sections. While Google Slides doesn’t natively support sections like PowerPoint, we can use variables to define what is a section so that when generating content we know what to show or hide.

I’d like a section on Gizmos, Widgets, and Gadgets. To do so, I’m going to put a tag on the page that indicates the slide is for that section. Just like before, we’ll use Handlebars. Our formatting is `{{tag_slide}}`, where “tag” is changed to whatever text tag you’d like to designate this section. In my case I’m using `{{gizmos_slide}}`, `{{widgets_slide}}`, and `{{gadgets_slide}}`, and I’m putting the tag at the center-bottom of the slide right over my footer. While you don’t have to put it there, just make sure you put your tag in a place that is unobtrusive while designing but easy for your designers to find and change depending on what section they’re on. Like before, save your slide tag names.

![A screenshot of the presentation where 3 additional slides have been added, each of which has a variable indicating that is is part of a section](../images/google-slides-presentation-generator/section-variable.png)  
*Adding Sections*

Notice that each of my sections are using different slide formats. As long as you have that tag somewhere on the page, it doesn’t matter what the slide looks like.

## Defining Dynamic Table of Contents

We probably want a table of contents that lists all of the different sections we’ll be presenting. With that in mind, add a new slide 2 with the variable TableOfContents. I titled mine “Agenda” but that’s up to you.

![A screenshot of the Agenda slide in which the TableOfContents variable has been added](../images/google-slides-presentation-generator/toc-variable.png)
*Table of Contents (Agenda)*

# Building the Form

So now that we have a baseline slide template that we want to generate from, let’s build the form that your team will fill out to generate their own bespoke presentation.

Back in the folder you created, go ahead and create a new Blank Form. I’ve named this form “Slide Generator” but you can name it whatever you’d like.

![A screenshot of the right-click menu in a Google Drive folder where Google Forms and Blank form have been selected](../images/google-slides-presentation-generator/creating-the-form.png)  
*Creating the Form*

Inside the form we are going to need to capture a few things:

1. The email address of the person who’s requesting it to be generated, so that we can make them the owner of their new deck  
2. The custom values that we’ll be populating the presentation with  
3. The sections that they want in their presentation

## Getting the Email

First off - let’s get the email. You may have this setting already applied, but make sure you’ve set your Form defaults (under Settings - Defaults) to collect email addresses. You know this is applied when you see the comment “This form is automatically collecting emails from all respondents” at the top of your form.

![A screenshot of the form defaults in which collecting email addresses is set to true](../images/google-slides-presentation-generator/form-defaults.png)  
![A screenshot of the description on the top of the Google Form that indicates it is automatically collecting respondent emails](../images/google-slides-presentation-generator/email-automatic.png)
*Collecting Email in the Form*

## Getting the Variables

Next, let’s populate some values. Here are the defaults that I’ve started with:

| Name | Type | Description | Required |
| ----- | ----- | ----- | ----- |
| File Name | Short Answer | The name of the file(s) generated, formatted "Presentation: **Your File Name**" | Yes |
| Company Name | Short Answer | The name of the company you’re presenting to | Yes |

![A screenshot of the Google Form editor with the File Name and Company Name variables added](../images/google-slides-presentation-generator/form-variables.png)  
*The form with some variables populated*

### Selecting a Folder

While allowing your team to specify a folder where they want their presentation to be generated is optional, it’s a nice way to save them an extra step of moving their file around. Since a folder picker isn’t supported in Google Forms, we’ll use the folder ID, which is on your Google Folder’s URL, instead.

First off, create a folder inside your other folder where you want the presentations to be generated by default. I’ve named mine “Generated Presentations”. Copy the ID of that folder, which you’ll find at the end of your Drive URL, for later.

![A screenshot of the Generated Presentations folder and the folder URL in which the ID is highlighted](../images/google-slides-presentation-generator/folder-id.png) 
*The Generated Presentations Folder*

Next: add the following question, being sure to change the values for your email address and drive folder link in the description.

| Name | Type | Description | Required |
| ----- | ----- | ----- | ----- |
| Folder ID | Short Answer | The ID of the folder these docs will be generated to, e.g. *https://drive.google.com/drive/folders/***ID  Share EDIT permission to this folder to [add your email] for the file to generate.**  If no folder is provided, generated documents will appear here:  [Add a Link to your Folder]  | No |

![A screenshot of the folder ID being collected within the form editor](../images/google-slides-presentation-generator/form-folder-id.png) 
*The Folder ID Variable*

## Getting the Sections to Generate

Our last question will be used to identify what sections your presenter wants to generate. At the bottom of your form, add the following question:

| Name | Type | Description | Required |
| ----- | ----- | ----- | ----- |
| Sections | Checkboxes | Select which sections you’d like in your presentation  | Yes |

Since I created the Gizmos, Widgets, and Gadgets sections I’ll add them here.

![A screenshot of the Checkbox field added to the Form that contains the sections](../images/google-slides-presentation-generator/form-sections.png)  
*Adding the Sections in the Form*

While not necessary, you may want to set a rule that ensures at least one section is selected. The generator isn’t that useful if it generates an empty deck! 

To do so, click the three dot menu next to “Required”, check “Response Validation”, and set the rule to “Select at least” 1. 

![A screenshot of the Form validation added to the Sections field that validates at least one selection must be selected.](../images/google-slides-presentation-generator/form-section-validation.png)
*Section Form Validation*

# Adding the Sheet

Now that we have our template and form ready to go, let’s link it up to our sheet that will run the generator script.

Under the Responses tab of your Form editor, click the link “Link to Sheets”. We’ll be using a Google Sheet so that we can track things like links to the generated documents and the status of the generation. When prompted, create a new spreadsheet and name it whatever you’d like - I’m using the default.

![A screenshot of the Form responses tab in which the Link to Sheets button is highlighted](../images/google-slides-presentation-generator/form-link-sheet.png)  
*Link to Sheets*

Once created, you should be redirected to the Google Sheet. Add two extra columns - one called “Generation Status” and the other called “File Link(s)”. It should look something like this:

![A screenshot of the initial linked sheet when created](../images/google-slides-presentation-generator/sheet-initial.png)  
*The Initial Spreadsheet*

To make it easier to update in the future, we’re going to use Sheet Tabs (i.e. “Sheets”) to store configuration information like variables and sections. This is what our script will look at to know how to read the form responses and populate default values.

At the bottom of the page click the plus button next to the tab that says “Form Responses 1” and add the following two sheets: **Variables** and **Sections**. Your sheet list should look like this:

![A screenshot of the Sheets (tabs) for Variables and Sections added to the linked spreadsheet](../images/google-slides-presentation-generator/sheet-sections.png)
*Adding the Sheets to the Spreadsheet*

**Note: Case and spelling is important here - make sure that *Variables* and *Sections* are written exactly as they appear here.**

## The Variables Sheet

The Variables sheet holds information about what variables we use in our generation script and inside our presentation - the ones we created in the [previous section](#defining-dynamic-variables). Values can be populated from your form or from a default value that you define here.

In the Variables sheet, add the following data:

| Variable | Default | Form Key | Form Index |
| :---- | :---- | :---- | ----- |
| TimeStamp |  | Timestamp | 0 |
| PresenterEmail |  | Email Address | 1 |
| FileName | My Presentation | File Name | 2 |
| FolderId |  | Folder ID | 3 |
| CompanyName | Customer | Company Name | 4 |
| Sections |  | Sections | 5 |
| Status |  | Generation Status | 6 |
| FileLinks |  | File Link(s) | 7 |
| TemplateTypes | Slides |  | -1 |
| SlideTemplateId |  |  | -1 |
| DocumentTemplateId |  |  | -1 |
| Year |  |  | -1 |

For the Form Index column, I like to use the following formula to auto-populate the number (add it to the second row and then hit “auto-apply” or drag the formula down to all of the rows):

```
=IFNA(MATCH(C2,'Form Responses 1'!$1:$1,0) - 1,-1)
```

That way, if the form or the spreadsheet is edited in any way the index value is still the same.

If you want to add a new variable, give it a unique variable name and put that name in the Variable column. 

* If you add a question to the form that populates this variable, put in the Form Key and the Form Index will auto-set for you.   
* If you give the variable a default value, and this form value isn’t populated, the default is used instead.   
* If you have a static variable you’d like to make sure everyone has, add it with a default value and no Form Key and it’ll be applied automatically.

For example, if I wanted to put a Presented On Date into my slides I might:

1. Create a Date input in my form called “Presentation Date”  
2. Add a row with the variable “PresentedOn” and the Form Key “Presentation Date”

Now, I can use the `{{PresentedOn}}` variable in my slides!

We’re going to need to point our generator to the right template. To do so, copy your Google Slides Template ID found in your URL and paste it into your default value for `SlideTemplateId`. The part you need to copy is bolded in this example link:

`https://docs.google.com/presentation/d/`**SlideTemplateId**

If you created a default folder, you’ll want to do the same thing for your `FolderId` variable - the part of the url is bolded in this example:

`https://drive.google.com/drive/folders/`**FolderId**

## The Sections Sheet

The Sections sheet holds information about what sections we want to show and hide and what slides those sections are related to based on the tags we set in the [previous section](#defining-dynamic-sections).

In the Sections sheet, add the following data:

| Form Value | Tag | Title |
| :---- | :---- | :---- |
| Gizmos | gizmos | Gizmos are Good |
| Widgets | widgets | Widgets are Weird |
| Gadgets | gadgets | Gadgets are Great |

The “Form Value” matches the Checkboxes we created in our form. The “Tag” links those sections to the appropriate slide (e.g. `{{gizmos_slide}}`). The “Title” is what we are going to show in our Table of Contents or wherever else we need a list of our sections.

If you want a new section:

1. Add a new value to the checkboxes in your Form  
2. Add that same value in the Form Value column with its own unique tag and title  
3. Use that tag to show/hide slides in your deck

For example, if I wanted a new section called “Thingamajigs”, I could add “Thingamajigs” to the Sections checkboxes in my form, create a new row with “Thingamajigs”, “thingamajigs”, and “Thingamajigs are Tacky”, and then I can set my sections in my slides with `{{thingamajigs_slide}}`.

# Adding the Script

So we have our template, we have a place to track our responses, and we have a means to map our responses to what is in our template - let’s put this all together and generate our slides!

## Load the Scripts

Inside your Spreadsheet up at the top of the page, click on Extensions → Apps Script to enter the script editor. 

![A screenshot of the "Extensions" tab in the top of the Google Sheet where Apps Script has been selected](../images/google-slides-presentation-generator/sheet-add-script.png)  
*Creating the Script*

Name your project something memorable - I’m calling mine Template Generator. Using the “+” button next to the “Files” tab, add in the files below (repository [here](https://github.com/gwizdala/google-template-generator/tree/main)):

| File (Linked) | Purpose |
| :---- | :---- |
| [Main.gs](https://github.com/gwizdala/google-template-generator/blob/main/apps-script/Main.gs) | Handles the main processing of the form.  |
| [Template.gs](https://github.com/gwizdala/google-template-generator/blob/main/apps-script/Template.gs) | The generic Template object. Allows extensions of this script to handle other Google Drive file types, like Google Docs. |
| [SlideTemplate.gs](https://github.com/gwizdala/google-template-generator/blob/main/apps-script/SlideTemplate.gs) | The extension of the Template that handles Google Slides. |
| [Utilities.gs](https://github.com/gwizdala/google-template-generator/blob/main/apps-script/Utilities.gs) | Generic helpful utility functions, ranging from sending email templates, to reading spreadsheets, to calculating durations between dates |
| [SuccessMessage.html](https://github.com/gwizdala/google-template-generator/blob/main/apps-script/SuccessMessage.html) | The email template sent to the user upon successful generation |
| [ErrorMessage.html](https://github.com/gwizdala/google-template-generator/blob/main/apps-script/ErrorMessage.html) | The email template sent to the admin upon failed generation |

Your tab list should look like this:

![A screenshot of the imported scripts into the Files section of the Apps Script](../images/google-slides-presentation-generator/script-list.png)  
*The Imported Scripts*

## Add the Trigger

Once your scripts are imported we’ll want to add our trigger so that the generator runs every time a new Form response is submitted. To do so, head to the “Main” file and run the function `attachTrigger`. You’ll be prompted to authorize a series of permissions that let you clone the template to a folder, modify the cloned template, update the generator sheet, and send a success/failure email.

![A screenshot of the "attachTrigger" script being run from the "Main.gs" file within the Apps Script Editor](../images/google-slides-presentation-generator/script-run-trigger.png)  
*Running the Trigger*

![A screenshot of the "Authorization required" prompt appearing from google before running the script](../images/google-slides-presentation-generator/script-permissions.png)
![A screenshot of the list of permissions you are required to authorize to run this script](../images/google-slides-presentation-generator/script-permissions-list.png)  
*Authorizing the Trigger*

After running this function, you should see an execution log indicating that the process ran as well as a “From spreadsheet - On form submit” trigger loaded for your project (you can see that in the “Triggers” section, indicated by a clock icon).

![A screenshot of the execution log that shows the attachTrigger function starting and completing](../images/google-slides-presentation-generator/script-execution.png)
![A screenshot of the Triggers view in the Apps Script editor in which the From spreadsheet - On form submit trigger has been added](../images/google-slides-presentation-generator/script-trigger-loaded.png)  
*The Loaded Trigger*

# Generating Slides

Our hard work is over, let’s get generating!

In the top-right corner of your Form, click the “Publish” button, set your sharing preferences, and copy your responder link.

Back in your Slides, add a new slide right at the beginning that teaches people how to generate. I usually add the following text: “How to Use: Fill out **This Form** to receive a generated template.” replacing the bolded underlined section with your unique responder link.

![A screenshot of the new first slide in your template that links out to the generator form](../images/google-slides-presentation-generator/slide-howto.png)  
*The First Slide in Your Template*

Have your presenters bookmark this presentation. That way if they need a new deck, they can click the link, fill out the form, and in a matter of seconds receive a slide deck customized just for them!

![A screenshot of the Slide Generator form being filled out with details from the presenter](../images/google-slides-presentation-generator/generator-form.png)  
*Filling out the form*

![A screenshot of the successful generation within the Google Sheet](../images/google-slides-presentation-generator/generator-sheet.png)  
*The Results in the Sheet*

![A screenshot of the generation results and the ownership change sent to the presenter's email](../images/google-slides-presentation-generator/generator-emails.png)  
![A screenshot of the generation results email that includes the list of generated files](../images/google-slides-presentation-generator/generator-emailbody.png)  
*The User’s Emails*

![A screenshot of the generated presentation](../images/google-slides-presentation-generator/generator-slides.png)  
*The Generated Presentation*

# Extending the Generator

The generator is built to be extended to fit whatever use cases you and your organization requires. That being said, here are some things built-in that you can try:

1. As mentioned before, add as many variables and sections as you want - as you update the Google Sheet, the generator will adapt automatically.  
2. In `Main.gs`, you’ll notice there are some additional commented-out arguments for start and end date. If you’re presenting on a topic that has a beginning and end (like a POC), add those arguments to your form and use the existing utility function called `calculateDuration` to get a duration between those two dates in days.  
3. Also in `Main` you’ll see that there’s another case commented out other than `Slides`. The generator is built using a parent `Template` class - meaning that you can create a generator for Google Docs, Google Sheets, Google Forms, etc. using the functions extended from that base class. As you add more classes, put them into the switch statement to be generated - and everything else will just work as expected.

# Conclusion

You now have a way to create dynamic content for your presenters based on a centralized template managed by your content team. This approach ensures that your branding and product information remains up-to-date while providing your audience with information catered to their exact needs. As your business grows, you can grow the template to match your requirements - and it’s as simple as adding a row to a spreadsheet.

The code required for generation can be found at [this Github repository](https://github.com/gwizdala/google-template-generator/tree/main). If you build extensions for other document types, put in a pull request! I’d love to keep making this system better.