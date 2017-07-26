---
layout:     post
title:      The Hidden Cost of Web Components
date:       2014-12-09T17:21:00-05:00
categories: webperf
permalink: /the-hidden-cost-of-web-components
---
_This post was originally published on the [Rigor Web Performance blog](http://rigor.com/blog/2014/12/hidden-costs-web-components). It is based on a talk I gave at the [Atlanta Web Performance Meetup](http://www.meetup.com/Atlanta-Web-Performance-Group/). [Here are the slides from that talk](http://www.slideshare.net/kconarro14/the-hidden-cost-of-web-components-atl-web-performance-meetup)._

Modern websites make a lot of requests. [And I mean a lot](http://httparchive.org/interesting.php#reqTotal). And many of these requests are to third-party resources. As [this trend continues](http://httparchive.org/trends.php#bytesTotal&amp;amp;reqTotal), it is important to routinely analyze the performance cost of your site's resources to identify areas for optimization.

One approach to such an analysis would be to aggregate requests at the domain-level. Using raw HAR data, the data that underlies [the popular waterfall chart](http://www.softwareishard.com/blog/har-viewer/), we can calculate the performance cost of each domain that our site uses.

Using [this HAR as an example](http://insights.rigor.com/example.har), our domain analysis for the five slowest domains would look like this:

![CNN Domain Analysis](http://rigor.com/wp-content/uploads/2014/11/screenshot-2014-11-26-at-1.38.31-PM-e1417027172979.png)

This approach makes it obvious which domains contribute the most to our overall load time. But now what? One option is to eliminate requests to a given domain to reduce its cost. For example, let's remove all requests to _z.cdn.turner.com_. A quick scan of the page source reveals nine references to this domain:

![CNN Page Source](https://docs.google.com/a/rigor.com/uc?id=0B4OqDVTQ1tMPQ1d3WmFYZHRNT3c)

Removing all nine of these should do the trick, right? Unfortunately, no. Looking back at our domain analysis, there are actually 30 requests being made to this domain. So where are the other 11 requests coming from?

## Tracking down requests with HTTP Referer

To find the 11 other requests, we can use the [HTTP Referer request header](http://en.wikipedia.org/wiki/HTTP_referer) to reevaluate our HAR data. This header [identifies the resource responsible for making a given request](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.36). Here is what the referer analysis looks like for our example HAR:

![CNN Referer Analysis](https://docs.google.com/a/rigor.com/uc?id=0B4OqDVTQ1tMPTU5GVUFMcFE2LTg)

Instead of aggregating requests by domain, we can now see the resources responsible for the majority of the site's requests. Not surprisingly, the base page (cnn.com, in this case) is often the main referer. But scanning the table reveals other expensive components, one of which is a resource loaded from _z.cdn.turner.com_. Expanding this referer reveals several requests to _z.cdn.turner.com_ that we weren't able to find in the page source:

![Second referer](https://docs.google.com/a/rigor.com/uc?id=0B4OqDVTQ1tMPN2R2TjZNSWcwQnc)

To make this new analysis even more powerful, we can search for all resources referering to or from _z.cdn.turner.com_. Any resource matching the search is either requesting additional resources from that domain or is hosted on that domain. Here is what our search results would look like using our same example HAR:

![Referer search](https://docs.google.com/a/rigor.com/uc?id=0B4OqDVTQ1tMPVHVBN0JjX2cxWms)

Using the power of HTTP Referer, we can now assign costs to each component we add to our site by seeing how many requests it makes. Instead of treating a new JavaScript library as a single resource, for example, we can now include all the dependent resources it requests in our cost analysis, giving us more insight into the cost of a given file.

To simplify this type of analysis, we've created a simple tool at [insights.rigor.com](http://insights.rigor.com). Simply upload a HAR file, and the tool will generate domain and referer reports to help you identify costly components. Next time you are adding resources to your website, consider using HTTP Referer to combat bloat and slow load times.
