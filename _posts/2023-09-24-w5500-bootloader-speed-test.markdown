---
layout: post
title:  "Fixing Connection Errors and Comparing TCP send speeds of Wiznet W5500 bootloaders"
date:   2023-09-25 20:00:00 -0400
categories: jekyll update
---

tl;dr [Test Results](#test-results)

In my [previous post]({% post_url 2023-09-21-w5500-intro %}) we got up and running with the W5500 and touched lightly on the various bootloader options for the board.  I had some issues on the client side of the [Wiznet releases](https://github.com/Wiznet/RP2040-HAT-MicroPython/releases), and in the article suggested just sticking to the [Official Micropython builds](https://micropython.org/download/W5500_EVB_PICO/), as they didn't seem to suffer from the same issues.  However, upon further testing I've found that the official Micropython release is *far* slower than the Wiznet ones, at least in my barebones TCP tests.

First, some good news.  I believe I have gotten to the bottom of what the issue is with the Wiznet releases.  Or at the very least I've got something that hasn't failed me yet across at least 15 test runs, whereas my success rate previously was less than 25%.  

Going back to the setup of our client in the bidirectional tests, we have the following chunk of code:

[[bidirectional.py]](https://github.com/sjhennion/gh_pages_materials/blob/main/w5500-intro/bidirectional.py)
{% highlight ruby %}
#...
    print("Attempt Loopback client Connect!")

    s = socket()
    s.connect(('192.168.1.20', 5000)) #Destination IP Address
    
    s.send('1')
#...
{% endhighlight %}

From what I can tell, it turns out that if attempting to connect via the socket too quickly after creating it, we can run into the two dreaded (and as far as I can tell, worthlessly un-google-able) errors:

{% highlight ruby %}
OSError: 4
# and
OSError: [Errno 13] EACCES
{% endhighlight %}

The fix is rather easy, albeit a bit of a hack.

{% highlight ruby %}
#...
    print("Attempt Loopback client Connect!")

    s = socket()
    utime.sleep_ms(5000)
    s.connect(('192.168.1.20', 5000)) #Destination IP Address
    
    s.send('1')
#...
{% endhighlight %}

I don't care for this too much.  Usually, the way these things work is that rather than some artificial sleep, we should be able to query the socket object to see whether it's ready to be used for a connection.  I'll investigate this further, but for now the sleep has been rock solid.  5 seconds works, I haven't had issue with 4 seconds, but 2 seconds was too short and reintroduced the errors.  

With that out of the way, let's take a look at why getting these Wiznet builds stable is so important.

While writing the previous article and testing things out, I never went back and reperformed the W5500 to W5500 client/server test after applying the official Wiznet bootloader.  When I sat down to start working on my enhancements to my examples, I found that I was getting really poor speeds when sending from the client W5500, regardless of whether it was to the server running on the W5500, or on my PC.  When performing the last tests in that article, I just assumed it was the LED value toggles or the print statements slowing things down, but even once I pulled that all out, no dice.  For a sanity check, I retried the Wiznet releases, both the `1.0.5 stable` build as well as the `2.0.0 prerelease`.  It turns out the `1.0.5` build was much quicker than the Micropython build (neither the stable or nightly builds made a difference, `v1.20.0 (2023-04-26) .uf2` and `v1.20.0-489-ga3862e726 (2023-09-20) .uf2` respectively as of this writing), and the `2.0.0` was *much* faster even than the `1.0.5` build.

Phooey.

This was pretty unfortunate, as obviously we're looking for reliability, as well as speed here, and the speeds I was seeing were nowhere near acceptable.  So, what to do?  At this point I had not yet discovered the sleep trick above, so as far as I knew the Wiznet builds were still unusable. That meant it was time to go looking around for *another* option.

## Circuitpython

Before I got my first batch of Pico Pis, I had done a little bit of prior research on the ecosystems around them.  I plan on utilizing the boards with a combination of off the shelf parts such as OLED screens, LCDs, 7-segment displays, as well as discrete components like switches, LEDs, as well as my own bespoke boards utilizing, SPI, I2C, etc.  Basically, the full range of hobbyist electronics "stuff".  During that initial googling around for "micropython + \[X\]" vs "circuitpython + \[X\]", the vibe I got was that there was more available on the micropython side.  I also really liked the CLI support with `rshell`, so I started down that path.  

After running into the issues described above, I knew I had to take another look at circuitpython, and when I found the github repository for the W5500 support I realized I might have jumped the gun on passing on circuitpython originally.  The project seems to be much more mature, as well as more recently and consistenly updated than the micropython equivalent.  But, the million dollar question remained, how does it compare in the stability and speed test?  Let's take a look.

## The Testing Rig

We're going to reuse the W5500 Client -> PC Server example from the previous article, with a few modifications to slim it down and perform some metric testing.  Our server remains unchanged and you can find it [here](https://github.com/sjhennion/gh_pages_materials/blob/main/w5500-speed/w5500_server.py).  Here's our modified client while running micropython:

[w5500_speed_client_micropython.py](https://github.com/sjhennion/gh_pages_materials/blob/main/w5500-speed/w5500_speed_client_micropython.py)

{% highlight ruby %}
from usocket import socket
from machine import Pin,SPI
import network
import time
import utime

led = Pin(25, Pin.OUT)

#W5x00 chip init
def w5x00_init():
    spi=SPI(0,2_000_000, mosi=Pin(19),miso=Pin(16),sck=Pin(18))
    nic = network.WIZNET5K(spi,Pin(17),Pin(20)) #spi,cs,reset pin
    nic.active(True)

    print('Client')
    nic.ifconfig('dhcp')
    
    print('IP address :', nic.ifconfig())
    while not nic.isconnected():
        time.sleep(1)
        print(nic.regs())
    
def client_loop():
    print("Attempt Loopback client Connect")

    s = socket()
    utime.sleep_ms(4000)
	# Connect to PC
    s.connect(('192.168.1.100', 5000)) #Destination IP Address

    s.send('1')

    start_time = time.ticks_ms() 
    print("Loopback client Connect!")
    for x in range(999):
        data = s.recv(16)
        print(data.decode('utf-8'))
        if data != 'NULL':
            data_int = int(data) + 1
            s.send(str(data_int))
    end_time = time.ticks_ms() 

    print("Total time: ", time.ticks_diff(end_time, start_time))

        
def main():
    # Toggle LED for sign of life
    for y in range(3):
        led.value(1)
        utime.sleep_ms(50)
        led.value(0)
        utime.sleep_ms(50)

    w5x00_init()
    
    client_loop()

if __name__ == "__main__":
    main()
{% endhighlight %}

The main difference here is that rather than sending messages back and forth until the program is force quit, we only run for 1000 iterations.  We collect the system ticks in milliseconds before and after, and take the difference of these two at the completion of the test and calculate our messages sent per second with:

{% highlight bash %}
1000 messages       1000 MS         X messages
-------------  *  -----------   =   -----------
 diff in MS          1 sec             1 sec
{% endhighlight %}

The `print(data.decode('utf-8'))` line during the test will slow the results down considerably.  I've left it in the code above for anyone recreating the test to confirm the send/receive of each value, but when performing the test for the results below, I've removed that line.

We also need to introduce circuitpython a bit here.  For the bootloader, I used the Pico [8.2.6](https://downloads.circuitpython.org/bin/wiznet_w5500_evb_pico/en_US/adafruit-circuitpython-wiznet_w5500_evb_pico-en_US-8.2.6.uf2) build, along with the Wiznet library found in the [adafruit-circuitpython-bundle-8.x-mpy-20230920](https://github.com/adafruit/Adafruit_CircuitPython_Bundle/releases/download/20230920/adafruit-circuitpython-bundle-8.x-mpy-20230920.zip) library collection.  Working off one of the [examples](https://github.com/Wiznet/RP2040-HAT-CircuitPython/blob/master/examples/Loopback/W5x00_Loopback.py) found from Wiznet (as the official Circuitpython examples didn't have working board configurations), I created the following:

[w5500_speed_client_circuitpython](https://github.com/sjhennion/gh_pages_materials/blob/main/w5500-speed/w5500_speed_client_circuitpython.py)

{% highlight ruby%}
import board
import busio
import digitalio
import time
from adafruit_wiznet5k.adafruit_wiznet5k import WIZNET5K
import adafruit_wiznet5k.adafruit_wiznet5k_socket as socket
from adafruit_ticks import ticks_ms, ticks_diff

##SPI0
SPI0_SCK = board.GP18
SPI0_TX = board.GP19
SPI0_RX = board.GP16
SPI0_CSn = board.GP17

##reset
W5x00_RSTn = board.GP20

print("W5500 Speed Test Client")

# Setup your network configuration below
# random MAC, later should change this value on your vendor ID
MY_MAC = (0x00, 0x01, 0x02, 0x03, 0x04, 0x05)
IP_ADDRESS = (192, 168, 1, 20)
SUBNET_MASK = (255, 255, 0, 0)
GATEWAY_ADDRESS = (192, 168, 0, 1)
DNS_SERVER = (8, 8, 8, 8)

led = digitalio.DigitalInOut(board.GP25)
led.direction = digitalio.Direction.OUTPUT

ethernetRst = digitalio.DigitalInOut(W5x00_RSTn)
ethernetRst.direction = digitalio.Direction.OUTPUT

cs = digitalio.DigitalInOut(SPI0_CSn)

spi_bus = busio.SPI(SPI0_SCK, MOSI=SPI0_TX, MISO=SPI0_RX)

# Reset W5500 first
ethernetRst.value = False
time.sleep(1)
ethernetRst.value = True

# Initialize ethernet interface with DHCP
eth = WIZNET5K(spi_bus, cs, is_dhcp=True, mac=MY_MAC, debug=True)

print("Chip Version:", eth.chip)
print("MAC Address:", [hex(i) for i in eth.mac_address])
print("My IP address is:", eth.pretty_ip(eth.ip_address))

# Initialize a socket for our server
socket.set_interface(eth)
client = socket.socket()  # Allocate socket for the server
server_ip = "192.168.1.247"  # IP address of server
server_port = 5000  # Port to listen on
client.connect((server_ip, server_port))
client.send('1')

start_time = ticks_ms()

for x in range(999):
    data = client.recv(16)
    #print(data.decode('utf-8'))
    if data != 'NULL':
        data_int = int(data) + 1
        client.send(str(data_int))
        
end_time = ticks_ms()
print("Total time:", ticks_diff(end_time, start_time))
print("Done")
{% endhighlight %}

We won't get into the details of the differences between standing the ethernet interface between the two underlying python systems, but at a high level this should look pretty familiar.  Initialize the interface, connect to the server, and then perform the test while measuring the time difference.

## Test Results

For each bootloader I ran the test three times.  These were all ran from the same W5500, and against the same machine running the server side.  Here are the raw results of tests:

{% highlight bash %}
Micropython v1.20.0 (2023-04-26) .uf2:
Test 1            | Test 2            | Test 3
Total time: 258.9s| Total time: 259.0s| Total time: 258.8s
~3.9 messages/sec | ~3.9 messages/sec | ~3.9 messages/sec 

Circuitpython v1.20.0 (2023-04-26) .uf2:
Test 1            | Test 2            | Test 3
Total time: 89.8s | Total time: 89.8s | Total time: 89.8s
 ~11 messages/sec |  ~11 messages/sec |  ~11 messages/sec 

Wiznet 1.0.5 (Stable):
Test 1            | Test 2            | Test 3
Total time: 26.3s | Total time: 26.3s | Total time: 26.3s 
 ~37 messages/sec |  ~38 messages/sec |  ~37 messages/sec 

Wiznet 2.0.0 (Prerelease):
Test 1            | Test 2            | Test 3
Total time: 1.377s| Total time: 1.383s| Total time: 1.388s 
~726 messages/sec | ~723 messages/sec | ~720 messages/sec 
{% endhighlight %}

For a more simplified version that gives us averages of:

{% highlight bash %}
1. Wiznet 2.0.0  - 723 messages/sec
2. Wiznet 1.0.5  -  37 messages/sec
3. Circuitpython -  11 messages/sec
4. Micropython   -   4 messages/sec
{% endhighlight %}

So, obviously there is an *enormous* range here, and I'm honestly baffled out how this is the case.  I haven't gone through the process of building the Wiznet bootloaders myself, but there's nothing glaringly obvious about the patches it applies to the micropython build that could cause such a difference.  I must be missing something about what it uses for its underlying socket connections (the same, I presumed), or the SPI setup/mode (what I suspect is a more likely culprit).

What surprises me even more, is that I had a hard time finding any other evidence online.  While the loopback example included a W5500 tcp client in the examples on the Wiznet github page, I had to write the Circuitpython version myself, as I found no one else with a premade tcp client example.  From what I've seen, most people seem to be using higher level functionality in these libraries, such as MQTT, HTTP, etc., and perhaps they don't have the expectation of speed that I'm looking for?  The Pico is running a 133MHz dual-core SoC, and the W5500 itself has a 33.3Mhz SPI guaranteed frequency; four messages a second seems like a bug. Some testers, like [javakys here](https://javakys.wordpress.com/2014/10/29/the-second-trial-on-enhancing-the-throughput-of-w5500/), have achieved transfer speeds of 13Mbs, although I'm not sure what the host device was in this case.

There's plenty more to investigate here for anyone interested.

## Wrapping Up

I'm not 100% sure where I will go from here in regards to the official python bootloaders.  Now that I've got a workaround for the Wiznet errors I was seeing before, I'll move forward with my own prototyping with the 2.0.0 prerelease.  It doesn't give me great feelings that the last commit on the repo was [7 months ago](https://github.com/Wiznet/RP2040-HAT-MicroPython/commit/aaffaf486def2012d743aa2a5ce7de702f33ebc4), with the prerelease build even older than that, but what I'm building is entirely self contained, and runs on a private network in a single subnet.  Once I have some redundancy around connection handling built-in, I think it'll serve my purposes fine, as simple and fast data transfer on a LAN is all I need. 

If you'd like more info or references here are my [notes and links](https://github.com/sjhennion/gh_pages_materials/blob/main/w5500-speed/notes_and_links.md) for this article.

As always, if you come across any errors, or have suggestions for how better to present anything, please don't hesitate to reach out to me at [ghp@stephanj.com](mailto:ghp@stephanj.com).  Happy hacking!