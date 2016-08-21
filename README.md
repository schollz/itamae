# itamae 0.1.1: 802.11 parser
![](logo/itamae.png?raw=true)

[![License: GPLv3](https://img.shields.io/pypi/l/itamae.svg)](https://github.com/wraith-wireless/itamae/blob/master/LICENSE)
[![PyPI Version](https://img.shields.io/pypi/v/itamae.svg)](https://pypi.python.org/pypi/itamae)
![Supported Python Versions](https://img.shields.io/pypi/pyversions/itamae.svg)
![Software status](https://img.shields.io/pypi/status/itamae.svg)

## 1 DESCRIPTION:
Itamae is a raw (packed binary data) 802.11 parser. Consider the OSI model:
```
 +-------------+
 | Application |
 +-------------+
 | Presentation|
 +-------------+
 | Session     |
 +-------------+     
 | Transport   |  
 +-------------+ 
 | Network     | /+-----------+   
 +-------------+/ | MSDU (LLC)|   
 | Data-Link   |  +-----------+
 +-------------+\ | MPDU (MAC)|
 | Physical    | \+-----------+
 +-------------+ 
```
Layer 2, the Data-Link layer can be subdivided into the MAC Service Data Unit 
(MSDU) or IEEE 802.2 Logical Link Control and the MAC Protocol Data Unit (MPDU).
Itamae is concerned with parsing the MPDU or 802.11 frame and parsing meta-data
about the Layer 1, Physical layer as found in <a href="http://www.radiotap.org">Radiotap</a>. 
ATT, itamae does not support Prism or AVS at Layer 1 and it does not parse 
anything at the LLC sublayer or above. In the future, I plan on extending it 
into the Network layer and include 802.1X parsing.
 
Itamae is not intended to be a substitute for Scapy. Use Scapy if you
 
 * need to parse TCP/IP, 
 * need to craft packets, or
 * need to inject packets.

Itamae is intended to meet a niche set of goals and is ideal if 

 * speed and efficiency is a requirement,
 * you only need 802.11 support, and
 * you are only parsing, not building packets.
 
When parsing raw data, Itamae is six times faster than Scapy and has a reduced 
overhead in terms of object size because it uses minimal classes. However, unlike 
Scapy, Itamae does not offer socket support (you'll have to bind and sniff your 
own sockets) and it is not layered. See Section 3: Using for an explanation.

## 2. INSTALLING:

### a. Requirements
Itamae requires Python 2.7. I have no plans on supported Python 3.x as doing so 
makes the code ugly. If at such a time as Python 3 becomes the defacto standard, 
I will move it to Python 3. Itamae has been tested on Linux but, as of yet, has
not been tested on Windows or MAC OS.

### b. Install from Package Manager
Install itamae through PyPI:

    sudo pip install itamae

### c. Install from Source
Itamae can also be installed from source. Download from http://wraith-wireless.github.io/itamae 
or https://github.com/wraith-wireless/itamae. Once downloaded, extract the files 
and from the itamae directory run:

    sudo python setup.py install

## 3. USING

Before using Itamae, you'll need a wireless card in monitor mode and a raw socket. 
You can use iw or, shameless plug follows, <a href="https://github.com/wraith-wireless/PyRIC">PyRIC</a> 
to create a virtual interface and use the Python socket module to bind the raw 
socket. Before showing Itamae examples, let's set up our card and socket. You'll
need to be root to do so.

```python
>>> import pyric.pyw as pyw
>>> import socket

>>> card = pyw.getcard('wlan0')
>>> pyw.phyadd(card,'mon0','monitor')
>>> sock = socket.socket(socket.AF_PACKET,
...                      socket.SOCK_RAW,
...                      socket.htons(0x0003))
>>> sock.bind((card.dev,0x0003))
```

Before showing how to parse with Itamae, it is best to describe how the 
radiotap and MPDU are handled. Each is a wrapper around a dict that exposes
certain fields using the '.' operator. And for each, the respective parse
function takes a byte stream and returns the appropriate layer dict. Unlike
Scapy and other protocol parsers, Itamae does not parse and/or treat radiotap
and MPDU as a layered hierarchy. That is, the parsed MPDU is not an object 
contained within the radiotap object.

After creating a raw socket ready, we can read the raw bytes and parse the raw
frame with Itamae. Let us start with radiotap. Radiotap exposes three fields, 
the version, the size and the present list. For any other field, you will have 
to use the bracket(s) operator.

```python
>>> import itamae.radiotap as rtap
>>> raw1 = sock.recv(7096)
>>> len(raw1)
171
>>> raw1
"\x00\x00\x12\x00.H\x00\x00\x00$l\t\xc0\x00\xb5\x01\x00\x00\x88A0\x00\x04\xa1Q
\xd0\xdc\x0f\xb04\x95n0\x02\x04\xa1Q\xd0\xdc\x0f\x00<\x00\x00\x07\x08\x00 \n
\x00\x00\x00\xa9\xe6\xfc\x98  T\xe4\xed\xf5\x01w`\xe76\x18@D.'\xaf:;\xa3\xff
\xf2\xb8\x88J\xe8\xeeL\x84\xaf\x08$\x1e\x87\xbc\x8e\xa0\x8e\x86\xd1\xce\xa26
\x84\xa4.\xf5#\xff\xc07`\xd4\xb2\xe4\xaf\n\x01\xcby\x9e4\xb5\xac:0a]\x9d\xfb
\xbf5X\xb3\xc5-f\xca0\xb77~4\xd5\xbf9\x8d\xf3oZ\xcb\xe6>t\xd35\x01\x1c%\x19
\x8cD+\xd6\xc7W\x81\xcb\xd6\x97O.\xde\x07\x11"
>>>
>>> dR1 = rtap.parse(raw1)
>>> dR1
{'sz': 18, 'vers': 0, 'antenna': 1, 'rx_flags': 0, 'antsignal': -75, 'rate': 36, 
'flags': 0, 'channel': [2412, 192], 'present': ['flags', 'rate', 'channel', 
'antsignal', 'antenna', 'rx_flags']}
>>>
>>> dR1.vers
0
>>> dR1.sz
18
>>> dR1.present
['flags', 'rate', 'channel', 'antsignal', 'antenna', 'rx-flags']
```

So far, we have read a raw frame of 171 bytes (which as we will see later is 
a data frame) off our monitor interface and parsed the radiotap "layer". We can 
print the version (0), size, in byrtes, (18) and the list of present fields (flags, 
rate, channel, antsignal, antenna and rx-flags). Not let us continue and look at 
the present fields (see http://www.radiotap.org/defined-fields) for a listing
of all defined fields:
 
 * flags: frame properties. see http://www.radiotap.org/defined-fields/Flags
 * rate: TX/RX data rate (in 500Kbps)
 * channel: a set (list) where the first item is the frequency and the second
  is the channel flags
 * antsignal: received signal power in dBM
 * antenna: the antenna index starting at 0. 
 * rx-flags: received frame properties. see http://www.radiotap.org/defined-fields/RX%20flags
 
In the above frame, the 2nd antenna read the frame (antenna = 1) at -75dBm. The
rate of the signal was 18Mbps (36 * 0.5) and the channel frequency was 2412.
There were no flags or rx-flags defined but there were channel flags defined.
To parse the channel flags we have to take an additional step:

```python
>>> rtap.chflags(dR['channel'][1])
['ism', 'cck']
```  

In this example, the frame is transmitted on 2 GHz ('ism') and it's a CCK 
channel (see http://www.radiotap.org/defined-fields/Channel for a full listing
of all channel flags).

Before moving on to MPDU parsing, let us look at another raw frame.

```python
>>> import itamae.mcs as mcs
>>> raw2
"\x00\x00\x15\x00*H\x08\x00\x00\x00\x9e\t\x80\x04\xc3\x01\x00\x00\x07\x00\x05
\x88\xc1,\x00\x08\x86;C\xf2h\x949\xe5i\xcf\x0b\xff\xff\xff\xff\xff\xff\x90\x1a
\x00\x00\xc0\xff\x00\xc0RM\x00 \x0b\x00\x00\x00!\xd0M$6\x15\x8f5;\xaf\xc1\xee_
\xae\x84 >\xc72\xdaJ-\xb6\xb61\x85+\xa1\xe4\xd1ys\xe9B\xe4\x8b%\xaa\xe0j\xdf\x86
\x04\xe6\x88\x89g\x11\x85\xb4\x0f\xbcI'Df\xcd\xbf\x83\xf5\x10\xec\x1b\xa1FMD\x81
\xcb\xbe\xa9qO\xc3\xb8\xecL\xb6[:\xe2h\xd5\x13\xbd\xdd\x94\xd6\xe2\xa6)\xd8\x9b
\xab"
>>>
>>> dR2 = rtap.parse(raw2)
>>> dR2
{'sz': 21, 'vers': 0, 'antenna': 1, 'rx-flags': 0, 'antsignal': -61, 'flags': 0, 
'present': ['flags', 'channel', 'antsignal', 'antenna', 'rx-flags', 'mcs'], 
'mcs': [7, 0, 5], 'channel': [2462, 1152]}
>>>
>>> rtap.chflags(dR2['channel'][1])
['ism', 'dcck']
>>>
>>> mcsflags = rtap.mcsflags_params(dR2['mcs'][0],dR2['mcs'][1])
>>> mcsflags
{'bw': 0, 'gi': 0}
>>> if mcsflags['bw'] == rtap.MCS_BW_20: bw = '20'
... elif mcsflags['bw'] == rtap.MCS_BW_40: bw = '40'
... elif mcsflags['bw'] == rtap.MCS_BW_20L: bw = '20L'
... else: bw = '20U'
... 
>>> gi = 1 if 'gi' in mcsflags and mcsflags['gi'] > 0 else 0
>>> index = dR2['mcs'][2]
>>> gi, ht, index
(0, 0, 5)
>>> width = int(bw[:2])
>>> rate = mcs.mcs_rate(index,width,gi)
>>> rate
57.8
```

The first thing to note is that there is no rate field present. That is because
we have an 'HT', 802.11n frame as can be seen by the presence of an mcs field.
(Also note, instead of a CCK channel, we have a Dyanamic CCK-OFDM channel). 
To get the rate, we have to do some additional parsing. Note that the mcs field
is a triple 7, 0, 5. These values correspond to known, flags and mcs index 
respectively (http://www.radiotap.org/defined-fields/MCS). Using the mcsflags_param
function with known and flags, returns 'bw' (bandwidth) with a value of 0 and 'gi' 
(guard interval) with a value of 0. The bandwidth of this signal is 20MHz 
(0 = 20, 1 = 40, 2 = 20L and 3 = 20U), and the guard interval is long (800ns).
Passing these along with our mcs index to mcs_rate we get a rate of 57.8Mbps.

Unfortunately, ATT, radiotap does minimal parsing. In most cases, your frames 
will have the same structure as in the examples above. However, if you come
across additional fields, radiotap provides functions for parsing each and
you can review http://www.radiotap.org for help along the way.

But, MPDU does the heavy work for you. Let us return to our orginal frame. 
To parse the MPDU layer, we need to pass the raw frame at the beginning 
of layer two. For that, we need to refer back to the size of the radiotap
frame. Additionally, depending on your interface's firmware the raw frame
may include the MPUD's FCS. If it is included, it will be set in the radiotap's
flags field.

```python
>>> import itamae.mpdu as mpdu
>>> raw1
"\x00\x00\x12\x00.H\x00\x00\x00$l\t\xc0\x00\xb5\x01\x00\x00\x88A0\x00\x04\xa1Q
\xd0\xdc\x0f\xb04\x95n0\x02\x04\xa1Q\xd0\xdc\x0f\x00<\x00\x00\x07\x08\x00 \n
\x00\x00\x00\xa9\xe6\xfc\x98  T\xe4\xed\xf5\x01w`\xe76\x18@D.'\xaf:;\xa3\xff
\xf2\xb8\x88J\xe8\xeeL\x84\xaf\x08$\x1e\x87\xbc\x8e\xa0\x8e\x86\xd1\xce\xa26
\x84\xa4.\xf5#\xff\xc07`\xd4\xb2\xe4\xaf\n\x01\xcby\x9e4\xb5\xac:0a]\x9d\xfb
\xbf5X\xb3\xc5-f\xca0\xb77~4\xd5\xbf9\x8d\xf3oZ\xcb\xe6>t\xd35\x01\x1c%\x19
\x8cD+\xd6\xc7W\x81\xcb\xd6\x97O.\xde\x07\x11"
>>>
>>> hasFCS = rtap.flags_get(dR1['flags'],'fcs')
>>> hasFCS 
0
>>> dM = mpdu.parse(raw1[dR1['sz']:],hasFCS)
>>> dM.present 
['framectrl', 'duration', 'addr1', 'addr2', 'addr3', 
'seqctrl', 'qos', 'l3-crypt']
```

## 4. ARCHITECTURE/HEIRARCHY:
Brief Overview of the project file structure. Directories and/or files annotated
with (-) are not included in pip installs or PyPI downloads

* itamae                  root Distribution directory
  - \_\_init\_\_.py       initialize distrubution module
  - logo (-)              logo directory
    + itamae.png (-)      image for README
  - setup.py              install file
  - setup.cfg             used by setup.py
  - MANIFEST.in           used by setup.py
  - README.md             this file
  - LICENSE               GPLv3 License
  - CHANGES               revision file
  - TODO                  todos for itamae
  - itamae                package directory
    + \_\_init\_\_.py     initialize itamae module
    + radiotap.py         parse radiotap
    + mpdu.py             parse the MPDU of layer 2
    + dot11u.py           constants for 802.11u
    + mcs.py              mcs index, modulation and coding
    + bits.py            bitmask related functions