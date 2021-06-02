---
title: Rate Limiter Design & Strategy
date: 2021-05-29 08:49:30 +0530
categories: [System Design, Rate Limiter, Design & Strategy]
tags: [api-gateway, design]
comments: true
seo:
  date_modified: 2021-06-03 04:35:18 +0530
---

## Overview
{: .themeBlue}

The current post explains about the design and strategy of Rate Limiter. I have also created a separate post for implementation of Rate Limiter using Java [here]({% post_url 2021-05-29-rate-limiter-implementation %})

Rate limiting is an API's configured behavior to reject sender's request when the number of requests over a window of time crosses a limit or threshold.

> **Example 1 : An API which generates an authentication token can cap the maximum number of token generation request sent by a specific client(or/and from a specific ip address) in a period of 1 hour. Let's say 1200 requests/hour. If the client sends more than 1200 requests in a span of 1 hour the server will reject the request with  HTTP 429 response code.**
{: .themeBlue}

> **Example 2 : As an API service provider one can set different rate-limiting threshold for different type of customers(paid/free).**
{: .themeBlue}

> **Example 3 : As an API service provider one can set maximum number of failed requests in an hour, let's say if user makes more than 3 failed credit card transactions in an hour, the server will reject the request.**
{: .themeBlue}

In terms of rate-limiting, following scenarios can take place when a sender sends a request to an API
1. _**Rate Limit not reached >>**_  In this case, the API sends a valid response(2xx, 5xx etc.) as per the request. Additionally, API can also send few headers which provide information about the current status of rate-limit like :

```
HTTP/1.1 200 OK
X-Rate-Limit-Limit: 1200
X-Rate-Limit-Remaining: 100
X-Rate-Limit-Reset: 1622603392
```

In the above example, the rate limit threshold for the API is set to 1200 and user can send only 100 more requests before Wednesday, 2 June 2021 3:09:52 AM GMT(1622603392) when the rate limit window gets reset.

2. _**Rate Limit reached >>**_ In this case, the API sends a 429 HTTP Code(Too Many Request). Additionally, API can also provide information about when this limit will be reset in the header.

```
HTTP/1.1 429
X-Rate-Limit-Limit: 1200
X-Rate-Limit-Remaining: 0
X-Rate-Limit-Reset: 1622603529
```

In the above example, the rate limit threshold for the API is set to 1200 and has already been reached. So, user can't make any more request before Wednesday, 2 June 2021 3:12:09 AM GMT(1622603529) when the rate limit window would be reset.

## High Level Design of Rate Limiter
