---
title: "Opensmtpd_postgresql"
date: 2023-04-16T14:58:09+09:00
draft: false
---
# OpenSMTPD and Dovecot with a shared PostgreSQL, Sieve and RSpamd on OpenBSD 6.6
Apr 17, 2020

I finally got around to setting up a new mailserver and i decided to give OpenSMTPD a try. It wasn’t a natural birth, i can tell you that. The switching of the configuration syntax makes for a lot of outdated Google Search results.

So what are we going to setup. Well the title gave it away i guess, so for the slow ones amongst you: we are building a Mailserver with OpenSMTPD, Dovecot, RSpamd and Sieve. The OpenSMTPD and the Dovecot will both be using the same authentication table and hashing scheme, making this a nifty solution.

## Installing the required components
~~~
pkg_add postgresql-server opensmtpd-extras opensmtpd-extras-pgsql opensmtpd-filter-rspamd opensmtpd-filter-senderscore rspamd dovecot dovecot-pigeonhole dovecot-postgresql redis
~~~
## Enabling them on boot
~~~
rcctl enable httpd
rcctl enable smtpd
rcctl enable postgresql
rcctl enable rspamd
rcctl enable dovecot
rcctl start dovecot
rcctl enable redis
rcctl start redis
~~~
## Setting up DNS
This has been explained in numerous posts on the Internet, you should by now know how to setup an MX Record (maybe SPF and DKIM).

