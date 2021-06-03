---
title: Rate Limiter Implementation
date: 2021-06-03 04:04:30 +0530
categories: [System Design, Rate Limiter, Implementation]
tags: [api-gateway, design]
comments: true
seo:
  date_modified: 2021-06-03 04:35:18 +0530
---

## Overview
{: .themeBlue}

## Types of Rate Limiting Algorithms
{: .themeBlue}


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