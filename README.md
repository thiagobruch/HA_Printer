# HA_Printer
Turn On and Off Printer based on cupsd queue

## ** This is based on the instructions from [Unixorn](https://unixorn.github.io/post/home-assistant-printer-power-management/)

# Printer Power Control
## mosquitto helper script

Instead of installing anything in the host, we will run the mosquitto_pub inside a container with the following c-mosquitto_pub helper script
You can download it from github [here](https://github.com/thiagobruch/HA_Printer/blob/main/c-mosquitto_pub) . 
Save this inside /usr/local/bin.

# cupsd Watcher
Configure the cupsd to share the printer - Instructions can be found in the end of this README - [Here](https://github.com/thiagobruch/HA_Printer/tree/main?tab=readme-ov-file#cupsd-server).<BR>
In this example the printer name will be Minolta.<BR>

Then a script will be checking the queue if it is empty or not. If there are jobs in the queue, the script will write "OFF" to the MQTT Topic /hass/printer/minolta. If the queue is empty, it writes "ON".
The reason for the "OFF" (when there are jobs) and "ON" (when there are no jobs) it is becasue we do not want Home Assistant to turn off the printer immediately once the queue becomes empty.
Instead, we will configure HA to restart a timer every time it sees the MQTT topic hass/printers/minolta switch from OFF to ON, and only turn the printer off after the queue has been empty for five continuous minutes.

Here’s the ha-check-for-print-jobs script source - you can download it from github [here](https://github.com/thiagobruch/HA_Printer/blob/main/ha-check-for-print-jobs.sh).

Save the script in /usr/local/bin on the same server you’re running the cupsd container on - it is designed to run a tool inside that container.

# Home Assistant Setup
Configure Home Assistant to watch a MQTT topic as a binary sensor. You can download this snippet [here](https://github.com/thiagobruch/HA_Printer/blob/main/printer-binary-sensor.yaml).
Now, when the watcher writes ON and OFF to the hass/printers/minolta queue, that binary sensor will change status and we can trigger an automation for it.
This automation will turn the printer power on every time the binary sensor is turned on, and turn it off five minutes after the last time the binary sensor switched from ON to OFF.
The outlet my printer is plugged into is controlled by HA and named switch.printerpower.
Add this to your automations.yaml file. Download it [here](https://github.com/thiagobruch/HA_Printer/blob/main/printer-automations.yaml).

# Test the pieces
Print server check
Confirm that you’ve got the print queue configured correctly by running:
```
docker exec -it cupsd-server lpq -P minolta
```
If there are no jobs, it should print something like:
```
Minolta is ready
no entries
```
# Automation test
Reload your automations, and you should now be able to test that the automations are correct by running c-mosquitto_pub -h mqtt.yourdomain.com -t hass/printers/minolta -m OFF or -m ON and watch HA turn the power to your printer off and on.

Once that is working, print a job, and if you run ha-check-for-print-jobs the printer power should get turned on.

# Run it all automatically
Now that you’ve confirmed that the power is being cycled properly when the MQTT queue recieves messages and that the print job checker is seeing the printer queue, we can add the checker job to cron.

Add
```
* * * * * PRINT_Q=Minolta MQTT_HOST=mqtt.example.com MQTT_TOPIC=hass/printers/minolta CONTAINER=cupsd_server /usr/local/bin/ha-check-for-print-jobs.sh | logger -t printserver
```
to your /etc/crontab, and you’re good to go. Now every minute, the checker script will get run by cron, and it will check every five seconds for print jobs and exit before the next invocation by cron.

# cupsd server
Make a directory to store your printer configuration. We’ll use /docker/cupsd/etc
export CUPSD_DIR='/docker/cupsd/etc'
touch $CUPSD_DIR/printers.conf
Run the cupsd server with
```
docker run -d --restart unless-stopped \
  -p 631:631 \
  --privileged \
  -v /var/run/dbus:/var/run/dbus \
  -v /dev/bus/usb:/dev/bus/usb \
  -v $CUPSD_DIR/printers.conf:/etc/cups/printers.conf \
  unixorn/cupsd
```
You can now connect to http://cupsd.example.com:631 and add printers using the web UI. (user/pass print/print)
When adding the printers, select Internet Printing Protocol and put in the IP or DNS name of your print server machine.
When entering the printer information into your printer settings, the queues should be entered as printers/printername, not printername.