## Setting up Let’s Encrypt SSL Certificates
/etc/httpd.conf
Configure httpd to do the acme challenges.
~~~
server "replace.with.host.name" {
listen on * port 80
location "/.well-known/acme-challenge/*" {
root "/acme"
request strip 2
}
location "/" {
block return 301 "https://$SERVER_NAME$REQUEST_URI"
~~~

And then start httpd:
~~~
rcctl start httpd
/etc/acme-client.conf
Now we go on to configure the acme-client.

api_url="https://acme-v02.api.letsencrypt.org/directory"
authority letsencrypt {
        api url $api_url
                account key "/etc/acme/letsencrypt-privkey.pem"
                }

domain replace.with.host.name {
        #alternative names { www.replace.with.host.name }
                domain key "/etc/ssl/private/replace.with.host.name.key"
                        #domain certificate "/etc/ssl/replace.with.host.name.crt"
                                domain full chain certificate "/etc/ssl/replace.with.host.name.crt"
                                        sign with letsencrypt
}
~~~
## Obtaining a certificate
~~~
acme-client -v replace.with.host.name
~~~
## Adding certificate renewal to cron

Enter the crontab with crontab -e and add the following line:
~~~
30      0       *       *       *       /usr/sbin/acme-client replace.with.host.name && /usr/sbin/rcctl restart smtpd && /usr/sbin/rcctl restart dovecot
~~~
## Preparations for our services
/etc/login.conf
Go ahead and add the following lines at the end of your /etc/login.conf:
~~~
dovecot:\
:openfiles-cur=1024:\
:openfiles-max=4096:\
:tc=daemon:

postgresql:\
:openfiles=768:\
:tc=daemon:
~~~
Once done, have the file cap_mkdb’d like this:
~~~
cap_mkdb /etc/login.conf
~~~
## /etc/sysctl.conf
Append the following values to /etc/sysctl.conf so PostgreSQL has a bit of breathing room:
~~~
kern.seminfo.semmni=60
kern.seminfo.semmns=1024
~~~
Then go on to actually setting them in the kernel:
~~~
sysctl -w kern.seminfo.semmni=60 kern.seminfo.semmns=1024
~~~
## Adding a vmail user and group
~~~
useradd -m -d /var/vmail -s /sbin/nologin vmail
~~~
## Preparing PostgreSQL
~~~
su - _postgresql
mkdir /var/postgresql/data
initdb -D /var/postgresql/data -U postgres -A scram-sha-256 -E UTF8 -W
exit
rcctl start postgresql
~~~
Next we are going to add a user, a database, two tables and three views:
~~~
psql -Upostgres <<EOF
CREATE USER mail WITH ENCRYPTED PASSWORD 'your.mail.password';
CREATE DATABASE mail OWNER mail;
EOF

psql -Umail mail <<EOF

-- this is the table for the users accounts
CREATE TABLE public.accounts (
id serial,
email character varying(255) DEFAULT ''::character varying NOT NULL,
password character varying(255) DEFAULT ''::character varying NOT NULL,
active boolean DEFAULT true NOT NULL
);

-- this is the table for the virtual mappings for email -> email
CREATE TABLE public.virtuals (
id serial,
email character varying(255) DEFAULT ''::character varying NOT NULL,
destination character varying(255) DEFAULT ''::character varying NOT NULL
);

-- this view is used to determine where to deliver things
CREATE VIEW public.delivery AS
SELECT virtuals.email,
virtuals.destination
FROM public.virtuals
WHERE (length((virtuals.email)::text) > 0)
UNION
SELECT accounts.email,
'vmail'::character varying AS destination
FROM public.accounts
WHERE (length((accounts.email)::text) > 0);

-- this view is used to determine which domains this server is serving
CREATE VIEW public.domains AS
SELECT split_part((virtuals.email)::text, '@'::text, 2) AS domain
FROM public.virtuals
WHERE (length((virtuals.email)::text) > 0)
GROUP BY (split_part((virtuals.email)::text, '@'::text, 2))
UNION
SELECT split_part((accounts.email)::text, '@'::text, 2) AS domain
FROM public.accounts
WHERE (length((accounts.email)::text) > 0)
GROUP BY (split_part((accounts.email)::text, '@'::text, 2));

-- this view should control the email addresses users can send with
CREATE VIEW public.sending AS
SELECT virtuals.email,
virtuals.destination AS login
FROM public.virtuals
WHERE (length((virtuals.email)::text) > 0)
UNION
SELECT accounts.email,
accounts.email AS login
FROM public.accounts
WHERE (length((accounts.email)::text) > 0);
EOF
~~~
## /etc/mail/postgres.conf
Next we configure the PostgreSQL lookups for smtpd:
~~~
conninfo host='localhost' user='mail' password='your.mail.password' dbname='mail'
query_alias SELECT "destination" FROM delivery WHERE "email"=$1;
query_credentials SELECT "email", "password" FROM accounts WHERE "email"=$1;
query_domain SELECT "domain" FROM domains WHERE "domain"=$1;
query_mailaddrmap SELECT "email" FROM sending WHERE "login"=$1;
~~~
Also since this file contains the password to the database, only _smtp should be able to read it:
~~~
chown _smtpd:_smtpd /etc/mail/postgres.conf
chmod o= /etc/mail/postgres.conf
~~~
## /etc/mail/smtpd.conf
Now we can go ahead and configure OpenSMTPD:
~~~
table aliases file:/etc/mail/aliases
table auths postgres:/etc/mail/postgres.conf
table domains postgres:/etc/mail/postgres.conf
table virtuals postgres:/etc/mail/postgres.conf
table sendermap postgres:/etc/mail/postgres.conf


pki replace.with.host.name cert "/etc/ssl/replace.with.host.name.crt"
pki replace.with.host.name key "/etc/ssl/private/replace.with.host.name.key"


filter check_dyndns phase connect match rdns regex { '.*\.dyn\..*', '.*\.dsl\..*' } \
disconnect "550 no residential connections"

filter check_rdns phase connect match !rdns \
disconnect "550 no rDNS is so 80s"

filter check_fcrdns phase connect match !fcrdns \
disconnect "550 no FCrDNS is so 80s"

filter senderscore \
proc-exec "filter-senderscore -blockBelow 10 -junkBelow 70 -slowFactor 5000"

filter rspamd proc-exec "filter-rspamd"


listen on all tls pki replace.with.host.name filter { check_dyndns, check_rdns, check_fcrdns, senderscore, rspamd }
#listen on all port smtps smtps pki replace.with.host.name auth <auths> senders <sendermap> masquerade
#listen on all port submission tls-require pki replace.with.host.name auth <auths> senders <sendermap> masquerade
listen on all port smtps smtps pki replace.with.host.name auth <auths>
listen on all port submission tls-require pki replace.with.host.name auth <auths>

action "receive_aliases"        lmtp "/var/dovecot/lmtp" rcpt-to alias <aliases>
match from local for local action "receive_aliases"

action "receive_vmail"          lmtp "/var/dovecot/lmtp" rcpt-to virtual <virtuals>
match from any for domain <domains> action "receive_vmail"

action "outbound"               relay helo replace.with.host.name
match from auth for any action "outbound"
~~~
And finally start the smtpd:
~~~
rcctl start smtpd
~~~
## /etc/rspamd/worker-proxy.inc
In this file i actually just changed the spam_header to X-Spam-Status, but this optional.

## /etc/rspamd/local.d/greylist.conf
~~~
greylist {
servers = "127.0.0.1:6379";
timeout = 1min;
}
~~~
Then we start rspamd:
~~~
rcctl start rspamd
~~~
## Diff style below here
I’ve chosen to only put in things you need to change or append, everything else should remain as is.

Why did i do this? Well since dovecot has evolved into this nice configuration-file layout, i decided that this is the most efficient way to keep this document clean and relevant.

## /etc/dovecot/conf.d/10-auth.conf
Towards the beginning of the file, disable plaintext authentication:
~~~
disable_plaintext_auth = yes
~~~
Then at the end of the file, there are several includes. We are going to comment the auth-system.conf.ext and are going to comment in the auth-sql.conf.ext instead:
~~~
#!include auth-system.conf.ext
!include auth-sql.conf.ext
~~~
## /etc/dovecot/conf.d/10-mail.conf
We change the mailbox location to our vmail directory.
~~~
mail_location = maildir:/var/vmail/%u
~~~
## /etc/dovecot/conf.d/10-master.conf
Next we are going to comment in our SSL listeners, feel free to leave in 143 and 110 as they are using STARTTLS:
~~~
service imap-login {
inet_listener imap {
port = 143
}
inet_listener imaps {
port = 993
ssl = yes
}
}

service pop3-login {
inet_listener pop3 {
port = 110
}
inet_listener pop3s {
port = 995
ssl = yes
}
}
~~~
Then make sure lmtp is configured with the correct permissions:
~~~
service lmtp {
unix_listener lmtp {
mode = 0660
user = vmail
group = vmail
}
}
~~~
## /etc/dovecot/conf.d/10-ssl.conf
Now we are going to configure SSL:
~~~
ssl = required

ssl_cert = </etc/ssl/replace.with.host.name.crt
ssl_key = </etc/ssl/private/replace.with.host.name.key

ssl_prefer_server_ciphers = yes
~~~
## /etc/dovecot/conf.d/15-lda.conf
Next up is our local delivery agent (LDA):
~~~
postmaster_address = postmaster@your.domain
hostname = replace.with.host.name

lda_mailbox_autocreate = yes
~~~
## /etc/dovecot/conf.d/20-lmtp.conf
Now we configure our LMTP for Sieve:
~~~
protocol lmtp {
# Space separated list of plugins to load (default is global mail_plugins).
mail_plugins = $mail_plugins sieve
}
~~~
## /etc/dovecot/conf.d/90-plugin.conf
Here we configure the Sieve plugin itself:
~~~
plugin {
sieve_plugins = sieve_imapsieve sieve_extprograms
sieve_global_extensions = +vnd.dovecot.pipe +vnd.dovecot.environment

imapsieve_mailbox1_name = Junk
imapsieve_mailbox1_causes = COPY APPEND
imapsieve_mailbox1_before = file:/usr/local/lib/dovecot/sieve/report-spam.sieve

imapsieve_mailbox2_name = *
imapsieve_mailbox2_from = Junk
imapsieve_mailbox2_causes = COPY
imapsieve_mailbox2_before = file:/usr/local/lib/dovecot/sieve/report-ham.sieve

imapsieve_mailbox3_name = Inbox
imapsieve_mailbox3_causes = APPEND
imapsieve_mailbox3_before = file:/usr/local/lib/dovecot/sieve/report-ham.sieve

sieve_pipe_bin_dir = /usr/local/lib/dovecot/sieve
}
~~~
## /usr/local/lib/dovecot/sieve/report-spam.sieve
Here we are going to fill in the default action for when we move files to spam folder. In our case we learn them as spam:
~~~
require ["vnd.dovecot.pipe", "copy", "imapsieve", "environment", "variables"];

if environment :matches "imap.user" "*" {
set "username" "${1}";
}

pipe :copy "sa-learn-spam.sh" [ "${username}" ];
/usr/local/lib/dovecot/sieve/sa-learn-spam.sh
Fill the file with the following contets:

#!/bin/sh
exec /usr/local/bin/rspamc -d "${1}" learn_spam
~~~
## /usr/local/lib/dovecot/sieve/report-ham.sieve
Here we are going to fill in the default action for when we move files out of the spam folder. In our case we learn them as ham:
~~~
require ["vnd.dovecot.pipe", "copy", "imapsieve", "environment", "variables"];

if environment :matches "imap.mailbox" "*" {
set "mailbox" "${1}";
}

if string "${mailbox}" "Trash" {
stop;
}

if environment :matches "imap.user" "*" {
set "username" "${1}";
}

pipe :copy "sa-learn-ham.sh" [ "${username}" ];
~~~
## /usr/local/lib/dovecot/sieve/sa-learn-ham.sh
Fill the file with the following contets:
~~~
#!/bin/sh
exec /usr/local/bin/rspamc -d "${1}" learn_ham
~~~
## making them both executable
To make them runnable by the system, we make them executable:
~~~
chmod +x /usr/local/lib/dovecot/sieve/*.sh
~~~
## /etc/dovecot/conf.d/90-sieve.conf
And here we configure the sieve folder that gets used for everyone.
~~~
sieve_before = /var/vmail/sieve/
~~~
## /var/vmail/sieve/junk.sieve
First create that folder:
~~~
mkdir -p /var/vmail/sieve
chown -R vmail:vmail /var/vmail
~~~
Then we sieve away the spam:
~~~
require "fileinto";
if header :contains "X-Spam-Status" "YES" {
fileinto "Junk";
stop;
}
~~~
## /etc/dovecot/conf.d/auth-sql.conf.ext
Here we configure our override fields, so we don’t have to do an ugly select:
~~~
userdb {
driver = sql
args = /etc/dovecot/dovecot-sql.conf.ext
override_fields = uid=vmail gid=vmail home=/var/vmail/%u
}
~~~
## /etc/dovecot/dovecot-sql.conf.ext
Now are almost done, we now configure how dovecot accesses the database. Append this to the end:
~~~
driver = pgsql
connect = dbname=mail user=mail password=your.mail.password
default_pass_scheme = CRYPT

password_query = SELECT email AS user, '{CRYPT}' || password AS password FROM accounts WHERE active = true AND email = '%u' AND email != '' AND password != ''
user_query = SELECT email FROM delivery WHERE email = LOWER('%u')
~~~
## /etc/dovecot/dovecot.conf
Last but not least we update the protocols we are going to use:
~~~
protocols = imap pop3 lmtp
~~~
And finally start the dovecot:
~~~
rcctl start dovecot
~~~
## Adding Accounts and Aliases
To generate securely hashed passwords, you can use “smtpctl encrypt” and then enter your password. The resulting hash can be used as replacement for PASSWORD:
~~~
INSERT INTO accounts (email,password) VALUES ('my@first.email.address','PASSWORD');
INSERT INTO virtuals (email,destination) VALUES ('my@second.mail.address','my@first.email.address');
~~~
## That’s it
You should now be able to use this setup as expected.

If you find any errors, you can find me on Twitter and let me know!

