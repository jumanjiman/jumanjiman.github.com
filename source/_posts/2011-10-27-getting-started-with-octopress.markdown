---
layout: post
title: "Getting started with Octopress"
date: 2011-10-27 16:07
comments: true
categories: Basics
---
This is my first post with [Octopress](http://www.octopress.org).
I followed the installation instructions provided with Octopress,
and it was pretty easy on my 64-bit [Fedora](http://fedoraproject.org) 15
laptop. In this post I describe:

* Two issues I encountered during setup along with their solutions
* Plugin configurations
* Reordering plugins in the sidebar
* Helpful references for Markdown

<!-- more -->

## Issues##

First, the version of *rake* required by Octopress did not match
what is installed by the gem bundle.
I fixed that by updating a single file in my git repo:

``` diff Fix rake version for Octopress
diff --git a/Gemfile.lock b/Gemfile.lock
index 6350698..aa5fff8 100644
--- a/Gemfile.lock
+++ b/Gemfile.lock
@@ -32,7 +32,7 @@ GEM
     pygments.rb (0.1.3)
       rubypython (>= 0.5.1)
     rack (1.3.2)
-    rake (0.9.2)
+    rake (0.9.2.2)
     rb-fsevent (0.4.3.1)
     rdiscount (1.6.8)
     rubypants (0.2.0)
```

The second issue did not manifest itself until I tried to 
add syntax highlighting to a code block. Luckily somebody else
had already identified the root cause and
[posted a solution](https://bitbucket.org/raineszm/rubypython/issue/7/libpython-fails-to-load-on-64-bit-centos). On my machine, I ran `cd ~/.rvm` and then
modified the offending file like this:

``` diff Fix Python path for 64-bit OS
diff --git a/gems/ruby-1.9.2-p0/gems/rubypython-0.5.1/lib/rubypython/pythonexec.rb b/gems/ruby-1.9.2-p0/gems/rubypython-0.5.1/lib/rubypython/pythonexec.rb
index d265cf9..c42c14d 100644
--- a/gems/ruby-1.9.2-p0/gems/rubypython-0.5.1/lib/rubypython/pythonexec.rb
+++ b/gems/ruby-1.9.2-p0/gems/rubypython-0.5.1/lib/rubypython/pythonexec.rb
@@ -59,6 +59,7 @@ class RubyPython::PythonExec
         locations << File.join("/opt/lib", name)
         locations << File.join("/usr/local/lib", name)
         locations << File.join("/usr/lib", name)
+        locations << File.join("/usr/lib64", name)
       end
     end
 
```

## Plugin configuration ##

Plugins were ridiculously easy to configure. For example,
enabling the Google Plus stuff was simply a matter of adding
my profile number:

``` diff Add Google Plus User ID
diff --git a/_config.yml b/_config.yml
index cdc7594..002b87b 100644
--- a/_config.yml
+++ b/_config.yml
@@ -75,7 +75,7 @@ google_plus_one_size: medium
 
 # Google Plus Profile
 # Hidden: No visible button, just add author information to search results
-googleplus_user:
+googleplus_user: 114044653383272567071
 googleplus_hidden: false
 
 # Pinboard
```

For Disqus (comments), I signed up at disqus.com and registered
the URL [http://jumanjiman.github.com](http://jumanjiman.github.com)
under the shortname *jumanjiman*.
Then I modified my config to be:

``` diff Add Disqus shortname
diff --git a/_config.yml b/_config.yml
index 002b87b..024933b 100644
--- a/_config.yml
+++ b/_config.yml
@@ -87,7 +87,7 @@ delicious_user:
 delicious_count: 3
 
 # Disqus Comments
-disqus_short_name:
+disqus_short_name: jumanjiman
 disqus_show_comment_count: false
 
 # Google Analytics
```

## Reordering plugins in the sidebar ##

I decided I wanted the sidebar plugins to appear
slightly differently than default.
I changed the main config:

``` diff Reorder sidebar order
diff --git a/_config.yml b/_config.yml
index 024933b..fbcb630 100644
--- a/_config.yml
+++ b/_config.yml
@@ -43,7 +43,7 @@ titlecase: true       # Converts page and post titles to tilecase
 
 # list each of the sidebar modules you want to include, in the order you want them to appear.
 # To add custom asides, create files in /source/_includes/custom/asides/ and add them to the list like 'custom/asides/custom_aside_name.html'
-default_asides: [asides/recent_posts.html, asides/github.html, asides/twitter.html, asides/delicious.html, asides/pinboard.html, asides/googleplus.html]
+default_asides: [asides/recent_posts.html, asides/googleplus.html, asides/twitter.html, asides/github.html, asides/delicious.html, asides/pinboard.html]
 
 # Each layout uses the default asides, but they can have their own asides instead. Simply uncomment the lines below
 # and add an array with the asides you want to use.
```

## References ##

Here are some references that I found useful for
getting acquainted with Markdown:

* [Markdown cheat sheet](http://warpedvisions.org/projects/markdown-cheat-sheet/)
* [Examples for Octopress](http://octopress.org/docs/blogging/plugins/)

