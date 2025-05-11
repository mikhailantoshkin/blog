+++
title="Local dev environment with LDAP server for pain and suffering"
date=2025-05-11
draft=false

[extra]
disable_comments = true
+++
At my previous workplace, we had a very specific DX problem to solve: we had a pool of shared VMs
used for testing. When you wanted to test out your PR you could grab a VM from the pool and
deploy your changes there. At some point, after a few QA engineers overwrote each other's VMs a few times,
we decided to write something that would allow you to "lease" the VM.
The first iteration of this was a simple Telegram bot. User experience was a bit lacking, but it got the job done.

But one day, frustrated by the clunkiness of the solution, my colleague and I decided that the time
for a makeover had come. After a brainstorming session, we ended up with a short list of what we wanted from this system.
The three main points for it were: 
- It needs to be a website (no more clunky messaging app interfaces)
- It needs to integrate with the company AD setup for auth
- It needs to be written in Rust

Easy enough, we already have experience with all of these, so this should not be a problem.
After a few evenings of hacking, the clunky Telegram bot was reborn as [tachikoma](https://github.com/udv-group/tachikoma).

More and more people started to use it, and after a while, we got a request to add the ability to assign VMs to a specific team.
Our junky at the time auth solution didn't cut it anymore - we now had to grab information about the groups the users are a part of. 

At the time dev setup for LDAP was borrowed from [ldap3 crate examples](https://github.com/inejge/ldap3/tree/00a513ece4ffa9a9782860c285f4c4c12bc07552/data).
With how our auth was set up, you could log in with empty credentials, very nice for dev environment.
But now we had to query the user from the LDAP server, and this example setup would not cut it anymore. 
As any reasonable dev would do, we just ran all requests against a real AD server on the company infra.
This meant that you had to have a VPN enabled when you were doing local development, but such is life.

Fast forward half a year or so, and I have now changed companies (and country of residency) and don't have
easy access to the AD server anymore. Now I had no choice but to add a test LDAP server setup to the project.

# LDAP

At first, I thought I could just adjust the `ldap` crate setup and I'd be golden. Not having worked with
LDAP servers before, I got very quickly overwhelmed. It was definitely way too complex for what I needed,
so, a few stackoverflow posts later, I was reading Arch docs of all places to see how to set up an `slapd`
from scratch. 

It netted good progress: I had an initial config that `slapd` would accept and start.

`config.ldif`
```toml,name=config.ldif
# The root config entry
dn: cn=config
objectClass: olcGlobal
cn: config
olcArgsFile: .ldap/run/slapd.args
olcPidFile: .ldap/run/slapd.pid

# Schemas
dn: cn=schema,cn=config
objectClass: olcSchemaConfig
cn: schema

include: file:///etc/openldap/schema/core.ldif

# The config database
dn: olcDatabase=config,cn=config
objectClass: olcDatabaseConfig
olcDatabase: config
olcRootDN: cn=Manager,dc=example,dc=org

# The database for our entries
dn: olcDatabase=mdb,cn=config
objectClass: olcDatabaseConfig
objectClass: olcMdbConfig
olcDatabase: mdb
olcSuffix: dc=example,dc=org
olcRootDN: cn=Manager,dc=example,dc=org
olcRootPW: test
olcDbDirectory: .ldap/db
olcDbIndex: objectClass eq
olcDbIndex: uid pres,eq
olcDbIndex: mail pres,sub,eq
olcDbIndex: cn,sn pres,sub,eq
olcDbIndex: dc eq

# Additional schemas
# RFC1274: Cosine and Internet X.500 schema
include: file:///etc/openldap/schema/cosine.ldif
# RFC2307: An Approach for Using LDAP as a Network Information Service
# Check RFC2307bis for nested groups and an auxiliary posixGroup objectClass (way easier)
include: file:///etc/openldap/schema/nis.ldif
# RFC2798: Internet Organizational Person
include: file:///etc/openldap/schema/inetorgperson.ldif
```

A few notes for me from the past: 
- `olcArgsFile` and `olcPidFile` need to point to an existing directory. 
	This looks like it was designed with `systemd` in mind, so if you are using it - you should point it 
	to `unit`s `run` directory.
- `olcDbDirectory` is a path to the directory where the main database is going to be stored, so be mindful about access rights.
- Take note of what value you used for `olcDatabase` for entry with `objectClass: olcMdbConfig`. 
	This will come into play with later configuration.
- Don't forget to set up `olcRootDN` and `olcRootPW`, this is your credential for admin access

Before applying this configuration, we need to create a base directory structure. In my case,
it's 

```bash
mkdir -p .ldap/db .ldap/config .ldap/run
```

Now to apply this configuration, run 

```bash
slapadd -n 0 -F .ldap/config -l ldap/config.ldif
```

where `-F` is a directory where server configuration is going to be stored and `-l` 
is a path to the configuration file.

The next step is adding some entries, which is also very straightforward: 

`config.ldif`
```toml,name=init.ldif
# example.org
dn: dc=example,dc=org
dc: example
o: Example Organization
objectClass: dcObject
objectClass: organization

# Manager, example.org
dn: cn=Manager,dc=example,dc=org
cn: Manager
description: LDAP administrator
objectClass: organizationalRole
objectClass: top
roleOccupant: dc=example,dc=org

# People, example.org
dn: ou=People,dc=example,dc=org
ou: People
objectClass: top
objectClass: organizationalUnit

# Groups, example.org
dn: ou=Group,dc=example,dc=org
ou: Group
objectClass: top
objectClass: organizationalUnit
```

To apply this, you first would need to start up the `slapd` daemon:

```
slapd -h ldapi://ldapi -F .ldap/config
```

After a second, you can add the base entries with 

```bash
ldapadd -x -D 'cn=Manager,dc=example,dc=org' -w test -H ldapi://ldapi -f ldap/init.ldif
```

Note the `-D` and `-w` arguments - this is your credentials you've set up when configuring the server.

Now we are ready to add a user and a group! This config defines a user and a group he is a part of. 

`user.ldif`
```toml,name=user.ldif
dn: uid=johndoe,ou=People,dc=example,dc=org
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: johndoe
cn: John Doe
sn: Doe
givenName: John
title: Guinea Pig
telephoneNumber: +0 000 000 0000
mobile: +0 000 000 0000
postalAddress: AddressLine1$AddressLine2$AddressLine3
userPassword: test
labeledURI: https://archlinux.org/
loginShell: /bin/bash
uidNumber: 9999
gidNumber: 9999
homeDirectory: /home/johndoe/
mail: jdoe@example.org
description: This is an example user

dn: cn=joe,ou=Group,dc=example,dc=org
objectClass: groupOfNames
objectClass: top
cn: joe
member: uid=johndoe,ou=People,dc=example,dc=org
```

To apply it, run 

```bash
ldapadd -x -D 'cn=Manager,dc=example,dc=org' -w test -H ldapi://ldapi -f ldap/user.ldif
```

At this point I'm feeling pretty good: I have a working LDAP server with a user and a group. 
Time to test the setup! Oh... User is missing `memberOf` property? 

# memberOf
Remember that we needed to grab the information about the groups the user is a member of?
This is done with `memberOf` attribute on the user. But when I query a user in does not have
such an attribute, though he is definitely a member of a group, so what is going on? 

As it turns out, there is a so-called "overlay" that back-populates users with `memberOf`
attributes. And we have to enable it somehow. 

Either Google is now useless, or there are no sane examples on how to do it (I suspect both),
but after a few hours of delirium, I ended up with these three config files. Don't ask me what
they do, I won't be able to answer that, I just know that they work and won't steal your crypto (probably).

`overlay.ldif`
```toml,name=overlay.ldif
# enable memberOf overlay, so infomation about user groups
# can be queried from the user
dn: cn=module,cn=config
cn: module
objectClass: olcModuleList
olcModuleLoad: memberof

dn: olcOverlay={0}memberof,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member
olcMemberOfMemberOfAD: memberOf
```

`refint.ldif`
```toml,name=refint.ldif
dn: cn=module{0},cn=config
changetype: modify
add: olcmoduleload
olcmoduleload: refint
```

`refint2.ldif`
```toml,name=refint2.ldif
dn: olcOverlay={1}refint,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
objectClass: top
olcOverlay: {1}refint
olcRefintAttribute: memberof member manager owner
```

If you apply these configs _before_ you add the user - you would have a `memberOf`
attribute for a user, that is a part of a group with `objectClass: groupOfNames`.

So the script to set it all up in one go looks like this:

```bash
#!/usr/bin/env bash

mkdir -p .ldap/db .ldap/config .ldap/run
slapadd -n 0 -F .ldap/config -l ldap/config.ldif
slapd -h ldapi://ldapi -F .ldap/config
sleep 1
ldapadd -x -D 'cn=Manager,dc=example,dc=org' -w test -H ldapi://ldapi -f ldap/init.ldif
ldapadd -x -D 'cn=Manager,dc=example,dc=org' -w test -H ldapi://ldapi -f ldap/overlay.ldif
ldapadd -x -D 'cn=Manager,dc=example,dc=org' -w test -H ldapi://ldapi -f ldap/refint.ldif
ldapadd -x -D 'cn=Manager,dc=example,dc=org' -w test -H ldapi://ldapi -f ldap/refint2.ldif
ldapadd -x -D 'cn=Manager,dc=example,dc=org' -w test -H ldapi://ldapi -f ldap/user.ldif
```

# Morale of the story

There is none. 
LDAP is a spawn of satan. 
For how ubiquitous it is - it's baffling how hard it is to find a good example configuration. 
Half of the time, I was just hitting digital rocks together. 

Props to Arch docs, it again proves to be the GOAT.
