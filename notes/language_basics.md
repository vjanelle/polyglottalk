Puppet for beginners
--------------------

# Language basics

## Terminology

### Domain specific language

Puppet represents resource configurations through a __Domain Specific Language__ which "is a type of programming language or specification language in software development and domain engineering dedicated to a particular problem domain, a particular problem representation technique, and/or a particular solution technique." -- Wikipedia 

Essentially this means that puppet has a fairly unique but similar syntax to other languages to abstract the low level interactions of Ruby away from you.  You still need to be aware that it's there, and you may elect to write functional pieces in Ruby in the future, but for now you won't need to know that it's there.

### Resources

<pre title="User resource">
user { 'vjanelle': <-- title
	ensure     => present, <-- attributes, which continue until the closing {
	uid        => 500,
	gid        => 500,
	shell      => '/bin/sh',
	home       => '/Users/vjanelle',
	managehome => true,
}
</pre>

The building blogs of a puppet run are a collection of **resources**.  

A resource is an instance of a **resource type** and is identified by a unique type+title(ie, a "user" titled "vjanelle"), and a number of attributes which operate as parameters, each having a value.  Puppet has a number of core types, which are distributed with the runtime.  You can extend upon this collection with modules, providers, and types.  I'll be covering these more later.  

This example demonstrates a large portion of the syntax in the language, and you'll be seeing this a lot.  Hopefully the above example lays this out in a straightforward way.  It's also worth noting that this alone is enough for the puppet user core type to start managing the user "vjanelle" - but we'll be continuing into manifests after this.

#### Titles

A resource **title** is a unique name across a **catalog**, which is a compilation of all the resources that will be applied to a given system and the relationships between those resources.  In a class, it is simply the name of the class.  In a resource type, the title is the part after the first curly brace and before the colon.  In the example, this would be _'vjanelle'_.  In a defined resource type, this'll be present as _$title_ as a variable.

It's worth noting that there is type of variable, called a _namevar_.  This normally defaults to _$name_ which can be specified as a attribute.  Typically this will default to the contents of _$title_, but you can override this in your declaration.  This represents a resources __unique identity__ on the target system.  Typically this is used to abstract packages, files, on different operating systems - say a Debian package (ntp) vs the same functional piece on RHEL (ntpd) will have different names, but be configured in the same way.  

#### Attributes

A attribute is a parameter, and will be available in your class or defined type as the same name, preceding with a _$_.  For example, the __home__ attribute in the example will be available as _$home_ in the DSL.

### Working example

<pre>
USAGE
-----
puppet resource [-h|--help] [-d|--debug] [-v|--verbose] [-e|--edit]
  [-H|--host &lt;host&gt;] [-p|--param &lt;parameter&gt;] [-t|--types] &lt;type&gt;
  [&lt;name&gt;] [&lt;attribute&gt;=&lt;value&gt; …]
</pre>

<pre>
puppet resource user
</pre>

Open a terminal window and execute the above command.  Puppet will enumerate a list of users present on your system.  Most of these should look familiar, or provided by the OS for various reasons.

This is known as the resource shell, and lets you query and modify your system from the command line, provided that the resource type supports this.  


Example of querying a specific user:

<pre>
puppet resource user root

user { 'root':
  ensure   => 'present',
  comment  => 'System Administrator',
  gid      => '0',
  groups   => ['admin', 'certusers', 'daemon', 'kmem', 'operator', 'procmod', 'procview', 'staff', 'sys', 'tty', 'wheel'],
  home     => '/var/root',
  password => '*',
  shell    => '/bin/sh',
  uid      => '0',
}
</pre>

> The above command output contains valid puppet DSL output - You can save this to a file and execute it with `puppet apply` later.  
> One thing to note are the groups attribute - you can pass an array as an argument.  We'll be going into this more later.

To create a user:

<pre>
puppet resource user vjanelle ensure=present shell="/bin/bash" home="/home/bash" managehome=true

notice: /User[vjanelle]/ensure: created

user { 'vjanelle':
    ensure => 'present',
    home => '/home/vjanelle',
    shell => '/bin/bash'
}
</pre>

> The 'notice: /User[vjanelle]/ensure: created' line is emitted by puppet as a log entry - it is not valid puppet syntax.  This is for informational purposes only that puppet created the user and it is now present on your system.

## Language elements

### Variables

Variables in puppet are much like most languages - preceeded by a dollar sign ($var), and assigned with the equal sign ($var = "bar").  Any __normal__ data type (ie, anything that isn't a regex value) can be assigned to a variable.  

Without getting into this too much, variables can only be assigned by their "short name" - ie, in their current scope.  What this means is that you cannot assign variables outside of the scope of the class, type, or node, or global scope where the variable is being assigned.

#### Resolution

Variables can also by assigned in place where you'd normally enter normal data:
<pre>
$onesies = 'alpha'
$twosies = 'beta'

$allsies = [ $onesies, $twosies ] 
</pre>

> The above code contains an array with the content [ 'alpha', 'beta' ]

#### Interpolation

Like many other dynamic languages, Puppet can resolve variables in double quoted strings:
<pre>
$species = 'human'
$world = 'Earth'

notice("Hello ${species}, from planet ${world}")
</pre>
> This will output `notice: Hello human, from planet Earth` in the puppet notice level logging output when executed

This can be used as a poor version of string concatenation - a feature puppet is currently missing.  You may need this in the future to combine variables together.

### Variable Scope

> Link to image from http://docs.puppetlabs.com/puppet/latest/reference/lang_scope.html

A __scope__ is a specific __area of code__, isolated from other areas of your manifest.  Their purpose is to limit the reach variables and other types of default attribute data you can assign to resource types.  

> This is a topic that easily confuses people, so don't be worried if you don't get it right away - if you're getting tripped up on it, try to simplify the readability of your code by using attributes for lookups instead of referencing variables outside of your scope.

They _do not affect_ resource titles, which are global, and references to those resources (see dependancy chaining).

To retrieve the value of a variable outside of your scope, use the `::` seperator:

<pre>
$topvar = 'foo' # available everywhere as $::topvar

class example {
   $var = 'one' # available locally as $var, and as $example::var
}
class secondexample {
   notice($example::var) # from the 'example' class variable, named 'var'
   notice($::topvar) # from the global scope
}
node default {
   $nodevar = 'two' # Local to the node scope, not any deeper or higher.  
                    # You can only pass this variable as an attribute assignment.
}
</pre>

Puppet will search the scope trees if you do not specify the explicit scope - this will matter more when we get into `facts` - information which is provided by the agent and is accessible for you in the `top scope`. 

> This behaviour has changed between puppet 2.6 and 3.x - this is important information to research if you're updating older puppet infrastructure, or considering using Puppet Enterprise!  Make sure you consult http://docs.puppetlabs.com/guides/scope_and_puppet.html for more information.  We'll be focusing on the puppet 3.x for this.

### Conditional statements

Conditional statements are like in most other languages, and allow your code to behave differently various situations by means of tests.  When combined with __facts__ or data retrieved from an external source, they are instrumental in making reusable components that you can extend and use in multiple places and environments.  These typically work off of boolean logic, and are very similiar to constructs in other languages.

As a side note, a number of words in puppet are __reserved__ - meaning you can't:

* Use them as bare words - variable contents without quotes
* Use them as names for custom ruby functions
* Use them as names of classes, custom resource types, or defined resource types
* In previous versions of puppet, use them as names of attributes - this changed in 3.x

Most of the reserved words are used for conditional statements:

1. if
2. else
3. and
2. or
3. case
3. true
4. false
6. in
7. or
8. unless
9. undef

> `undef` is a 'special' value - it's only a reserved word by aggreement, and commonly used as a default variable that one tests for as a variable that needs to be forcibly assigned in a class or type.

Most conditionals contain: 
1. a keyword
2. a conditional test, typically a `boolean` (true, false)
3. a pair of curly braces ({ and }) which contain puppet code to be executed


#### `if` statements

This tests for a boolean true conditional, and executes the code block if the test evaluates to `true`.
<pre>
if $::fqdn == 'localhost' {
  notice("My host is ${::fqdn}") # Outputs "My host is localhost"
}
elsif $::fqdn == 'database' {
  notice("My host is the database server")
}
else {
  notice("We're somewhere strange")
}
</pre>

Along with `if`, there is `elsif` (acts an else/if test), and a pure `else` operator.

> $::fqdn is a top level variable that contains `fact` - a variable provided by the agent that is gathered from the system.

#### `unless` statements

The `unless` statement is basically the reverse of an if statement, and will only execute code if the test is `false`.  `unless` statement do not have any branching - they cannot be followed up with an `elsif` or `else` block.

<pre>
unless $::memorysize > 1024 {
  $maxclient = 500
}
</pre>

> In this example we're testing to see if the machine the agent is running on has more than 1024 MB of ram - a `fact` that calculates the memory size into a human usable number.  If the machine has less than 1GB, it'll set an arbritrary variable to 500.

#### `case` statement

Like `if` statements, `case` statements use a list of cases and code blocks and will execute the first block that matches the control expression as a boolean `true`.

<pre>
case $::operatingsystem {
  'Solaris':           { notice("We are running solaris") }
  'RedHat', 'CentOS':  { notice("We are running redhat, or centos") }
  /^(Debian|Ubuntu)$/: { notice("Through the power of regex, we are running ${::operatingsystem}") }
  default:             { exit("Unknown ${::operatingsystem") }
}
</pre>

> The `default` statement is a catch-all bare word - code that is executed if none of the other tests match.  As you can see in the previous example, you are not limited to normal data types in this test(strings, numbers), and you can apply ruby regular expressions as well (or even execute functions).

#### Selectors

Selector statements are similiar to `case` statements, but instead of executing a block of puppet code, they return a variable:

<pre>
$rootgroup = $::osfamily ? {
  'Solaris'          => 'wheel',
  /(Darwin|FreeBSD)/ => 'wheel',
  default            => 'root',
}
</pre>

> Various operating systems have the concept of a __group__ that allows for various super-user level privileges, or acts as a control gate for elevating privileges, or may not have a super-user group specific to the administrative user at all - Solaris, MacOS and FreeBSD being examples of this.
>  This code is determining based off a `fact` that provides a higher level overview of common operating system family traits and is available to you as a top level string.

These are useful for determining variable content based off of another content, and avoiding chained if/elsif statements which could get rapidly tedious.  For readability's sake, you should only do this when assigning variables, ie:

<pre>
$grp = $::osfamily ? {
  default => 'root',
}
user { 'vjanelle':
	group => $grp,
}
</pre>

You should really, **really** avoid doing something like this:
<pre>
user { 'vjanelle':
  group => $::osfamily ? { 
     'SomeOS' => 'foo',
     default => 'root',
  },
}
</pre>

Inlining the selectors in your resource is possible, but can rapidly produce unreadable code.  Many auto-formatting tools also can't handle this.

It can also lead to duplication of code, especially when you're creating many similiar resources.  

#### Regular expressions

`if`, `case`, and selectors all support regex as expression options in various fashions.  These can be extended even further to capture sections of the variable you're testing against, and are available inside your block:

<pre>
$tz = $timezone ? {
  /(PST|PDT)/ => "America/Vancouver is currently $1",
  default => "Somewhere else"
}
if $::hostname =~ /www(\d\+)\./ {
  notice("We are on webserver # $1")
}
</pre>

These are not normal variables, and have unique behaviours:   Their variable scope is is not persisted outside the content being tested (outside of the `if` block, etc).

> [Rubular regex](http://rubular.com/) is an awesome resource for testing ruby regexes if you're not familiar with their syntax.


