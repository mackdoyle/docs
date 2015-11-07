#Mounting Shares for Unbuntu Virtualbox on OS X

##Using Samba

####Step1:
Install Samba on guest

```bash
sudo apt-get update
sudo apt-get install samba
```

####Step 2:
Set a password for your user in Samba. (use the same password you log into guest with)

```bash
sudo smbpasswd -a <user_name>
```

####Step3:
Create a directory to be shared

```bash
mkdir /home/<user_name>/<folder_name>
```

####Step 4:
Make a safe backup copy of the original smb.conf file to your home folder, in case you make an error

```bash
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.orig
```

####Step 5:
Edit the file "/etc/samba/smb.conf"

```bash
sudo vim /etc/samba/smb.conf
```

####Step 6:
Add this to smb.conf. (replace <> sections with desired settings)

```bash
[<folder_name>]
path = <full_path_to_folder>
available = yes
valid users = <user_name>
read only = no
browseable = yes
public = yes
writable = yes
```

####Step 7:
Restart the samba:

```bash
sudo service smbd restart
```

####Step 8:
Once Samba has restarted, use this command to check your smb.conf for any syntax errors

```bash
testparm
```

####Step 9:
Mount your network share on OS X
In Finder's menu bar, go to Go -> Connect to Server
Enter: `smb://<HOST_IP_OR_NAME>/<folder_name>/`

You can now access your share via the finder or in your terminal at `cd /Volumes/<folder_name>`

#NOTES:
##Step 1: Adding Share in Virtualbox Client

Shares you add in Virtualbox will be located in `/media` on the guest machine

##Step 2: Guest Configuration

```bash
sudo usermod -a -G vboxsf <userName>
```
You must **log out and back in** for the new permissions to take effect.


fstab

```bash
//adams-mbp/Transmission /mnt/adams-mbp cifs credentials=/root/secret.txt,iocharset=utf8,uid=1000,gid=1000,file_mode=0777,dir_mode=0777,nounix 0 0
```

