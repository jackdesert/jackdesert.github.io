source "https://rubygems.org"

# Hello! This is where you manage which Jekyll version is used to run.
# When you want to use a different version, change it below, save the
# file and run `bundle install`. Run Jekyll with `bundle exec`, like so:
#
#     bundle exec jekyll serve
#
# This will help ensure the proper Jekyll version is running.
# Happy Jekylling!
#
# Based on https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/ I am removing the jekyll gem in favor of the github-pages gem
#gem "jekyll", "~> 3.8.3"

# This is the default theme for new Jekyll sites. You may change this to anything you like.
gem 'jekyll-theme-leap-day' # YASS  -- has a Table of Contents of the Post
gem 'jekyll-theme-tactile'  # Yass -- great syntax highlighting
gem 'jekyll-theme-hacker'   # Maybe -- black with green text
gem 'jekyll-theme-midnight' # Maybe -- beautiful, unassuming

# If you want to use GitHub Pages, remove the "gem "jekyll"" above and
# uncomment the line below. To upgrade, run `bundle update github-pages`.
# gem "github-pages", group: :jekyll_plugins

# If you have any plugins, put them here!
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.6"
  gem "github-pages"
end

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem "tzinfo-data", platforms: [:mingw, :mswin, :x64_mingw, :jruby]

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.0" if Gem.win_platform?

