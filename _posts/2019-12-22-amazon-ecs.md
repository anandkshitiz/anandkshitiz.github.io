---
title: AWS ECS
date: '2019-12-22 11:41:00'
categories: [AWS, ECS]
tags: [aws]
published: true
seo:
  date_modified: 2019-12-22 23:30:32 +0530
---

## Containers - Overview

Imagine a scenario where an application needs to run in different environments. In order for this application to run successfully its dependency with correct version needs to be installed in every environment.

<img src="{{ site.baseurl }}/assets/img/posts/amazon-ecs/Container-Overview-1.png"
     style=" float: center;
    display: block;
    margin-left: auto;
    margin-right: auto;" />

A container is a standard unit of software that packages up code and all its dependencies; so the application runs quickly and reliably from one computing environment to another.

## Container Orchestration

**1**. What if we need to scale, i.e. need multiple multiple containers and multiple ec2 instances ?
**2**. What if our ec2 instance goes down ? 
**3**. What if we need to schedule few other service and batch jobs?
**4**. How do we ensure zero downtime ?
**5**. How do I get analytics and monitor performance of resources ?

## Amazon ECS - Very High Level

<img src="{{ site.baseurl }}/assets/img/posts/amazon-ecs/ecs-high-level.png"
     style=" float: center;
    display: block;
    margin-left: auto;
    margin-right: auto;" />

## Running a container with Amazon ECS

**Create Task Definition** &rarr; **Define Service** &rarr; **Run Task**

1. Create Task Definition
* Define Container specifications.
* Memory
* CPU
* Environment Variables
* Logger
* Network Mode etc.
2. Define Service
* Number Of Tasks
* Deployment Strategies
* Placement Strategies
* Load Balancing
* Service Discovery
* Cluster to run a task
3. Run Task 
* One or multiple Docker containers run as a task, defined by Task Definition and Service

## Task Definition - What to run
> **_It specifies the container information for your application, such as how many containers are part of your task, what resources they will use, how they are linked together, and which host ports they will use._**

<img src="{{ site.baseurl }}/assets/img/posts/amazon-ecs/Task-Definition.png"
     style=" float: center;
    display: block;
    margin-left: auto;
    margin-right: auto;" />

## Service - How to run
> **_It specifies how many copies of your task definition to run and maintain in a cluster. You can optionally use an Elastic Load Balancing load balancer to distribute incoming traffic to containers in your service.Amazon ECS maintains that number of tasks and coordinates task scheduling with the load balancer._**

<img src="{{ site.baseurl }}/assets/img/posts/amazon-ecs/ECS-Service.png"
     style=" float: center;
    display: block;
    margin-left: auto;
    margin-right: auto;" />

## Task - What runs