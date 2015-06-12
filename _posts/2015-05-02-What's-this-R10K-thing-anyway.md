---
layout: post
title: What's this R10K thing anyway
categories: [general, code, puppet]
tags: [code, git, puppet, r10k]
fullview: true
---
So you have been using Puppet for a while, and your code is everywhere!.

Infrastructure as Code allows you to do amazing things, so soon enough you'll realize that it can also become a nightmare, if you haven't been "doing it right". So if you haven't seen Gary Larizza's talk in PuppetConf 2014 called **Doing the refactor dance** [I'd strongly suggest you do that](https://www.youtube.com/watch?v=v9LB-NX4_KQ), and this article will be waiting for you once you're done.

Now that your code is actually **structured** in a sort of logic way, let's be honest, it's not going to stay that way much longer. Code evolves, and so your infrastructure, and we need a way to track those code changes, evolve those modules from Testing, to QA, to Production. What about their dependencies?.
So if this are the kind of issues you're starting to face, R10K is the keyword you should be googling. If you have Puppet Enterprise, good news, its actually included as part of Code Manager from 3.8 onwards. If not, go ahead and issue a gem install r10k. It won't bite.

R10K takes a **Control Repository**, which would hold a **Puppetfile**, describing what modules are required, your **Hiera data**, your site.pp, etc. ... Everything but the modules. R10K will go and deploy a Puppet Environment out of each **branch** of that Control Repository.

Consider the following example, available on my [github repository](https://github.com/ncorrare/environments):


- My repository has two branches, production and testing.
  
  
  - On the production branch, I've a Puppetfile that looks like this:
  {% highlight ruby %}
  forge "http://forge.puppetlabs.com"

  # Modules from the Puppet Forge
  mod 'puppetlabs/apache'
  mod 'puppetlabs/ntp'

  # Modules from Github using various references
  mod 'notifyme',
    :git => 'https://github.com/ncorrare/ncorrare-notifyme.git',
    :tag => '0.0.3'
  {% endhighlight %}
  This Puppetfile is telling R10K how my production environment looks like. In this case, I'm pulling two modules from the forge, and I've one module from my internal repository.
  
  
  - In the testing branch, my file looks slightly different.
  {% highlight ruby %}
  forge "http://forge.puppetlabs.com"

  # Modules from the Puppet Forge
  mod 'puppetlabs/apache'
  mod 'puppetlabs/ntp'

  # Modules from Github using various references
  mod 'notifyme',
    :git => 'https://github.com/ncorrare/ncorrare-notifyme.git',
    :tag => '0.0.4'
  {% endhighlight %}  
  So I'm actually using a different module version for the notifyme module.

If you need a complete base environment to start with, I'd strongly recommend you head to Terri Harber's repository, where she has a [very good example of one](https://github.com/terrimonster/puppet-control).

Once your control repo is ready, well that's half the job. You probably want to start with a blank Puppet Master, since R10K by default will delete every module and every environment that's not specified in the Control Repository.
{% highlight yaml %}
# The location to use for storing cached Git repos
cachedir: '/var/cache/r10k'

# A list of git repositories to create
sources:
  # This will clone the git repository and instantiate an environment per
  # branch in the directory you've specified. Check your puppet.conf file.
  ncorrare:
    remote: 'https://github.com/ncorrare/environments.git'
    basedir: '/etc/puppetlabs/puppet/environments/'
{% endhighlight %}

If you now proceed to run r10k deply environment -pv, voila!, R10K is already setting up your Puppet Environments for you.
Now that you understand the basics behind R10K, look into the r10k module [in the Puppet Forge](https://forge.puppetlabs.com/zack/r10k) for instructions into how to have Puppet manage R10K (that will eventually manage your Puppet environments, and there we go full circle again).
