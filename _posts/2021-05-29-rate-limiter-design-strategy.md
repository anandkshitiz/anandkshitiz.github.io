---
title: Rate Limiter Design & Strategy
date: 2021-05-29 08:49:30 +0530
categories: [System Design, Rate Limiter, Design & Strategy]
tags: [api-gateway, design]
comments: true
pin: true
seo:
  date_modified: 2021-06-03 08:54:15 +0530
---

## Introduction
{: .themeBlue}

Recently, I got a chance to implement Rate Limiter for the purpose of throttling API requests coming to the servers. The servers in the system have been benchmarked against maximum and average rps(requests per second) and there is an auto-scaling in place which can cater to an increase in load.

Even with this setup in place and accepting the fact that the client can retry for any failed transaction, in case of high load if auto-scaling takes more time, the new servers will get overwhelmed with the requests which could result in resource starvation and result in more failed transactions and total chaos in the system. Moreover, auto-scaling can also not be infinite as it can lead to an unplanned infrastructure cost. 

This is a good use case for implementing throttling using rate-limiting for individual APIs.

In this post I have tried to share my understanding of the concept of Rate Limiter and have provided a high level design and strategy while implementing Rate Limiter. I have also created a separate post for coding implementation of Rate Limiter using Java [here]({% post_url 2021-06-03-rate-limiter-implementation %}) .

## What is rate limiting in software systems ?
{: .themeBlue}
Rate limiting is an API's configured behavior to reject sender's request when the number of requests over a window of time crosses a limit or threshold. The sender here could be another system identified using user id, client id, ip address or other such identifier.

Moreover, a system can have a cap or rate-limiting set which could be independent of clients' context. This helps in throttling requests for any unplanned surge in traffic whether real or because of an attack.

## Why do software systems need rate limiting ?
{: .themeBlue}

I have already explained one of the rationale behind using rate-limiter. In fact, any system should always protect itself against tremendous rps as it directly impacts its availability, cost,latency and it's overall experience. So, rate limiting should be implemented as per design to every system which provides services used by other system(s). 

Let's understand following typical examples where Rate Limiter can be used:

> Example 1 : An API which generates an authentication token can cap the maximum number of token generation request sent by a specific client(or/and from a specific ip address) in a period of 1 hour. Let's say 1200 requests/hour. If the client sends more than 1200 requests in a span of 1 hour the server will reject the request with  HTTP 429 response code.
{: .themeBlue}

> Example 2 : As an API service provider one can set different rate-limiting threshold for different type of customers(paid/free).
{: .themeBlue}

> Example 3 : As an API service provider one can set maximum number of _failed_ requests in an hour, let's say if user provides more than 3 incorrect PIN for online financial transactions in an hour, the server will rate-limit further requests.
{: .themeBlue}

Based on above examples, it is clear that rate-limiter as a design choice is fit for following use cases:
1. Keep infrastructure cost as per the planned budget.
2. Gracefully reject client's request in case of high load.
3. Implement different revenue model for different kind of user's subscription.
4. Security against attack and suspicious activities

## Scenarios with Rate Limiter
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

## High Level Designs of Rate Limiter
{: .themeBlue}

### 1. _Simple rate limiter design_

![Simple stateless rate-limiter]({{ site.baseurl }}/assets/img/posts/rate-limiter/stateless-rate-limiter.png)
_Simple rate limiter design_

A simple design of rate limiter will be to cap the requests sent on a particular server if the requests per second goes above the benchmarked number. As shown in the above diagram, the rate limiter is part of the application and will cap the requests on the server based on only the individual server's state where it is running.

Pros:
* Very simple and easy to implement.
* Effective when there is a need to throttle to reduce infrastructure cost and chaos because of excessive load.

Cons:
* It can't be used to rate-limit requests for a particular user, ip, user-groups, specific clients etc.

### 2. _Stateful rate limiter design_

Above mentioned design can be extended and the rate-limiting state(s) can be shared across servers by caching it using some caching solutions like Redis, Memcache etc.

As shown in below diagram, here, the clients' ip address or user id can be used as a key with the current rate-limiting state variables as value. Since, the cache can be used across servers this solution can be used to limit requests across servers in a cluster unlike the previous design.

![Stateful rate-limiter]({{ site.baseurl }}/assets/img/posts/rate-limiter/stateful-rate-limiter.png)
_Stateful rate limiter design_

Pros:
* It can be used to rate-limit requests for a particular user, ip, user-groups, specific clients etc.

Cons:
* The rate-limiting system is tightly coupled with the server.

### 3. _Decoupled Rate Limiter_

Above mentioned design can be decoupled by running the Rate Limiter system outside of application server as shown in below diagram.

![Decoupled rate-limiter]({{ site.baseurl }}/assets/img/posts/rate-limiter/rate-limiter.png)
_Decoupled Rate Limiter_

Pros:
* Rate Limiter solution is decoupled and can be managed or maintained outside the application servers independently.

Cons:
* Management and maintenance of Rate Limiting system.

### 4. _API gateway based rate limiter design_

The above design can be further simplified by making the Rate Limiter part of the API Gateway system or using a 3rd party Cloud provider's API Gateway service which provides rate limiting.

![API Gateway based rate-limiter]({{ site.baseurl }}/assets/img/posts/rate-limiter/api-gw-rate-limiter.png)
_API gateway based rate limiter design_

## Conclusion
{: .themeBlue}

In my opinion, every system should implement some kind of rate-limiting solution(s) to minimize un-planned cost, attacks from hackers, un-warranted use by clients. This will also help in making the system highly available and provide a better experience to the client system(s).

## How to develope a simple rate limiting sdk in Java ?

You can refer [this post]({% post_url 2021-06-03-rate-limiter-implementation %}) for coding implementation of a simple rate limiter in Java.