---
title: About

# The About page
# © 2017-2019 Cotes Chung
# MIT License
---

I am reachable via. following means:
1. LinkedIn <a href="https://www.linkedin.com/in/{{ site.linkedin.username }}" target="_blank">
    <i class="fab fa-linkedin"></i>
  </a>
2. Twitter  <a href="https://twitter.com/{{ site.twitter.username }}" target="_blank">
    <i class="fab fa-twitter"></i>
  </a>
3. Email {% assign email = site.social.email | split: '@' %}
  <a href="javascript:window.open('mailto:' + ['{{ email[0] }}','{{ email[1] }}'].join('@'))">
    <i class="fas fa-envelope"></i>
  </a>