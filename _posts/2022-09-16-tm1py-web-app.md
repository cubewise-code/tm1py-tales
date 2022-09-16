---
layout: post
title:  "Build a web app to interact with TM1py"
date:   2022-09-16 18:30:00
categories: introduction
---

## The Problem

TM1py makes it easy to interact with TM1 programmatically. Typically you would write and execute your TM1py code from a regular Python file or Jupyter Notebook. While this works, would it not be great to have a graphical user interface to simplify running your TM1py scripts?

## The Solution

You can build a web app to interact with TM1py using streamlit. Streamlit is a library that allows you to create interactive applications in Python. It is designed to be easy to use, making it possible to create beautiful applications with very little code.
There are many benefits to using Streamlit, including:

- It is straightforward to use. You can create an application in just a few lines of code.
- It is open source and free to use.
- It is very reliable and well-tested.
- It has excellent documentation.

You can install streamlit via pip:

```powershell
pip install streamlit
```

Once streamlit is installed, let us create a very basic web app.

As the first step, let us create a new Python script. Let us call it `app.py`.

Open `app.py` in your favourite IDE or text editor, then add these lines:

```python
import streamlit as st

st.title("I love TM1py")
```

With that in place, we can run Streamlit from the command line:

```powershell
streamlit run app.py
```

Eh voilà, you just have created a web app:

![Screenshot - Streamlit Basic Website](https://github.com/cubewise-code/tm1py-tales/blob/master/_images/2022-09-16-streamlit-demo.png?raw=true)

Have a look at the official [streamlit documentation](https://docs.streamlit.io/) to see all features.

## Combine Streamlit + TM1py

Now that you know how to create a basic web app using streamlit, let us combine it with TM1py.

For this example, let us synchronize dimensions between TM1 instances, as shown in a [previous article](https://cubewise-code.github.io/tm1py-tales/2021/sync-dimensions.html) on TM1py tales.
The goal is that the user can select the dimension from the TM1 source instance that should be synchronized with the target instance.

Screenshot:

![Screenshot - TM1py Dashboard](https://github.com/cubewise-code/tm1py-tales/blob/master/_images/2022-09-16-streamlit-tm1py-demo.png?raw=true)

Source code:

``` python
import streamlit as st
from TM1py import TM1Service
 
st.title("Synchronize Dimensions between TM1 instances") 

# Initialization of streamlit session state
# Reference: https://docs.streamlit.io/library/api-reference/session-state
if "dimensions" not in st.session_state:
    st.session_state["dimensions"] = ""
 

# 1. Dimension Selection
with st.form("dimension_selection"):
    st.write("1. Step: Get Dimensions From Source")
    submitted = st.form_submit_button("Get All Dimensions From Source")
    if submitted:
        with TM1Service(address='localhost', port=12354, user='admin', password='apple', ssl=True) as tm1_source:
            # Get all dimensions from source
            dimensions = tm1_source.dimensions.get_all_names() 

            # Store dimensions in session state
            st.session_state["dimensions"] = ",".join(dimensions)
           
# Only show the second step if dimensions are available
if not st.session_state["dimensions"]:
    st.stop()
 
# 2. Dimension Synchronization
with st.form("dimension_sync"):
    st.write("2. Step: Synchronize Dimension")
    # Get dimensions from session state and display them
    dimension_name = st.selectbox("Select Dimension:", options=st.session_state["dimensions"].split(","))
    submitted = st.form_submit_button("Synchronize Dimension")
    if submitted:
        with st.spinner("Synchronizing dimensions..."):
            with TM1Service(address='localhost', port=12354, user='admin', password='apple', ssl=True) as tm1_source:
                with TM1Service(address='localhost', port=12354, user='admin', password='apple', ssl=True) as tm1_target:
                    dimension = tm1_source.dimensions.get(dimension_name=dimension_name)
                    tm1_target.dimensions.update_or_create(dimension)
        st.success("Dimension successfully synchronized")
```

## Further Thoughts

As shown in this example, you can turn your TM1py scripts into an interactive web app with just a couple of lines. You could expand on this example and build an entire dashboard to execute your favourite TM1py scripts from the web app.

If you have the available infrastructure, you could deploy your streamlit app to an on-premises server or cloud instance, making it available to all your TM1 users.

_____

Written by [![GitHub](https://i.stack.imgur.com/tskMh.png) Sven Bosau](https://github.com/sven-bo)
