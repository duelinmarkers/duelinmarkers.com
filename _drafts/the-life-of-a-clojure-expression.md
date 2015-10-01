---
title: 'The Life of a Clojure Expression: A Quick Tour of Clojure Internals'
---

This is a written version of
[my Clojure/West 2015 presentation]({% post_url 2015-04-22-my-clojurewest-presentation %}),
for those who'd rather read than watch a video.

This is basically a code walkthrough of a thin slice of [Clojure](http://clojure.org/)'s
reader and compiler.
These aren't things a Clojure developer needs to think about day in and day out,
but it can't hurt to understand what's going on behind the scenes.
It also might help demystify the compiler enough to get some people who are already
comfortable with the JVM to warm up to Clojure.


Overview
--------

At a high level, we'll be looking at the "R" and "E" in "REPL."

- `clojure.core/read` and `clojure.lang.LispReader#read` [&darr;](#read),
- `clojure.core/eval` and `clojure.lang.Compiler#eval` [&darr;](#eval),
  which breaks down into
  - `Compiler.analyze` [&darr;](#analyze) and
  - `Compiler$Expr.emit` [&darr;](#emit).
- Then, *runtime*[&darr;](#runtime)!

*[REPL]: Read Eval Print Loop


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

It's the map literal `{:foo "bar" :baz v}`,
appearing in the context of a `fn`-definition where `v` is an argument to the function.
That context in a function is important,
but at certain points I'll hand-wave past the details of
what gets the function compiled, because it's just too much.
But my goal is *not* to hand-wave past any details of taking that map-literal
from a series of characters, through Java bytecode, to pumping out fresh maps at runtime.

We'll also discuss some variations that can lead down different paths.

{% highlight clojure %}
;; a constant
{:foo "bar" :baz 23}

;; w/ a runtime-calculated key
{v "bar" :baz 23}

;; w/ (> (count kv-pairs) 8)
{:a 1 :b 2 :c 3 :d 4 :e 5 :f 6 :g 7 :h 8 :i v}
{% endhighlight %}


Read
----

The reading process is about consuming characters from a
[`java.io.PushbackReader`](https://docs.oracle.com/javase/8/docs/api/java/io/PushbackReader.html)
(i.e., a `Reader` that lets you unread characters) and producing *forms*.
The key takeaway is that Clojure is homoiconic, so the forms it returns are instances of
the same types of data structures we use all the time in idiomatic Clojure:
lists, vectors, symbols, strings, etc.

> ### Why a PushbackReader?
>
> There are several cases where the reader needs to back up.
> One example is when it finds a `+` or `-` at the beginning of a form.
> That could indicate the start (or entirety) of a symbol (as in `clojure.core/+`)
> *or* it could be part of a number literal (like `-2`). After consuming a `+` or `-`,
> it reads the next character.
> If [it's a digit](https://docs.oracle.com/javase/8/docs/api/java/lang/Character.html#isDigit-char-),
> it pushes the digit character back into the `PushbackReader` and goes down the `readNumber` path.
> (That's the same path it would have gone down if it had encountered a digit at the beginning of a form.)

Awkwardly, `clojure.core/read` is just a passthrough to the `LispReader` java class,
so those data structures we use all the time in idiomatic Clojure are created
and manipulated in non-idiomatic Java.

{% highlight clojure %}
(defn read
  ,,, ; many boring arities elided
  ([stream ,,,] ; stream is a java.io.PushbackReader
   (clojure.lang.LispReader/read stream
                                 ,,,)))
{% endhighlight %}

Here are some highlights from `LispReader.java`.

{% highlight java %}
static IFn[] macros = new IFn[256];
static {
  ,,,
  macros['{'] = new MapReader();
  ,,,
}
{% endhighlight %}

First off we have a static array of `IFn`s&mdash;the Java interface representing Clojure
functions&mdash;indexed by the character indicating the type of data they read.
(I thought that was pretty clever.
Who needs a `Map<Character,IFn>` when you can just use `char`s as indices in an `IFn` array?
Not Rich Hickey.)

So when the reader encounters an open curly brace `{` starting a new form,
it finds `MapReader` in the `macros` array and passes the `PushbackReader` along to it
(along with some other stuff I've elided below).
`MapReader`, along with similar classes for other syntactic forms,
is nested inside `LispReader`.

{% highlight java %}
static class MapReader extends AFn {
  public Object invoke(Object reader, ,,,) {
    PushbackReader r = (PushbackReader) reader;
    Object[] a = readDelimitedList('}', r, ,,,);
    if((a.length & 1) == 1)
      throw Util.runtimeException("Odd # of forms!");
    return RT.map(a);
  }
}
{% endhighlight %}

`MapReader` uses `readDelimitedList` to read everything up to the matching close curly brace
into an `Object` array,
ensures it found an even number of forms,
and creates a map using `RT.map`.

{% highlight java %}
static public IPersistentMap map(Object... init){
  if(init == null)
    return PersistentArrayMap.EMPTY;
  else if(init.length <= PersistentArrayMap.HASHTABLE_THRESHOLD) // 16
    return PersistentArrayMap.createWithCheck(init);
  return PersistentHashMap.createWithCheck(init);
}
{% endhighlight %}

Since the `Object` array is small, it uses `PersistentArrayMap.createWithCheck`.

{% highlight java %}
static PersistentArrayMap createWithCheck(Object[] init){
 for(int i=0; i < init.length; i += 2) {
  for(int j=i+2; j < init.length; j += 2) {
   if(equalKey(init[i], init[j]))
    throw new IllegalArgumentException("Duplicate key:" + init[i]);
  }
 }
 return new PersistentArrayMap(init);
}
{% endhighlight %}

Ensure keys are unique and create a `PersistentArrayMap` around the array.

{% highlight java %}
public PersistentArrayMap(Object[] init){
  this.array = init;
  this._meta = null;
}
{% endhighlight %}

We now have the equivalent of this quoted form.

{% highlight clojure %}
'(defn m [v] {:foo "bar" :baz v})
{% endhighlight %}

It's a list with two symbols followed by a one-element vector followed by a map.
The map has two keyword keys, one string value, and one symbol value.
The reader hasn't done anything to correlate the symbol `v` in the vector
with the symbol `v` in the map.

The fact that Clojure data structures are used to represent Clojure source
at this stage is what makes macros both powerful and easy to write.
This is the payoff of LISP's
[homoiconicity](https://en.wikipedia.org/wiki/Homoiconicity).

So much for `read`. Next we get into the good stuff!


Eval
----

Again we have a handy function in `clojure.core` that just passes through
to Java code.

{% highlight clojure %}
(defn eval [form]
  (clojure.lang.Compiler/eval form))
{% endhighlight %}

Now, `clojure.lang.Compiler/eval` is a LOT to take in.
So before we look at the real thing, here's a pseudo-code overview of `eval`.

{% highlight scala %}
eval(form) -> Object {
  form = macroexpand(form);
  Expr expr = analyze(form);
  return expr.eval()
}
{% endhighlight %}

It takes in a form, like the reader just gave us, and returns an `Object` result.
It gets from form to object by

- macroexpanding the form,
- analyzing the form into an `Expr` ("expression"), and
- having the `Expr` eval itself as it sees fit. (That's *not* a recursive call to `eval`.)

So what's an `Expr`? It's an interface that's implemented for each type of expression
in the language. (There are about forty types.)

{% highlight java %}
interface Expr {
  Object eval();
  void emit(C ctx, ObjExpr objx, GeneratorAdapter gen);
  boolean hasJavaClass();
  Class getJavaClass();
  // And often there's this static fn:
  //   static Expr parse(C ctx, CORRECT_TYPE form);
  // For most special forms, there's an IParser:
  //   interface IParser{
  //     Expr parse(C ctx, Object form) ;
  //   }
}
{% endhighlight %}

With a lot of detail elided, here's the "real" eval method.
Some lines have intentionally been left super-long, because you can just read
the comments I've inserted at the beginning. But if you're curious, you can scroll right.

{% highlight java %}
public static Object eval(Object form, boolean _) {
  ,,,
  form = macroexpand(form);
  if(/* form is a (do ...) */ form instanceof ISeq && Util.equals(RT.first(form), DO))
  { /* eval each form, returning the last. */
    ISeq s = RT.next(form);
    for(; RT.next(s) != null; s = RT.next(s))
      eval(RT.first(s), false); // recursive call
    return eval(RT.first(s), false);
  }
  else if(/* form is not a "def" */ (form instanceof IType) || (form instanceof IPersistentCollection && !(RT.first(form) instanceof Symbol && ((Symbol) RT.first(form)).name.startsWith("def"))))
  {
   /* wrap it in a 0-arity fn and invoke that */
   ObjExpr fexpr = (ObjExpr) analyze(C.EXPRESSION,
     RT.list(FN, PersistentVector.EMPTY, form), "eval" + RT.nextID());
   IFn fn = (IFn) fexpr.eval();
   return fn.invoke();
  } else {
   Expr expr = analyze(C.EVAL, form);
   return expr.eval();
  }
}
{% endhighlight %}

First, forms will be macroexpanded. If they're not macro invocations,
the `macroexpand` call will just return the input form.

Next, if the form is a `(do...)` it's "unrolled" and each form of its body is `eval`'d as if
it were a top-level form, and the result of `eval`ing the last form is returned.
That's important for macros to be able to return `do` forms where some forms create
global state (e.g., `def`ing a var)
and later forms depend on that state in order to compile
(e.g., referring to the var just `def`ed).

If the form being `eval`'d is not a "def" of some kind, it's wrapped in a zero-arity `fn`
and that `fn` is invoked.
I don't know why that's necessary.
(If you do, please [let me know](http://twitter.com/duelinmarkers).)

Regardless of why, I find it interesting to see *how* the compiler wraps a form in a zero-arity `fn`.
It could use some internal API for doing that, but it just does what you'd do in
Clojure code, albeit awkwardly from Java code.
Instead of analyzing the input `form`, it analyzes this:

{% highlight java %}
RT.list(FN, PersistentVector.EMPTY, form)
{% endhighlight %}

That's just a list with the symbol `fn` at the beginning, then an empty args vector,
then the input form as the `fn`'s body. It's the java version of this Clojure syntax.

{% highlight clojure %}
(fn [] form)
{% endhighlight %}

Finally, if the form we're analyzing *is* some type of "def," as most top-level forms are,
we analyze it into an `Expr` and return the result of its `eval` method.

Let's get back to *our* form. The `macroexpand` at the beginning will turn our

{% highlight clojure %}
(defn m [v] {:foo "bar" :baz v})
{% endhighlight %}

into (more or less)

{% highlight clojure %}
(def m (fn [v] {:foo "bar" :baz v}))
{% endhighlight %}

So that's the form that will go through the `eval` we looked at above.
Since it's a `def`, it will follow the straightforward `analyze`-and-`expr.eval` path.


Analyze
-------

Here's a taste of `analyze`.

{% highlight java %}
static Expr analyze(C ctx, Object form, String name) {
  Class fclass = form.getClass();
  if(fclass == Symbol.class)
    return analyzeSymbol((Symbol) form);
  else if(fclass == Keyword.class)
    return registerKeyword((Keyword) form);
  ,,, /* etc, etc */
  else if(form instanceof ISeq)
    return analyzeSeq(ctx, (ISeq) form, name);
  else if(form instanceof IPersistentMap)
    return MapExpr.parse(ctx, (IPersistentMap) form);
  ,,, /* etc, etc */
}
{% endhighlight %}

Depending on the type of our form, we dispatch to some type-specific analysis.
All the branches I haven't edited out above are branches needed to fully analyze
our expression.

Our outermost `def` is wrapped in a list, so off to `analyzeSeq`!

{% highlight java %}
static Expr analyzeSeq(C ctx, ISeq form, String name) {
  Object op = RT.first(form);
  /* elided nil-check, inline */
  if(op.equals(FN))
    return FnExpr.parse(ctx, form, name); // our fn
  IParser p;
  else if((p = (IParser) specials.valAt(op)) != null)
    return p.parse(ctx, form); // our def
  else
    return InvokeExpr.parse(ctx, form);
}
{% endhighlight %}

It looks at the `op` at the beginning of the form.
If it's `fn`, that's handled specially due to `fn`s having names that often come from
outside their `fn` form.
If it's in the `specials` map (special forms), we hand off to the corresponding `IParser`.
Otherwise, it must be a plain old `fn`-invoke.

Now for some hand-waving.

Our `def` is in the `specials` map. It gets analyzed into a `DefExpr`,
whose `eval` evals its init expression (our `fn`). Trust me.

Our `fn` is analyzed into a `FnExpr`, whose eval compiles the `fn`-body,
each `Expr` of which gets analyzed and emitted. Trust me.

![Hand-waving](/images/hand-waving-fallon.gif)

Our function body is a map. You can see in `analyze` above that will mean
a call to `MapExpr.parse`. Here we are.

{% highlight java %}
public static class MapExpr implements Expr{
 // Each MapExpr has a vector of keyvals.
 public final IPersistentVector keyvals;
 // ...
 static public Expr parse(C ctx, IPersistentMap form) {
  IPersistentVector keyvals = PersistentVector.EMPTY;
  // Iterate through the entries in the unevaluated map.
  for(ISeq s = RT.seq(form); s != null; s = s.next()) {
   IMapEntry e = (IMapEntry) s.first();
   // Analyze each key and value, adding the result to keyvals.
   Expr k = analyze(ctx, e.key());
   Expr v = analyze(ctx, e.val());
   keyvals = (IPersistentVector) keyvals.cons(k);
   keyvals = (IPersistentVector) keyvals.cons(v);
   // elided constantness, k uniqueness checks
   ,,,
  }
  Expr ret = new MapExpr(keyvals);
  // elided special cases:
  // map with metadata, non-unique keys, entirely constant maps
  ,,,
  return ret;
 }
}
{% endhighlight %}

`MapExpr` has a vector of keys and values, populated with `Expr`s representing each form.
One of the "special cases" I elided above checks to see if the map is *entirely* constant
(i.e., everything in `keyvals` is a literal), in which case `parse` returns a `ConstantExpr`
and a fn with such a map in its body would just read a static constant value rather than
creating a fresh map on each run.

In our case, we end up with `keyvals` containing
a `KeywordExpr`, a `StringExpr`, another `KeywordExpr`, and a `LocalBindingExpr`.
The first three are our literals.
The `LocalBindingExpr` is the internal representation of the symbol `v` from the
unevaluated map the reader produced.
When the symbol was analyzed, the compiler looked at a dynamic var of in-scope locals
and found `v`, so the `v` in the map was analyzed into a use of that local.

As I mentioned earlier, `FnExpr`'s `eval` calls `emit` on each `Expr` of its body
to emit bytecode for a Java class.


Emit
----

JVM bytecode is emitted using a repackaged copy of the
[ASM bytecode library](http://asm.ow2.org/).

*[ASM]: According to the ASM user guide, "the ASM name does not mean anything: it is just a reference to the __asm__ keyword in C, which allows some functions to be implemented in assembly language."

A map literal compiles to a static call to either `RT.mapUniqueKeys(Object[])`,
if the compiler can guarantee the keys are unique,
or `RT.map(Object[])` if key uniqueness needs to be checked at runtime.
`MapExpr.emit` does a bit more analysis of its keys to determine which.

{% highlight java %}
public static class MapExpr implements Expr{
 public final IPersistentVector keyvals;
 static Method mapMethod = Method.getMethod(
   "clojure.lang.IPersistentMap map(Object[])");
 static Method mapUniqueKeysMethod = Method.getMethod(
   "clojure.lang.IPersistentMap mapUniqueKeys(Object[])");

 public void emit(C ctx, ObjExpr objx, GeneratorAdapter gen){
  // elided: iterate through keyvals to determine:
  boolean allKeysConstant = /* is every k instanceof LiteralExpr? */,,,;
  boolean allConstantKeysUnique = /* no two literal k.eval() results equal */,,,;
  ,,,
  MethodExpr.emitArgsAsArray(keyvals, objx, gen);
  if((allKeysConstant && allConstantKeysUnique)
     || (keyvals.count() <= 2))
   gen.invokeStatic(RT_TYPE, mapUniqueKeysMethod);
  else
   gen.invokeStatic(RT_TYPE, mapMethod);
  if(ctx == C.STATEMENT) gen.pop();
 }
}
{% endhighlight %}

The `Method` and `GeneratorAdapter` classes referred to above are ASM stuff.
The particulars above show a tiny example of how JVM bytecode (and ASM) work.
You emit your arguments, then your static invocation.
Then, if this expression occurs in a "STATEMENT" context
(i.e., it's not the last expression in a function body or `do` block),
you emit a bytecode to discard the return value of that static invocation.

If you put that bytecode emission together with my earlier hand-waving,
our `fn` has now been compiled into the equivalent of this Java class
and loaded into an in-memory classloader.

{% highlight java %}
import clojure.lang.AFunction;
import clojure.lang.Keyword;
import clojure.lang.RT;

public final class a_map$m extends AFunction {

  public static final Keyword FOO = RT.keyword(null, "foo");
  public static final Keyword BAZ = RT.keyword(null, "baz");

  @Override
  public Object invoke(Object arg) {
    return RT.mapUniqueKeys(new Object[] {FOO, "bar", BAZ, arg});
  }
}
{% endhighlight %}

Easy!


Runtime
-------

Somewhere else, we hope, will be a call to our function. It might look like this.

{% highlight clojure %}
(m "Thanks")
{% endhighlight %}

That piece of code will run through some of the same things we've already looked at,
as well as some things we haven't looked at, and end up as bytecode equivalent to
this Java code.

{% highlight java %}
// TODO expand this to an abbreviated class def?
M_VAR               // a static constant in the calling fn's class
  .getRawRoot()     // reads a volatile field in the clojure.lang.Var
  .invoke("Thanks") // invokeinterface
{% endhighlight %}

All our class's invoke does is create an Object array and call `RT.mapUniqueKeys`.

{% highlight java %}
static public IPersistentMap mapUniqueKeys(Object... init){
  if(init == null)
    return PersistentArrayMap.EMPTY;
  else if(init.length <= PersistentArrayMap.HASHTABLE_THRESHOLD) // 16 again
    return new PersistentArrayMap(init);
  return PersistentHashMap.create(init);
}
{% endhighlight %}

We've already seen that the `PersistentArrayMap` constructor just uses the `init`
array to back the array-map, so that's it. We've seen it all!

{% highlight clojure %}
{:foo "bar" :baz "Thanks"}
{% endhighlight %}

Notice that our `fn`'s call to `mapUniqueKeys` is an excellent candidate for inlining by the JVM.
Since the `Object` array is created right there with a length of 4, it can skip the
null-check and length-check and go straight to creating the `PersistentArrayMap`.


What Else?
----------

I'm just about out of material here. *I promise!*


### Literals are Faster

I found it interesting to discover that map-literals with
compile-time-known-unique keys create maps with a mechanism more efficient
than any other supported means. There is no other call to `RT.mapUniqueKeys`
in the Clojure codebase and no supported way for you get at it other than
via a `MapExpr`. (Do note though that your macros can return maps,
which will hit the same fast code path as a literal,
so you can have fast map creation without actually using a map-literal in your code.)

There are other methods like that in `RT`, by the way.
Open up clojure itself in IntelliJ or another IDE that analyzes
a whole project, and it will tell you that `mapUniqueKeys` and several
other methods are unused.
It (understandably) has no idea that there are calls to those methods "hidden"
in the compiler's `Expr#emit` implementations.


### clojure.tools Stuff

For delving deeply into Clojure code, it's worth knowing about
[tools.reader](https://github.com/clojure/tools.reader) and
[tools.analyzer.jvm](https://github.com/clojure/tools.analyzer.jvm).
Tools.analyzer basically stands in for `clojure.lang.Compiler.analyze`,
which we discussed earlier, but instead of returning an `Expr`, it returns
a clojure map describing the form it analyzed.
Among other useful things, for performance-sensitive code,
it provides visibility into what function-invocations have been
inlined into static method calls,
and whether values are being treated as primitives or being boxed.

However, tools.analyzer didn't do me any good figuring out how a map-literal
worked: the result of analysis is a map similar to this.
(Some noisy details have been removed.)

{% highlight clojure %}
{:op :map
 :children [:keys :vals]
 :keys [{:op :const :literal? true
         :val :foo :tag clojure.lang.Keyword :form :foo}
        {:op :const :literal? true
         :val :baz :tag clojure.lang.Keyword :form :baz}]
 :vals [{:op :const :literal? true
         :val "bar" :tag java.lang.String :form "bar"}
        {:op :local
         :name v__#0
         :local :arg
         :arg-id 0
         :variadic? false
         :tag java.lang.Object
         :form v}]
 :tag clojure.lang.PersistentArrayMap
 :form {:foo "bar" :baz v}}
{% endhighlight %}

Basically it tells you you've got a literal and leaves it at that.

There is also [tools.emitter.jvm](https://github.com/clojure/tools.emitter.jvm),
which supports a `{:debug? true}` argument to output bytecode details (somewhat)
human-readably.
That comes very close to giving you exactly what we looked at above,
which is pretty awesome.
Do keep in mind, though, that all of these libraries provide *alternative
implementations* of Clojure's internals, not insight into its *actual* internals.
(For example, see tools.emitter.jvm's
[Differences from Clojure](https://github.com/clojure/tools.emitter.jvm#differences-from-clojure).)


### javap

What I used to examine compiled Clojure code was mostly `javap`, which is a great tool.


Wrap it up already!
-------------------

Ok, ok!

Maybe next time something doesn't behave as you'd expect, you'll roll up your sleeves
and dig into the internals.
At the very least,
hopefully you understand a little more now about how your Clojure code works
than you did before.

Thanks for reading.
If you have questions, feel free to [hit me up on Twitter](http://twitter.com/duelinmarkers)
or elsewhere.
If the answer doesn't fit in 140 characters, maybe I'll write another ridiculously long
blog post.
