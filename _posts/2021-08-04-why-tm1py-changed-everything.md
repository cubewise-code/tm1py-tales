---
layout: post
title:  "Why TM1py changed everything!"
date:   2021-08-04 9:43:27
categories: personal-story
---

I want to share with you my personal view of TM1py and why it has probably the biggest impact on our beloved functonal database 


The Dark Ages 
=====

Before TM1py there was only one way of getting data from your TM1 database. And it was (and still is) a messy one. You had to export it as a flat file with all the disadvantages of a flat file. Let me just say one word: `encoding`! 
Lets ignore for the moment the option to right click on a cube and export a view as a text file. So the only real option is a TurboIntegrator process which exports a subsection of the cube via `ASCIIOutput` into a text file. Hopefully it does not mix up the decimal separator and the column delimiter is not used in an element name. 
If you got lucky you end up with a usable file with the requested data as content.

The Enlightenment
=====

When TM1py arrived, it had no real impact on the big world of Python. But in the small corner of functional databases, namely TM1, it changed everything. Suddenly everyone was able to easily export data from TM1. Three lines of code and you had your cube data in a Pandas DataFrame and with it all the possibilities of 'batteries included'. Those three lines is all a data analyst needs to know about the inner workings of TM1. 

``` python
from TM1py import TM1Service

with TM1Service(address='localhost', port=12354, ssl=True, user='admin', password='apple') as tm1:
    df = tm1.cubes.cells.execute_view_dataframe(cube_name='plan_BudgetPlan', view_name='Budget Input Detailed')

```
From here on out everything is Python and Pandas or any other Python package you can think of. 



_____

Written by [![GitHub](https://i.stack.imgur.com/tskMh.png) Christoph Hein](https://github.com/scrumthing)