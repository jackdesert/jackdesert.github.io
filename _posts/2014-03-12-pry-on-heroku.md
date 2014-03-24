Pry-Debugger on Heroku
=============

What? Pry on Heroku? But that would be my production environment!

Yes. Sometimes you want to run some queries on production. And sometimes you need to inspect values inside methods. 
The beauty of running pry-debugger in production is that you do not actually have to change any code to do your debugging.  

Setup
-----

To get pry-debugger up and running on Heroku is almost the same as setting up the same for local development. 

Add the 'pry-debugger' and 'pry-rails' gems to your gemfile for all groups where you want it accessible.
(If running Ruby 2.x, use the pry-byebug gem instead of pry-debugger)

    group :staging, :production, :development, :test do
      gem 'pry-debugger'
      gem 'pry-rails'
    end

Bundle locally and push to heroku.

    $ bundle
    $ git commit -am'<your message>'
    $ git push heroku master -app <app_name>

If any of the gems fail to build on heroku, try updating these gems:

    $ bundle update debugger debugger-linecache debugger-ruby_core_source pry-debugger


Usage
-----

First, fire up the heroku console

    $ heroku run rails console --app <your_app_name>

Then set a breakpoint right where you are. This keeps you in the console after you exit
a breakpoint.

    > binding.pry
    
Set a breakpoint on your method of choice.

    > user = User.first
    > break user.whats_my_name  # Alternatively, `break User#whats_my_name`

Then call the method, and it will halt at the beginning of that method

    > user.whats_my_name

Now you can look around and see what you want to see. When you are done inspecting, exit.

    > exit

And it will bring you right back to that first binding.pry that you typed.
From there you can run the same method again, set a different breakpoint, or
type `exit` again to leave the console completely.

Other Options
-------------

For more pry-debugger options like clearing all breakpoints, access the help screen
    
    > help


