---
layout: post
title: The Lake Water Irrigation Project
categories: [Projects, HomeAutomation, Coding, Bash]
---
[![](/images/lake_water_IMG_4306.jpg)](/images/lake_water_IMG_4306.jpg){:target="_blank"}

## Background

The house I live in has access to "lake water", that is, a parallell water grid with irrigation water directly from lake [Mälaren](https://en.wikipedia.org/wiki/Mälaren){:target="_blank"}, in addition to the normal utilities. There are rules to when and how this water can be used, which this project automates.

The water grid is provided by a non-profit  association named “Samfällighetsföreningen Sörgärdets Sjövatten” . The rules that govern the usage are:

- We can use the water on days with even dates. 2, 4, 6…30.
- Only one sprinkler can be used at any time due to pressure issues.

Some additional factors are:

- The property has pavements at its border and soaking people is not appreciated.
- At one of the pavements there are people returning home from the pub at night. They sometimes get peculiar ideas…

## The water network

The picture below shows how the water network in the garden is constructed. The red part is constantly pressurized, the blue parts are pressurized when the main valve is open and the black parts are only pressurized whenever its corresponding valve is open.

Lake water contains a lot of biological material such as algae, humus etc. so filtration is needed to keep the valves functioning. The filter provides this for the system.

[![](/images/lake_water_Irrigation.png)](/images/lake_water_Irrigation.png){:target="_blank"}

In case of a valve malfunction the water will still be turned off since two valves, the main **and** the corresponding sprinkler valve must both be open for the water to flow.

A valve:

[![](/images/lake_water_IMG_4324.jpg)](/images/lake_water_IMG_4324.jpg){:target="_blank"}

## Electronics

These are the basic connections:

[![](/images/lake_water_raspberry.png)](/images/lake_water_raspberry.png){:target="_blank"}



Here are some actual pictures of the build:

[![](/images/lake_water_IMG_4296.jpg)](/images/lake_water_IMG_4296.jpg){:target="_blank"}



[![](/images/lake_water_IMG_4297.jpg)](/images/lake_water_IMG_4297.jpg){:target="_blank"}



## Code

The system is currently controlled by a bash script. The order of the sprinklers are specific to our garden.

The script is scheduled to run at night when few people are active and it runs the sprinklers closest to the pavements first in order to avoid soaking anyone later in the cycle.

At very early Saturdays and Sundays there are people returning home from the pub at night that pass our garden. They sometimes get peculiar ideas and a running sprinkler in the summer heat can provoke undesired behaviour. Therefore the cycle is started later in the morning those days.

### water.sh

```bash
#/bin/bash

i2cset="/usr/sbin/i2cset -y "
i2cget="/usr/sbin/i2cget -y "

# Relay card 1
allrc1="1 0x20 0xa 0xff"
sprinkler1="1 0x20 0xa 0x01"
sprinkler2="1 0x20 0xa 0x02"
sprinkler3="1 0x20 0xa 0x04"
sprinkler4="1 0x20 0xa 0x08"
sprinkler5="1 0x20 0xa 0x10"
sprinkler6="1 0x20 0xa 0x20"
sprinkler7="1 0x20 0xa 0x40"
sprinkler8="1 0x20 0xa 0x80"

 
# Relay card 2
allrc2="1 0x21 0xa 0xff"
main="1 0x21 0xa 0x01"
root="1 0x21 0xa 0x02"
sprinkler9="1 0x21 0xa 0x04"


WATERTIME=1800
#WATERTIME=30
COOLDOWNTIME=10

function log() {
	echo `/bin/date +%Y-%m-%d\ %H:%M:%S` $1 >> /home/pi/irrigation.log
}

function change_state() {
	local bus=$1
	local chip=$2
	local addr=$3
	local bits=$4
	local state=$5

	local curr=`$i2cget $bus $chip $addr`

	if [ "$state" == "off" ]; then
		local new=$(($curr & ~$bits))
		$i2cset $bus $chip $addr $new
	fi

	if [ "$state" == "on" ]; then
		local new=$(($curr | $bits))
		$i2cset $bus $chip $addr $new
	fi
}

function off() {
	local bus=$1
	local chip=$2
	local addr=$3
	local bits=$4

	change_state $bus $chip $addr $bits "off"
	/bin/sleep 1
	change_state $main "off"
	/bin/sleep $COOLDOWNTIME
}

function on() {
	local bus=$1
	local chip=$2
	local addr=$3
	local bits=$4

	change_state $main "on"
	/bin/sleep 1
	change_state $bus $chip $addr $bits "on"
	/bin/sleep $WATERTIME
}

function go() {
	local bus=$1
	local chip=$2
	local addr=$3
	local bits=$4

	on $bus $chip $addr $bits
	off $bus $chip $addr $bits
}



if [ "$1x" == "x" ]; then
	log "Normal schedule"
	log "Turning off all in relay card 1"
	off $allrc1
	log "Turning off all in relay card 2"
	off $allrc2

	log "Sprinkler 8 active"
	go $sprinkler8

	log "Sprinkler 6 active"
	go $sprinkler6

	log "Sprinkler 7 active"
	go $sprinkler7

	log "Sprinkler 9 active"
	go $sprinkler9

	log "Sprinkler 5 active"
	go $sprinkler5

	log "Sprinkler 3 active"
	go $sprinkler3

	log "Sprinkler 2 active"
	go $sprinkler2

	log "Sprinkler 1 active"
	go $sprinkler1

	log "Sprinkler 4 active"
	go $sprinkler4

	log "Irrigation done!"
else
#	echo "Manual $1 $2"
	eval bit=\$$1

	change_state $bit $2
fi

exit
```

### root.sh

Runs the in-soil irrigation.

```bash
# !/bin/bash

function log() {
	echo `/bin/date +%Y-%m-%d\ %H:%M:%S` $1 >> /home/pi/irrigation.log
}

log "Root irrigation active"
/home/pi/water.sh main on
/bin/sleep 1
/home/pi/water.sh root on
/bin/sleep 60
/home/pi/water.sh root off
/bin/sleep 1
/home/pi/water.sh main off
log "Root done!"
```

### crontab

You can't combine time and day of month and day of week conditions in crontab. The solution here is to have two script files that are executed at specific times and days of month, and the day of week condition is checked in the scripts.

```bash
0 3 2-30/2 * * /home/pi/weekday.sh
0 4 2-30/2 * * /home/pi/holiday.sh
```

## weekday.sh

```bash
# !/bin/bash

function log() {
	echo `/bin/date +%Y-%m-%d\ %H:%M:%S` $1 >> /home/pi/irrigation.log
}

dayofweek=`/bin/date +%u`

if [ "$dayofweek" -ne "6" -a "$dayofweek" -ne "7" ]; then
	log "Weekday!"
	/home/pi/water.sh
fi
```

## holiday.sh

```bash
# !/bin/bash

function log() {
        echo `/bin/date +%Y-%m-%d\ %H:%M:%S` $1 >> /home/pi/irrigation.log
}

dayofweek=`/bin/date +%u`

if [ "$dayofweek" -eq "6" -o "$dayofweek" -eq "7" ]; then
	log "Holiday!"
	/home/pi/water.sh
fi
```



## Parts

(Links are to both Swedish and international sites)

- [A Raspberry Pi](https://www.electrokit.com/en/product/raspberry-pi-3-1gb-model-b-3/){:target="_blank"}.
- [Manual valve](https://www.clasohlson.com/se/Kulventil-inv.-utv.-/Pr307106000){:target="_blank"}.
- [Filter](https://www.biltema.se/fritid/tradgard/tradgardspumpar/vattenfilter-2000023583){:target="_blank"}.
- [Electric valves](https://www.electrokit.com/en/product/plastic-water-solenoid-valve-1-2-12v/){:target="_blank"}.
- [Sprinklers](https://www.biltema.se/fritid/tradgard/bevattning/vattenspridare/vattenspridare-2000020693){:target="_blank"}.
- [8-channel relay board for Raspberry Pi and Arduino](https://www.tindie.com/products/jap/8-channel-relay-board-for-raspberry-pi-and-arduino/){:target="_blank"} (I2C version).
- [Power supply](https://www.electrokit.com/produkt/switchat-nataggregat-5v-12v-4a-1a-mean-well-rd-35a/){:target="_blank"}.
- [USB-A Female breakout board](https://www.electrokit.com/en/product/usb-a-female-breakout-board/){:target="_blank"}.
