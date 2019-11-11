# Home monitoring with Raspberry Pi, 1-2-3

## Basic settings

This first part focuses on setting up and securing the Raspberry Pi.

### Prepare with a Macbook Air

Download a Raspbian image (lite will do). Official builds are available at <https://www.raspberrypi.org/downloads/>.

Insert an SD card and write the image to the card. Replace `[n]` with correct disk number refering to the SD card.

```bash
diskutil list
diskutil unmount /dev/disk[n]s1
sudo dd bs=1m if=[image] of=/dev/disk[n] conv=sync
```

Make ssh available for headless config by adding an empty file `ssh` on the boot volume.

```bash
sudo touch /Volumes/boot/ssh
```

Enable Wi-Fi for first boot by adding the following into the file `wpa_supplicant.conf` on the boot volume:

```bash
sudo cat <<EOF > /Volumes/boot/wpa_supplicant.conf
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
network={
   ssid="[network_ssid]"
   psk="[network_password]"
   key_mgmt=WPA-PSK
}
EOF
```

Eject the card, plug it in to the Raspberry, and boot.

```bash
diskutil unmount /dev/disk[n]s1
```

### First login

Ssh into Raspberry as user pi. Set password and install updates.

```bash
passwd
sudo apt update
sudo apt upgrade
```

Set up the environment.

```bash
cat <<EOF >.bash_aliases
alias ls='ls -F'

function ll() { ls -la "\$@" | more; }
EOF
```

Install some basic tools.

```bash
sudo apt install vim git locate
sudo updatedb
```

Configure Wi-Fi country, define locales, and enable 1-Wire interface.

```bash
sudo raspi-config
```

Ready to go. Reboot.

### Securing login

Create new user which will be used to log in. This user will only be able to log in and do absolutely nothing else – not even write to his home directory. Replace `[user]` with a desired username.

```bash
sudo useradd -m [user] -s /bin/bash
sudo passwd [user]
sudo -iu [user]
ssh-keygen -t ed25519 -b 4096
cat <<EOF > .ssh/authorized_keys
[copy-paste]
EOF
chmod u-w .ssh/* .[bpsv]* .
logout
```

Configure sudo. Login user must sudo in order to do anything. Again, replace `[user]` with the username you created in the previous step.

```bash
sudo -i
cat <<EOF >/etc/sudoers.d/010_[user]
[user] ALL=(pi) /bin/bash
EOF
```

Test. If any errors are detected fix them.

```bash
sudo visudo -c # if there are ANY errors, fix them before logging out!
logout
```

### Securing ssh

Install `ssh-audit` and run it to see weak points of the current installation.

```bash
mkdir -p src/git; cd src/git
git clone https://github.com/arthepsy/ssh-audit.git
python ssh-audit/ssh-audit.py localhost
cd
```

Edit `/etc/ssh/sshd_config` with following options:

```bash
ListenAddress 0.0.0.0
ListenAddress ::

PermitRootLogin no

AllowUsers [user]

# key exchange algorithms
KexAlgorithms [list the green ones from ssh-audit's output]

# host-key algorithms
HostKeyAlgorithms [the green ones]

# encryption algorithms (ciphers)
Ciphers [greens]

# message authentication code algorithms
MACs [greens]

PasswordAuthentication no
ChallengeResponseAuthentication no

UsePAM yes

X11Forwarding yes

PrintMotd no

AcceptEnv LANG LC_*

Subsystem sftp /usr/lib/openssh/sftp-server
```

Test. If any errors are detected fix them.

```bash
sshd -t # if there are ANY errors, fix them!
sudo systemctl restart ssh
```

Make sure you can still ssh into the box. And sudo to `pi` user.

### Firewall

Configure firewall to allow all connections from local private network and from external networks allow only ssh on IPv6 inteface. Default is deny everything else. Replace `192.168.1.0/24` with your network.

```bash
sudo apt install ufw
sudo ufw allow ssh
sudo ufw limit ssh/tcp
sudo ufw allow from 192.168.1.0/24
sudo ufw delete 1
sudo ufw enable
```

### Time

Edit `/etc/systemd/timesyncd.conf` with:

```bash
NTP=ntp1.funet.fi ntp2.funet.fi
```

Restart not-anymore-ntpd. Oddly, timesyncd still requires ntpd…

```bash
sudo apt install ntp
sudo systemctl restart systemd-timesyncd
```
