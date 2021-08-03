---
layout: post
title:  "Why TM1py changed my world"
date:   2021-08-04 9:43:27
categories: personal-story
---

Why TM1py changed everything
=====

Create an open platform for the TM1 community to share and discuss TM1py use cases. stories and ideas.

Promote a set of good practices how to do things with TM1py such as to load a dimension from a CSV datasource or to query TM1 data for PowerBI.

How to add new posts?
=====

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

How to include code in your post?
=====

Follow the markdown conventions:

``` python
from TM1py import TM1Service

with TM1Service(address='localhost', port=12354, ssl=True, user='admin', password='apple') as tm1:
    print(tm1.server.get_product_version())

```
_____

Written by [![GitHub](https://i.stack.imgur.com/tskMh.png) Christoph Hein](https://github.com/scrumthing)