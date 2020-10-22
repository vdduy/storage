Install samba on Centos 7

```
systemctl disable --now firewalld
setenforce 0
sed -i s/SELINUX=enforcing/SELINUX=disabled/g /etc/selinux/config
```
```
yum install -y samba samba-client
```
```
systemctl enable --now smb.service
systemctl enable --now nmb.service
```
```
mkdir /samba

groupadd sambashare 
chgrp sambashare /samba
useradd -M -d /samba/josh -s /usr/sbin/nologin -G sambashare josh
mkdir /samba/josh
chown josh:sambashare /samba/josh
chmod 2770 /samba/josh
smbpasswd -a josh
smbpasswd -e josh
useradd -M -d /samba/users -s /usr/sbin/nologin -G sambashare sadmin
smbpasswd -a sadmin
smbpasswd -e sadmin

mkdir /samba/users
chown sadmin:sambashare /samba/users
chmod 2770 /samba/users
```
```
vi /etc/samba/smb.conf
...
[users]
    path = /samba/users
    browseable = yes
    read only = no
    force create mode = 0660
    force directory mode = 2770
    valid users = @sambashare
```

    
The options have the following meanings:

   - [users] and [josh] - The names of the shares that you will use when logging in.
   - path - The path to the share.
   - browseable - Whether the share should be listed in the available shares list. By setting to no other users will not be able to see the share.
   - read only - Whether the users specified in the valid users list are able to write to this share.
   - force create mode - Sets the permissions for the newly created files in this share.
   - force directory mode - Sets the permissions for the newly created directories in this share.
   - valid users - A list of users and groups that are allowed to access the share. Groups are prefixed with the @ symbol.

```
systemctl restart smb.service; systemctl restart nmb.service
```

