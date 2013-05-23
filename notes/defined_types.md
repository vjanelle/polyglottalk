# Defined Types

Defined resource types are blocks of puppet code that can be instantiated multiple times with different parameters.  Once evaluated, they act like a new resource type - you can execute the block of puppet code by declaring a resource of your own type.

Think of these as a lightweight way of developing sophisticated descriptive resources that contain other resources.

## Defining a type

From the puppet documentation:

<pre>
define apache::vhost (
  $port,
  $docroot,
  $servername = $title,
  $vhost_name = '*',
) {
  include apache # contains Package['httpd'] and Service['httpd']
  include apache::params # contains common config settings
  
  $vhost_dir = $apache::params::vhost_dir
  
  file { "${vhost_dir}/${servername}.conf":
     content => template('apache/vhost-default.conf.erb'), 
     # This template can access all of the parameters and variables from above.
     owner   => 'www',
     group   => 'www',
     mode    => '644',
     require => Package['httpd'],
     notify  => Service['httpd'],
  }
}
</pre>

This will create a new resource type called `apache::vhost`.

The general form of a type definition is:

* The `define` keyword
* The name of the defined type (`apache::vhost`)
* An optional set of parameters, which consists of:
  * An opening parenthesis
  * A comma-separated list of parameters, each of which consists of:
    * A new variable name, including the $ prefix
    * An optional equals sign and default value (any data type)
  * An optional trailing comma after the last parameter
  * A closing parenthesis
* An opening curly brace
* A block of arbitrary Puppet code, which generally contains at least one resource declaration
* A closing curly brace

###  A word about style 

Puppet's pretty flexible about what you feed it - bare words, numbers, etc.  It will accept many forms of 'code compression', ie removing redundant statements, selectors in type/class declarations etc… For now, please avoid them - try to keep your manifests using local variables as possible, and don't worry about line counts.

The above example shows parameters declared one per line - this is in general to aid you with SCM systems.  If you add, change, or remove a parameter this will show up clearly in a diff.  The `=>` variable assignment operator is generally all lined up together in a resource for clarity.  Parameters without default values are first in the list, and ones with default values are declare in the latter half.  These standards are entirely optional and up to you to utilize and puppet will not care if you use them or not.

The [puppet style guide](http://docs.puppetlabs.com/guides/style_guide.html) goes over in more detail what puppet labs suggests as good practices for you and your organization.  The [puppet-lint](http://puppet-lint.com/) tool will give you more information and is useful as a warning hook with your SCM or your editor may have integration with it.

## Declaring an instance

Instances of a defined type (often just called **resources**) can be declared the same a normal resource is declared - ie, with a type, title, and a set of attribute/value pairs.

The **parameters** used when defining the type become **attributes** (without the `$` prefix) used when declaring resources of that type, and will be present as variables in that type, any resources underneath it, templates, etc…

To declare a resource of the `apache::vhost` type from the example above:

<pre>
apache::vhost { 'homepages': # 'homepages' becomes $title
  port    => 8081, # becomes $port
  docroot => '/var/www-testhost', # becomes $docroot
}
</pre>

### $title and $name

Every defined type gets two default parameters that will always be present, and do not have to be explicitly defined in the instantiation:

* $title is always set to the `title` of the instance, which is unique across the entire catalog.  It is useful for making sure contained resources are unique.  We'll be going into this shortly.
* $name defaults to the value of $title, but users can optionally change this for a different value.  This is useful for things like `package` resources, where you may want a title of **apache2**, but the name is different (*apache2* on Debian, *httpd* on RHEL).  If you are wondering which value to use, use $title.

The values of `$name` and `$title` are also available **inside the the parameter list**.  This means you can use $title as the default value for another attribute:

<pre>
define apache::vhost ($port, $docroot, $servername = $title, $vhost_name = '*') { ...
</pre>


### Resource uniqueness

If you define multiple instances of a defined type in your manifest, you will come across an issue where the resources inside your type conflict because of duplicate titles in the catalog - a **duplicate resource declaration** error.

In the above example:

<pre>
file { "${vhost_dir}/${servername}.conf":
… 
}
</pre>

What this is doing is combining the values of `$vhost_dir` and `$servername` as parts of the path to create the configuration file.  If you had a generic `vhost.conf` file that contained your virtual hosts, you'd have a conflict very quickly when you tried to add a second config.

> Sometimes having a single file that multiple resources needs to address is unavoidable.  The [ripienaar-concat](http://forge.puppetlabs.com/ripienaar/concat) module is invaluable for building configuration files out of fragments in a `stage` - a group of isolated puppet manifests that run without context (other than uniqueness in the graph) from other stages.

