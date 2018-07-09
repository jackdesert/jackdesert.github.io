Blog
====

I store this in ~/r/blog, but the actual github url is the standard one
for github pages:

https://github.com/jackdesert/jackdesert.github.io


Variables
---------

see https://jekyllrb.com/docs/variables/

  page.path  # post's github url



Linking to Posts Internally
---------------------------

see https://jekyllrb.com/docs/templates/#linking-to-posts

Here's the basic form:

    [Link Text]({% link _posts/<full_filename> %})

The amazing news is that jekyll will complain if the filename changes
and you have not updated your link.

Also at the bottom of the page you can set up link variables:

[home]: /

Then you can reference "home" like this:

    [Link Text][home]


Post Titles
-----------

Titles are inferred from the filename if none is given in the metadata
at the top of the file.

Inferred from the filename means standardized capitalization (Not good
for uWSGI).

My recommendation is to include the title in the metadata,
and to print that metadata in the TOC.



Generate Site and Serve Locally
-------------------------------

    # Optionally, remove the _site directory
    rm -r _site
    be jekyll serve




CSS
---

I do like this site: https://jekyllrb.com/docs/github-pages/
Whose CSS is here: https://jekyllrb.com/css/screen.css

Closer to the source, the CSS is in this folder:

    ~/r/jekyll/docs/_sass$

(where jekyll is `git clone git@github.com:jekyll/jekyll)

In order to see what markup they used alongside a rendered version,
compare these two pages:

    https://github.com/jekyll/jekyll/blob/master/docs/_tutorials/orderofinterpretation.md
    https://jekyllrb.com/tutorials/orderofinterpretation/#order-of-interpretations

But the jekyll-theme-hacker is serving me nicely.

Intended modifications:

  * Fix so "Written by" shows up
  * Change "View on Github" to "Back to index"
  * Or just change page title to href to "/"




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



Permalinks
----------

See configuration docs at https://jekyllrb.com/docs/permalinks/

Note using without .html extensions requires nginx to have this enabled:

    try_files $uri $uri.html $uri/ =404;


