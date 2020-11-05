---
title: Designing a movie recommendation system
date: '2020-03-30 17:04:00'
categories: [System Design]
tags: [oops, design]
comments: true
seo:
  date_modified: 2020-04-02 08:27:12 +0530
---

# Problem Statement

Design a movie recommendation system which will recommend movies based on user's movie browsing history.

# Functional Requirements

Recommendation for any movie will be following

Top 5 movies
  1. in the same genre
  2. with the same main cast
  3. in the same language

# Evolutionary Solutions

## Using In Memory Data Structure

```java
List<Movies>
```

```puml
A -> B
```

{% plantuml %}
[First] - [Second]
{% endplantuml %}

## Using a Relational Database

## Advanced solution using RDS, NO SQL , Caching etc.