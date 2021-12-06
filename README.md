# Setting up home monitoring with a Raspberry Pi and Ruuvitags

Easy way to start up is to clone this repository to your computer. These instructions are written for macOS.

## Bring life to the Pi

### Prepare the drive

Browse to [Pi OS official download page](https://www.raspberrypi.org/downloads/) and get a suitable **Pi Imager software**. Prepare an SD card according to the instructions given. The Pi Imager will unmount the card upon finishing so remove and re-insert it to mount it again because we're performing a headless setup and therefore need to do a few tricks before booting the Pi OS:

1. Make ssh available for headless configuration by adding an empty file `ssh` on the boot volume.

   ```bash
   sudo touch /Volumes/boot/ssh
   ```

2. Enable Wi-Fi for the first boot by adding the following into the file `wpa_supplicant.conf` on the boot volume:

   ```bash
   sudo cat <<EOF > /Volumes/boot/wpa_supplicant.conf
   ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
   network={
      ssid="[your network ssid]"
      psk="[your network password]"
      key_mgmt=WPA-PSK
   }
   EOF
   ```

Eject the card, plug it in to the Raspberry Pi, and boot it up.

### First login

Ssh into the Raspberry as user _pi_ with the default password _raspberry_. Set a new password and plant your ssh public key in.

```bash
passwd
mkdir -m 0700 .ssh
cat <<EOF >.ssh/authorized_keys
[your public key]
EOF
```

Configure the Wi-Fi country, define locales, and do the other things, if needed.

```bash
sudo raspi-config
```

All done and ready to rock. Reboot.

```bash
sudo shutdown -r 0
```

### (optional) Harden ssh

You may want to harden the sshd. This probably does not make a huge difference, but it's fairly quick and easy so why not.

Log in to Raspberry, get _ssh-audit_ tool and then run it against the local sshd.

```bash
mkdir git && cd git && git clone https://github.com/arthepsy/ssh-audit.git
python ssh-audit/ssh-audit localhost
```

The green lines indicate safe algorithms to be used, so allowing _sshd_ to use only those lower the (already low) possibility of someone getting access to your Raspberry. So, include `KexAlgorithms`, `HostKeyAlgorithms`, `Ciphers`, and `MACs` configuration options populated with the green algorithms (comma separated) into the `/etc/ssh/sshd_config`. And while you are at it, also change `PermitRootLogin` to `no`. You should probably also limit who can log in and from where by adding `AllowUsers`, e.g. `AllowUsers *@192.168.1.0/24 pi@host`.

While still logged in, restart _sshd_ and make sure you can log in before you log out from the Raspberry Pi.

```bash
sudo systemctl restart ssh
```

## Ruuvi for you

 This is the easy part but requires some manual configuration.

### Setup your inventory

You need to define your inventory, i.e. the hosts (servers) you are going to use.

0. Clone this repository if you haven't done so yet.

1. In the `./hosts` file replace `raspberrypi` with the hostname or IP address of your Raspberry Pi.

   1. If you did change the hostname you need to rename `host_vars/raspberrypi` accordingly:

   ```bash
   ( cd host_vars && mv raspberrypi [your hostname] )
   ```

2. This configuration assumes you have another host somewhere which will store and do visualization of the data the RuuviTags will generate. Again, in the `./hosts` file replace `host-4` and rename `host_vars/host-4` accordingly.

### Configuration and building

There are a few places for variables you need to check and modify based on your setup.

1. Variables related to the software and the hosts running it are located as follows:

   | file                    | purpose                                          |
   |:----------------------- |:------------------------------------------------ |
   | `group_vars/all`        | common settings for all hosts                    |
   | `host_vars/raspberrypi` | settings for the RuuviCollector host             |
   | `host_vars/host-4`      | settings for the database and visualization host |

   The variable names in these files should be self-explanatory enough.

2. You need to define which Ruuvi tags you want to read. You can obtain the ID's of the tags e.g. with Ruuvi mobile app and enter them in the `templates/ruuvi-names.properties` file. Ruuvi tags not listed in this file will not be read and no data from those tags are fed into the database. You can define whatever name you want for the tags.

3. Currently, (March 2021) the RuuviCollector software does not support InfluxDB v2, but the templates include a configuration to set it up. However, InfluxDB v1.8 is being used. If you want to have InfluxDB v2 you need to modify the `main.yml` file.

4. After all the above is OK, just run

   ```bash
   ansible-playbook -i hosts main.yml
   ```

   and you should have everything build, configured and fired up automatically.

At this point the Grafana visualization web interface is functional and accessible at your visualization host's local port 3000, `http://localhost:3000`. You should set up a web proxy (e.g. with nginx or Apache) with suitable access control mechanisms. Also, the InfluxDB API is open at `http://localhost:8086` and should be proxied accordingly. Installation and configuration of the proxy is not part of this documentation, but some basic instructions can be found [proxy.md](here).
