---
layout: post
title: Fun with uIP
---

*The repo for this project is located [here](https://github.com/davidday99/tm4c-uip)*.

[uIP](https://github.com/adamdunkels/uip) is an extremely lightweight TCP/IP implementation initially created by Adam Dunkels. 
It is intended for resource constrained 8- and 16-bit processors and embedded systems. Just a few kilobytes of code and a 
few hundred bytes of RAM are required for an RFC compliant stack that supports ICMP, TCP, UDP, and other integral networking protocols. 
Of course, this comes at the cost of the performance you’d get from a general purpose TCP/IP implementation. 
uIP is single-threaded and uses a single buffer for both receiving and transmitting so don’t expect significant throughput.

uIP is now part of Contiki, and the original codebase doesn’t seem to be maintained, but I thought it would be fun 
to try running it on one of the TM4C microcontrollers I have lying around.

## Compiling (for Linux)

So I've cloned the repo and skimmed over the documentation. Unfortunately, there don't seem to be any instructions for compiling. 
There is a director called *unix* which contains some starter code for running uIP on Linux or FreeBSD. 
There's also an *apps* directory containing some example applications using uIP such as a webserver and a telnet server. 

Before I compile this for the MCU, I'll try to get a working executable on my PC, since that seems easier to debug. 
Without anything else to go off of, I'll just start by compiling *unix/main.c*, seeing as it contains our `main`, and go from there.
Before I do that though, I'll modify the file to call `hello_world_init()` instead of `httpd_init()` as it only seems appropriate to 
start with *hello-world*.

```
gcc -o uip.elf unix/main.c
```

And of course, errors ensue.

```
unix/main.c:39:10: fatal error: uip.h: No such file or directory
   39 | #include "uip.h"
      |          ^~~~~~~
compilation terminated.
```

Passing a few extra options to `gcc` resolves this and one other similar issues.

```
gcc -o uip.elf unix/main.c -Iuip -Iunix
```

But now I'm getting the following error:

```
unix/uip-conf.h:149:10: fatal error: webserver.h: No such file or directory
  149 | #include "webserver.h"
      |          ^~~~~~~~~~~~~
compilation terminated.
```

Looking through *unix/uip-conf.h*, I see this is another location where the user application is selected. I'll just update this file
so that it includes *hello-world.h* instead. This also requires another option be passed to `gcc`.

```
gcc -o uip.elf unix/main.c -Iuip -Iunix -Iapps/hello-world
```

Now I'm getting numerous undefined reference errors; progress... I'll just find in which file each symbol is defined and include it in the compilation.

After some time, I've found all the files needed to successfully compile the executable using the following command:

```
gcc -o uip.elf unix/*.c apps/hello-world/hello-world.c $(find uip -mindepth 1 -name "*.c" -not -name "uip-split.c" -printf "%p ") -Iapps/hello-world/ -Iuip -Iunix
```

Essentially, it's all the source files in `unix/`, `apps/hello-world`, and `uip`, EXCEPT for *uip-split.c*, which expects a user-defined function `tcpip_output`. 
But this purpose of this file is, according to the documentation, "for splitting outbound TCP segments in two to avoid the delayed ACK throughput degradation". 
The code compiles without it, and I don't feel like figuring out how to implement `tcpip_out`, so I'm just going to leave this file out. 
Maybe it'll come back to bite me but if things don't work as expected this will be the first place I check.

But now I have an executable! Running it without permissions gives an error, but it's because part of the code in *unix* creates a tap device that serves as the NIC for the protocol stack (see *tapdev.c*).
Sufficiently confident that Adam Dunkels isn't trying to hack my PC, I'm going to run the executable as root. That works, and the executable is running. 
But aside from a few log messages it prints to `stdout`, what does it do...?

A peek into *hello-world.c* indicates that it's listening for connections on port 1000 and will send a message to any connections it makes.

To connect to port 1000 I'll use a simple Python script:

```
import socket

HOST = "192.168.0.2"
PORT = 1000  

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect((HOST, PORT))
    data = s.recv(1024)

print(f'Received {data}')

```

The IP address comes from the initialization code in `main`:

```
  uip_init();

  uip_ipaddr(ipaddr, 192,168,0,2);
  uip_sethostaddr(ipaddr);
  uip_ipaddr(ipaddr, 192,168,0,1);
  uip_setdraddr(ipaddr);
  uip_ipaddr(ipaddr, 255,255,255,0);
  uip_setnetmask(ipaddr);
```

where `192.168.0.2` is the server address and `192.168.0.1` is the default router.

I run the Python script, and what do you know!

```
$ python client.py                      
Received b'Hello. What is your name?\n'
```

Not only that, but the server is pingable!

```
$ ping 192.168.0.2 
PING 192.168.0.2 (192.168.0.2) 56(84) bytes of data.
64 bytes from 192.168.0.2: icmp_seq=1 ttl=64 time=0.192 ms
64 bytes from 192.168.0.2: icmp_seq=2 ttl=64 time=0.160 ms
64 bytes from 192.168.0.2: icmp_seq=3 ttl=64 time=0.157 ms
^C
--- 192.168.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2016ms
rtt min/avg/max/mdev = 0.157/0.169/0.192/0.015 ms
```

This looks good enough to try on the MCU.

## Compiling for the TM4C

I'm going to use a [template repo](https://github.com/davidday99/tm4c-bare-metal-template.git) I have for the TM4C and modify it to include the uIP sources. 
I'll start by copying over the source and header files and updating the Makefile. Before I make any calls to uIP I want to get this project to build. But compiling yields
the following error:

```
In file included from uip/timer.c:48:
inc/uip/clock.h:55:10: fatal error: clock-arch.h: No such file or directory
   55 | #include "clock-arch.h"
      |          ^~~~~~~~~~~~~~
compilation terminated.
```

One of the functions uIP expects us to define is a reference clock required to drive TCP mechanisms like delayed acknowledgments and retransmissions. 
One implementation is included in the start file *unix/clock-arch.c/*, so I'm just going to copy that over to this project. 
The Unix implementation looks like this:

```
clock_time_t
clock_time(void)
{
  struct timeval tv;
  struct timezone tz;

  gettimeofday(&tv, &tz);

  return tv.tv_sec * 1000 + tv.tv_usec / 1000;
}
```

In the embedded context this would be implemented with a timer, but right now I'm just going to make a dummy implementation that returns 0. If the program doesn't run as expected this will be yet 
another spot to look into later...

Attempting to compile again, I now get a plethora of undefined references for functions like `strlen`, `memcpy`, and `printf`. Time to implement those:

```
void memset(void *dst, char c, int n) {
    while (n--)
      *((char *) dst++) = c; 
}

void memcpy(void *dst, const void *src, int n) {
    while (n--)
      *((char *) dst++) = *((char *) src++);
}

int strlen(char *s) {
    int n = 0;
    while (*s++)
        n++;
    return n;
}

char *strncpy(char *dst, char *src, int sz) {
    while (*src && sz--)
      *((char *) dst++) = *((char *) src++);
    return dst;
}

int printf() {
    return 0;
}
```

With these implemented, most of those undefined reference errors are gone, but there's still another missing reference for `uip_log`, which is used throughout the uIP code. After adding another dummy
implementation for it in *main.c*, the project compiles! Now it's a matter of setting up our main loop to repeatedly handle incoming and outgoing packets and--oh yeah. 
The MCU needs some way of communicating with the network.

## The Network Interface 

To connect to my LAN, I'm going to use a breakout [ENC28J60](https://www.microchip.com/en-us/product/ENC28J60) ethernet controller from Microchip that I have on hand. 
It supports 10Base-T speeds (more than fast enough for what uIP can handle) and has a SPI interface for communicating with a host. Writing the driver for the device 
would probably be the most time-consuming part of this project. 

Luckily, I have one that I wrote a few years ago. It has scattered functionality, including blocking functions for reading and writing ethernet frames. 
The blocking could pose an issue on a multi-threaded system, but since this program is going to be one big main loop processing packets it shouldn't be much of an issue, 
especially considering that uIP only has one buffer for both receiving and transmitting. The SPI commands are implemented using TI's *driverlib* library for the TM4C. 
The driver code is contained in *enc28j60.c*, but I'm going to hide away its ugliness and create a simpler interface similar to the one implemented in *unix/tapdev.c* of the original uIP codebase.

```
#include <stdint.h>
#include "nic.h"
#include "enc28j60.h"
#include "uip_arp.h"

#define BUF ((struct uip_eth_hdr *)&uip_buf[0])
#define ETH_SENDER_MAC_ADDR_OFFSET 6
#define ARP_SENDER_HW_ADDR_OFFSET 22

static uint8_t mac[6];

static struct ENC28J60 *pENC = &ENC28J60;

int nic_init(void) {
    if (!ENC28J60_init(pENC))
        return 0;
    ENC28J60_get_mac_address(pENC, mac);
    ENC28J60_enable_receive(pENC);
    return 0;
}

int nic_read(uint8_t *buf) {
    int size = 0;
    if (ENC28J60_get_packet_count(pENC) > 0) {
        size = ENC28J60_read_frame_blocking(pENC, buf);
        ENC28J60_decrement_packet_count(pENC);
    }
    return size;
}

void nic_write(uint8_t *buf, int size) {
    /* uIP expects the NIC to insert the MAC, so we
     * always insert it in the sender field of the ethernet frame,
     * and if the buffer contains an ARP packet we insert it 
     * into the sender HW address field too.
     */
    memcpy(buf + ETH_SENDER_MAC_ADDR_OFFSET, mac, sizeof(mac));
    if (BUF->type == htons(UIP_ETHTYPE_ARP))
        memcpy(buf + ARP_SENDER_HW_ADDR_OFFSET, mac, sizeof(mac));
    ENC28J60_write_frame_blocking(pENC, buf, size);
}

```

`nic_read` and `nic_write` will allow the MCU to send and receive data on the network.  

## Main Loop
 
The last thing to do is to write the main loop for processing packets. I essentially copied over the `main` function from *unix/main.c* and replaced all the calls to 
`tapdev_read` and `tapdev_send` with `nic_read` and `nic_write`, respectively. I also changed the host address of the MCU to something that made sense on my LAN. 
With that finished, the project compiles and gives me my binary for flashing.

Now for the moment of truth. I wire up the TM4C to the ENC28J60 which then connects to the 50 ft. spare CAT6 cable lying in my room. 
I connect the other end of the cable to a port in the wall, flash the MCU, and test it out.

```
$ python client.py                      
Received b'Hello. What is your name?\n'
```

```
$ ping 192.168.1.150
PING 192.168.1.150 (192.168.1.150) 56(84) bytes of data.
64 bytes from 192.168.1.150: icmp_seq=2 ttl=64 time=4.56 ms
64 bytes from 192.168.1.150: icmp_seq=3 ttl=64 time=4.45 ms
64 bytes from 192.168.1.150: icmp_seq=4 ttl=64 time=3.95 ms
64 bytes from 192.168.1.150: icmp_seq=5 ttl=64 time=3.73 ms
^C
--- 192.168.1.150 ping statistics ---
5 packets transmitted, 4 received, 20% packet loss, time 4012ms
rtt min/avg/max/mdev = 3.734/4.173/4.560/0.341 ms
```

Somewhat to my surprise, it works without any issues! 

![Rate my setup](/assets/img/tm4c_enc28j60.jpeg)
*"Rate my setup"*


What's so cool to me about uIP is its elegant simplicity. It doesn't take much effort to port the stack to any system. And all that functionality is 
contained in less than 3000 lines of code. This program also didn't really do anything useful, but the main packet processing loop could be moved into 
its own thread on a multi-threaded system or even replaced altogether by timer and packet-driven interrupts. 
I'm looking forward to trying out a few of the other example projects and adding the stack to some other projects in the future.


---
**Further Reading**:

[uip (micro IP)](https://en.wikipedia.org/wiki/UIP_(micro_IP))

[Contiki](https://en.wikipedia.org/wiki/Contiki)

