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
