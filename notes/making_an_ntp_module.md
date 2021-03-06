# Making an NTP Module

## Autoloader

Puppet uses an autoloader concept to reduce the amount of code it needs to parse and load at compilation time.  It loads manifests and other files on demand as it compiles the catalog, searching for file locations from the `modulepath` 

<pre>
puppet master --configprint modulepath
/etc/puppet/modules
</pre>

## Directory layout

* /etc/puppet/modules
  * ntp/
    * manifests/init.pp # contains `class ntp ( … ) { … }`
    * templates/ntp.conf.el
    * templates/ntp.conf.debian
    * files/
    
## Basic init.pp contents

<pre>
    class ntp (
      $ntpservers = [ 
      '0.centos.pool.ntp.org',
      '1.centos.pool.ntp.org',
      '1.centos.pool.ntp.org',
       ],    
    ){
    
      $conf_file = '/etc/ntp.conf'
      
      case $operatingsystem {
        centos, redhat: { 
          $service_name = 'ntpd'
        }
        debian, ubuntu: { 
          $service_name = 'ntp'
        }
      }
      
      package { 'ntp':
        ensure => installed,
      }
      
      service { 'ntp':
        name      => $service_name,
        ensure    => running,
        enable    => true,
        subscribe => File['ntp.conf'],
      }
      
      file { 'ntp.conf':
        path    => '/etc/ntp.conf',
        ensure  => file,
        require => Package['ntp'],
        source  => template($conf_file),
      }
    }
</pre>

## ntp.conf contents

<pre>
# ntp.conf(5), ntp_acc(5), ntp_auth(5), ntp_clock(5), ntp_misc(5), ntp_mon(5).

driftfile /var/lib/ntp/drift

# Permit time synchronization with our time source, but do not
# permit the source to query or modify the service on this system.
restrict default kod nomodify notrap nopeer noquery
restrict -6 default kod nomodify notrap nopeer noquery

# Permit all access over the loopback interface.  This could
# be tightened as well, but to do so would effect some of
# the administrative functions.
restrict 127.0.0.1 
restrict -6 ::1

# Hosts on local network are less restricted.
#restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap

# Use public servers from the pool.ntp.org project.
# Please consider joining the pool (http://www.pool.ntp.org/join.html).
<% ntp.servers.each do | ntpserver | %>
server <%= ntpserver %>
<% end %>

#broadcast 192.168.1.255 autokey	# broadcast server
#broadcastclient			# broadcast client
#broadcast 224.0.1.1 autokey		# multicast server
#multicastclient 224.0.1.1		# multicast client
#manycastserver 239.255.254.254		# manycast server
#manycastclient 239.255.254.254 autokey # manycast client

# Undisciplined Local Clock. This is a fake driver intended for backup
# and when no outside source of synchronized time is available. 
#server	127.127.1.0	# local clock
#fudge	127.127.1.0 stratum 10	

# Enable public key cryptography.
#crypto

includefile /etc/ntp/crypto/pw

# Key file containing the keys and key identifiers used when operating
# with symmetric key cryptography. 
keys /etc/ntp/keys

# Specify the key identifiers which are trusted.
#trustedkey 4 8 42

# Specify the key identifier to use with the ntpdc utility.
#requestkey 8

# Specify the key identifier to use with the ntpq utility.
#controlkey 8

# Enable writing of statistics records.
#statistics clockstats cryptostats loopstats peerstats
</pre>
