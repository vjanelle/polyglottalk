# Classes

Classes are named blocks of Puppet code, which are not applied until they are invoked by name.  These are typically medium to large size chunks of functionality that only need to run once (such as package resources, services, global file templates, etc…).  More fine grained or repeatable pieces of information can be configured through resources.  Try to think of these as singletons that can only be invoked (with parameters, more into this later) once.

## Defining a class

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

### Include-like

The `include`, `require`, and `hiera_include` functions let you safely declare a class **multiple** times.  No matter how many times you declare it, a class will only be added to the catalog *once*.  This can allow classes or defined types to manage their own dependancies, and lets you create overlapping 'role' classes where functionality (such as requiring a package for a specific language, database connectivity, etc) can exist in multiple roles.

Include-like behavior relies on external data and defaults for parameter data - if your class does not use defaults, or include functions in the declaration to lookup these resources (via `extlookup` or via `hiera_lookup()`), catalog compilation will fail:

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

### Resource-like

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

The way this works is through a special `class` pseudo resource-type.

From the puppet manual: "Why Do Resource-Like Declarations Have to Be Unique?""
 > This is necessary to avoid paradoxical or conflicting parameter values. Since overridden values from the class 
> declaration always win, are computed at compile-time, and do not have a built-in hierarchy for resolving conflicts, 
> allowing repeated overrides would cause catalog compilation to be unreliable and parse-order dependent. 
> This was the original reason for adding external data bindings to include-like declarations: since external data is set 
> before compile-time and has a fixed hierarchy, the compiler can safe safely rely on it without risk of conflicts.

When puppet talks about parse-order, it is specifically talking how the puppet DSL is read off the disk.  As data is interpreted, it loads in resources.  Certain functions can be used (such as `defined()` which will make a decision based off of the current state of the catalog as it's being compiled, and can change the way the DSL is being interpreted.  Mostly this is from a top-down perspective.  This is a highly complicated topic and it isn't something you're expected to know at the beginning, but it's useful for defined types and custom resources where you may need to depend on a class, but don't necessarily need it instantiated in a certain order.

### Using the `include` function

The `include` function is the standard way to declare classes.  You can include multiple classes as arguments, 	and it relies on external data (i.e., from data sources or default parameters) for parameters.  It can accept a single class:

<pre>
include common
include base
include ntp
</pre>

A comma separated list of classes:

<pre>
include common,ntp,base
</pre>

Or an array of classes:
<pre>
$my_classes = ['common,'ntp','base']
include $my_classes
</pre>

### Using the `require` function

The `require` function declares one or more classes, then enforces a child relationship on the surrounding container.

> This is different from the `require` meta-parameter, which is a common parameter all defined types can accept as an argument, which provides a case-by-case dependancy.

<pre>
define ntp::server (
 $port = 123,
 $bind = '0.0.0.0'
) {
  require ntp
  …
}
</pre>

What this will do is ensure that every `ntp` resource gets applied **before** every `ntp::server` resource.

### Which one should I use?

If you're prototyping out a system, I'd suggest using the resource type - it allows you to maintain all your changes and prototyping inside's puppet DSL, avoiding some of the complexity external data systems can produce.  However, it can lead to a lot more boilerplate; once you start producing systems that are designed to be replicated and used in multiple environments, you should start migrating your manifests to utilize external data for parameters (params pattern, hiera, etc).




