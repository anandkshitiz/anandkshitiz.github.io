---
title: Rate Limiter Implementation
date: 2021-06-03 04:04:30 +0530
categories: [System Design, Rate Limiter, Implementation]
tags: [api-gateway, design]
comments: true
seo:
  date_modified: 2021-06-03 04:11:34 +0530
---

## Overview
{: .themeBlue}

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



## Design custom API Rate Limiter
{: .themeBlue}

### Rate Limiter for 
{: .themeBlue}

Before designing an actual full-fledged API Rate Limiter, let's code for a simple rate-limiter.

#### Implementation of a custom rate-limiter
{: .themeBlue}

```java
public class MyCustomRateLimiter {
    private final int MAX_SIZE = 100; // This caps the maximum number of APIs for which Rate Limiting can be configured by an application
    private final Map<String, APIRateLimiter> registeredAPIs;

    private static Logger log = LoggerFactory.getLogger(MyCustomRateLimiter.class);

    MyCustomRateLimiter() {
        registeredAPIs = new HashMap<>();
    }

    boolean configure(RateLimiterConfig rateLimiterConfig) {
        // TO DO : do validation
        if (registeredAPIs.size() < MAX_SIZE) {
            registeredAPIs.put(rateLimiterConfig.name(), new APIRateLimiter(rateLimiterConfig));
            return true;
        } else {
            return false;
        }

    }

     boolean isRateLimited(String name) {

        if (registeredAPIs.containsKey(name)) {
            APIRateLimiter apiRateLimiter = registeredAPIs.get(name);
            long currentTime = System.currentTimeMillis();
            long timeElapsedSinceBucketStart = currentTime - apiRateLimiter.getStartTime(); // This isn't thread safe
            log.info("Count = {}, Elapsed = {} ms", apiRateLimiter.getCount(), timeElapsedSinceBucketStart);
            if (apiRateLimiter.getStartTime() == 0L) {
                log.debug("Started");
                apiRateLimiter.setStartTime(currentTime);
                apiRateLimiter.setCount(apiRateLimiter.getCount() + 1);
                return false;
            } else if (timeElapsedSinceBucketStart >= apiRateLimiter.rateLimiterConfig.limitPeriodInSeconds() * 1000L) {
                log.debug("Rate Limiting Period Time Elapsed. Resetting the count");
                apiRateLimiter.setStartTime(currentTime);
                apiRateLimiter.setCount(1);
                return false;
            } else if (timeElapsedSinceBucketStart < apiRateLimiter.rateLimiterConfig.limitPeriodInSeconds() * 1000L
                    && apiRateLimiter.getCount() > apiRateLimiter.getRateLimiterConfig().limitForPeriod()) {
                log.debug("Rate limit count exceeded");
                return true;
            } else if (timeElapsedSinceBucketStart < apiRateLimiter.rateLimiterConfig.limitPeriodInSeconds() * 1000L
                    && apiRateLimiter.getCount() < apiRateLimiter.getRateLimiterConfig().limitForPeriod()) {
                log.debug("Rate Limiting Period Time Elapsed.");
                apiRateLimiter.setCount(apiRateLimiter.getCount() + 1);
                return false;
            } else {
                log.debug("Current Count equals maximum count in the bucket");
                apiRateLimiter.setCount(apiRateLimiter.getCount() + 1);
                return false;
            }
        } else {
            log.debug("rate limit not configured");
            return false;
        }
    }
}
```

#### APIRateLimiter
{: .themeBlue}
```java
public class APIRateLimiter {
    RateLimiterConfig rateLimiterConfig;
    private long startTime = 0L;
    private long count = 0L;

    APIRateLimiter(RateLimiterConfig rateLimiterConfig) {
        this.rateLimiterConfig = rateLimiterConfig;
    }

    public RateLimiterConfig getRateLimiterConfig() {
        return rateLimiterConfig;
    }

    public void setRateLimiterConfig(RateLimiterConfig rateLimiterConfig) {
        this.rateLimiterConfig = rateLimiterConfig;
    }

    public long getStartTime() {
        return startTime;
    }

    public void setStartTime(long startTime) {
        this.startTime = startTime;
    }

    public long getCount() {
        return count;
    }

    public void setCount(long count) {
        this.count = count;
    }
}

```

#### Define RateLimiter Configuration
{: .themeBlue}

This should be implemented by the sender to define the configurations

```java
public interface RateLimiterConfig {

    int limitForPeriod();

    int limitPeriodInSeconds();

    String name();
}
```
From the API perspective, define an APIRateLimiter class which will store the rate-limiter configuration of a particular API and its current state

``` java
public class APIRateLimiter {
    RateLimiterConfig rateLimiterConfig;
    private long startTime = 0L;
    private long count = 0L;

    APIRateLimiter(RateLimiterConfig rateLimiterConfig) {
        this.rateLimiterConfig = rateLimiterConfig;
    }

    public RateLimiterConfig getRateLimiterConfig() {
        return rateLimiterConfig;
    }

    public void setRateLimiterConfig(RateLimiterConfig rateLimiterConfig) {
        this.rateLimiterConfig = rateLimiterConfig;
    }

    public long getStartTime() {
        return startTime;
    }

    public void setStartTime(long startTime) {
        this.startTime = startTime;
    }

    public long getCount() {
        return count;
    }

    public void setCount(long count) {
        this.count = count;
    }
}
```

## Open Source Rate Limiters
{: .themeBlue}
{:toc}