# wifigen

Shows which Wi-Fi generation (4, 5, 6, 6E or 7) your Linux connection actually
uses. Decodes `iw` link info into readable band, signal quality, PHY rate,
channel width, spatial streams and MCS for RX and TX, and lists what the access
point advertises. Handy for spotting mesh repeaters that downgrade your link.

## Example

```
$ wifigen
=== Link (wlp2s0) ===
Connected to 02:00:00:00:01:01 (on wlp2s0)
	SSID: MyNetwork
	freq: 5975.0
	signal: -64 dBm
	rx bitrate: 1441.3 MBit/s 160MHz HE-MCS 7 HE-NSS 2 HE-GI 0 HE-DCM 0
	tx bitrate: 1729.6 MBit/s 160MHz HE-MCS 8 HE-NSS 2 HE-GI 0 HE-DCM 0

=== Decoded Connection ===
Wi-Fi Version  : Wi-Fi 6E (802.11ax, 6 GHz)
Band           : 6 GHz (5975 MHz)
Signal         : -64 dBm
Signal Quality : Good

                 RX               TX
PHY Rate       : 1441.3 MBit/s    1729.6 MBit/s
Channel Width  : 160MHz           160MHz
Streams (NSS)  : 2                2
MCS            : 7                8

=== Access Point Capabilities ===
Band     Freq      Signal   Offers
2.4 GHz  2412 MHz  -43 dBm  Wi-Fi 4, Wi-Fi 5, Wi-Fi 6
5 GHz    5180 MHz  -51 dBm  Wi-Fi 4, Wi-Fi 5, Wi-Fi 6
6 GHz    5975 MHz  -50 dBm  Wi-Fi 6E  <- connected
Note: Wi-Fi 5 on 2.4 GHz is a vendor extension (256-QAM), not 802.11ac.

=== Other APs named "MyNetwork" (mesh repeaters?) ===
BSSID              Band     Freq      Signal   Offers
02:00:00:00:02:01  2.4 GHz  2437 MHz  -72 dBm  Wi-Fi 4, Wi-Fi 5
06:00:00:00:02:01  5 GHz    5500 MHz  -80 dBm  Wi-Fi 4, Wi-Fi 5

02:00:00:00:03:01  2.4 GHz  2412 MHz  -75 dBm  Wi-Fi 4, Wi-Fi 5
06:00:00:00:03:01  5 GHz    5180 MHz  -78 dBm  Wi-Fi 4, Wi-Fi 5
```

## Requirements

- Linux with `bash`, `iw`, `awk`, `grep`
- NetworkManager (`nmcli`) for scanning without root; otherwise `sudo iw` is
  used when a scan is needed

## Installation

```sh
git clone https://github.com/efasel/wifigen.git
ln -s "$PWD/wifigen/wifigen" ~/bin/wifigen   # any directory on your PATH
```

## Usage

```sh
wifigen              # auto-detects the wireless interface
wifigen wlp2s0       # explicit interface
wifigen --scan       # force a fresh scan for access points (~8 s)
wifigen --help       # show usage
```

Exit code is `0` on success, `1` on an unknown option, or if `iw` is missing,
no interface is found, `iw` fails, or you are not connected.

## How it decodes

| Bitrate tag | Wi-Fi version                                |
|-------------|----------------------------------------------|
| `EHT-MCS`   | Wi-Fi 7 (802.11be)                           |
| `HE-MCS`    | Wi-Fi 6 (802.11ax), Wi-Fi 6E on 6 GHz        |
| `VHT-MCS`   | Wi-Fi 5 (802.11ac)                           |
| `MCS`       | Wi-Fi 4 (802.11n), NSS derived as MCS/8 + 1  |
| (none)      | Legacy (802.11a/b/g)                         |

- **Band** from frequency: < 2500 MHz = 2.4 GHz, < 5925 MHz = 5 GHz, else 6 GHz
- **Channel width** defaults to 20 MHz when `iw` omits it
- **Signal quality**: ≥ -55 dBm Excellent, ≥ -67 Good, ≥ -75 Fair, else Weak
- **AP capabilities** come from the HT/VHT/HE/EHT capability elements in the
  scan cache (see below)
- **Same AP, other bands**: radios with the same SSID whose BSSID matches in
  octets 2-5 (vendors vary either the first or the last octet per band)
- **Other APs**: same SSID, different BSSID family, typically mesh repeaters

## Scan cache

`iw dev <if> scan dump` reads the kernel's (cfg80211) in-memory list of
recently seen access points. Nothing is stored on disk.

- **Filled by** background scans from NetworkManager / wpa_supplicant and by
  beacons from the associated AP.
- **Expires** entries ~30 s after an AP was last seen, except the associated
  AP, which is always kept.
- **No root needed** to read it.
- **Freshness**: capabilities reflect the AP's last beacon or probe response.
  These rarely change, so the cached data is reliable.

Cache entries expire one by one and background scans can be partial, so
wifigen treats the cache as fresh only if it holds at least one other radio
with the connected SSID and none of them was last seen more than 20 s ago.
Otherwise (or with `--scan`) it triggers a fresh scan and says so on stderr.
It uses `nmcli dev wifi list --rescan yes`, which needs no root (polkit) but
takes ~8 s, and falls back to `sudo iw dev <if> scan`. Running wifigen again
within ~20 s uses the fresh cache and is instant. Networks with a single
radio, or a different SSID per band, will rescan on every run.

Inspect it yourself:

```sh
iw dev wlp2s0 scan dump | grep -E '^BSS|SSID:|freq:|last seen'
```

## Notes

- On 6 GHz, access points do not advertise HT/VHT, so only Wi-Fi 6E/7 can show
  on that band.
- Some APs advertise VHT on 2.4 GHz (a vendor extension for 256-QAM), so
  Wi-Fi 5 may appear there.
- A lower negotiated version than the AP offers usually means you are connected
  through a mesh repeater, or your client or band does not support it.
- A single scan can miss radios, so the AP tables may differ between runs:
  - **6 GHz** radios are often missing. Clients may only scan 6 GHz passively
    or after learning about it from 2.4/5 GHz radios, so discovery is less
    reliable, even right next to the AP.
  - **Weak or distant** radios (below about -70 dBm) may drop out of a scan.

  Run `wifigen --scan` again if something you expect is missing.
- The signal in "Decoded Connection" is averaged over data frames; the AP
  table shows the signal of the last beacon. They can differ by 10 dB or more.
- The RX channel width and rate reflect the last received frame. With little
  traffic this is often a 20 MHz control frame, so RX can look much lower than
  TX. Generate traffic (e.g. a download) for representative values.
- A fresh scan briefly takes the radio off-channel; expect a short latency
  blip on the active connection.
- Rates shown are PHY (link) rates, not real throughput.

## License

[MIT](LICENSE)
