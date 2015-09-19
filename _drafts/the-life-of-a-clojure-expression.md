---
title: 'The Life of a Clojure Expression: A Quick Tour of Clojure Internals'
---

This is a written version of
[my Clojure/West 2015 presentation]({% post_url 2015-04-22-my-clojurewest-presentation %}),
for people like me, who'd rather read than watch a video.

This is basically a code walkthrough of a thin slice of [Clojure](http://clojure.org/)'s
reader and compiler.
These aren't things a Clojure developer needs to think about day in and day out,
but it couldn't hurt to understand what's going on behind the scenes.
It also might help demystify the compiler enough to get some people who are already
comfortable with the JVM to warm up to Clojure.


Disclaimers
-----------

- I'm not an expert on compilers, in general, or on the Clojure compiler in particular.
- We'll mostly be looking at internals, not supported API, so it's all subject to change.
- Code samples used herein will be mangled for readability and to focus on what's important.
  When you see `,,,` in the code that means I've chopped something out.


What Expression?
----------------

Let's follow the life of a single Clojure expression.
Here it is:

{% highlight clojure %}
(defn m [v]
  {:foo "bar" :baz v})
;;^---- this one ---^
{% endhighlight %}

It's the map literal `{:foo "bar" :baz v}`.
It appears in the context of a `fn`-definition, and that's important,
but at certain points I'll hand-wave past the details of
what gets the function compiled, because it's just too much.
But my goal is *not* to hand-wave past any details of taking that map-literal
from a series of characters through Java bytecode to pumping out fresh maps at runtime.

We'll also discuss some variations:

{% highlight clojure %}
;; a constant
{:foo "bar" :baz 23}

;; w/ runtime-calculated k
{v "bar" :baz 23}

;; w/ (> (count kv-pairs) 8)
{:a 1 :b 2 :c 3 :d 4 :e 5 :f 6 :g 7 :h 8 :i v}
{% endhighlight %}


Overview
--------

At a high level, we'll be looking at the "RE" in "REPL." Here's how it breaks down.

- `clojure.core/read` and `clojure.lang.LispReader#read`,
- `clojure.core/eval` and `clojure.lang.Compiler#eval`, which breaks down into
  - `Compiler.analyze` and
  - `Compiler$Expr.emit`.
- Then, *runtime*!


Read
----

The reading process is about as boring as you'd expect, so we won't look at every detail.
The key takeaway is that Clojure is homoiconic, so reading mostly results in instances of
the same types of data structures we use all the time in idiomatic Clojure.

Awkwardly, `clojure.core/read` is just a passthrough to the `LispReader` java class,
so "the same types of data structures we use all the time in idiomatic Clojure" are created
and manipulated in very non-idiomatic Java.

{% highlight clojure %}
(defn read
  ,,,
  ([stream ,,,]
   (clojure.lang.LispReader/read stream
                                 ,,,)))
{% endhighlight %}

Here are some highlights.

{% highlight java %}
static IFn[] macros = new IFn[256];
static {
  macros['{'] = new MapReader();
  ,,,
}
{% endhighlight %}

First off we have an array of `IFn`s&mdash;the Java interface representing Clojure
functions&mdash;indexed by the character indicating what type of data we're about to read.
(I thought that was pretty clever.
Who needs a `Map` with `char` keys when you can just use `char`s as array indices?
Not Rich Hickey.)

So when the reader encounters an open curly brace, it uses `MapReader` to read it.
Here's `MapReader`.

{% highlight java %}
static class MapReader extends AFn {
  public Object invoke(Object reader, Object _) {
    PushbackReader r = (PushbackReader) reader;
    Object[] a = readDelimitedList('}', ,,,);
    if((a.length & 1) == 1)
      throw Util.runtimeException("Odd # of forms!");
    return RT.map(a);
  }
}
{% endhighlight %}
