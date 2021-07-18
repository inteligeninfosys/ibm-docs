## Adding a User
```
adduser newuser
```
Adding the New User to the Sudo Group
```
groups newuser
usermod -aG sudo newuser
```
Specifying Explicit User Privileges in /etc/sudoers
```
sudo visudo
root    ALL=(ALL:ALL) ALL
newuser ALL=(ALL:ALL) ALL
```
## list users
```
less /etc/passwd
awk -F: '{ print $1}' /etc/passwd
```
Each line in the file is  a single account identity… there are seven fields delimited by colons that contain the following:

User name\
Encrypted password (x means that the password is stored in the /etc/shadow file)\
User ID number (UID)\
User’s group ID number (GID)\
Full name of the user (GECOS)\
User home directory\
Login shell (defaults to /bin/bash)

## delete user account command
## create a dir to store backups ##
```
$ sudo mkdir /oldusers-data
$ sudo chown root:root /oldusers-data
$ sudo chmod 0700 /oldusers-data
$ sudo deluser --remove-home --backup-to /oldusers-data/ ubuntu
```
How to temporarily disable user login instead of deleting a user account Use usermod command as follows:
```
$ sudo usermod -L -e 1 {username}
$ sudo usermod -L -e 1 jerry
```
You can specify expiry date too:
```
$ sudo usermod -e {YYYY-MM-DD} {username}
$ sudo usermod -e 2018-02-24 jerry
```
If you had previously configured sudo privileges for the user you deleted, you may want to remove the relevant line again by typing:

```
sudo visudo

root    ALL=(ALL:ALL) ALL
newuser ALL=(ALL:ALL) ALL   # DELETE THIS LINE
```
This will prevent a new user created with the same name from being accidentally given sudo privileges.

