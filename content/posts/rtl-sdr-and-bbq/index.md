---
title: "RTL-SDR and BBQ"
date: 2022-06-24
draft: true
---

When I first got started with BBQ smoking I needed some temperature probes that morning so I raced down to the ACE hardware near my house and picked up a Big Green Egg Dual-Probe Wireless Thermometer. This project is my attempt to turn that simple thermometer output into a graph of temp over time.

<!--more-->

{{< figure 
src="bge-thermometer.jpg" 
caption="Big Green Egg Dual-Probe Wireless Thermometer" 
attr="Big Green Egg"
class="float-image-right"
attrlink="https://biggreenegg.com/product/temperature-gauge-dual-probe-wireless-thermometer/" 
>}}

At first, the simple wireless thermometer was just fine for my smoking needs but as I have continued to hone those skills I started to think it would be nice to have a graph of the temperatures over time to see if I could notice trends faster. For example, just a point in time "temperature of cook chamber is 220" isn't as useful as "temperature was 225 and now it is 220" which signals to me that the temperature is dropping.

As I started to look around, I found that there are a number of really awesome paid and OSS solutions out there for some pretty sophisticated BBQ setups. However, since I already had a wireless thermometer system I first started by looking at how I could use it to acheive my goal of graphing my temperatures.

## Learning About Radio Signals

With almost no radio hardware background, I thought that all I wanted to do would be to intercept the radio signal between the "base" with the temp probes attached and the "remote" which displays the temperatures. Once I could get the data then I could send the temperatures as a time series to a database like InfluxDB and graph it with Grafana.

In my head it looked something like this:
```goat
                                                           o
            intercept  --->  DB  <---  frontend  <--+--- --|--
                ^                                   |      |
                |                                   |     / \
                |                                   |     user
                |                                   |
                |                                   |
transmitter  <--+-->  receiver  <-------------------+
     |
     |
   bbq 🔥 🍖
```

As I started digging into how I could get the radio signals, I found that the BGE thermometer that I had was actually just a rebranded Maverick ET-733 which was fairly well known having been on the market for a while. I stumbled upon an excellent Hackaday project: [Reverse Engineering the Maverick ET-732](https://hackaday.io/project/4690-reverse-engineering-the-maverick-et-732/details) which helped me understand a lot about how deep this rabbit hole really goes. Highlights of the things I learned:

1. **Methodology for observing and decoding radio signals** This brought me down rabbit holes on wikipedia learning about various radio signal encoding mechanisms
1. **Probe tramsits every 12 seconds** Will be useful when I am building graphs in Grafana
1. **Probe always transmits in Celsius** I will need to find a method for converting my data for my graphs in F
1. **If probe is not attached, temperature will be -532C** Will need to somehow discard these erroneous readings

## RTL-SDR

I have been reading about software defined radio projects for a number of years and thought that this project would be a good candidate for me since I needed a receiver that could pull the temperatures from the 433mhz band. I looked around at all of the various people selling a sdr device with the RTL2832U chipset but not really having much experience in this area I decied to go with the rtl-sdr.com branded setup which came with some antennas and mounts.

![rtl-sdr.com software defined radio starter kit]()

Once I got my kit I needed a machine to plug it into so I could start capturing signal. I decided to use a RaspberryPi 4 Model B that I had lying around. As I was verifing that my rtl-sdr kit would work on a Pi 4 with Rasbian I came across a really cool project, rtl_433.

## rtl_433

As it turns out, 433mhz is a very common frequency range for a number of consumer devices like my temperature probe and some folks decided to try to decode everything on that frequency in a single project: [merbanan/rtl_433](https://github.com/merbanan/rtl_433).

This was pretty exciting to find (and even with a debian package, of course!) because it meant I wouldn't have to write my own application to handle the signal processing I had learned in the hackaday article.

After booting the Pi and installing the prerequisites I started rtl_433 not knowing what I was going to get and...

{{< figure 
src="rtl_433-working.png" 
caption="Success! Wait, what is an Acurite-Rain899?"
class="mid-image" 
>}}

I was looking at getting the JSON output and piping that into a time series database and I saw that one of the output formats was influxdb which solved that problem as well.

## InfluxDB

I have a lot of experience running InfluxDB v1 and Telegraf which were some of the first movers in the modern timeseries database space. Since that time, InfluxDB has moved to v2 and changed quite a bit. Notably they moved from the SQL-like dialect to it's own Flux query language. The Flux language is incredibly powerful but it took me a bit longer to get the hang of it.

After installing InfluxDB and hooking up the output from rtl_sdr I briefly looked at deploying Grafana as well but I decided to check out another new feature of InfluxDB: integrated dashboards.

## Building the Dashboard

{{< figure 
src="dashboard.jpg" 
caption="InfluxDB Dashboard of Wireless Temperature Probe"
>}}

## Finished Product

Once everything was built out I put the entire setup upstairs and it works perfectly. The antennas and stand came with the rtl-sdr.com kit which just made this part easy.

{{< figure
src="sdr-antenna.jpg" 
caption="Raspberry Pi and SDR"
class="mid-image"
>}}

### Systemd

For good measure I also set up everything with systemd so it would start capturing when the Pi turns on. I am always happily surprised at how easy it is to set up a service in linux these days.

{{<highlight systemd>}}
[Unit]
Description=rtl_433 service
After=network.target

[Service]
ExecStart=rtl_433 -M time:unix:usec:utc -F "influx://influxdb/api/v2/write?org=home&bucket=rtl_433,token=key"
Restart=on-failure

[Install]
WantedBy=multi-user.target
{{</highlight>}}

{{<highlight bash>}}
$ vi /etc/systemd/system/rtl_433.service
$ systemctl daemon-reload
$ systemctl enable --now rtl_433
{{</highlight>}}
