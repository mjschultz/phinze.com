--- 
wordpress_id: 3
layout: post
title: If <code>YAML.dump</code> can't produce valid YAML...
wordpress_url: http://www.beyond-syntax.com/?p=3
---
Yesterday I had fun with ruby's YAML module not loading a piece of YAML I needed it to load.  <code>Syntax Error</code>... fine--all in a day's work. But then I started looking at  the YAML in question and I realized, this was GENERATED by the module itself!

<!--more-->

First of all, the reason I was using YAML was as a quick, simple, human-readable backend for a tiny little side-project.  I chose YAML over SQLite or CSV because neither have the human-readable thing going for them, plus I'm working in Ruby and it seemed to have nice built-in support for YAML.

You can take any object you like (with obvious exceptions like streams and sockets) and just <code>YAML.dump</code> it, creating a happy string waiting to be written to the nearest file.  So given this class:
<pre lang="ruby">class Foo
  def initialize
    @bar = 'one'
    @baz = 2
    @qux = [3, 'four', 5]
  end
end</pre>
You can do something like this:
<pre lang="ruby">&gt;&gt; f = Foo.new
=&gt; #
&gt;&gt; YAML.dump(f)
=&gt; "--- !ruby/object:Foo \nbar: one\nbaz: 2\nqux: \n- 3\n- four\n- 5\n"</pre>
And that YAML string at the end there looks like this:
<pre lang="yaml">--- !ruby/object:Foo
bar: one
baz: 2
qux:
- 3
- four
- 5</pre>
So you can stuff that wherever you'd like (<code>file.yml</code> let's say) and then at some later date all you need to do is:
<pre lang="ruby">&gt;&gt; new_f = YAML.load_file('file.yml')
=&gt; #
&gt;&gt; f == new_f
=&gt; true</pre>
Fun times, no?  And most importantly fun times that don't require one learns YAML in great detail.  Well, let me show you exactly where the fun times ended for me.  For an example, let's make a simple hash with our <code>f</code>.
<pre lang="ruby">&gt;&gt; h = { f =&gt; 2 }
=&gt; {#=&gt;2}
&gt;&gt; YAML.dump(h)
=&gt; "--- \n!ruby/object:Foo ? \n  bar: one\n  baz: 2\n  qux: \n  - 3\n  - four\n  - 5\n: 2\n\n"</pre>
Seems fine, no?
<pre lang="yaml">---
!ruby/object:Foo ?
  bar: one
  baz: 2
  qux:
  - 3
  - four
  - 5
: 2</pre>
So we go about our business, right?
<pre lang="ruby">&gt;&gt; YAML.load(YAML.dump(h))
ArgumentError: syntax error on line 2, col -1</pre>
Boo.  So now I have to go and do what I was trying to avoid doing in the first place: learn about YAML.   Well after wading through the schema documents for YAML, which are long, full of BNF, and no fun.  At the end of the day it turns out that <code>YAML.dump</code> produces invalid YAML whenever an object is used as a key in a Hash.  Without going into too much detail (partially because I don't fully understand it), the question mark needs to come before the ruby object tag for it to be valid.

Here's a workaround that just does some regexp munging on the string :
<pre lang="ruby">&gt;&gt; YAML.load(YAML.dump(h).gsub(/^(!ruby.*) \? *$/) { |m| "? #{$1}" })
=&gt; {#=&gt;2}</pre>
Looks like I'll be submitting my first bug report to rubylang.
