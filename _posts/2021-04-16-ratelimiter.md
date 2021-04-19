---
title: Rate Limiter
date: "2021-04-16 08:49:30 +0530"
categories: [System Design]
tags: [api-gateway, design]
comments: true
seo:
  date_modified: 2021-04-16 08:49:30 +0530
---

## Overview
{: .themeBlue}

Rate limiting is an API's configured behaviour to reject sender's request in case the number of request over a window of time crosses a limit.

Example : An API which generates an authentication token can cap the maximum number of token generation request sent by a specific client(or/and from a specific ip address) in a period of 1 hour. Let's say 1200 request/hour.

In terms of rate-limiting, following scenarios can take place when a sender sends a request to a REST API
1. Rate Limit not reached : In this case, the API sends a valid response(2xx, 5xx etc.) as per the request. Additionally, API can also send few headers which provide information about the current status of rate-limit like :

```
HTTP/1.1 200 OK
X-Rate-Limit-Limit: 1200
X-Rate-Limit-Remaining: 100
X-Rate-Limit-Reset: 1618728633175
```
2. Rate Limit reached : In this case, the API sends a 429 HTTP Code(Too Many Request). Additionally, API can also provide information about when this limit will be reset in the header.

```
HTTP/1.1 429
Date: Sun, 18 April 2018 21:33:25 GMT
X-Rate-Limit-Limit: 1200
X-Rate-Limit-Remaining: 0
X-Rate-Limit-Reset: 1618728633175
```

## Design an API Rate Limiter
{: .themeBlue}

### Simple Rate Limiter
{: .themeBlue}

Before designing an actual full-fledged API Rate Limiter, let's write a very simple rate-limiter which caps the number of request to a Java method to 10/min.

#### Implementation of a custom rate-limiter
{: .themeBlue}

```java
public class MyCustomRateLimiter {
    private final int MAX_SIZE = 10;
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
            long timeElapsedSinceBucketStart = currentTime - apiRateLimiter.getStartTime();
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
From the API perspective, define an APIRateLimiter classs which will store the rate-limiter configuration of a particular API and its current state

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