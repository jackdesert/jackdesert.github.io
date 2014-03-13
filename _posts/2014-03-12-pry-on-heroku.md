Pry-Debugger on Heroku
=============

What? Pry on Heroku? But that would be my production environment!

Yes. Sometimes you want to run some queries on production. And sometimes you need to inspect values inside methods. 

Setup
-----

To get pry-debugger up and running on Heroku is almost the same as setting up the same for local development. 

Add the 'pry-debugger' and 'pry-rails' gems to your gemfile for all groups where you want it accessible (for me this is all 

    group :staging, :production, :development, :test do
      gem 'pry-debugger'
      gem 'pry-rails'
    end

Bundle locally and push to heroku.

    bundle
    git commit -am'<your message>'
    git push heroku master -app <app_name>

If any of the gems fail to build on heroku, try updating these gems:

    bundle update debugger debugger-linecache debugger-ruby_core_source \
    pry-debugger


Usage
-----

The beauty of pry-debugger in production is that you do not actually have to 
change any code to do your debugging.  
