---
layout: post
title:  'TM1py Offline Install'
date:   2022-05-18 20:20:00
categories: install
---

# 7 simple steps to install Python & TM1py without internet

1. Download Python from https://www.python.org/downloads/
2. Install Python in the target environment and in a secondary environment where access to the internet is available _(e.g., on your desktop)_. The same Python version must be installed in both environments.
3. In the secondary environment install tm1py with pip: `C:\Python310\scripts\pip install tm1py[pandas]`
4. In the secondary environment create a snapshot of your installed packages with pip: `C:\Python310\scripts\pip freeze > C:/temp/offline_install/requirements.txt`
5. In the secondary environment  download the tm1py packages and its dependencies as files with pip: `pip download tm1py[pandas]`
6. Now move the requirements.txt and the package file to the target environment
7. On the target environment install the packages with pip: `C:\Python310\scripts\pip install --no-index --find-links C:/temp/offline_install/ -r requirements.txt`

![the installation folder](https://github.com/cubewise-code/tm1py-tales/blob/master/_images/2022-05-18-offline_install.png?raw=true)  
