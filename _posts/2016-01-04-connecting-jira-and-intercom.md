---
layout:     post
title:      Connecting JIRA and Intercom
date:       2016-01-04T11:17:00-05:00
summary:    Using webhooks to link customer support cases and development tasks
categories: workflow support
permalink: /connecting-jira-and-intercom
---

At [Rigor](http://rigor.com) we use [JIRA](https://www.atlassian.com/software/jira) to track our development tasks and [Intercom](https://www.intercom.io/) to handle customer support. When a support case comes in that requires development work, we create an issue in JIRA. To connect the systems, we add a private note to any related Intercom support cases with a link to the issue in JIRA.

[As we've grown](https://twitter.com/TeamRigor/status/657253617976672256), it's gotten more difficult to keep these two systems in sync. To automate some of the manual effort, I built [a Sinatra-based web service to connect JIRA and Intercom](https://github.com/kconarro14/jira_intercom_webhook).

## How it works
1. Deploy the web service to your favorite platform (we use [Heroku](https://www.heroku.com/))
[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/kconarro14/jira_intercom_webhook)

1. [Add the web service as a webhook in JIRA](https://developer.atlassian.com/jiradev/jira-apis/webhooks#Webhooks-jiraadmin) and register the "issue created" and "issue updated" events
![Register webhook events](https://silvrback.s3.amazonaws.com/uploads/93f8ad1d-4e57-42b0-ba6c-39095558a776/WebHooks_-_JIRA_medium.jpg)

1. Include a link to an Intercom conversation in your JIRA issue descriptions
![Create a JIRA issue](https://silvrback.s3.amazonaws.com/uploads/b96cfcc3-e6bb-4405-9894-43be1115b568/jira-intercom_medium.jpg)

1. A private note will be posted to the Intercom conversation with a link to the JIRA ticket created in step 2
![Automated Intercom private note](https://silvrback.s3.amazonaws.com/uploads/17ccdf49-6071-437b-bd26-4f6979401687/intercom-jira-note_medium.jpg)

*For more on setup and configuration, see [the project's README](https://github.com/kconarro14/jira_intercom_webhook/blob/master/README.md).*

## What's next
Currently the web service handles the `jira:issue_created` and `jira:issue_updated` webhook events and looks for Intercom URLs in the issue description. Future enhancements might include:

* Listening for new or updated comments that include Intercom links
* Adding support for [post-functions](https://developer.atlassian.com/jiradev/jira-apis/webhooks#Webhooks-Addingawebhookasapostfunctiontoaworkflow) to add Intercom notes when a linked JIRA issue's status changes
* Tagging Intercom conversations with the issue ID to simplify finding all conversations related to a specific JIRA issue (Intercom doesn't support adding tags via API as of yet)

I'll post blog updates as any major features are added, but be sure to check out [the project on Github](https://github.com/kconarro14/jira_intercom_webhook) for updates.
