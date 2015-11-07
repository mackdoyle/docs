 
#Setup FTP server on Ubuntu 14.04

##Step 1:  Install VsFTPD package using the below command.

```bash
sudo apt-get update
sudo apt-get install vsftpd
```

##Step 2:  After installation open /etc/vsftpd.conf file for editing

```bash
sudo vim /etc/vsftpd.conf
```

Uncomment the following lines...

```bash
write_enable=YES
local_umask=022
chroot_local_user=YES
```
...and add the following line at the end.

```bash
allow_writeable_chroot=YES
#enable passive mode.
pasv_enable=Yes
pasv_min_port=40000
pasv_max_port=40100
```

##Step 3:  Restart vsftpd service using the below command.

```bash
sudo service vsftpd restart
```

##Step 4:  Create a FTP user 
Create a user with the command below. Use /usr/sbin/nologin shell to prevent access to the bash shell for the ftp users .

```bash
sudo useradd -m ftproot -s /usr/sbin/nologin
sudo passwd ftproot
```

##Step 5:  Allow login access for nologin shell . Open /etc/shells for editing

```bash
sudo vim /etc/shells 
```
Add the following line at the end.

```bash
/usr/sbin/nologin
```

##Step 6: Verify connection
Now try to connect this ftp server with the username on port 21 using a FTP client.
