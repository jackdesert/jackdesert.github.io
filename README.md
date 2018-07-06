Blog
====

Github URL: https://github.com/jackdesert/jackdesert.github.io




CSS
---

I do like this site: https://jekyllrb.com/docs/github-pages/
Whose CSS is here: https://jekyllrb.com/css/screen.css


But the jekyll-theme-hacker is serving me nicely.

Intended modifications:

  * Fix so "Written by" shows up
  * Change "View on Github" to "Back to index"
  * Or just change page title to href to "/"


Generate Site and Serve Locally
-------------------------------

    # Optionally, remove the _site directory
    rm -r _site
    be jekyll serve


CNAME
-----

The CNAME file tells github where to redirect, and where to allow hosting from.



Tag Line
--------

If you’re having trouble or if some part isn’t clear or incorrect, don’t hesitate to drop me an email or leave a comment below.


Changing Themes
---------------

Each theme seems to have a different name of their layouts.

In index.md, a layout is expected.

If you home page is not showing up, change the layout specified in index.md.

Hacker Theme
------------

I copied this file:

    _layouts/default.html

from the gem so I could edit it.
