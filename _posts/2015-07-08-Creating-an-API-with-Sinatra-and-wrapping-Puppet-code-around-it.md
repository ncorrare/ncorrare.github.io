---
layout: post
title: Creating an API with Sinatra and wrapping Puppet Code around it
categories: [general, puppet, code]
tags: [ruby, sinatra, things I write on trains]
fullview: true
---
Another week, another commute to a different country. You do have to love Europe!.
Anyway, with an hour on my hands, I starting thinking back to, well, probably the question I get asked the most: "What can you do with Puppet?".
There is probably no short or straight answer to it, so I tend to start enumerating anything that might be relevant, and lately I've been finishing up with, "and also, if you have anything that expose resources through an API, well you can wrap Puppet Code around it!". An excellent example of this, could be Gareth's AWS module. It is also, too complicated to illustrate a principle, since it's a fairly complex module, with quite a bit of Ruby code. For those who don't know me, I don't do Ruby... well... kind of.

Ruby does have a number of brilliant gems (see what I did there?), and since I have been hearing about Ruby, the one that always caught my eye, was Sinatra.

![Not this Sinatra. Thanks wikimedia commons for the image!](https://upload.wikimedia.org/wikipedia/commons/2/2d/Frank_Sinatra2,_Pal_Joey.jpg)

It is one of those things that look amazingly simple, that are really powerful, and of course, I always wanted to play around with. With all these in mind, I decided to write an extremely simple REST API, and a Puppet Module around it with very simple curl calls, to demo how this would look like, and of course, I did it my way.

The absolutely basic Sinatra syntax is:

{% highlight ruby %}

httpverb '/httppath' do
  rubycode
end
{% endhighlight %}

So let's look at a commented example:

{% highlight ruby %}
# HTTP GET verb to retrieve a specific item. Returns 404 if item is not present.
# HTTP Verb and Path definition
get '/items/:key' do
  # Check if the string is actually present in the file and give a quick message in case this is being opened with a browser. Sinatra returns 200 OK as default.
  unless File.readlines("file.out").grep(/#{params['key']}/).size == 0 then
    "#{params['key']} exists in file!"
  else
  # or return the appropiate http code.
    status 404
  end
end
{% endhighlight %}

So with a few bits and bolts of Ruby code here and there, here's my full API example, with three basic verbs.

{% highlight ruby %}
#Extremely Simple API with three basic verbs.
require 'sinatra'
require 'fileutils'
require 'tempfile'

# HTTP PUT verb creates an item on the file. Returns HTTP Code 422 if already exists.
put '/items/:key' do
  unless File.readlines("file.out").grep(/#{params['key']}/).size != 0 then
    "Created #{params['key']}"
    open('file.out', 'a') { |f|
      f.puts "#{params['key']}\n"
    }
  else
    status 422
  end
end

# HTTP GET verb to retrieve a specific item. Returns 404 if item is not present.
get '/items/:key' do
  unless File.readlines("file.out").grep(/#{params['key']}/).size == 0 then
    "#{params['key']} exists in file!"
  else
    status 404
  end
end

# HTTP GET verb to retrieve all the items.
get '/items' do
  file = File.open("file.out")
  contents = ""
  file.each {|line|
            contents << line
  }
  "#{contents}"
end

# HTTP DELETE verb to remove a specific item. With a horrendous hack to recreate the file without the specific key. Returns 404 if the item is not present.
delete '/items/:key' do |k|
  unless File.readlines("file.out").grep(/#{params['key']}/).size == 0 then
    tmp = Tempfile.new("extract")
    open('file.out', 'r').each { |l| tmp << l unless l.chomp ==  params['key'] }
    tmp.close
    FileUtils.mv(tmp.path, 'file.out')
    "#{params['key']} deleted!"
  else
    status 404
  end
end
{% endhighlight %}

So I'm sure there will be a lot of Ruby experts criticizing how that code is written, to be honest, and taking into account is the first piece of Ruby code I wrote, and that I lifted some examples from everywhere, I'm quite happy with my Frankenstein API.

So let's curl around to see how my API works.

If I PUT an item there, it saves it on the file, but of course if the item exists, it won't allow me to do it again:

{% highlight bash %}
[ncorrare@risa ~]# curl -vX PUT http://localhost:4567/items/item2
* Hostname was NOT found in DNS cache
*   Trying ::1...
* connect to ::1 port 4567 failed: Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 4567 (#0)
> PUT /items/item2 HTTP/1.1
[...]
>
< HTTP/1.1 200 OK
< Content-Type: text/html;charset=utf-8
[...]
* Connection #0 to host localhost left intact

[ncorrare@risa ~]# curl -vX PUT http://localhost:4567/items/item2
* Hostname was NOT found in DNS cache
*   Trying ::1...
* connect to ::1 port 4567 failed: Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 4567 (#0)
> PUT /items/item2 HTTP/1.1
[...]
>
< HTTP/1.1 422 Unprocessable Entity
[...]
<
* Connection #0 to host localhost left intact
[ncorrare@risa ~]#
{% endhighlight %}

Note that when I try to PUT the item the second time, it returns an HTTP code of 422. By the way, I spent a bit of time researching which should be the right http code to return in this case. I think I got it right but if anyone has a table documenting how to map these, please do send it over.

By the way, depending on the sinatra / webrick version, you may need to specify a Content-Length (even if its zero) or you might get an HTTP/1.1 411 Length Required response. It would look like this

{% highlight bash %}
/usr/bin/curl -H 'Content-Length: 0' -fIX PUT http://localhost:4567/items/item1
{% endhighlight %}

Now how about a couple of GETs
{% highlight bash %}
[ncorrare@risa ~]# curl -vX GET http://localhost:4567/items
* Hostname was NOT found in DNS cache
*   Trying ::1...
* connect to ::1 port 4567 failed: Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 4567 (#0)
> GET /items HTTP/1.1
[...]
>
< HTTP/1.1 200 OK
< Content-Type: text/html;charset=utf-8
[...]
<
item1
item2
* Connection #0 to host localhost left intact
[ncorrare@risa ~]# curl -vX GET http://localhost:4567/items/item2
* Hostname was NOT found in DNS cache
*   Trying ::1...
* connect to ::1 port 4567 failed: Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 4567 (#0)
> GET /items/item2 HTTP/1.1
[...]
>
< HTTP/1.1 200 OK
< Content-Type: text/html;charset=utf-8
[...]
* Connection #0 to host localhost left intact
item2 exists in file!
[ncorrare@risa ~]# curl -vX GET http://localhost:4567/items/item3
* Hostname was NOT found in DNS cache
*   Trying ::1...
* connect to ::1 port 4567 failed: Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 4567 (#0)
> GET /items/item3 HTTP/1.1
[...]
< HTTP/1.1 404 Not Found
< Content-Type: text/html;charset=utf-8
[...]
<
* Connection #0 to host localhost left intact
{% endhighlight %}

Please note that GET, of course, if the default action, but I'm specifying it just for documentation purposes. A GET to /items is returning the whole contents of the file, while a GET to /items/specificitem, checks if the items (a string in this case) exists on the file. A GET to a non-existent item returns 404. 

Finally, let's delete something.

{% highlight bash %}
[ncorrare@risa ~/puppetlabs/demo/razor]# curl -vX DELETE http://localhost:4567/items/item2
* Hostname was NOT found in DNS cache
*   Trying ::1...
* connect to ::1 port 4567 failed: Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 4567 (#0)
> DELETE /items/item2 HTTP/1.1
[...]
>
< HTTP/1.1 200 OK
< Content-Type: text/html;charset=utf-8
[...]
<
* Connection #0 to host localhost left intact
item2 deleted!
[ncorrare@risa ~/puppetlabs/demo/razor]# curl -vX DELETE http://localhost:4567/items/item2
* Hostname was NOT found in DNS cache
*   Trying ::1...
* connect to ::1 port 4567 failed: Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 4567 (#0)
> DELETE /items/item2 HTTP/1.1
[...]
>
< HTTP/1.1 404 Not Found
< Content-Type: text/html;charset=utf-8
[...]
<
* Connection #0 to host localhost left intact
[ncorrare@risa ~/puppetlabs/demo/razor]#
{% endhighlight %}

For all intent and purposes, that's a great API, it even has error checking!.
But the best is yet to come, now is time to actually wrap a Puppet module around it. As I'll be using exec, I've to be spot on, around managing errors. Now here's the kicker... in order for curl to return an HTTP error code >= 400 as an exit code > 0, you have to use the -f parameter in curl.

So in this case, I basically created a module (which is available along with the full API code in https://github.com/ncorrare/ncorrare-itemsapi). The key here, is the resource definition:

{% highlight ruby %}
define itemsapi (
  $ensure,
)
  {
    validate_re($ensure, ['present','absent'])
    if $ensure=='present' {
      exec { 'create item' :
        command => "/usr/bin/curl -H 'Content-Length: 0' -fIX PUT http://localhost:4567/items/${name}",
        unless  => "/usr/bin/curl -fIX GET http://localhost:4567/items/${name}",
      }
    }
    elsif $ensure=='absent' {
      exec { 'delete item' :
        command => "/usr/bin/curl -fIX DELETE http://localhost:4567/items/${name}",
        onlyif  => "/usr/bin/curl -fIX GET http://localhost:4567/items/${name}",
      }
    } else {
      fail('ensure parameter must be present or absent')
    }
  }
{% endhighlight %}

That's it! You know have a new type, to manage an API, which is fully idempotent, so you can basically:

{% highlight ruby %}
itemsapi { "whatever":
  ensure => "present",
}
{% endhighlight %}

or:

{% highlight ruby %}
itemsapi { "whatever":
  ensure => "absent",
}
{% endhighlight %}

By the way, can you find the references to Sinatra (the actual one) in the blog post? There are two (I think!)
