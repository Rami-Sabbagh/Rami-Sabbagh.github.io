---
layout: post
title: "Hacking my router settings"
image: images/posts/2019-08-08-Hacking-my-router-settings/post_image.png
categories:
  - Hacking
  - Wifi
  - Wifislax
  - Wireshark

tags:
  - Hacking
  - Wifi
  - Wifislax
  - Wireshark
  - TP-Link
  - Settings
  - Configuration
---

Hello everyone, it's been 2 years since the last blog post, yea, I'm a lazy blogger :P

Much changes has been made into my home's network, my dad took control of all of it.

We have 2 TP-Link routers at home:
- The main one, which connects with ADSL, is named `InGodWeTrust_HA`,
- The slave one, which is connected with the main one via an ethernet cable, is named `InGodWeTrust_Boost`.

What my dad has made was:
- Changed the main router SSID (which was `InGodWeTrust_Main`) to `InGodWeTrust_Main`
- Changed the main router wifi password, and never gave it to anyone else in the family :(
- Enabled bandwidth control, gave the full speed for himself, and half the speed only for the `Boost` router, which we use, and never told anyone about that.
- Made firewall rule for automatically cutting the internet for everyone (including himself) from 11 PM until 4 AM
- Disabled the firewall rule when he wanted to use the internet after 11 PM....

So he basically took full control of the main router, setting unfair rules for the rest of the family, and not telling anyone about the bandwidth one...

Our ADSL speed is 1 Mbps, which is usually 102kbyte/s when stable
And now with his bandwidth rule, all the rust of the family (3 members) gets 50kbytes/s to fight each other in...

It's also that the main router is connnected with a battery, so it stays working when the power is off, but the other router (Boost) is not, so we are also left with no internet when the power is off (About 4 hours everyday).

I once had this conversation with him:
```
Me: Dad, could you please disable the bandwidth control ? the internet is really slow :(
He replied: There is no bandwidth control!, It's a general issue for the whole building, ask the neighboors.
I replied: No, I know you dad, you have set us to only have half the speed.
He replied: I'm no lying, ask the nighboors.
I replied: No you are, please, disable it, at least when you are not here..
He replied: This is the situation, deal with, it's not changing.
```

50kb/s is such a pain, everything is slow, youtube lags even at 144p...

So I decided it's time to wear my black hat...

He leaves his laptop at home, and it's locked with a password all the family knows, so, when he was out, I openned the laptop, logged in, and got the `HA` wifi password ;)

![00-wireless-password.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/00-wireless-password.png)

I'm now in, that's good, I could enjoy having internet when the power is off, but the 50kb/s limit is still there...

I went and started doing some research, I already played with wifislax 3 years ago.

It's a special linux distro with many wifi tools pre-installed, and ready for hacking, easily installable on a usb stick.

I know there's kali linux, but it's 2GB download, and you know, the 50kb/s won't help, I have wifislax already downloaded, it's a 3 years old copy, but those things still work.

## Knowing the router configuration/settings webpage

As most routers, the router settings page is at http://192.168.1.1/, and those settings pages, are accessed throw http, and you know, that's not encrypted at all.

Unlike classic routers, this router gives a page with a fields for username and password

![00-router-login-page.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/00-router-login-page.png)

I downloaded the page, inspected it's content, and found the javascript code for the login button:

```js
function PCSubWin()
{	
	if (isLocked == true)
	{
		return ;
	}
		
	var auth;
	var password = $("pcPassword").value;
	var userName = $("userName").value;
	auth = "Basic "+Base64Encoding(userName+":"+password);
	document.cookie = "Authorization=" + auth;
	window.location.reload();
}
```

It's just a simple plaintext cookie with the username and password base64 encoded (because of UTF-8 characters support).

That should be simple, I only have to sniff the wifi packet containing the cookie,
But it turned out it's not that simple, but still easy, I need to capture the connection handshake so I could decrypt the wifi frames.

## Sniffing the authorization cookie

> Note: In those pictures I'm displaying myself capturing my own mobile sending a fake password, because the original footage was not screenshotted.

After dad has returned to the house, and the clock was 10:30 PM, it's the time to do the wifi work:

### 1. Booting Wifislax
I've install wifislax into my usbstick, and booted it with `CopyToRam` option, so it runs faster, and lets me use the usb for storing files.

### 2. Figuring out the router ESSID and it's channel
`Aircrack-ng` is such an awesome wifi hacking suit, it contains all the lovely tools you want.

By executing the following command, I'll be able to know the router SSID and it's MAC, which I need both for the next steps:
```
airodump-ng wlan0
```

![01-list-networks.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/01-list-networks.png)

> (Press `Control-C` to get back into the prompt)

I've highlighted my router entry, but blured all the second halves of the mac addresses, for privacy reasons ;)

### 3. Killing the network manager
I've read in multiple places, and aircrack itself mentions in the terminal, that the network manager and some other services might cause problems with the sniffing proccess, so it's much better to kill them:

```
airmon-ng check kill
```

![02-kill-manager.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/02-kill-manager.png)

### 4. Enabling monitor mode
Thankfully, my wifi adapted, which is `TP-Link TL-WN722N_V1`, supports monitor mode and packets injection.

```
airmon-ng wlan0 11
```

![03-enable-monitor-mode.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/03-enable-monitor-mode.png)

> `11` Is the wifi network channel number.

### 4. Start the capturing proccess, and find my dad's mobile MAC address
I've mounted the usb stick, and created a folder named `captures`, to store the the wifi frames.

Openned a terminal in that folder, and executed the following command for starting the capture proccess:
```
airodump-ng mon0 -c 11 --bssid C0:4A:00:XX:XX:XX -w HAC05
```

![04-device-found.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/04-device-found.png)

As you can see, the highlighted line is the target device.

### 5. Deauthunticate the target device
Inorder to successfully decrypt the wifi frames into packets, I have to know the wifi password, and to capture the 4 steps handshake.

The device is already connected, so I have to deauthunticate it first, inorder to do that, I openned another terminal and executing the following:
```
aireplay-ng --deauth 10 -a C0:4A:00:XX:XX:XX -c 50:9E:A7:XX:XX:XX mon0
```
The `-a` flag specifies the router ESSID, the `-c` specifies the target device MAC address, `--deauth 10` specifies that it's a dauth attack with 10 packets.

![05-deauth.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/05-deauth.png)

### 6. Checking if handshake was captured
I've closed the second terminal, and return to the first one to check if the handshake was captured, and yes it was :)

![06-handshake-captured.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/06-handshake-captured.png)

### 7. Making dad open the router settings page
It's 11:00 PM now, and my dad is asking me to sleep, yea I would usually sleep, but this time I want him to cut the internet and force me to sleep,

So I opened youtube on my phone, and started watching some youtube videos, he got angry and cutted the internet, yeah !

### 8. Shut down the system
Okay now, the cookie mas have been sniffed, so I terminated the terminal, and shutdown the whole system for the next day,

If you want to continue using it, you could turn off the monitor mode by typing

```
airmon-ng stop mon0
```

![07-disable-monitor-mode.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/07-disable-monitor-mode.png)

And start the network manager back from the start menu entry

![08-start-network-manager.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/08-start-network-manager.png)

## Reading the sniffed cookie packet

The next day, I've booted back into Windows, and pulled the files out of the usb stick.

### 1. Open the .cap file in wireshark

![09-wireshark.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/09-wireshark.png)

### 2. Add in the wifi password

Go `Edit -> Perferences`

![10-preferences.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/10-preferences.png)

Then `Protocols -> IEEE 802.11`, press `Edit...` near the `Decyption keys` text

![11-protocol.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/11-protocol.png)

Add a new entry, set the type into `wpa-pwd`, then fill in the wifi password and SSID in the key feed with this format `password:ssid`

![12-key.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/12-key.png)

Press `Ok` on each windows until you return back to the main wireshark window

### 3. Search for http packets while includes cookies

That could be accomplished by using this filter `http.cookie && ip.dst == 192.168.1.1`

Any packet would work, right click it, then go `Follow -> Follow TCP Stream`

![13-follow-tcp-steam.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/13-follow-tcp-steam.png)

### 4. Find the authorization cookie and decode it

![14-cookie-found.png]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/14-cookie-found.png)

Now take that base64 encoded string and decode it

```
aGVsbG86ZGVhciByZWFkZXIgIQ== -> hello:dear reader !
```



![]({{ site.url }}/images/posts/2019-08-08-Hacking-my-router-settings/)