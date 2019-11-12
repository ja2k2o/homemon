# Setting up Ruuvi Collector

## Collecting data

This second part will configure Rasberry to collect the data.

## Install software

Ruuvi provides both Python and Java libraries to collect data from tags. However, the python code is not so great so let's use Java. Thus, a Java and Maven are needed.

```bash
sudo apt install default-jdk maven
```

Bluetooth libraries are needed, too. Raspbian uses Linux SE, so need to set capabilities also.

```bash
sudo apt install bluez bluez-hcidump
sudo setcap 'cap_net_raw,cap_net_admin+eip' `which hcitool`
sudo setcap 'cap_net_raw,cap_net_admin+eip' `which hcidump`
```

## Build and configure RuuviCollector

Ok, ready to build RuuviCollector.

```bash
cd /src/git
git clone https://github.com/Scrin/RuuviCollector
cd RuuviCollector
mvn clean package
```

If there were no errors we're good to set RuuviCollector up.

```bash
cd; mkdir ruuvitag
cp src/git/RuuviCollector/target/ruuvi-collector-0.2.jar ruuvitag/
cp src/git/RuuviCollector/ruuvi-collector.properties.example ruuvitag/ruuvi-collector.properties
cp src/git/RuuviCollector/ruuvi-names.properties.example ruuvitag/ruuvi-names.properties
cd ruuvitag/
touch ruuvicollector.service
```

Edit `ruuvi-collector.properies` with following options (replace obvious part with valid data):

```bash
influxUrl=https://[remote_host]/influxdb/

influxDatabase=house

influxUser=ruuvi
influxPassword=[secretpassword]

influxGzip=false
```

Edit `ruuvi-names.properties` to taste (get tag-ids with `hcidump`).

## Make sure Collector runs at any time

Put the following into `ruuvicollector.service` file:

```bash
[Unit]
Description=RuuviCollector Service
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi/RuuviCollector
ExecStart=/usr/bin/java -jar /home/pi/RuuviCollector/ruuvi-collector-0.2.jar
Restart=always

[Install]
WantedBy=multi-user.target
```

Finally, set up a systemctl service and fire it up.

```bash
sudo cp ruuvicollector.service /etc/systemd/system/
sudo systemctl enable ruuvicollector.service
sudo systemctl start ruuvicollector
sudo systemctl status ruuvicollector.service
```

## Raspberry configuration is done

At this point Raspberry should be configured, up and running and pushing collected measurements into remote database. Only that this remote database does not exist yet so all you get are errors.
