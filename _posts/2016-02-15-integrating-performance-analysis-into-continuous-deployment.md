---
layout:     post
title:      Integrating Performance Analysis into Continuous Deployment
date:       2016-02-15T18:13:00-05:00
categories: workflow webperf
permalink: /integrating-performance-analysis-into-continuous-deployment
---
_This post was originally published on the [Rigor Web Performance blog](http://rigor.com/blog/2016/02/integrating-performance-analysis-continuous-deployment)._

As a performance company, we're always looking for ways to incorporate performance into our development process. With [the recent release of version 2 of Zoompf's API](http://rigor.com/blog/2016/01/announcing-the-new-zoompf-developer-api), we've been exploring methods of automating some of the manual performance analysis we do as a team. While playing with the new API endpoints, it occurred to us: when we push new code, we automatically run tests to catch functional regressions. Why can't we do the same to catch _performance_ regressions?

&gt; When we push new code, we automatically run tests to catch functional regressions. Why can't we do the same to catch performance regressions?

*Spoiler alert: we can (and did)*

To continuously analyze performance, we need two things:

  1. A tool to analyze performance (Zoompf, our performance analysis product)
  2. A way to notify said tool of code changes

We use [Semaphore CI](https://semaphoreci.com/), a hosted continuous integration service, to build and deploy our application. Semaphore's platform has a handful of integrations to enable notifications of builds and deployments. For our use case, [Semaphore's post-deploy webhooks](https://semaphoreci.com/docs/post-deploy-webhooks.html) are the answer. Post-deploy webhooks allow us to notify an arbitrary endpoint when an application is deployed, giving us the second item required for continuous performance analysis.

![Semaphore post-deploy webhook](https://files.slack.com/files-pri/T03LML849-F0M0H497B/screenshot_2016-02-11_at_2.17.32_pm.png?pub_secret=c71d3dfb86)

# Connecting the Dots

With the two ingredients in hand, all we need is a web service to receive webhooks from Semaphore and trigger performance snapshots via Zoompf's API.

To accomplish this, we built a simple [Sinatra-based](http://www.sinatrarb.com/) web service that:

  1. Receives webhook notifications from Semaphore on each staging deployment
  2. Triggers a snapshot for one of our Zoompf tests (in this case, a test against our staging app)
  3. Posts a link to the latest snapshot in Slack

Here is what our Slack notification looked like:

![Semaphore to Zoompf Slack notification](https://files.slack.com/files-pri/T03LML849-F0LVCDPKN/screenshot_2016-02-11_at_12.25.58_pm.png?pub_secret=5f4e648f4b)

With this in place, we now had automatic snapshots for each staging deployment, giving us a good idea of how each shipment impacted our performance. But receiving a Slack notification of a new snapshot isn't all that helpful. In order to see what changed, we had to click the link and manually inspect our performance test results. Not only that, we were getting a lot of noise in our Slack channel, as our staging environment gets deployed several times a day.

# Detecting Regressions

To avoid manual inspection and cut down on noisy notifications, we decided to automate the regression detection. Using [Zoompf's snapshots API](https://api.zoompf.com/Explorer/index.html?url=/swagger/docs/v2#!/Snapshots/v2_Snapshots_Get_0), we can retrieve all snapshots for a given test. To detect changes, all we need to do is compare the latest snapshot to the previous snapshot.

![Zoompf Snapshots API](https://files.slack.com/files-pri/T03LML849-F0M0JT9RN/screenshot_2016-02-11_at_2.20.00_pm.png?pub_secret=a61feff336)

The API has a couple of handy parameters to make this easy: `p_per_page` and `p_order_by`. These parameters allow you to specify the number of snapshots you want to see and sort by a given attribute, respectively. For our use case, we only need the two most recent snapshots, so we can set `p.per_page=2` and `p.order_by=ScanAddedUTC`. Here is an example of what that request looks like:

```bash
curl "https://api.zoompf.com/v2/tests/:test_id/snapshots?p.per_page=2&amp;p.order_by=ScanAddedUTC"
```

Armed with the two latest snapshots, comparing them is easy. In our case, we compare the Zoompf scores of each snapshot to measure the change. However, automating this comparison within our web service required us to make some changes. Instead of simply triggering a new snapshot, we now have to:

  1. Trigger a snapshot
  2. Wait until the snapshot is complete (i.e. poll the snapshot's status)
  3. Get the latest two snapshots and compare their Zoompf scores

The first version of our web service triggered the snapshot to Zoompf within the request/response cycle. This was a quick solution for our original needs, but it wasn't ideal. Adding the logic required for automated regression detection would have introduced a fair amount of overhead that would bog down the web server. To avoid this problem, we added [Sidekiq](http://sidekiq.org/), a Redis-backed asynchronous worker framework written in Ruby, to our application. Moving the core logic into asynchronous workers shifted the bulk of the work out of the request/response cycle, keeping our web server fast and responsive.

With the Sidekiq changes added, our web service now:

  1. Receives webhook notifications from Semaphore on each staging deployment
  2. Enqueues a Sidekiq worker
  3. Returns [a 202 "Accepted"](http://whysitedown.com/http_codes/202) response

And our Sidekiq worker:

  1. Triggers a performance snapshot
  2. Waits until the snapshot is complete
  3. Gets the latest two snapshots and compares their Zoompf scores
  4. Posts in Slack if performance has regressed (or improved)

Here is what the  Slack notifications look like with the regression detection in place:

![Regression detection Slack notification](https://files.slack.com/files-pri/T03LML849-F0LMKQRTK/screenshot_2016-02-09_at_10.27.36_am.png?pub_secret=efad4a5a92)

The regression detection update yields much more useful Slack notifications. If a staging deployment causes a performance regression (or improvement), we'll get notified immediately via Slack. This notification links to the comparison of the last two snapshots in Zoompf, giving us one-click access to the performance changes. We can also click on the "Commit" link to see what code change was deployed by Semaphore, reducing the steps necessary for tracking down the root cause of any regressions.

Furthermore, the new workflow _reduces_ the number Slack notifications by suppressing snapshots that did not impact performance. As anyone who's ever been on call knows, figuring out what notifications _not_ to send is important for avoiding alert fatigue.

# Conclusion
How should a continuous performance analysis tool work? We identified the following useful features:
  * Automated performance analysis on every successful deployment
  * Detection of regressions (and improvements) in the latest version of the application
  * Integration with notification tools (Slack, in our case)

Automating the performance analysis process has helped our team by:
  * Reducing time spent manually inspecting performance
  * Improving code coverage from a performance standpoint (i.e. it **guarantees** that all changes trigger a performance analysis)
