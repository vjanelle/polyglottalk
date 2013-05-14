## Classes

Classes are named blocks of Puppet code, which are not applied until they are invoked by name.  These are typically medium to large size chunks of functionality that only need to run once (such as package resources, services, global file templates, etc…).  More fine grained or repeatedable pieces of information can be configured through resources.  Try to think of these as singletons that can only be invoked (with parameters, more into this later) once.

### Defining a class

The most simple class is one declared by a name, with no parameters and no content:

<pre>
class polyglot {
}
</pre>

To invoke this code you invoke it.  Puppet has two methods of doing so.  A more complicated class has parameters, with optional defaults:

<pre>
class polyglot (
  $student = "Robert Paulson"
) {
  notice("His name was ${student}")
}
</pre>

Invoking this class has an optional parameter of "student", which will change the output in the notice function.  You can do this in a few ways…

> On a side note, declaring a parameter in the class without a default will cause the compilation to **fail**.  This is useful as a circuit-breaker to make sure that parameter contents are defined, typically with the resource-like declaration in pure puppet, or with using hiera or an ENC in the master-agent setup.  You may wish to set the default to `undef`, and add an if test, with the `fail()` function to provide a more descriptive error message.

#### Include-like

The `include`, `require`, and `hiera_include` functions let you safely declare a class **multiple** times.  No matter how many times you declare it, a class will only be added to the catalog *once*.  This can allow classes or defined types to manage their own dependancies, and lets you create overlapping 'role' classes where functionality (such as requiring a package for a specific language, database connectivity, etc) can exist in mulitple roles.

Include-like behaviour relies on external data and defaults for parameter data - if your class does not use defaults, or include functions in the declaration to lookup these resources (via `extlookup` or via `hiera_lookup()`), catalog compilation will fail:

<pre>
test.pp:
class polyglot ( $parameter ) { }
include polyglot
</pre>

Will result in:
<pre>
[0] user@host:~$ puppet apply test.pp
Error: Must pass parameter to Class[Polyglot] at test.pp:7 on node host.local
Error: Must pass parameter to Class[Polyglot] at test.pp:7 on node host.local
</pre>

> The [example42-puppi](https://github.com/example42/puppi) module provides some helper functions which can make transitioning between multiple styles of default variables easier.

#### Resource-like

With resource-like declarations you can specify parameters in the puppet DSL.  This allows you to override parameters at compilation time, and will fall back to external data (extlookup, enc, hiera, etc) for any data you don't specify.  The caveat of this is that you can only declare the class once:

<pre>
class polyglot (
  $student = "Robert Paulson"
) {
  notice("His name was ${student}")
}

class { 'polyglot':
	student => 'Vincent Janelle',
}
</pre>
Produces:
<pre>
Notice: Scope(Class[Polyglot]): His name was Vincent Janelle
</pre>

From the puppet manual: "Why Do Resource-Like Declarations Have to Be Unique?""
 > This is necessary to avoid paradoxical or conflicting parameter values. Since overridden values from the class 
> declaration always win, are computed at compile-time, and do not have a built-in hierarchy for resolving conflicts, 
> allowing repeated overrides would cause catalog compilation to be unreliable and parse-order dependent. 
> This was the original reason for adding external data bindings to include-like declarations: since external data is set 
> before compile-time and has a fixed hierachy, the compilercan safe safely rely on it without risk of conflicts.

When puppet talks about parse-order, it is specifically talking how the puppet DSL is read off the disk.  As data is interpreted, it loads in resources.  Certain functions can be used (such as `defined()` which will make a decision based off of the current state of the catalog as it's being compiled, and can change the way the DSL is being interpreted.  Mostly this is from a top-down perspective.  This is a highly complicated topic and it isn't something you're expected to know at 


