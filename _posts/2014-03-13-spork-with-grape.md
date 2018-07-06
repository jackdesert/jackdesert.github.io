Spork with Grape
================

Spork saves you a lot of time in the feedback loop of test-driven development. But in order for it to work with Grape, spork needs to reload your files between test runs.

Spork Order of Events
---------------------

Here's the order of events when running spork:

1. Spork.each_run block
1. config.before(:suite) block
1. config.before(:each) block
1. Your test

If you manually load your files in the Spork.each_run block, they still get overwritten by Grape's cached copy. So to get this to work, load them at a later stage. What worked for me was to load them in spec-helper's config.before(:suite) block:

Loading Files Later in the Game
-------------------------------

    # spec/spec_helper.rb
    Spork.prefork do
      ...
      config.before(:suite) do
        # Explicitly loading files in grape api so that spork
        # does not use a cached copy. (Note this does not have
        # the same effect if used in the Spork.each_run block)
        if ENV['DRB']
          Dir["#{Rails.root}/lib/api/**/*.rb"].map{|f| load f }
        end
      end
    end

    Spork.each_run do
      ...
    end


