# Networking

I wish there was a nice way to prove networking works under qemu; unfortunately, bridging internet access to a qemu guest is kind of painful. Instead we'll go through a lot of effort just to ping our host machine (but also get dhcp running).

Busybox ships with a tiny dhcp client, `udhcpc`. When you start udhcpc or anything important happens thereafter, it tries to run a script (`/usr/share/udhcpc/default.script` by default). We need to make that script.

So [here's](scripts/dhcpc.conf) my `/usr/share/udhcpc/default.script`. Copy it there and make sure you `chmod +x` it.

You can test this by just running `udhcpc` or `udhcpc -i eth0` (if you want to be more specific). I get output like this:

````
udhcpc (v1.19.4) started
[   10.679512] e1000: eth0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
Sending discover...
Sending select for 10.0.2.15...
Lease of 10.0.2.15 obtained, lease time 86400
````

And then you can put `udhcpc -i eth0` in your `/etc/start`.

Under qemu, the host machine resides at 10.0.2.2. We can ping it:

````
~ # ping 10.0.2.2
PING 10.0.2.2 (10.0.2.2): 56 data bytes
64 bytes from 10.0.2.2: seq=0 ttl=255 time=36.247 ms
64 bytes from 10.0.2.2: seq=1 ttl=255 time=4.620 ms
64 bytes from 10.0.2.2: seq=2 ttl=255 time=2.622 ms
64 bytes from 10.0.2.2: seq=3 ttl=255 time=2.092 ms
64 bytes from 10.0.2.2: seq=4 ttl=255 time=2.139 ms
64 bytes from 10.0.2.2: seq=5 ttl=255 time=2.130 ms

--- 10.0.2.2 ping statistics ---
6 packets transmitted, 6 packets received, 0% packet loss
round-trip min/avg/max = 2.092/8.308/36.247 ms
````

And if you have a web server (or even just python) on the host machine...

````
# make a little index.html
echo "hello from the qemu host" > index.html
# start python's SimpleHTTPServer
python -m SimpleHTTPServer 80 # or python -m http.server 80 under python 3
````

and, while that's running, you should be able to `wget 10.0.2.2` on the guest.


There are a couple of other things you may want to do:

* Set up /etc/hosts and the loopback device
* Configure a wireless card
* Make sure dns works correctly
* A nice way to set up static ip addresses

But in general these are unecessary and difficult to test under virtualization.


