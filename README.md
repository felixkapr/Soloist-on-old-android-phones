# Soloist-on-old-android-phones
# Turn Any Old Android Phone into a Dedicated Spotify Connect Streamer

Got an old Android phone sitting in a drawer? Instead of letting it collect dust, you can turn it into a dedicated, headless **Spotify Connect receiver** for your stereo, soundbar, or powered speakers. 

Most older Android phones still feature a **3.5mm headphone jack** with surprisingly good built-in DACs (Digital-to-Analog Converters). By stripping away Android's heavy background clutter and running Spotify's lightweight command-line client (**Spotify Soloist**) inside a native Linux environment, your phone will boot up in seconds and automatically appear as a speaker in your Spotify app—no screen interaction required.

---

## Table of Contents
- [How It Works (Under the Hood)](#how-it-works-under-the-hood)
- [Prerequisites](#prerequisites)
- [Step 1: Prepare Your Android Device](#step-1-prepare-your-android-device)
- [Step 2: Set Up the Linux Environment (chroot)](#step-2-set-up-the-linux-environment-chroot)
- [Step 3: Install & Configure soloistd (Auto-Updater)](#step-3-install--configure-soloistd-auto-updater)
- [Step 4: Create the Fully Automatic Boot Script](#step-4-create-the-fully-automatic-boot-script)
- [Step 5: First Boot & Audio Tuning](#step-5-first-boot--audio-tuning)
- [Troubleshooting](#troubleshooting)

---

## How It Works (Under the Hood)

* **Spotify Soloist:** Instead of the heavy official Spotify Android app, we use Spotify's official, ultra-lightweight CLI client.
* **Native Linux `chroot`:** Standard Android Linux apps use `PRoot`, which translates system calls and causes severe audio stuttering on older CPUs. By using a rooted **`chroot`**, Debian runs directly on your phone's Linux kernel with **zero performance overhead**.
* **Automatic Updates (`soloistd`):** Official Soloist builds expire every 90 days for security reasons. We include `soloistd`, a small background manager that automatically downloads and installs the latest version whenever an update is available—making the setup completely maintenance-free.

---

## Prerequisites

1. **An old Android phone** (Android 7.0+ recommended, ideally with a 3.5mm audio jack).
2. **Root access** (e.g., via Magisk, KernelSU, or built-in LineageOS root).
3. **A clean Android ROM** (Optional, but a de-googled ROM like LineageOS without GApps makes boot times dramatically faster).
4. **A Spotify Premium account** and a free **Soloist API Key** (`spak_...`) from the [Spotify Developer Dashboard](https://spotify.com).
5. **Two free apps from F-Droid:**
   * **Termux** (Terminal emulator)
   * **Termux:Boot** (Runs scripts automatically when the phone powers on)
   * *Important: Install both apps from the same source (F-Droid) so their security signatures match.*

---

## Step 1: Prepare Your Android Device

Before touching the terminal, optimize Android so it boots fast and doesn't kill background audio:

1. **Enable Airplane Mode, then turn Wi-Fi back on:** This shuts down the cellular modem, saving battery and preventing heat.
2. **Disable the Lock Screen:** Go to **Settings > Security > Screen lock** and set it to **None**.
3. **Register Termux:Boot:** Open the **Termux:Boot** app once from your app drawer. Nothing will visibly happen, which is normal—this just registers the app with Android's boot system.
4. **Disable Battery Optimization:** Go to **Settings > Apps / Battery > Battery optimization** and set both **Termux** and **Termux:Boot** to **Don't optimize**. Otherwise, Android will kill the music stream after a few minutes.

---

## Step 2: Set Up the Linux Environment (chroot)

Open the **Termux** app. Whenever you see a code block below, you can copy the entire block, paste it into Termux, and press **Enter**.

### 1. Install Required Termux Packages
First, update Termux and install `proot-distro` (to download a clean Debian filesystem) and `pulseaudio` (to route Linux audio to Android's headphone jack):

```bash
pkg update && pkg upgrade -y
pkg install -y proot-distro pulseaudio
```

### 2. Download Debian
Download the minimal Debian root filesystem (this will be stored in Termux's internal folder):

```bash
proot-distro install debian
```

### 3. Enter the Native `chroot` Environment
Now we switch to root (`su`), mount Android's hardware interfaces into the Debian folder, fix Android's network restrictions for `apt`, and enter the Debian shell:

```bash
su -c '
DEBIAN_PATH=/data/data/com.termux/files/usr/var/lib/proot-distro/containers/debian/rootfs

mount -t proc /proc $DEBIAN_PATH/proc
mount -t sysfs /sys $DEBIAN_PATH/sys
mount -o bind /dev $DEBIAN_PATH/dev
mount -t devpts devpts $DEBIAN_PATH/dev/pts

echo "nameserver 8.8.8.8" > $DEBIAN_PATH/etc/resolv.conf
echo "APT::Sandbox::User \"root\";" > $DEBIAN_PATH/etc/apt/apt.conf.d/01-android-network

chroot $DEBIAN_PATH /bin/su - root
'
```
*Note: Your terminal prompt should now change to `root@localhost:~#`. You are now inside Debian!*

---

## Step 3: Install & Configure soloistd (Auto-Updater)

While still inside Debian (`root@localhost:~#`), we will compile `soloistd` directly for your phone's CPU architecture (works on both 32-bit ARMv7 and 64-bit ARM64) and link your Spotify API key.

### 1. Install Go and Git
```bash
apt update && apt install -y golang-go git ca-certificates
```

### 2. Download and Compile `soloistd`
*(This step takes 1–2 minutes depending on your phone's processor speed).*
```bash
git clone https://github.com
cd soloistd
go build -o /usr/local/bin/soloistd .
```

### 3. Configure Your Speaker Name & API Key
Replace `YOUR_API_KEY` with your actual `spak_...` key from Spotify, and change `"Living Room"` to whatever name you want to see in your Spotify app:

```bash
SOLOIST_API_KEY='YOUR_API_KEY' soloistd setup --device-name "Living Room"
```
`soloistd` will automatically detect your CPU, download the latest official Soloist binary from Spotify, and save your configuration.

### 4. Clean Up Build Files & Exit Debian
To save storage space on your phone, remove the temporary compiler and source files, then exit back to Termux:

```bash
cd /root && rm -rf soloistd /root/go
apt purge -y golang-go git && apt autoremove -y && apt clean
exit
```
*Your prompt should now be back to Termux (`~ $`).*

---

## Step 4: Create the Fully Automatic Boot Script

Now we create a startup script that **Termux:Boot** will run automatically every time you turn on the phone. 

This script handles several important tricks automatically:
* **Prevents static hiss/noise:** Configures PulseAudio with `--exit-idle-time=60` so the phone's analog audio amplifier powers down when music is paused.
* **Prevents Wi-Fi dropouts:** Disables Wi-Fi power saving (`power_save off`) so skip and volume commands from your main phone arrive instantly.
* **Zero wasted boot time:** Instead of a fixed delay, a fast loop checks every 0.2 seconds if Wi-Fi is connected and launches `soloistd` the exact millisecond an IP address is assigned.

Copy and paste this entire block into **Termux** (`~ $`):

```bash
mkdir -p ~/.termux/boot

cat << 'EOF' > ~/.termux/boot/start-spotify.sh
#!/data/data/com.termux/files/usr/bin/sh

# Keep CPU awake so audio doesn't stutter
termux-wake-lock

# Save logs with timestamps
exec > /data/data/com.termux/files/home/spotify.log 2>&1
echo "$(/data/data/com.termux/files/usr/bin/date "+%Y-%m-%d %H:%M:%S.%3N"): --- STARTING SPOTIFY STREAMER ---"

# 1. Start PulseAudio (disable-shm prevents errors on older Android kernels)
pulseaudio --start --disable-shm --load="module-native-protocol-tcp auth-ip-acl=127.0.0.1 auth-anonymous=1" --exit-idle-time=60
echo "$(/data/data/com.termux/files/usr/bin/date "+%Y-%m-%d %H:%M:%S.%3N"): PULSEAUDIO STARTED"

# 2. Switch to root to configure network and launch Debian chroot
su -c '
iw dev wlan0 set power_save off 2>/dev/null
dumpsys deviceidle disable 2>/dev/null
echo "$(/data/data/com.termux/files/usr/bin/date "+%Y-%m-%d %H:%M:%S.%3N"): WI-FI POWER SAVE DISABLED"

DEBIAN_PATH=/data/data/com.termux/files/usr/var/lib/proot-distro/containers/debian/rootfs

mount -t proc /proc $DEBIAN_PATH/proc 2>/dev/null
mount -t sysfs /sys $DEBIAN_PATH/sys 2>/dev/null
mount -o bind /dev $DEBIAN_PATH/dev 2>/dev/null
mount -t devpts devpts $DEBIAN_PATH/dev/pts 2>/dev/null

echo "nameserver 8.8.8.8" > $DEBIAN_PATH/etc/resolv.conf
echo "$(/data/data/com.termux/files/usr/bin/date "+%Y-%m-%d %H:%M:%S.%3N"): DEBIAN CHROOT MOUNTED"

# Wait up to 30 seconds for Wi-Fi IP address
COUNT=0
while [ $COUNT -lt 150 ]; do
  if ifconfig wlan0 2>/dev/null | grep -q "inet "; then
    echo "$(/data/data/com.termux/files/usr/bin/date "+%Y-%m-%d %H:%M:%S.%3N"): WI-FI CONNECTED"
    break
  fi
  sleep 0.2
  COUNT=$((COUNT + 1))
done

echo "$(/data/data/com.termux/files/usr/bin/date "+%Y-%m-%d %H:%M:%S.%3N"): LAUNCHING SOLOISTD"
chroot $DEBIAN_PATH /bin/bash -c "export PULSE_SERVER=127.0.0.1 && export HOME=/root && /usr/local/bin/soloistd run"
' < /dev/null
EOF

chmod +x ~/.termux/boot/start-spotify.sh
```

### Optional: Add a Quick `log` Shortcut
To easily check if everything is running or see what track is currently playing, add a `log` command to Termux:

```bash
echo 'alias log="tail -n +1 -f ~/spotify.log"' >> ~/.bashrc
source ~/.bashrc
```
Now you can simply type `log` in Termux at any time to watch the live output (press `Ctrl + C` to exit the log view).

---

## Step 5: First Boot & Audio Tuning
1. **Reboot your phone** completely (`su -c reboot`).
2. **Initial Pairing:** Once the phone finishes booting, open Spotify on your main phone or PC, tap the Devices icon, and select "Living Room". Soloist will save your login session permanently—on future reboots, it will appear automatically without needing to pair again.
3. **Maximize Audio Quality:** Turn the physical media volume buttons on your old Android phone all the way up to 100%, and control your listening volume strictly through Spotify or your stereo amplifier. This maximizes the signal-to-noise ratio and eliminates any analog background hiss.

## Troubleshooting

• **Controls (Skip / Volume) stop working when the screen turns off:**
Some older Wi-Fi chips aggressively filter incoming network packets when the screen goes completely into hardware sleep mode. To prevent this while still turning the screen off, install the free open-source app FakeStandby from F-Droid:

	1. Open FakeStandby and grant it the required Display over other apps permission.
	2. In the app's settings, enable Start on boot (and optionally Close on unlock / Start directly) so it automatically blacks out the screen every time the phone powers on.
	3. Make sure to also exclude FakeStandby from Android's Battery optimization so the system doesn't close it in the background.
	
• **Slower boot times:**
You can safely disable unused stock apps (like Camera, Calendar, Calculator, or Sound Recorder) via root using pm disable <package_name> to speed up Android's boot sequence. Avoid disabling core system packages (systemui, settings, captiveportallogin, or trebuchet).
