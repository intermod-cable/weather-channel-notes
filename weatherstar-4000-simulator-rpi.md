# WeatherStar 4000 on a Raspberry Pi with Composite Video
 
A complete guide to running the Python/Pygame WeatherStar 4000 recreation [wesellis/weatherstar-4000](https://github.com/wesellis/weatherstar-4000) on a Raspberry Pi running Raspberry Pi OS Lite, with output to a CRT over composite video. It is based on a real setup session, including the problems that came up and how each was handled.
 
## 1. What you are building
 
A Raspberry Pi with no desktop and no browser that boots straight into a live, 1990s Weather Channel style display. Weather data comes from NOAA/NWS (US only by default), and the picture goes out over composite to a 4:3 CRT. Audio plays smooth jazz through the 3.5 mm jack.
 
Why this project? Most WeatherStar simulators (such as ws4kp) are web apps and need a browser. This one is a native Python app drawn with Pygame, so it can run on a bare console.
 
## 2. Hardware Check
 
The project README targets a Pi 3B+, Pi 4, or Zero 2 W with at least 1 GB RAM. Its own benchmarks:
 
| Platform | FPS | RAM | CPU |
| --- | --- | --- | --- |
| Pi 4 | 45-60 | 200 MB | 15-25% |
| Pi 3B+ | 30-45 | 200 MB | 30-40% |
| Pi Zero 2 W | 20-30 | 180 MB | 50-60% |
 
This guide was tested on a Pi 3 Model B (1 GB), which was first mistaken for a Pi 2 because the 32-bit kernel (`rpt-rpi-v7`) runs on both. Check your own board with `grep Revision /proc/cpuinfo`: `a02082` and `a22082` are Pi 3 Model B. A Pi 3 is a comfortable target, a Pi 2 is borderline since it is slower than a Zero 2 W, and a Pi 1 is well below the stated requirements.

Note that the Pi 2 _does_ work, it just has a much reduced frame rate.
 
You also need: a microSD card, a composite cable (a yellow RCA jack on the Pi 1; a 4-pole TRRS 3.5 mm cable on the Pi 2/3, with the correct pinout), a 4:3 CRT, and a network connection (Ethernet, or Wi-Fi on models that have it).
 
## 3. Flash the OS with SSH enabled
 
Use Raspberry Pi Imager and choose Raspberry Pi OS Lite (32-bit). Open the OS customisation settings and set:
 
- hostname
- username and password
- Wi-Fi details, if needed
- Enable SSH (Services tab)
Manual alternative: after flashing, on the boot partition create an empty file named `ssh` and a `userconf.txt` containing `username:hashed-password` (generate the hash with `openssl passwd -6`).
 
Boot the Pi and connect: `ssh username@hostname.local`.
 
## 4. Composite Output
 
Older Pis fall back to composite automatically when no HDMI display is detected at boot, so you may see a picture with no configuration at all. That is normal. However, it is best to manually set the configuration to use composite.
 
Edit `/boot/firmware/config.txt` (on older images, `/boot/config.txt`):
 
```
dtoverlay=vc4-kms-v3d,composite
```
 
Then append to the single line in `/boot/firmware/cmdline.txt` (keep it all on one line):
 
```
video=Composite-1:720x480@60ie vc4.tv_norm=NTSC video=HDMI-A-1:d
```
 
The `video=HDMI-A-1:d` entry disables the HDMI output, which matters later (see section 9). On the legacy firmware driver, use `sdtv_mode=0` (NTSC, or 2 for PAL) and `sdtv_aspect=1` (4:3) in `config.txt` instead.
 
Symptoms and fixes:
 
- Black and white or rolling picture: PAL/NTSC mismatch.
- Squashed or stretched picture: set the aspect to 4:3.
- Text cut off at the edges: adjust the overscan or margins.
## 5. Install Dependencies
 
```
sudo apt update
sudo apt install -y git python3-venv python3-pip python3-dev python3-pygame python3-pil python3-requests libsdl2-2.0-0 libsdl2-image-2.0-0 libsdl2-ttf-2.0-0 libsdl2-mixer-2.0-0 alsa-utils
sudo usermod -aG video,render,input,audio $USER
```
 
Log out and back in so the group changes apply.
 
## 6. Get the Project
 
Clone into `/opt` rather than your home directory, so the app is not tied to your login. Your user owns the folder for now so you can install and test, and section 12 hands it to a dedicated service account.
 
```
sudo mkdir /opt/weatherstar-4000
sudo chown $USER: /opt/weatherstar-4000
git clone https://github.com/wesellis/weatherstar-4000.git /opt/weatherstar-4000
cd /opt/weatherstar-4000
python3 -m venv --system-site-packages .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install ephem
```
 
`ephem` is optional and improves the astronomy data. The venv uses `--system-site-packages` so it can see the apt-installed pygame if you need it.
 
## 7. First Run
 
Run it from the Pi's own console if you can, with a keyboard attached:
 
```
source .venv/bin/activate
export SDL_VIDEODRIVER=kmsdrm
python run_weatherstar.py
```
 
In the session this guide is based on, the log showed automatic location detection working and then a display error. The following sections cover every problem that came up, in the order it happened.
 
## 8. Problem: EGL not initialized
 
Symptom:
 
```
ERROR: Could not restore CRTC
pygame.error: EGL not initialized
```
 
Cause: SDL's KMS/DRM driver needs the EGL and GBM libraries, which are not installed on a Lite image by default.
 
Fix:
 
```
sudo apt install -y libgbm1 libegl1 libgles2 libdrm2 libgl1-mesa-dri mesa-utils-bin
sudo usermod -aG video,render,input $USER
```
 
Log out and back in (or reboot) and run again. Check that the display device exists:
 
```
ls /dev/dri
cat /sys/class/drm/*/status
```
 
You should see `card0`. In the tested setup this fixed the display error.
 
If EGL still fails, try the distro's pygame by running `pip uninstall -y pygame` in the venv so the apt version is used.
 
The neon warning (`pygame was not built with support for it`) is only a minor performance note. Ignore it. The `pyttsx3 not installed` message only affects voice narration, which is off by default.
 
## 9. Problem: pageflip errors (-28)
 
Symptom, repeated in the log:
 
```
ERROR: Could not queue pageflip: -28
```
 
`-28` is ENOSPC. In SDL's KMS driver this typically means the flip was queued on an output that cannot take it. In the tested setup `/sys/class/drm` showed two connectors as connected, and SDL uses the first connected one, which may not be the composite output.
 
Diagnosis:
 
```
for f in /sys/class/drm/card0-*; do echo $f: $(cat $f/status); done
```
 
Fix: disable the HDMI connector in `cmdline.txt` with `video=HDMI-A-1:d` (already included in section 4), keep the composite entry, and reboot. If it still fails, try the non-interlaced mode `video=Composite-1:720x480@60e`. You can test the display path outside the app with a small pygame script that fills the screen and flips repeatedly. If the script also errors, the cause is the driver, not WeatherStar.
 
After disabling HDMI the display worked and the errors stopped in the tested setup.
 
## 10. Picture position and the black bar
 
The picture was shifted too far to the left, and it did the same at boot, so the cause was the composite signal timing, not the app.
 
What does not work: KMS margins (`margin_left`, `margin_right` on the `video=` entry) only shrink the active area, so they can move the left edge in but can never push the right edge out. Do not rely on them for a true shift.
 
What worked: make SDL scale and centre the picture. Change the `set_mode` call (line 438 of `run_weatherstar.py`) with this one-liner:
 
```
sed -i.bak 's/pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))/pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT), pygame.SCALED | pygame.FULLSCREEN)/' run_weatherstar.py
grep -n set_mode run_weatherstar.py
```
 
The `-i.bak` option saves a backup (`run_weatherstar.py.bak`). To undo, run `mv run_weatherstar.py.bak run_weatherstar.py`.
 
After this, any margin you added in `cmdline.txt` shows up as a black bar, so remove the `margin_left` and `margin_right` parts and reboot. If the bar remains, use the TV's horizontal position control (some CRTs hide it in a service menu). If the picture is too wide, add small, equal margins on both sides.
 
## 11. Audio
 
On the Pi 2 and 3 the 3.5 mm jack carries both composite video and audio. Run `sudo raspi-config`, go to System Options, then Audio, and select the headphone output. The app has more than 75 smooth jazz tracks that play through pygame's mixer. Volume is in the in-app settings menu.
 
## 12. Start on boot with systemd
 
First create a dedicated service account and give it the app folder. The account has no login shell, and it is in the groups the display and audio need:
 
```
sudo useradd --system --home-dir /opt/weatherstar-4000 --shell /usr/sbin/nologin --groups video,render,input,audio weatherstar
sudo chown -R weatherstar:weatherstar /opt/weatherstar-4000
```
 
Then create the service file. Replace the example coordinates with your own (see section 13):
 
```
sudo tee /etc/systemd/system/weatherstar.service >/dev/null <<'EOF'
[Unit]
Description=WeatherStar 4000
After=network-online.target
Wants=network-online.target
 
[Service]
User=weatherstar
WorkingDirectory=/opt/weatherstar-4000
Environment=SDL_VIDEODRIVER=kmsdrm
ExecStart=/opt/weatherstar-4000/.venv/bin/python run_weatherstar.py --lat 39.9151 --lon -75.3502
Restart=always
RestartSec=5
SupplementaryGroups=video render input audio
 
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now weatherstar
journalctl -u weatherstar -f
```
 
No input piping is needed, because this version detects the location automatically instead of prompting. The `set_mode` patch lives in the script, so the service uses it, and the `cmdline.txt` changes apply at boot.
 
Under systemd there is no logged-in session, so the display can fail to start. If you see a black screen or a DRM/EGL error in the journal:
 
- Add `TTYPath=/dev/tty1`, `StandardInput=tty`, and `TTYReset=yes` under `[Service]`, then run `sudo systemctl disable getty@tty1`.
- Add `After=systemd-user-sessions.service` and `RestartSec=10`.
- Confirm your user is in the `video` and `render` groups with `id`.
Reboot and check that it starts with nothing typed. If not, run `journalctl -u weatherstar -b --no-pager | tail -40` to see why. Note that the service had not yet been tested at the end of the session this guide is based on.
 
## 13. Changing your location
 
The README describes a startup prompt for latitude and longitude, but in the tested version the app went straight to IP-based detection. In the source (`run_weatherstar.py`), line 375 shows that auto-detection runs only if no coordinates are provided, and an `argparse` block near line 1562 means coordinates can be passed on the command line.
 
Check the exact option names:
 
```
cd /opt/weatherstar-4000
source .venv/bin/activate
python run_weatherstar.py --help
```
 
The app takes the coordinates as command-line arguments, `--lat` and `--lon`. Test it by running it by hand with your own coordinates, for example:
 
```
SDL_VIDEODRIVER=kmsdrm python run_weatherstar.py --lat 39.9151 --lon -75.3502
```
 
The startup log should print the coordinates you passed, followed by the matching city and state, and IP-based auto-detection no longer runs. To make it permanent, add the same arguments to the `ExecStart` line of the service file so it reads:
 
```
ExecStart=/opt/weatherstar-4000/.venv/bin/python run_weatherstar.py --lat 39.9151 --lon -75.3502
```
 
Apply it with `sudo systemctl daemon-reload` and `sudo systemctl restart weatherstar`. Replace the example coordinates with your own (latlong.net works). If a negative longitude is ever read as a flag, write it as `--lon=-75.3502`.
 
NWS data covers the US only. For other countries use the International Weather option in the settings menu (right-click or `M`, then `8`). The radar fetch in the log used different coordinates from the detected location, probably a nearby radar site, so check that the radar looks right after you set a location.
 
## 14. Performance tuning
 
The README benchmarks a Pi 3B+ at 30-45 FPS, so a Pi 3B should be close to that. If it feels sluggish, find the cause before changing things:
 
```
vcgencmd get_throttled
vcgencmd measure_clock arm
top -H
```
 
`get_throttled` returning `0x0` means no throttling. Any other value (for example `0x50005`) means undervoltage or thermal throttling, and a better 2.5 A power supply may fix the sluggishness on its own. `top -H` shows which thread is using the CPU during a stutter.
 
Options, in order of likely benefit:
 
1. **Power and cooling.** Fix any throttling reported above first.
2. **News toggles.** Turn off MSN, Reddit, and local news in the settings menu. Fetching and processing run in Python alongside drawing, so stutters often line up with refreshes. Turn off the radar next if needed.
3. **Music.** Set the volume to zero briefly to see how much MP3 decoding costs.
4. **The SCALED patch.** Scaling 640x480 to the output mode costs some CPU. Restore the backup (`mv run_weatherstar.py.bak run_weatherstar.py`) and compare. If it is much faster, fix the picture position with the TV's H-position control instead.
5. **64-bit OS.** The startup warning that pygame was not built with NEON support exists because the 32-bit pip wheel (pygame 2.6.1) was built without NEON blitters. On 64-bit Raspberry Pi OS Lite, NEON is always available, so the standard wheel should include it. This is untested here, and it needs a reflash. Every other step in this guide still applies.
6. **Build pygame with NEON on 32-bit** (untested, and slow to compile on a Pi): install `build-essential` and the SDL2 development packages, then run `pip uninstall -y pygame` followed by `CFLAGS='-mfpu=neon -O3' pip install pygame --no-binary pygame` in the venv.
7. **Mild overclock.** Adding `arm_freq=1300` and `over_voltage=2` to `config.txt` can help, but only with a heatsink and a good supply. Skip it if `get_throttled` shows problems.
8. **Service tweaks.** Add `Nice=-5` under `[Service]` to raise the app's priority, and disable unused services (for example `sudo systemctl disable --now triggerhappy`).
NEON blitters only speed up some pygame operations (alpha blending and scaling), so expect a modest gain rather than a fix on their own.
 
## 15. Controls
 
| Key | Action |
| --- | --- |
| Space | Pause or resume auto-play |
| Left / Right | Navigate displays |
| Right-click or M | Settings menu |
| R (in menu) | Refresh weather data |
| Escape | Exit |
 
## 16. Troubleshooting quick reference
 
| Symptom | Likely cause | Fix |
| --- | --- | --- |
| EGL not initialized | Missing graphics libraries | Install libgbm1, libegl1, libgles2, libdrm2, libgl1-mesa-dri; add user to video/render groups |
| Could not queue pageflip: -28 | SDL using the wrong connector | Disable HDMI with video=HDMI-A-1:d; try a non-interlaced mode |
| Picture shifted left | Composite timing | Use pygame.SCALED, remove margins, then the TV H-position |
| Black bar after scaling | Leftover margin in cmdline.txt | Remove the margin settings |
| Black screen under systemd | Display held by getty or a session issue | Add TTY settings, disable getty on tty1 |
| Choppy video | Hardware limit | Use a Pi 3 or newer |
| Wrong city or radar | IP geolocation is approximate | Pass explicit coordinates |
 
## 17. Final configuration summary
 
- OS: Raspberry Pi OS Lite, 64-bit, SSH enabled at first boot
- `config.txt`: `dtoverlay=vc4-kms-v3d,composite`
- `cmdline.txt`: `video=Composite-1:720x480@60ie vc4.tv_norm=NTSC video=HDMI-A-1:d`
- Packages: libgbm1, libegl1, libgles2, libdrm2, libgl1-mesa-dri, SDL2 libraries, pygame
- Environment: `SDL_VIDEODRIVER=kmsdrm`
- Patch: `pygame.SCALED | pygame.FULLSCREEN` in `set_mode`
- Service: `weatherstar.service` with the `/opt` path
