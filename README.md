# Setting up an SMB, Bonjour server for MacOS

This article describes the following:

* installing smb (sambda) for file sharing server
* configuring Avahi for Bonjour discovery
* tuning sambda for Apple smb implementation
* configuring the Pi to work with overlay (RAM only)
  * well, this is optional, I do it as I need it for temporary storage only for my network scanner

## Install Raspberry Pi OS Lite (64-bit if you have Pi 3/4/400, otherwise 32-bit)

* Download the Raspberry Pi Imager
* Select the right OS and memory card
* Click the "Settings" cog icon
  * Configure your username, password, ssh access and (if using WiFi) WiFi details
* Hit "Write" and go make yourself a hot beverage

## Boot up your Pi and ssh to it

This article assumes you know how to either:

* check your Pi IP address from your router
* or connect the Pi to your monitor, login and run `ip addr sh`

## Install the required packages:

```bash
ssh pi@RASPBERRY_IP
sudo -i
apt update
apt upgrade
apt install avahi-daemon samba samba-vfs-modules
```

## Configure Avahi for Bonjour

Add configuration file:

```bash
cat <<'EOF' > /etc/avahi/services/samba.service 
<?xml version="1.0" standalone='no'?><!--*-nxml-*-->
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">

<service-group>
  <name replace-wildcards="yes">%h</name>
  <service>
    <type>_smb._tcp</type>
    <port>445</port>

  </service>
</service-group>
EOF
```

## Add user and directory

```bash
mkdir /scans
useradd -m -U -p 'some-user-password' scanner
chown -R scanner:scanner /scans
echo -e 'some-user-password\nsome-user-password' |smbpasswd -s -a scanner
```

## Configure sambda for sharing

Backup smb.conf:

```bash
cd /etc/sambda
mv /etc/samba/smb.conf /etc/samba/smb.conf.backup
```

Configure smb.conf:

```bash
cat <<'EOF' > /etc/samba/smb.conf
[global]
	workgroup = WORKGROUP
	log file = /var/log/samba/log.%m
	max log size = 1000
	logging = file
	panic action = /usr/share/samba/panic-action %d
	server role = standalone server
	obey pam restrictions = yes
	unix password sync = yes
	passwd program = /usr/bin/passwd %u
	passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
	pam password change = yes
	map to guest = bad user
	min protocol = SMB2
	ea support = yes
	vfs objects = fruit streams_xattr
	fruit:metadata = stream
	fruit:model = Tower # icon
	fruit:posix_rename = yes 
	fruit:veto_appledouble = no
	fruit:zero_file_id = yes
	fruit:wipe_intentionally_left_blank_rfork = yes 
	fruit:delete_empty_adfiles = yes
	fruit:nfs_aces = no ### this setting (setting to 'no') prevents MAC from modifying the ACLs of uploaded files
### uncomment the below to enable time machine backup
### remember to add a user and modify the user/path here
# [TimeMachineBackup]
# 	path = /scans
# 	valid users = scanner
# 	read only = no
# 	vfs objects = fruit streams_xattr
# 	fruit:time machine = yes
# 	fruit:time machine max size = SIZE
[scans]
	path = /scans
	valid users = scanner
	read only = no
EOF
```

## Restart samba and avahi:

```bash
service nmbd restart
service smbd restart
journalctl -u nmbd -n 10
journalctl -u smbd -n 10
service avahi-daemon restart
journalctl -u avahi-daemon -n 10
```

## Add monitoring for WiFi drops and auto-reboot

Download the pingtest tool:

```
cd /opt
# replace arm64 with arm in the below commands if using Pi Lite 32 bit OS
wget https://github.com/rglonek/pingtest/releases/download/latest/pingtest.linux.arm64
mv pingtest.linux.arm64 pingtest
chmod 755 pingtest
```

Setup crontab (replace 192.168.0.1 with your router IP you wish to ping against):

```
echo '* * * * *  root  /opt/pingtest -host 192.168.0.1 -interval 14s -number 4 -privileged -threshold 100 || /usr/sbin/reboot' >> /etc/crontab
service cron restart
```

## Follow the below to overlayroot

This will essentially make the filesystem read only and store changes in RAM only

[Click here to follow the manual](https://github.com/rglonek/raspberry-pi-wifi-ethernet-bridge/blob/main/steps/overlayroot.md)
