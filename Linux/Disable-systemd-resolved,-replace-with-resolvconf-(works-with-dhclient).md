```
systemctl disable systemd-resolved.service
systemctl stop systemd-resolved.service

# check if resolv.conf is pointing to resolvconf
ls -la /etc/resolv.conf
# lrwxrwxrwx 1 root root 27 May  7 16:15 /etc/resolv.conf -> /run/resolvconf/resolv.conf
# if not, delete /etc/resolv.conf and symlink it like this: 
rm /etc/resolv.conf
ln -s /run/resolvconf/resolv.conf /etc/resolv.conf

# this will remove the resolved stub resolver entry from resolv.conf
resolvconf -d systemd-resolved

# fix dhclient scripts
chmod +x /etc/dhcp/dhclient-enter-hooks.d/resolvconf

# ifdown/ifup your interface to regenerate resolv.conf (or systemctl restart ifup@eth0)
ifdown eth0; ifup eth0

# check /etc/resolv.conf has the right settings
```
my post from https://askubuntu.com/a/1336755 (slightly modified to remove unnecessary steps)