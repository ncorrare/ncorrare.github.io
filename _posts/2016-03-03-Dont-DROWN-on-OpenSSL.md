---
layout: post
title: Don't DROWN own OpenSSL
categories: [general, puppet, code, security]
tags: [puppet, drown, openssl]
fullview: true
---

Yet another OpenSSL vulnerability in the loose. While the vendors release fixes, Puppet to the rescue!. Let's start by disabling those weak ciphers.

For the apache https server, you can use the puppetlabs-apache module to disable weak ciphers:


{% highlight puppet %}

class { 'apache::mod::ssl':
  ssl_cipher           => 'EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384
                           EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256
                           EECDH+aRSA EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5
                           !EXP !PSK !SRP !DSS !EXPORT',
  ssl_protocol         => [ 'all', '-SSLv2', '-SSLv3' ], #Default value for the module
  ssl_honorcipherorder => 'On', #Default value for the module
}

{% endhighlight %}

More information on https://forge.puppetlabs.com/puppetlabs/apache/readme#class-apachemodssl

Using Postfix? No problem, the camptocamp-postfix module can help you there:


{% highlight puppet %}

postfix::config {
    'smtpd_tls_security_level':            value => 'secure';
    'smtpd_tls_mandatory_protocols':       value => '!SSLv2, !SSLv3';
    'smtpd_tls_mandatory_exclude_ciphers': value => 'aNULL, MD5'
}

{% endhighlight %}

More information on https://forge.puppetlabs.com/camptocamp/postfix

How about IIS 7? We can use the puppetlabs-registry module to disable weak ciphers:

{% highlight puppet %}

class profile::baseline {
  registry_value { 'HKLM\System\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 2.0\Server\Enabled':
    ensure => present,
    type   => dword,
    data   => 0x00000000,
  }

  registry_value { 'HKLM\System\CurrentControlSet\Control\SecurityProviders\SCHANNEL\Protocols\SSL 3.0\Server\Enabled':
    ensure => present,
    type   => dword,
    data   => 0x00000000,
  }
}

{% endhighlight %}

More information on https://forge.puppetlabs.com/puppetlabs/registry 
