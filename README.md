# 🎧 DIY Sonos Port (Turntable → Sonos via Raspberry Pi)

**Goal:**

Play a USB turntable through Sonos (like a Sonos Arc) without buying a Sonos Port

## 🧰 What you need

- Raspberry Pi (Pi 4 or Pi 5 recommended)
  - Monitor
  - Keyboard + Mouse
  - Internet Connection
- USB turntable (e.g. Audio-Technica AT-LP120XBT-USB)
- Ethernet or solid WiFi
- Sonos speaker with AirPlay (Arc, One, etc.)

## 🧠 Overview (what we’re building)

Install pv

```bash
sudo apt update
sudo apt install pv -y
```

`Turntable → USB → Raspberry Pi → OwnTone → AirPlay → Sonos`

## ⚙️ Step 1 - Install Prequisites

## ⚙️ Step 2 - Install OwnTone

Add Repository Key

```bash
# Add OwnTone repository key
wget -q -O - https://raw.githubusercontent.com/owntone/owntone-apt/refs/heads/master/repo/rpi/owntone.gpg | sudo gpg --dearmor --output /usr/share/keyrings/owntone-archive-keyring.gpg

# Add the repository matching your distribution (substitute DIST with either trixie, bookworm, bullseye or buster)
sudo wget -q -O /etc/apt/sources.list.d/owntone.list https://raw.githubusercontent.com/owntone/owntone-apt/refs/heads/master/repo/rpi/owntone-DIST.list


# Install
sudo apt update
sudo apt install owntone -y
```

Then:

```bSH
sudo service owntone restart
```

And logs:

```bash
tail -f /var/log/owntone.log
```

## ⚙️ Step 2 - Configure OwnTone (use ALSA, not PulseAudio)

Edit config:

```bash
sudo nano /etc/owntone.conf
```

Find the audio section and set:

```conf
audio {
type = "alsa"
}
```

Restart:

```bash
sudo systemctl restart owntone
```

## ⚙️ Step 4 - Create input pipe

```bash
sudo mkdir -p /srv/music
sudo mkfifo /srv/music/turntable.pcm
sudo chmod 666 /srv/music/turntable.pcm
sudo chmod 777 /srv/music
```

## ⚙️ Step 5 - Tell OwnTone about the pipe

Edit config again:

```bash
sudo nano /etc/owntone.conf
```

Find:

```conf
library {
```

Add:

```conf
    directories = { "/srv/music" }
```

Restart:

```bash
sudo systemctl restart owntone
```

## ⚙️ Step 6 - Find your turntable device

Plug in your turntable, then:

```bash
arecord -l
```

You’ll see something like:

```bash
card 2: USB AUDIO CODEC, device 0
```

👉 Your device = `hw:2,0`

## ⚙️ Step 7 - Install required tools

```bash
sudo apt install ffmpeg pv -y
```

## ⚙️ Step 8 - Test audio (IMPORTANT)

Run:

```bash
ffmpeg -f alsa -thread_queue_size 4096 -i hw:2,0 \
  -ac 2 -ar 44100 -f s16le - \
  | pv -qL 176400 \
  > /srv/music/turntable.pcm
```

Then:

1. Open OwnTone in browser: `http://<your-pi-ip>:3689`
2. Select your Sonos speaker
3. Play turntable.pcm
4. Drop the needle 🎶

👉 You should hear audio

---

### 🧠 Why this works (important)

- `ffmpeg` = stable audio capture
- `pv` = enforces real-time pacing
- prevents buffer overruns (the #1 failure point)

## ⚙️ Step 9 - Make it automatic (systemd)

Create service:

```bash
sudo nano /etc/systemd/system/turntable.service
```

Paste:

```INI
[Unit]
Description=Stream USB turntable audio into OwnTone pipe
After=owntone.service sound.target
Requires=owntone.service

[Service]
Type=simple
ExecStart=/bin/bash -c '/usr/bin/ffmpeg -f alsa -thread_queue_size 4096 -i hw:2,0 -ac 2 -ar 44100 -f s16le - | /usr/bin/pv -qL 176400 > /srv/music/turntable.pcm'
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable turntable.service
sudo systemctl start turntable.service
```

Check:

```bash
systemctl status turntable.service
```

## 🎯 Final Result

Now your system:

- Starts automatically on boot
- Streams turntable audio continuously
- Appears in OwnTone
- Plays on Sonos via AirPlay

## 💡 Tips

### Name it nicely

Rename in OwnTone:

- “Turntable”
- “Vinyl”

### Set default speaker

Set your Sonos as default output so you don’t have to select it every time.

## ⚠️ Common Issues

### ❌ “overrun!!!” errors

Fix: use `ffmpeg + pv` (don’t use raw `arecord`)

---

### ❌ Service fails with status=127

Fix:

```bash
sudo apt install pv
```

---

### ❌ No audio

- Check `arecord -l`
- Confirm correct `hw:X,0`
- Make sure OwnTone is playing the pipe

---

## 🧠 Why this is better than Sonos Port

- Costs basically nothing
- Uses USB (clean signal path)
- No Bluetooth compression
- Fully customizable

## 💬 Final Thoughts

Most guides stop at:

```bash
arecord > pipe
```

…but that’s unstable.

👉 The real key is:

```bash
ffmpeg + pv = stable real-time streaming
```
