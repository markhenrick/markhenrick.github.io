---
layout: post
title: "Dark theme is now live!"
date: 2021-06-05 22:35:00 +0100
tags: tech coding meta
---

This site should now use `prefers-color-scheme` to correctly switch to dark mode. Apologies for blinding you

<!--more-->

[Minima](https://github.com/jekyll/minima) partially supported this. There's an official dark scheme, but it uses SCSS variables and hence has to be specified at compile time. I've converted those to native CSS variables/custom properties, but I did have to inline the use of `lighten`/`darken`. It would have been possible to work around this by indirecting the variables, but I didn't see it as worth it

Here's a code block to test syntax highlighting, since I don't have any yet

{% highlight ruby %}
def foo
  puts 'foo'
end
{% endhighlight %}