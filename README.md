# wifigen

Shows which Wi-Fi generation (4, 5, 6, 6E or 7) your Linux connection actually
uses. Decodes `iw` link info into readable band, signal quality, PHY rate,
channel width, spatial streams and MCS for RX and TX, and lists what the access
point advertises. Handy for spotting mesh repeaters that downgrade your link.

## Example

```
$ wifigen
=== Link (wlp2s0) ===
Connected to 02:00:00:00:00:01 (on wlp2s0)
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

=== Access Point Capabilities (6 GHz) ===
Offers         : Wi-Fi 6 (802.11ax)
Note           : 6 GHz has no HT/VHT; capabilities shown are for this band only.
```

## Requirements

- Linux with `bash`, `iw`, `awk`, `grep`
- `sudo` only if the AP is not in the scan cache

## Installation

```sh
git clone https://github.com/efasel/wifigen.git
ln -s "$PWD/wifigen/wifigen" ~/bin/wifigen   # any directory on your PATH
```

## Usage

```sh
wifigen              # auto-detects the wireless interface
wifigen wlp2s0       # explicit interface
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
  scan cache (see below); a fresh `sudo iw scan` is the fallback

## Scan cache

`iw dev <if> scan dump` reads the kernel's (cfg80211) in-memory list of
recently seen access points. Nothing is stored on disk.

- **Filled by** background scans from NetworkManager / wpa_supplicant and by
  beacons from the associated AP.
- **Expires** entries ~30 s after an AP was last seen, except the associated
  AP, which is always kept.
- **No root needed** to read it; triggering a fresh scan needs `CAP_NET_ADMIN`,
  hence the `sudo` fallback (rarely used, since the associated AP is kept).
- **Freshness**: capabilities reflect the AP's last beacon or probe response.
  These rarely change, so the cached data is reliable.

Inspect it yourself:

```sh
iw dev wlp2s0 scan dump | grep -E '^BSS|SSID:|freq:|last seen'
```

## Notes

- On 6 GHz, access points do not advertise HT/VHT, so only Wi-Fi 6/7 can show.
  The AP may still offer Wi-Fi 4/5 on its 2.4/5 GHz radios.
- A lower negotiated version than the AP offers usually means you are connected
  through a mesh repeater, or your client or band does not support it.
- Rates shown are PHY (link) rates, not real throughput.

## License

[MIT](LICENSE)
