OpenWRT Xtables Firewall
========================

The default firewall configuration as shipped with OpenWRT is just
horrible.  It is a twisty maze of custom chains which are mostly empty
and thus serve no purpose.  The only purpose of it is to be configured
by the GUI.

The proper way to load ip[6]tables rules is via iptables-restore and
ip6tables-restore.  That's what these init scripts do.

Installation
------------

Shell access to your OpenWRT router is assumed.  Deactivate the
default OpenWRT firewall.

```
/etc/init.d/firewall disable
```

Copy each file to /etc/init.d on the OpenWRT router.

```
scp ipset openwrt:/etc/init.d
scp iptables openwrt:/etc/init.d
scp ip6tables openwrt:/etc/init.d
```

Activate the new initscripts.

```
/etc/init.d/ipset enable
/etc/init.d/iptables enable
/etc/init.d/ip6tables enable
```

Note that you may need to use opkg to install ipset before you can
use that service.

Configuration
-------------

To configure each of these services, just create new rules files in
/etc using iptables-save/restore (and ipset -S/-R) syntax.  Each
service reads a file named /etc/$SERVICE.save.

```
ipset     => /etc/ipset.save
iptables  => /etc/iptables.save
ip6tables => /etc/ip6tables.save
```

One way you can build new configurations is to use iptables and ipset
to construct your rules one line at a time, then use iptables-save and
ipset -S to write out your config files.  However, it is recommended
that you write a clean ruleset for each and upload them to the
appropriate file location.  Here are some examples you can use to help
get you started.  These examples assume eth1 is your public interface
and br-lan is your LAN interface.

/etc/ipset.save
```
-N Bogons hash:net -!
-A Bogons 0.0.0.0/8
-A Bogons 10.0.0.0/8
-A Bogons 100.64.0.0/10
-A Bogons 127.0.0.0/8
-A Bogons 169.254.0.0/16
-A Bogons 172.16.0.0/12
-A Bogons 192.0.0.0/29
-A Bogons 192.0.2.0/24
-A Bogons 192.168.0.0/16
-A Bogons 198.18.0.0/15
-A Bogons 198.51.100.0/24
-A Bogons 203.0.113.0/24
-A Bogons 224.0.0.0/3
```

/etc/iptables.save
```
*nat
-A POSTROUTING -o eth1 -j MASQUERADE
COMMIT

*filter
:INPUT DROP
:FORWARD DROP
:OUTPUT ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i br-lan -s 192.168.1.0/24 -j ACCEPT
-A INPUT -i eth1 -m set --match-set Bogons src -j DROP
-A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i br-lan -o eth1 -j ACCEPT
COMMIT
```

/etc/ip6tables.save
```
*filter
:INPUT DROP
:FORWARD DROP
:OUTPUT ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A INPUT -i br-lan -s fe80::/10 -j ACCEPT
-A INPUT -p icmpv6 -j ACCEPT
-A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i br-lan -o eth1 -j ACCEPT
-A FORWARD -p icmpv6 -j ACCEPT
COMMIT
```

Details
-------

The more generalized distros like Debian and Fedora ship with scripts
for automatic setup of firewall rules by calling iptables-restore on a
stored rules file.  OpenWRT's firewall script, however, is based on
configuring the firewall with the web interface.  This leads to a
difficult-to-understand ruleset and overall functionality which is
limited to whatever is provided by the web interface.  There is a
facility to add a customization script, but it's exactly that, a
script.

These services restore your ability to treat the firewall on your
OpenWRT router just like it is on any other distro.  There are no
default rules included with these services, you must write your own.
Examples are provided in the previous section of this document.

Using scripts to insert your ruleset is discouraged because it does
not provide an atomic function to insert rules.  What does this mean?
iptables-restore works in one single atomic operation.  Any syntax
error in one line causes the entire operation to fail.  Compare this
to using scripts to insert one rule at a time.  A syntax error in one
rule could leave you missing a critical piece, possibly locking you
out of your router or leaving it completely open due to a misplaced
ALLOW statement.  Furthermore, every time you call iptables to insert
a rule, the kernel actually copies the entire ruleset, makes a one
line modification, and reloads the whole set.  This is what
iptables-restore does but only once, so it's better to get the whole
ruleset in one sweep.

Additionally, using scripts encourages you to use shell tricks like
loops to add similar rules where you would be better off using ipsets
or multiport matches.
