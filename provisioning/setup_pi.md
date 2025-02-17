# Setting up Raspberry Pi units for use with ODM360

We provide pre-configured images for the various components (Parent, Child, and Timeserver) of the ODM360 kit __(TODO: THAT)__, but for developers or people wishing to customize their setup, this is the full install procedure.

## Buy stuff and get ready
- Get a the appropriate Raspberry Pi for the component you are making.
  - For a TimeServer or a Child, this is a Raspberry Pi Zero W (or the WH, which is the same device but has pre-soldered headers, which will save you some work).
    - Note that the non-W version does not have a WiFi chip, and will be more complicated to set up (it can be done, specifically by sharing network over USB from your computer, but you don't want to).
  - For a Parent, you probably want a heavier Raspberry Pi. __TODO: MENTION WHICH ONE__
- Get a decent quality micro SD card. Absolute minimum 8GB, but wouldn't recommend bigger than 64GB (compatibility issues can come up). Read [this](https://www.raspberrypi.org/documentation/installation/sd-cards.md) or do some research if you want to fiddle with it, otherwise buy a bog-standard brand-name SD card and get on with it. I'm using SanDisk Extreme 64GB cards. They seem fine.
- Work somewhere where there's a WiFi router without any complicated settings. You'll need the SSID and password of the router.
  - If there's a captive portal/login page or other complexities (as in a lot of hotels and some co-working spaces) be warned: your life will suck.
  - If need be (for example if the only WiFi you can find has a captive portal) you can do it all tethered to a WiFi Hotspot from your phone.
- You'll need a computer with an SD card reader.

## Set up base infrastructure on the Pi
- Download the Raspberry Pi OS (32-bit) Lite image from [here](https://www.raspberrypi.org/downloads/raspberry-pi-os/). The Lite image doesn't have a graphical interface (it's "headless") and is therefore smaller to download and will run more efficiently on the limited Pi hardware.
- Unzip the archive and flash the image to your SD card using [Balena Etcher](https://www.balena.io/etcher/). You can mess around with other ways to burn images if you are feeling masochistic; Etcher just works, and it's open source.
- After flashing the image, it's a good idea to eject the card. You'll need to put it back in momentarily, but it seems that sometimes things go wrong if you don't eject and re-insert the card.
- Now you need to get the Pi on the WiFi, enable SSH, and connect to it. The SD card you just flashed should appear in your file explorer, open it up.
  - The SD card should now have two partitions, a small one called ```boot``` and a larger one called ```rootfs``` (if you're on Windows you might not see the ```rootfs``` partition; too bad for you—you should be using Linux anyway—but it doesn't matter for now). Put a ```wpa_supplicant.conf``` file in the ```boot``` partition of the SD card. Explanation of that file [here](https://www.raspberrypi.org/documentation/configuration/wireless/headless.md).

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=TZ

network={
 ssid="OMDTZ-Guest"
 psk="apassword"
}

```

I copy the contents of that example, navigate to the root of the ```boot``` partition on the SD card and type ```nano wpa_supplicant.conf``` and paste in the contents, replacing the SSID, password, and country code with the appropriate ones.
  - Also in the ```boot``` partition, you need to place a file called ```ssh```. It doesn't matter what's in it (an empty file is fine) and there's no extension on the filename. This enables [SSH](https://en.wikipedia.org/wiki/Secure_Shell) on the Pi (disabled by default). You can do this on Linux or Mac by navigating to the root of the ```boot``` partition and typing ```touch ssh```.
- Eject the SD card and remove it from your computer, put it in the Pi, and power it up. Wait about 90 seconds to make sure it has time to boot up before you try to connect to it.
- Make sure your computer is connected to the same WiFi network as the Pi, open a terminal, and type ```ssh pi@raspberrypi.local```.
  - Didn't work? ```Could not resolve hostname...``` or something like it? Sigh. Ok, try to figure out if the Pi is on the WiFi. You can try [nmap](https://nmap.org/) to attempt to identify every device on your local subnet with ```nmap -sP 192.168.X.0/24```, but I recently found a lovely FOSS application called [Angry IP Scanner](https://angryip.org/); it's great. Try it. If you're lucky, you'll see the Pi on the network along with its IP address, and maybe you can ssh into it using the IP address instead of the ```raspberrypi.local``` alias. If not, you'll have to dive into the world of Google, StackExchange, and the Raspberry Pi forums until you get it sorted.
- Log into the Pi using the default password, which is ```raspberry```. Once you're in, immediately type ```passwd``` (_without_ ```sudo```) and—at the prompts—enter first the old and then the new password (twice). Try not to forget the new password.
- Get everything up to date with ```sudo apt update && sudo apt upgrade -y```. This will take a few minutes, more if your Internet connection is slow.
- You might as well install a few more basic infrastructure bits while you're at it:

```
sudo apt install -y git python3-pip libgphoto2-dev libatlas-base-dev gfortran
```
- Fetch and install the ODM360 code

```
git clone https://github.com/OpenDroneMap/odm360
cd odm360
pip3 install -e .

```


From here you have a basic Raspberry Pi configuration; you can make it into a Parent, a Child, or a TimeServer with the next steps.

## Postgresql and accessing it from Python

Install posgresql:

```
sudo apt install postgresql postgresql-contrib
```

Set up a user and database.

- Create a postgres user by
```
sudo su postgres
createuser --interactive
```
Give the user a sensible name like ```odm360``` and make it a superuser

- Run psql and set up that user and a database
```
psql
# GIVE USER A PASSWORD, CREATE A DATABASE, SET DATABASE OWNER TO THE USER, ETC
```

Install psycopg2, an adapter (sort of like a driver) for Python to connect to Postgres

```
sudo apt install libpq-dev
pip3 install psycopg2
```

Try connecting to the database in Python with a connection and a cursor. Proper documentation [here[(https://www.psycopg.org/docs/usage.html). Sample code:

```
import psycopg2

con = psycopg2.connect('dbname=odm360 user=odm360 host=localhost password=mypassword')
cur = con.cursor()
```

Try making a table with the cursor:

```
cur.execute("CREATE TABLE photos(uuid serial PRIMARY KEY, project VARCHAR (50) NOT NULL, survey_run VARCHAR (50) NOT NULL, device VARCHAR (50) NOT NULL, photo_filename VARCHAR (100) NOT NULL, photo_id VARCHAR (50) NOT NULL)")
con.commit()
```

Try sticking some data into that table:

```
cur.execute("INSERT INTO photos VALUES (1,'stonetown', '2020-09-11', 'cam3', 'in/that/folder', 'DCIM123456.jpg');")
con.commit()
```

Try reading it:

```
cur.execute("SELECT * FROM photos")
data = cur.fetchall()
data
```

Clean up after yerself:
```
cur.close()
con.close()
```


## Physical wiring

You will need to connect the serial communication and power pins on the GNSS receiver to the Raspberry Pi Zero (or other Raspberry Pi). Here's an example with an Ardusimple connected to a Zero (note: this cable harness was made with a kit of DuPont connecters and a crimper, which is the way to go if you are doing more than a little of this).

![Pi_wiring](Photos/Pi_wiring.jpg)
![ublox_wiring](Photos/ublox_wiring.jpg)
![Pi_and_ublox_wiring](Photos/Pi_and_ublox_wiring.jpg)

*TODO: photo of Ardusimple with header soldered onto PPS pin and connected to an appropriate Pi GPIO pin*

## Setting up NTP for a server and client device on the local network

### On all devices

Raspberry Pi OS comes with a lightweight version of NTP client called systemc-timesyncd. We want the full NTP service. Set it up with ```sudo apt install ntp```.

Then become root ```sudo su -``` (you should now be root, and have a hashmark rather than a dollar sign as your terminal prompt).

Stop the default lightweight NTP service (systemd-timesyncd) and start the full NTP:

```
systemctl stop systemd-timesyncd
systemctl disable systemd-timesyncd
​/etc/init.d/ntp stop
​/etc/init.d/ntp start
```

Test it with ```ntpq -pn```. You should see an infodump that lists what timeservers are being contacted and which of them is the reference being used, marked with an asterisk (*). It may take a few minutes after starting NTP for the asterisk to appear (it takes a while for NTP to be sure of the timing over a network; there's actually a lot of sophisticated stuff going on under the hood to account for network latency)

The configuration to choose other timeservers (which we'll use on the client Pi's to select the timeserver Pi), you edit ```/etc/ntp.conf```.

Stop being root with ```exit``` (you should now be the pi user again).

### Server-specific setup (to share time with others)

Again you have to log in as the root user with ```sudo su -``` (and ```exit``` when you're done).

Edit the ```/etc/ntp.conf``` file again, adding the lines:
```
restrict 192.168.1.0 mask 255.255.255.0

broadcast 192.168.1.255
broadcast 224.0.1.1
```

### Client-specific setup (assuming you already have a server on the network)

Again you have to log in as the root user with ```sudo su -``` (and ```exit``` when you're done).

You should already have the IP address of the timeserver on your network (if not, you can use the hostname).

Edit the ```/etc/ntp.conf``` file, commenting out the existing servers (which will probably be listed as a pool rather than specific servers). Comment them out, and add a line directing NTP to the server on the network. The relevant section of the ```/etc/ntp.conf``` file should look like this when you're done (of course with the correct IP address at the end):

```
#pool 0.debian.pool.ntp.org iburst
#pool 1.debian.pool.ntp.org iburst
#pool 2.debian.pool.ntp.org iburst
#pool 3.debian.pool.ntp.org iburst 
server 192.168.1.15
```

Test with ```ntpq -p```. It may take a few minutes for the client to sync with the server, but when it does, the results of this command should show an asterisk (*) in front of the line for the timeserver.

## TimeServer (now implemented on the Parent)

The TimeServer is a dedicated Raspberry Pi that listens to the GNSS unit and syncs to the ([extremely precise](https://gssc.esa.int/navipedia/index.php/Precise_Time_Reference)) timing pulse from it and provides timing information via [Network Time Protocol (NTP)](https://en.wikipedia.org/wiki/Network_Time_Protocol) to the rest of the kit.

All devices in the ODM360 kit need to have very precise time, _milliseconds count_. Even moving at a walking speed (around 1.5 m/s), more than a few tens of milliseconds of timing offset will already introduce errors larger than the accuracy of a well-functioning GNSS setup. On a vehicle (car, motorcycle, Bajaj, or drone) which moves faster, this is much worse. We aim for synchronization of less than 5ms. 

To work well, the TimeServer must listen to both the [NMEA stream](https://en.wikipedia.org/wiki/NMEA_0183) coming from the GNSS, which provides timestamps per reading (not particularly accurate, but close enough for setting time to the second) as well as the [Pulse Per Second (PPS)](https://en.wikipedia.org/wiki/Pulse-per-second_signal) signal that is sub-millisecond precise.

Once its own clock is set from the GNSS device (making it a [stratum 1 device](https://en.wikipedia.org/wiki/Network_Time_Protocol#Clock_strata)), it provides an NTP server to the other devices. This ideally requires a short-wire, very low-latency, symmetrical (same signal time both directions) network connection—we're still working on getting this right.

The PPS signal needs to be on a separate pin (__TODO: PICTURES OF THE PIN WIRING__) and for best precision should use an interrupt instead of a loop.

### Steps to configure the Pi Zero as a TimeServer

*Note: Much of this can be done while setting up the base Pi just after flashing. It's separated out here so that it can be scripted and eventually added to a [.deb package](https://en.wikipedia.org/wiki/Deb_(file_format)) specific to the TimeServer setup.*

- Change the name of the Pi to "timeserver" ```sudo hostnamectl set-hostname timeserver``` and inform the Pi of this in the hosts file ```sudo sed -i 's/127.0.0.1\tlocalhost/127.0.0.1\tlocalhost timeserver/g' /etc/hosts```

- Disable IPv6 ```sudo sed -i '$s/$/ ipv6.disable=1/' /boot/cmdline.txt```

- Disable the default serial console to free up UART serial line ```sudo sed -i 's/console=serial0,115200 //g' /boot/cmdline.txt```

