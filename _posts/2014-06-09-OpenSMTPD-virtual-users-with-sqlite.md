---
layout: default
title: OpenSMTPD virtual users using SQLite
tags: [sqlite,OpenSMTPD,virtual users,how-to]
---

# {{ page.title }}

So boys and girls I managed to get virtual users with username@domain.tld to work with OpenSMTPD, and let me explain it off to you all.  

<!--more-->

## Prereqs:
  
1. This assumes you already have virtual users setup for your selected LDA/MDA/IMAP/etc server and it understands how to deal with a blowfish password since OpenSMTPD is really just acting as an MTA.  
2. SQLite3, I may do an LDAP and postgresql one down the line but for this I made an executive call and used SQLite since that backend was the most stable at the time I started this oddessy.  
3. A Working knowledge of computers.
  
So there are a few basic parts to getting this working, the first part that is beyond the scope of this is getting dovecot to accept your virtual users using SQLite and do user@domain.tld for auth. The second part of it is setting up the database, below is the schema I have for mine, It needs to be tuned a lot but this works.

## SQLite Database Schema

```sql
CREATE TABLE users (username VARCHAR(128) NOT NULL, 
domain VARCHAR(128) NOT NULL, 
home VARCHAR(256) NOT NULL,
password VARCHAR(64) NOT NULL, 
uid INTEGER NOT NULL, gid INTEGER, 
active CHAR(1) DEFAULT 'Y' NOT NULL);

CREATE TABLE alias (user VARCHAR NOT NULL, 
alias VARCHAR NOT NULL);  
CREATE TABLE domains (domain VARCHAR(256) NOT NULL, 
active CHAR(1) DEFAULT 'Y' NOT NULL);  
```
  
Each of the tables handle different things,  

* users: All users, passwords, and user info are stored here.  
* alias: Just like it sounds its a replacement for the alias file anything in user will be resolved to alias  
* domains: a list of domains that are accepted for delivery  
  
So the other moving part is telling OpenSMTPD how to create the tables which is done in a simple flat configuration file.


## SQLiteTables.conf
```config
dbpath            /etc/mail/authdb.sqlite;
query_alias       select alias from alias where user=?;
query_domain      select domain from domains where domain=? and active="Y";
query_userinfo    select uid,gid,home from users where username=? and active="Y";
query_credentials select username, password, from users where (username||'@'||domain)=? and active="Y";
```

So lets take it line by line and explain it:  
  
`dbpath                  /etc/mail/authdb.sqlite`  
This line tells you simply where the sqlite database is going to be.  

`query_alias             select alias from alias where user=?;`  
This will respond when SMTPD does look up call for ALIAS. It will run the SQL command replacing ? with the left hand side of the SMTP address. so if you look up foo@bar.com and it is aliased to bar@bar.com the query will be select alias from where user='foo'; and return bar@bar.com. it expects 1 field to be returned as a varchar.
  
`query_domain            select domain from domains where domain=? and active="Y";`  
This will be run when SMTPD does a lookup call for DOMAIN. It will replace the ? with the right hand side of the SMTP address, so if you have foo@bar.com it will become select domain from domains where domain='bar.com' and active="Y"; and if both conditions are met in the query it will return bar.com. This expects 1 field to be returned as a varchar. Now you will notice I have a static value at the end of the query active="Y", That is completely optional, what it allows me to do is disable a domain without removing it from the table. In theory with that and a comments column I could keep track of why a domain was removed from the default domains table.

`query_userinfo          select uid,gid,home from users where username=? and active="Y";`
 This will be run when SMTPD does a lookup call for USERINFO. It will replace the ? with the left hand side of the SMTP address to produce a valid user, if it returns 0 or -1 the look up will fail. So the query becomes select uid,gid,home from users where username=foo and active="Y"; and it expects 3 fields to be returned the first is an int which will be used as a UID, the second is an int to be used as a gid, and the third is a varchar to be used as the path to the users mail home. This also has the active="Y" for the same reason as domain and it is completely optional.

`query_credentials       select username, password from users where (username||'@'||domain)=? and active="Y";`  
This will be run when SMTPD does a lookup call for CREDENTIALS. This replaces ? with whatever is given as the username. This does have 2 required fields a varchar for username and a varchar for password. But this is where some of the magic comes in, if you have a virtual setup that requires users login as user@domain.com you will need to concatenate 2 of the columns together to get the log on name as user@domain.com, `(username||'@'||domain)=?` will do this.
  
The last moving part is the SMTPD.conf  
  
## SMTPD.CONF
  
so near the top of the config define all your table declarations or at least the auth one, but I like to keep them together. The defs would look like:  

    table domains sqlite:/etc/mail/sqlite  
    table users sqlite:/etc/mail/sqlite  
    table sqalias sqlite:/etc/mail/sqlite  
   
the next part to deal with is getting auth with your selected port, I require auth on submission and I also try to hide the clients IP address, it looks somthing like:
  
`listen on all port submission tls auth pki mail.serversave.us mask-source`
  
then you have to tell your server to accept mail for the virtual user base:
  
`accept from any for domain alias userbase deliver to mbox`
  
and with all that in place you should be authing and delivering to virtual users.
