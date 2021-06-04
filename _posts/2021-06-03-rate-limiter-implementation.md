---
title: Rate Limiter Implementation
date: 2021-06-03 04:04:30 +0530
categories: [System Design, Rate Limiter, Implementation]
tags: [api-gateway, design]
comments: true
seo:
  date_modified: 2021-06-03 06:14:15 +0530
---

## Overview
{: .themeBlue}
For a detailed overview of Rate Limiting, you can refer my earlier post [here]({% post_url 2021-05-29-rate-limiter-design-strategy %}) .

## Implementation of a custom rate-limiter
{: .themeBlue}

Before implementing a custom rate limiter, let's see the use case under which we want to apply the rate limiting.
Let's assume that we have a java method called ``` doSomething() ``` which we want to execute only 2 requests in a span of 1 second.

### How to use the Rate Limiter ?
{: .themeBlue}

In order to do that, we will define the Rate Limit configurations like ```limit for the period``` and ```limit period in seconds``` as shown below and configure the RateLimiter.
``` java 
DoSomethingRateLimiterConfig doSomethingRateLimiter =
            new DoSomethingRateLimiterConfig("doSomething", 2, 1);
RateLimiter rateLimiter = new RateLimiter();
rateLimiter.configure(doSomethingRateLimiter);
```
Once this configuration is done, we can first check whether, the rate limit is reached or not by calling ```rateLimiter.isRateLimited("doSomething")``` and then execute the logic of the method ```doSomething()```. 

The full source code to demonstrate how to use Rate Limiter in Java method can be seen in below test class called ```RateLimiterTest```. In this test case, I have created a fixed thread pool of size 2 so that we can test for concurrent scenarios as well.

``` java
import org.anandkshitiz.ratelimiter.config.RateLimiterConfig;
import org.junit.Test;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class RateLimiterTest {

    RateLimiter rateLimiter = new RateLimiter();
    private static final Logger LOGGER = LoggerFactory.getLogger(RateLimiterTest.class);
    static class DoSomethingRateLimiterConfig implements RateLimiterConfig {

        String name;
        int limit;
        int durationInSeconds;

        DoSomethingRateLimiterConfig(String name, int maxRequest, int durationInSeconds) {
            this.name = name;
            this.limit = maxRequest;
            this.durationInSeconds = durationInSeconds;
        }

        @Override
        public String name() {
            return name;
        }

        @Override
        public int limitForPeriod() {
            return limit;
        }

        @Override
        public int limitPeriodInSeconds() {
            return durationInSeconds;
        }

    }

    @Test
    public void isRateLimited() {
        DoSomethingRateLimiterConfig doSomethingRateLimiter = new DoSomethingRateLimiterConfig("doSomething", 2, 1);
        rateLimiter.configure(doSomethingRateLimiter);
        ExecutorService executorService = Executors.newFixedThreadPool(2);
        for (int i = 0; i < 200000; i++) {
            int finalI = i;
            executorService.execute(() -> {
                try {
                    doSomething(finalI);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }
        executorService.shutdown();
        while (!executorService.isTerminated()) {
        }
        LOGGER.debug("Finished all threads");
    }

    private void doSomething(int i) throws InterruptedException {
        if (!rateLimiter.isRateLimited("doSomething")) {
            Thread.sleep(1000);
            LOGGER.debug("Processing {}",i);
        } else {
            LOGGER.debug("{} This is rate limited",i);
        }
    }

}
```

### Code for RateLimiter package implementation
{: .themeBlue}

We will first define an interface which should be implemented by the sender to define the configurations.
#### Define RateLimiter Configuration
{: .themeBlue}
```java
/**
 * @author anandkshitiz
 * @since 15/04/21
 */
public interface RateLimiterConfig {

    int limitForPeriod();

    int limitPeriodInSeconds();

    String name();
}
```

