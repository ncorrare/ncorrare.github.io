---
layout: post
title: My CI Workflow
categories: [general, code, puppet]
tags: [code, git, puppet, CI]
fullview: true
---
Don't get me wrong, I like Jenkins, unlike every other Java app. But I'm a Sysadmin (at least in spirit), and there are still things that I need to get used to from the development world. But Jenkins is:

- Extremely flexible (or extremely complicated)
- Another App I'd need to host somewhere (not that I'm laking IT resources)
- As my manager likes to put it, using a Ferrari to drive at 20 miles per hour (that's been taken out of context by the way)

I needed something extremely simple, since I've a module published in the Puppet Forge (hopefully more soon enough), and I want to run Unit Tests on it before I actually publish new releases.
In comes **Travis CI**. It's probably way more powerful that what I'm using it for, but it's simple enough for me go get it integrated on my development workflow because:

- It integrates with GitHub, so I don't need to write and troubleshoot any custom hooks.
- It email's me when my build fails
- I've a nice tag that I can add into my README.md to force me to fix my builds
- It's extremely simple to configure

How simple you may ask?


- Head onto the [Travis-CI Website](http://www.travis-ci.org)
- Log-in with your GitHub Account, and enable the repo's you want to manage

Now those repositories need a bit of extra love to be tested with Travis. You need to create a .travis.yml file with the configuration.

{% highlight yaml %}
---
language: ruby
bundler_args: --without system_tests
script: "bundle exec rake validate && bundle exec rake lint && bundle exec rake spec SPEC_OPTS='--format documentation'"
matrix:
  fast_finish: true
  include:
  - rvm: 1.9.3
    env: PUPPET_GEM_VERSION="~> 3.0"
  - rvm: 2.0.0
    env: PUPPET_GEM_VERSION="~> 3.0"
  - rvm: 2.0.0
    env: PUPPET_GEM_VERSION="~> 4.0"
notifications:
  email: nicolas@mymailserver.com
{% endhighlight %}

Now I'm assuming here that you have your Gemfile and Rakefile properly set up, in my case my jobs are as follows:

{% highlight ruby %}
require 'rubygems'
require 'puppetlabs_spec_helper/rake_tasks'
require 'puppet-lint/tasks/puppet-lint'
PuppetLint.configuration.send('disable_80chars')
PuppetLint.configuration.ignore_paths = ["spec/**/*.pp", "pkg/**/*.pp"]

desc "Validate manifests, templates, and ruby files"
task :validate do
  Dir['manifests/**/*.pp'].each do |manifest|
    sh "puppet parser validate --noop #{manifest}"
  end
  Dir['spec/**/*.rb','lib/**/*.rb'].each do |ruby_file|
    sh "ruby -c #{ruby_file}" unless ruby_file =~ /spec\/fixtures/
  end
  Dir['templates/**/*.erb'].each do |template|
    sh "erb -P -x -T '-' #{template} | ruby -c"
  end
end

task :default => [:validate, :spec, :lint]
{% endhighlight %}

Good news if you don't understand any of that, puppet module generate actually set's that up for you!

So in my case, I'm doing parsing validation, linting and spec testing on my module. Even if you don't write spec tests, just doing parser validation and linting is a great start!

##Keeping you honest
You need to write spec tests to cover your cases, the more spec tests you write, the more you can be sure that you're not breaking anything when changing your code. Luckily there is a great tool to help you with that, and its Coveralls.

![How much of my code do my tests cover](/assets/media/coveralls/code-coverage.png "How much of my code do my tests cover")
![Where do I need to work a bit more](/assets/media/coveralls/more-work.png "Where do I need to work a bit more")

To set up Coveralls, [the instructions are pretty clear](https://coveralls.zendesk.com/hc/en-us/articles/201769485-Ruby-Rails), but basically you'd need to:

- Include the Coveralls gem in the Gemfile
{% highlight ruby %}
source 'https://rubygems.org'

puppetversion = ENV.key?('PUPPET_VERSION') ? "= #{ENV['PUPPET_VERSION']}" : ['>= 3.3']
gem 'puppet', puppetversion
gem 'puppetlabs_spec_helper', '>= 0.1.0'
gem 'puppet-lint', '>= 0.3.2'
gem 'facter', '>= 1.7.0'
gem 'rake'
gem 'rspec-puppet'
gem 'dpl'
gem 'coveralls', require: false
{% endhighlight %}
(That last lines add coveralls by the way).

And add Coveralls to your testing suite
{% highlight ruby %}
require 'coveralls'
Coveralls.wear!
require 'puppetlabs_spec_helper/module_spec_helper'
{% endhighlight %}

The only gotcha so far, is that you need to add an environment variable (USENETWORK=true) in Travis to allow the test to post the result to Coveralls. You can do this through the Web Interface, or in your .travis.yml file.

![Setting the environment variable in Travis](/assets/media/coveralls/envvar.png "Setting the environment variable in Travis")
