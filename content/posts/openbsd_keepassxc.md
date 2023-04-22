---
title: "Self-host a password manager on OpenBSD"
date: 2023-01-08T21:45:17+09:00
draft: false
---

## The big picture 

1. The big picture 

1. Hosting the password database

1. Importing and managing passwords from a laptop

1. Synchronizing Server and Laptop(s)

1. Using the password database on iOS device(s)

1. Notes on using NextCloud

1. Conclusion

I’ve been using Rubywarden to store and access my passwords  from OpenBSD workstations and iOS toys. But recent redondant failures from the iOS App and rubywarden not being maintained anymore led to the need for a new solution.

I was investing on pass+pgp+git but it was quite complex. Following a toot from Solene@, I tried KeePassXC and it does cover my requirements:

- Access and manage passwords from iOS devices ;
- Filling credentials in iOS apps ;
- Accessible and manageable from an OpenBSD and MacOS workstation.
- The big picture
- Self-hosted KeePassXC database
- There are a few bricks to assemble.

The KeePassXC database is an encrypted file. It is hosted on the OpenBSD server and replicated on every devices. The file is encrypted using a pass-phrase and an extra “physical” key. Without both, you shall not be able to decrypt the file.

The KeePassXC file is kept synced between the Server and the Laptop using Syncthing. The KeePassXC file is also accessed using the Strongbox iOS App and kept synced between the Server and the iOS mobile devices using SFTP.

From either KeePassXC and Strongbox, the data can be updated and synced without closing any of those. Very nice.

## Hosting the password database

A dedicated user allows access and synchronization of the password database. It is chrooted in its home directory and can only gain SFTP access ; no SSH login.

~~~
# useradd -c "Strongbox access" -d /home/_strongbox -g =uid -s /sbin/nologin -u 11111 _strongbox
# usermod -G _strongbox <my_uid>

# chown root:_strongbox /home/_strongbox
# chmod 0755 /home/_strongbox

# vi /etc/ssh/sshd_config
(...)
Match User _strongbox
  ChrootDirectory /home/_strongbox
  ForceCommand    internal-sftp

# rcctl reload sshd

# doas -u _strongbox sh -c 'ssh-keygen -C "Strongbox access key" -t ed25519 -f ~/.ssh/strongbox'
# doas -u _strongbox sh -c 'cat ~/.ssh/strongbox.pub >> ~/.ssh/authorized_keys'

# install -o _strongbox -g _strongbox -m 0770 -d KeePassXC
~~~

The SSH private key will have to be installed on the iOS devices ; and/or any device needing to access the password database using SFTP.

The KeePassXC directory and content need to be writable by both the user and the group. iOS devices will log as that SFTP user to sync the database. Syncthing runs as <my_uid>  to synchronize the database.

## Importing and managing passwords from a laptop

On the OpenBSD laptop, KeePassXC can be installed from packages with a browser add-on:

~~~
# pkg_add keepassxc-2.6.1-browser
# vi /etc/firefox/unveil.main
(...)
/usr/local/bin/keepassxc-proxy rx
~~~

Using Firefox, the Bitwarden database hosted via Rubywarden, can be exported as JSON and/or CSV. In my testings, all data are mostly well exported. The JSON is easier to read if you have many fields in the objects. On the Bitwarden plugin, select “Settings” and “Export Vault”. Select the CSV file format and enter the “Master Password”. Select a filename and save it.

Using KeePassXC, select the “Import from CSV” option. Enter the database name and description, then click “Continue”. Review the “Encryption Settings” and “Database format”, then click “Continue”. Enter and confirm the pass-phrase that will protect the database file.

The following steps are optional. Do as you like. Click “Add Additional Protection”. Click “Add Key File”. Click “Generate” and specify the file name of the key. Click “Done”.

Wait, moving your mouse to add entropy, and save the password database file when proposed.

Review the column association to match all fields you need and click “Ok”. I used:

~~~
Group    = Not Present
Title    = Column 4
Username = Column 8
Password = Column 9
URL      = Column 7
Notes    = Column 5
~~~

Note that my “Credit Card” and “Secured Notes” objects were not exported. Those objects don’t exist in KeePass. I had to create/copy/paste manually from the Bitwarden plugin GUI to KeePassXC.

Once done, in KeePassXC, I went to Tools / Settings / Browser Integration and checked “Enable browser integration” and “Firefox”. Then I installed the “KeePassXC-Browser” Firefox plugin and associated it to KeePassXC. As far as I understand, KeePassXC has to be started so that Firefox can access the password database.

## Synchronizing Server and Laptop(s)

Using syncthing(1), the password database hosted on the server is synchronized with a copy hosted on the laptop.

The server shares the /home/_strongbox/KeePassXC directory.

The laptop sync it in ~/KeePassXC.

Using laptops running OpenBSD 6.8 or MacOS Big Sur work the same:

* Use Syncthing to synchronize the password database.
* Use KeePassXC to manage the passwords.
* Use the Firefox plugin to deal with web sites authentication.
* So far, using a Syncthing 1.9 server with Syncthing 1.11 clients doesn’t seem to raise errors. But note that Syncthing recommends using the same version on all hosts.

## Using the password database on iOS device(s)

On the iOS devices, I use Strongbox to manage and synchronize the password database using SFTP. The “paid” version is required to be fully usable.

Click the “+” icon and select “Add an existing database”. Select the “SFTP” location.

Fill-in the server name, credentials for the chrooted user created in the previous steps and enter the SSH private key. Path can be set to “/KeePassXC” or left empty. Click “Connect”, select the database file and click “Add”.

From the Strongbox list, select the recently added password database, enter the master pass-phrase, select the key file used to secure the db and click “Unlock”.

I’m using Face ID to avoid typing the pass-phrase to unlock the database.

In the Settings / Passwords / Prefill passwords section of the iOS device, authorize Strongbox to be used by applications.

## Notes on using NextCloud

In my first experiments, I used a password database hosted in a NextCloud instance. Using iOS Files, Strongbox was able to access the NextCloud storage. But I had issues with synchronization. I regularly had to access the NextCloud share from Files to have Strongbox update it. Another reason to leave NC, but that’s another story.

## Conclusion

So far, everything looks stable. I never access the same database from several devices at the same time. But every update got available to all devices within a few seconds.

Bye bye Bitwarden, you served me well. Thanks again jcs@ for Rubywarden. That was a great self-hosted solution.


