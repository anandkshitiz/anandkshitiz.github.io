---
title: Amazon Elastic Compute Service (AWS ECS)
date: '2018-12-23 11:41:00'
categories: [AWS, ECS]
tags: [aws]
comments: true
# image: " ../../assets/img/posts/amazon-ecs/Container-Overview-1.png"
seo:
  date_modified: 2019-12-26 15:02:03 +0530
---

## Containers - Overview

We often face the problem where in an application shared by the Dev team doesn't work in QA environmet or we are unable to run an applicatiom shared by our friend in our own laptop because we don't have the right version of dependencies used by the application.

Well, containers are the solution to this problem. As, with a container, an application and its dependencies are shipped together under a container image.This container image can run without any dependencies issue in any environmet whether it is a private data center, public cloud or a personal laptop.


> **A container is a standard unit of software that packages up code and all its dependencies; so the application runs quickly and reliably from one computing environment to another.**

So, in order to successfully run a springboot application in any environmet, we need to create a container image which contains details about all its dependences like the Java runtime, maven, git etc. This container image can then be shipped to any environment where it can successfully run as a container.

## Container Orchestration

In real life scenarios, running a single container might not be sufficient. We will encounter following situations :

1. What if there is need to scale horizontally?
2. What if the ec2 instance goes down ?
3. What if there is a need to schedule few other service and batch jobs?
4. How to ensure zero downtime even during the time of deployment ?
5. How to get useful analytics and monitor performance of the resources ?

All, or most of the above problems, can be solved using some Container Orchestration system. There are a number of Container Orchestration system availale today. Most famous are Kubernetes, Amazon ECS, Marathon, Docker Swarm etc.

In this article let's try to learn what is Amazon ECS and how to run different applications in ECS.

## Amazon ECS (Elastic Container Service)

ECS is a regional service which allows running containers in a multi availability zones within a region.

![ECS High Level Diagram]({{ site.baseurl }}/assets/img/posts/amazon-ecs/ecs-high-level.png)

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
> **_It specifies the container information for your application, such as how many containers are part of your task, what resources they will use, how they > are linked together, and which host ports they will use._**

![Task Definition]({{ site.baseurl }}/assets/img/posts/amazon-ecs/Task-Definition.png)

## Service - How to run
> **_It specifies how many copies of your task definition to run and maintain in a cluster. You can optionally use an Elastic Load Balancing load balancer
> to distribute incoming traffic to containers in your service.Amazon ECS maintains that number of tasks and coordinates task scheduling with the load
> balancer._**

![Service]({{ site.baseurl }}/assets/img/posts/amazon-ecs/ECS-Service.png)

## Task - What runs
