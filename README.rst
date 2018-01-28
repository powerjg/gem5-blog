:Authors: Jason Lowe-Power

This repository contains the code for the gem5 blog.
The blog is hosted at http://blog.gem5.org.

Installation of required tools
------------------------------

The gem5 blog is built using Jekyll.
I struggled choosing a blogging platform, but decided to go with Jekyll because it seems to be the most popular platform.
Choosing the most popular platform means that it is easier to deply, extend, find themes, etc.
The biggest downside to Jekyll is that it is written in Ruby whereas most of the gem5 developers are familiar with Python.
However, I'm hoping we won't have to make many modification to the site generation scripts.

Before you begin, you need to install Ruby_, and then install Jekyll.
Note: You can install this for only the current user with ``--user-install``.

.. code-block:: sh

    gem install jekyll bundler

Needed plugins
~~~~~~~~~~~~~~
* jekyll-rst: https://github.com/xdissent/jekyll-rst. You will have to source venv/bin/activate to build.

.. code-block:: sh

    git submodule add https://github.com/xdissent/jekyll-rst.git _plugins/jekyll-rst
    virtualenv venv
    source venv/bin/activate
    pip install -r requirements.txt


Tags
----

* announcements: Used for general announcements about the blog or gem5 in general.
* best-practices: Used when describing a gem5 best practice. These usually are more about using gem5 and less about gem5 code.
* howto: Describes how to do something specific.
* code: A documentation-like post that describes how some gem5 code works.
