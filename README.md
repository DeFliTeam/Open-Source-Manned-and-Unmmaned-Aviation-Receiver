# Open-Source-Manned-and-Unmmaned-Aviation-Receiver
Full hardware and software specifications / installation for a device that tracks aircraft and drones using a variety of Rf techniques 

## The following guide explains how you can build a device that will track aircraft using ADS-B, ACARS packets using L-Band, DJI Specific packets, RemoteID (WiFi + BLE) & Passive radar. 

# Notes 

### If you just wish to track drones you can omit the passive radar section, note that this will also remove the ability to track manned aircraft on ADS-B and L-Band. 
### This is a hobbyist project, this device cannot be used to provide service to the DeFli Network. 

## Hardware Requirements 

1 x Jetson Nano Development Board 
1 x Intel 8265 Network Card 
1 x Panda Wireless PAU05 
2 x Dual-Band Antennas	2.4 GHz/5 GHz	Omnidirectional;Gain at 2.4 GHz: 5 dBi; Gain at 5 GHz: 9 dBi
2 x Low-Noise Amplifier	0.5–8 GHz	Low noise: 1.4 dB @ 2 GHz; High IP3: +34 dBm; Gain flatness: ±0.9 dB over 0.5 to 7 GHz @6V 
1 x Cavity Bandpass Filter	2.4 GHz	Center frequency: 2437 MHz; Bandwidth: 25 MHz; Maximum insertion loss: 1.5 dBm 
1 x Bandpass Filter	2.4–2.5 GHz	Bandwidth: 400 MHz; Insertion Loss: 1 dB; Power capacity: 5 W; Voltage standing wave ratio (VSWR): 1.50:1 
1 x Bandpass Filter	5.725–5.875 GHz	Bandwidth: 150 MHz; Maximum insertion loss: 1 dB; Maximum power: 5 W 
1 x Power Splitter/Combiner	350–6000 MHz	Power capacity (as splitter): 25 W; Insertion loss: 0.9 dB 


### For Passive Radar, ADSB and L-Band 

1 x HackRF 
1 x ADS-B 1090 Mhz Antenna 
1 x L-Band Antenna 
1 x L-Band LNA (Optional)

## Hardware Setup 

Please refer to the below diagram for how to set up your hardware  

<a href="https://ibb.co/ZM0BLP5"><img src="https://i.ibb.co/QK4PbRL/Untitled-design-9.png" alt="Untitled-design-9" border="0"></a>

## The Rf Theory 

Wi-Fi signals can be strongly attenuated with distance and the presence of obstacles. To ensure drone detection from significant distances, the detection system should detect weak Wi-Fi signals. An RF amplification chain is then required. The implemented RF architecture is tailored to the system’s specifications. Indeed, the RemoteID uses a 10 MHz Wi-Fi channel fixed at 2437 MHz, while the DJI Drone ID uses a 5 MHz bandwidth frequency channel, which is on an unknown hopping channel in the 2.4 GHz or 5.8 GHz frequency bands. The three amplification stages account for these frequencies—the 2437 MHz channel, the full 2.4 GHz, and 5.8 GHz bands. The 2.4 GHz stage is divided into two parts by a two-way RF splitter, whereas the 5.8 GHz stage is transmitted without splitting, as illustrated in the above image. A Wi-Fi channel 6 cavity filter is included to capture only RemoteID packets. In addition, the remaining architecture is used to receive other communication packets, including the DJI Drone ID. Low-noise amplifiers (LNAs) are employed to amplify the RF signals received by the antennas, maximizing the signal-to-noise ratio (SNR). Following amplification, the signal is routed to a filter that eliminates non-useful frequencies according to the channels, making it ready for processing. The ADS-B captures traffic on the 1090 Mhz channel whilst also acting as receiver / illuminator for passive radar, the L-Band collects data at 1537Mhz to 1550Mhz for ACARS packets whilst laso acting as illuminator and receiver for the Passive Rf. 

## Software Requirements 

The independent software packages that you will require are 

Jaero 
Aircrack-ng (inc airodump-ng)
Wireshark 
Kismet  
SDRPlay 
Tar1090

Rather than install all these packages separately, we advise downloading DragonOS (https://cemaxecuter.com/) to an SD card and booting the Jetson Nano from the SD Card. DragonOS carries these packages as default (https://sourceforge.net/projects/dragonos-focal/)

Once in DragonOS, open up a terminal window (this is a Linux terminal). All packages can be found in  /usr/src 

### ADSB 

To install ADSB please open the "Flightview_Gui" package and follow the installation instructions 

### LBand 

To install L-Band please open the "Jaero" package. Once open please choose the frequencies listed at the bottom of this page (https://thebaldgeek.github.io/L-Band.html) 

### RemoteID 

The first thing we need to do is put our Panda and Intel Network cards in to monitor mode, this is achieved using aircrack-ng with the following command 

```
sudo airmon-ng start $<name_wireless_card>
```
Next enter the wireshark package and enter the following commands from the command line

```
wlan.fc.type_subtype == 0x08
```
We then filter using this command 

```
wlan.bssid[0:3] == 60:60:1F
```

To check for the presence of RemoteID packets check whether the OUI tag “6a:5c:35” is present in the decoded packet

<a href="https://ibb.co/zJFyH8S"><img src="https://i.ibb.co/5FkNxY5/Untitled-design-10.png" alt="Untitled-design-10" border="0"></a>

### DJI Packets 

For this we will use Kismet and the Kismet REST_API. 

Enter the Kismet directory and run the following bash command from the terminal 

```
kismet -c wlan0man:name=channel10, ht_channels=false, vht_channels=,
channels=\"1W5, 2W5, 3W5, 4W5, 5W5, 6W5, 7W5, 8W5, 9W5, 10W5, 11W5, 12W5, 13W5, 14W5, 140W5, 149W5, 153W5, 157W5, 161W5, 165W5, 169W5, 173W5, 177W5\"
```

We can then use the Kismet REST_API to retrieve the DJI Drone ID packet data from the Kismet server in the form of a JavaScript object notation (JSON) file. 

```bash
def per_device(d):
    print d['kismet.device.base.macaddr'],
    print d['dot11.device']['dot11.device.last_beaconed_ssid'],
    print d['uav.device']['uav.serialnumber'],
    print d['uav.device']['uav.last_telemetry']['uav.telemetry.location']['kismet.common.location.lat'],
    print d['uav.device']['uav.last_telemetry']['uav.telemetry.location']['kismet.common.location.lon']

...

kr = KismetRest.KismetConnector(uri)

regex = [
    [ "uav.device/uav.serialnumber", ".+" ]
]

kr.smart_device_list(callback = per_device, regex = regex)
```

### Passive Radar 

For passive radar we advocate using Blah2's system, the installation instructions are as follows 

```
sudo docker pull ghcr.io/30hours/blah2:latest
vim docker-compose.yml
--- build: .
+++ image: ghcr.io/30hours/blah2:latest
sudo docker compose up -d
```

You then need to enter the config files

```
cd /config
sudo nano config-hackrf.yml
```
You will need to change all the parameters that apply to you. The adsb address is the one created by the ADSB installation.




