---
layout: post
title: "The Devex NewsLab: Using Google Docs as a CMS"
date: 2015-01-12 13:00:00 +0100
author: dsager
comments: true
categories:
- Insight

---

This article describes our current content management solution and gives some insight on the parts of the implementation we think are interesting.

As you might know a big part of our website is dedicated to [News content covering various aspects of international development](https://www.devex.com/news).
At Devex we usually publish around 10 articles per day which are sometimes planned and worked on several days or even weeks in advance.
There are different people involved in the process including both internal and external writers, editors, researchers or marketing folks.
Some of them contribute to the article's content while others just need to be aware what is going on to coordinate their own work like social media campaigns.

<!-- more -->

## Where we come from

Editors maintained individual spreadsheets to schedule and coordinate the work of several writers.
These spreadsheets contained article meta data like estimated publication date, category, assigned persons or the newsletter this article should appear in.
A writer would work on an assigned article as a Word document and send the draft to editors or collaborating writers.
Changes were made to the article, a new version was saved (e.g. `awesome_article_v7.doc`) and sent again via email.
I guess I don't have to elaborate on the downsides of this workflow - just think briefly about transparency, revision control or the overhead of individual email and file organization...

Once an article reached a "to be published" state the editor would send it to one of our web publishers who then would extract and upload the images to Amazon S3, copy the content to a web-based CMS and format it (TinyMCE).
If everything goes well, the article could be published and would appear on the website.
Any further editorial changes or fixes needed to be done in the web CMS.
Again I think that the pitfalls of this publishing workflow are obvious (steps that are repeated manually up to 10 times a day, lack of transparency, data inconsistency and so forth).

## Where we figured we should be

When we sat down to think about requirements of a new solution, we came up with the following:

- Everyone needs easy access to the article content in progress
- Everyone should be able to publish an article (i.e. decrease the overhead)
- Any change to the article content needs to be recorded (revision history)
- One general overview of all articles being worked on
- Transparency regarding the assignments or estimated publication dates
- Improved task overview for individuals (i.e. personalized task list)
- Each step of the workflow should be accessible as a mobile-friendly version
- Unify the look and feel of an article's presentation on the website
- Writers and editors shouldn't need to know HTML or CSS

While regarding several CMS solutions we stumbled upon a
[Google Apps case study about the Huffington Post](https://docs.google.com/fileview?id=0B5Y-fwYJF2hLNzM5ZTIwODMtNzgwOS00MzYyLWE0MTYtMDJlNTBjYjZmMzMy)
which states that their blog and editorial teams use Google Docs for collaboration and share editorial calendars.
Giving this some thought it seemed like the perfect solution:

- Editors and writers can work with a "Word-like" UI they are used to
- All changes are saved instantly in the cloud and each document has it's own revision history
- Sharing and access management is fairly easy, no more email attachments!
- Google Docs can be exported to HTML or ODF (i.e. XML), so the content can be processed by an external app
- By using shared calendar events every person gets his personal agenda
- There are official mobile applications from Google for editing documents or checking your calendar

Great, what are we waiting for, let's do it!

## How we got there

The first problem we encountered was pretty obvious: We are not using Google Apps!
So migrating our roughly 120 users to Google Apps was the first step (which was considered for some time anyway).
I spare you the details...

To give our News team an idea of how they would be working in the future we put together a prototype using various Google services:
A group for the editorial staff, a metadata spreadsheet, some calendars and a folder for articles.
All tied together by a simple Google site and a web application written in Google Apps Script.
This prototype was far from perfect as it was rather slow and the final article still had to be copied manually in order to be published.
Also the spreadsheet was not providing the level of overview we had in mind.
But we were able to get early feedback from the writers and editors and also gain valuable experience with the Google Apps ecosystem.

The next step was to link the Ruby application powering our website to the Google services.
This integration is currently based on three different types of communication with Google APIs:

- **Manage documents**: As soon as someone creates an article on our website an API call to Google Drive creates a corresponding document.
  The unique document Id is then stored with the article object in our database.
- **Manage calendar events**: Each article can have a deadline (when it should be completed by the writer) and an estimated publication date.
  An API call creates these events in a specific Google calendar and sends invites to the people who work on the article (assignees, editors & watchers).
  In case the dates change (e.g. an article is postponed) the calendar events are updated accordingly.
- **Process the content**: For each article there's a button on our website called "Pull latest Content from Google".
  Clicking it does exactly what it says, it grabs an HTML export of the document, processes it and saves a clean HTML snippet that can be displayed on our site.
  The processing detects various elements like images, video (YouTube & Vimeo) or slideshow links, alignments, quotes or headings.
  For example have a look at [this document](https://docs.google.com/document/d/10pXRPT9J4VzvfdR5vSj4-NJWfHVRwkGAdSn7S4Ew-xQ/edit?usp=sharing) which results in [this article](https://www.devex.com/news/kennedy-johnson-and-the-early-years-83339).
  The content processing is continuously tested using a reference document to detect changes in the HTML structure provided by Google.

Apart from the integration with Google the application has some additional features like notifications for writers, editors and watchers.
It also provides the general overview via filterable list and calendar views.

So far we are really happy with our new CMS although we have heaps of improvements in our Backlog.
Since we deployed it 10 months ago over 2,200 articles were created and published.