#### Define individual APIs state
{: .themeBlue}
```java
import org.anandkshitiz.ratelimiter.config.RateLimiterConfig;

/**
 * @author anandkshitiz
 * @since 15/04/21
 */
public class APIRateLimiterState {
    RateLimiterConfig rateLimiterConfig;
    private long startTime = 0L;
    private int availableTokens;

    APIRateLimiterState(RateLimiterConfig rateLimiterConfig) {
        this.rateLimiterConfig = rateLimiterConfig;
        this.availableTokens = rateLimiterConfig.limitForPeriod();
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

    public int getAvailableTokens() {
        return availableTokens;
    }

    public void setAvailableTokens(int availableTokens) {
        this.availableTokens = availableTokens;
    }
}
```
#### Rate Limiter Implementation class
{: .themeBlue}

In this post, we will be implementing a token based rate limiter, where each API request will try to fetch a token, if there are available tokens for the API that request will be allowed to happen, if not, the request will be rate limited. The tokens will be regenerated after the rate limiting window is over.

So, following scenarios can take place :
1. For the very first API call, we will set the start time to current time, decrease the available token count by 1 and then allow the request to happen as we have available tokens.
2. If the Rate Limiting Period Time has elapsed, we will reset the start time to current time and available token count to maximum token count - 1 and allow the request to happen as this is a new window.
3. If there are available tokens and the rate limiting window has not passed, we will allow the request to happen by decreasing the available tokens by 1.

```java

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.anandkshitiz.ratelimiter.config.RateLimiterConfig;

import java.util.HashMap;
import java.util.Map;

/**
 * @author akshitiz
 * @since 15/04/21
 */
public class RateLimiter {

    private final Map<String, APIRateLimiterState> registeredAPIs;
    private static final Logger LOGGER = LoggerFactory.getLogger(RateLimiter.class);

    RateLimiter() {
        registeredAPIs = new HashMap<>();
    }

    void configure(RateLimiterConfig rateLimiterConfig) {
        // TO DO : do validation
        registeredAPIs.put(rateLimiterConfig.name(), new APIRateLimiterState(rateLimiterConfig));
    }

    synchronized boolean isRateLimited(String name) {

        if (registeredAPIs.containsKey(name)) {
            APIRateLimiterState apiRateLimiterState = registeredAPIs.get(name);
            long currentTime = System.currentTimeMillis();
            long timeElapsedSinceBucketStart = currentTime - apiRateLimiterState.getStartTime();
            LOGGER.debug("Available Tokens = {}, Elapsed = {} ms", apiRateLimiterState.getAvailableTokens(), timeElapsedSinceBucketStart);
            if (apiRateLimiterState.getStartTime() == 0L) {
                LOGGER.debug("First API call. Setting the start time to current time, decreasing the available token count by 1 and allowing the request");
                apiRateLimiterState.setStartTime(currentTime);
                apiRateLimiterState.setAvailableTokens(apiRateLimiterState.getAvailableTokens() - 1);
                return false;
            } else if (timeElapsedSinceBucketStart >= apiRateLimiterState.rateLimiterConfig.limitPeriodInSeconds() * 1000L) {
                LOGGER.debug("Rate Limiting Period Time Elapsed. Resetting the start time to current time and token count to original token count -1 and allowing the request";
                apiRateLimiterState.setStartTime(currentTime);
                apiRateLimiterState.setAvailableTokens(apiRateLimiterState.getRateLimiterConfig().limitForPeriod() - 1);
                return false;
            } else if (timeElapsedSinceBucketStart < apiRateLimiterState.rateLimiterConfig.limitPeriodInSeconds() * 1000L
                    && apiRateLimiterState.getAvailableTokens() >= 1) {
                LOGGER.debug("Rate limit not reached, decreasing the available token counts by 1 and allowing the request");
                apiRateLimiterState.setAvailableTokens(apiRateLimiterState.getAvailableTokens() - 1);
                return false;
            } else {
                LOGGER.debug("Rate Limit threshold reached");
                return true;
            }
        } else {
            LOGGER.debug("Rate limit not configured for the endpoint {}" , name);
            return false;
        }
    }
}
```