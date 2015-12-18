---
layout: post
title: Extending your Puppet Language Dictionary for Application Orchestration
categories: [general, puppet, code, apporch]
tags: [puppet, apporch]
fullview: true
---

Time to extend your Puppet Language skills! The new features for application orchestration introduce a number of language constructs you'll want to know about if you're planning to describe your applications and supporting infrastructure in the Puppet language.

## The Environment Service Resource Type
If you haven't worked with Puppet types and providers before, I'd strongly suggest you read the documentation on custom types and providers (available here [https://docs.puppetlabs.com/guides/custom_types.html](https://docs.puppetlabs.com/guides/custom_types.html) and here [https://docs.puppetlabs.com/guides/provider_development.html](https://docs.puppetlabs.com/guides/provider_development.html), respectively).

It is worth noting that, unlike your traditional Puppet types and providers, an environment service resource type is far simpler to develop, as you’ll see in the first example below.

The environment service resource is the key to defining the relationship between different software layers. It effectively extends a traditional Puppet type (like file, services, or packages) to describe application services (an SQL database, web server, API layer or a microservice, for example). Like Puppet node types, the environment service resource can (optionally) have a provider, which in this case implements a number of service-related methods — for instance, how to check if an environment service produced by an application component is actually ready for work before continuing the deployment.

The environment service type looks pretty much like any other Puppet type, but it has the property of being a capability. A very simple example could be an SQL resource, as described below:

{% highlight ruby %}
    Puppet::Type.newtype :sql, :is_capability => true do
      newparam :name, :namevar => true
      newparam :dbname
      newparam :dbhost
      newparam :dbpass
      newparam :dbuser
    end
{% endhighlight %}
What I'm doing here is telling the Puppet resource abstraction layer (or rather the service abstraction layer) that I have modeled an SQL database that can be produced by a server (managed with Puppet), for use by one or more servers that need an SQL database. You can think of it as a contract between the servers on your network: What information do they need to exchange in order to interoperate?

## The Application Layers
The component in your applications should be iterable — that is to say, you should be able to deploy multiple instances of your layers (multiple web servers, for example). What better way to do that than the way you've been doing it historically, with defined types?

Here's where you can leverage your existing modules, profiles and roles. Remember, Puppet is about code reusability; you don't need to reinvent the wheel every time you do something.

Just to show you a quick example, here's a define type for my database layer:

{% highlight puppet %}
    define apporch_blog::db (
      $dbuser,
      $dbpass,
      $host = $::fqdn,
      ){
       $override_options = {
         'mysqld' => {
           'bind-address' => '0.0.0.0',
         }
       }
        firewall { '101 Allow Connections to Database':
          dport    => 3306,
          proto    => 'tcp',
          action  => 'accept',
        }
        # For simplicity, I’m declaring the mysql::server class here but you’d normally do this outside of the define (Preferably in Hiera).

        class {'::mysql::server':
          root_password    => 'whatever',
          override_options => $override_options,
        }
        mysql::db { $name:
          user     => $dbuser,
          password => $dbpass,
          host     => '%',
          grant    => ['ALL PRIVILEGES'],
        }
      }
    }

{% endhighlight %}
As you can see, this is pretty much standard; I'm creating a new type that defines a database. Now going back to the environment service resource I described before, I could potentially produce an SQL resource out of this database, so let's modify the code to do that:

{% highlight puppet %}
    define apporch_blog::db (
      $dbuser,
      $dbpass,
      $host = $::fqdn,
      ){
       $override_options = {
         'mysqld' => {
           'bind-address' => '0.0.0.0',
         }
       }
       firewall { '101 Allow Connections to Database':
         dport    => 3306,
         proto    => 'tcp',
         action  => 'accept',
       }
       # It is a bad idea to instantiate a class is this way, since if I'm trying to apply the db class to multiple nodes the catalog compilation may fail, but for simplicity:
       class {'::mysql::server':
         root_password    => 'whatever',
         override_options => $override_options,
       }
       mysql::db { $name:
         user     => $dbuser,
         password => $dbpass,
         host     => '%',
         grant    => ['ALL PRIVILEGES'],
       }
     }
     Apporch_blog::Db produces Sql {
       dbuser => $dbuser,
       dbpass => $dbpass,
       dbhost => $host,
       dbname => $name,
     }
{% endhighlight %}


Now I've told the Puppet parser that this particular define type is producing a resource, and I'm mapping the parameters of this type to the parameters of this resource.

Just as this define type produces the resource, I've created another defined type, to consume it:

{% highlight puppet %}
    define apporch_blog::web (
      $webpath,
      $vhost,
      $dbuser,
      $dbpass,
      $dbhost,
      $dbname,
      ) {
        package {['php','mysql','php-mysql','php-gd']:
          ensure => installed,
        }
        firewall { '100 Allow Connections to Web Server':
          dport    => 80,
          proto    => 'tcp',
          action  => 'accept',
        }

        include ::apache
        include ::apache::mod::php
        apache::vhost { $vhost:
          port    => '80',
          docroot => $webpath,
          require => [File[$webpath]],
        }

        file { $webpath:
          ensure => directory,
          owner => 'apache',
          group => 'apache',
          require => Package['httpd'],
        }
        class { '::wordpress':
          db_user        => $dbuser,
          db_password    => $dbpass,
          db_host        => $dbhost,
          db_name        => $dbname,
          create_db      => false,
          create_db_user => false,
          install_dir    => $webpath,
          wp_owner       => 'apache',
          wp_group       => 'apache',
        }
      }
    Apporch_blog::Web consumes Sql {

    }
{% endhighlight %}
Please note two things in this particular code extract:

* I'm declaring variables in the defined type that are going to be filled with parameters from the environment service resource. 
* I'm also declaring that this defined type consumes the environment service resource, but I'm not mapping any variables. That's because I'm using the same names for the variables.

## The Application Construct
Now let's glue everything together. The application construct allows us to model the application and declare the dependencies within the layer:

{% highlight puppet %}
    application apporch_blog (
      $dbuser  = 'wordpress',
      $dbpass  = 'w0rdpr3ss',
      $webpath = '/var/www/wordpress',
      $vhost   = 'wordpress.puppetlabs.demo',
      ) {
        apporch_blog::db { $name:
          dbuser => $dbuser,
          dbpass => $dbpass,
          export => Sql[$name],
        }
        apporch_blog::web { $name:
          webpath => $webpath,
          consume => Sql[$name],
          vhost   => $vhost,
        }
    }
{% endhighlight %}

As you can see, it's here where I'm actually defining the variables to configure the services and create the environment service resource.

## The Site Construct
As with your layers, you should be able to deploy your application in multiple instances. That's where the site construct comes into play. In my particular example, I'm defining it in the `site.pp` file for the particular environment. But here's where you actually instantiate your application and glue it to the specific nodes (using the hostnames for the specific nodes):

{% highlight puppet %}
    site {
      apporch_blog { 'example':
          nodes           => {
            Node['db.demo'] => [ Apporch_blog::Db[ 'example' ]],
            Node['www.demo'] => [ Apporch_blog::Web[ 'example' ]],
        }
      }
    }
{% endhighlight %}


In all fairness, I had the unfair advantage of using the Puppet language and modules for quite a while now, but as you can see, once again, the code is human readable and draws from the existing modules on the Puppet Forge. As you can imagine, while this is an extremely simple example, the possibilities are endless:

* Think of your microservices running in Docker containers — this is a great way of actually tying the whole application together. See an example [here](https://github.com/ncorrare/ncorrare-worldofcontainers/blob/master/manifests/profile/infoapi.pp).
* Load balancing? Sure, your web servers can produce resources that are consumed by the load balancer.
* You can easily replace the nodes in your application, assuming you have current backups. You might be able to re-deploy your database and have Puppet automatically update your web servers.

Now that you have an idea on how the language looks like, you should read the [full documentation](http://docs.puppetlabs.com/pe/latest/app_orchestration_overview.html#language-extensions-for-application-orchestration-an-overview). It’s worth noting that the [Puppet Orchestrator](https://docs.puppetlabs.com/pe/latest/orchestrator_intro.html), which is the set of tools that will actually make the orderly deployment possible, is not referenced here. But you can learn about it from the documentation linked above and below.

[*] This article was originally authored for the Puppet Labs Blog (http://blog.puppetlabs.com).
