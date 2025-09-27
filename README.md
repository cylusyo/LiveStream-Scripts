# LiveStream-Scripts
Script for 24/7 Livestream GCP Config

#!/bin/bash

# Ubuntu Streaming Setup Script for GCP
# Optimized for OBS streaming to YouTube
# Based on working configuration with error handling

set -e  # Exit on error

echo "Starting Ubuntu OBS streaming setup..."

(
# set password
echo "Setting user password for RDP access..."
sudo passwd $USER
)

(
# install xrdp service 
echo "Installing XRDP and desktop environment..."
sudo apt update
sudo apt install -y xrdp

# install ubuntu & cinnamon desktop
sudo DEBIAN_FRONTEND=noninteractive \
    apt install -y ubuntu-desktop
sudo DEBIAN_FRONTEND=noninteractive \
    apt install --assume-yes cinnamon-core desktop-base dbus-x11
)

(
# enable and start xrdp service
echo "Configuring XRDP..."
echo "cinnamon" > ~/.Xclients
chmod +x ~/.Xclients
sudo systemctl restart xrdp.service
sudo systemctl enable xrdp.service
)

(
# fix "authentication required" bug - FIXED TYPOS
echo "Fixing authentication issues..."
sudo mkdir -p /etc/polkit-1/localauthority/50-local.d/

cat <<EOF | \
  sudo tee /etc/polkit-1/localauthority/50-local.d/xrdp-NetworkManager.pkla
[NetworkManager]
Identity=unix-group:sudo
Action=org.freedesktop.NetworkManager.network-control
ResultAny=yes
ResultInactive=yes
ResultActive=yes
EOF

cat <<EOF | \
  sudo tee /etc/polkit-1/localauthority/50-local.d/xrdp-packagekit.pkla
[PackageKit]
Identity=unix-group:sudo
Action=org.freedesktop.packagekit.system-sources-refresh
ResultAny=yes
ResultInactive=auth_admin
ResultActive=yes
EOF

sudo systemctl restart polkit 
)

(
# install audio system and streaming software
echo "Installing audio system and OBS..."
sudo apt install -y pulseaudio pavucontrol

# install obs via PPA for better performance
sudo add-apt-repository -y ppa:obsproject/obs-studio
sudo apt update
sudo apt install -y obs-studio

# install chrome
echo "Installing Chrome..."
cd /tmp
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y

# install other tools
sudo apt install -y actiona ffmpeg
rm -f google-chrome-stable_current_amd64.deb
)

(
# Create a 4GB swapfile
echo "Creating swap file..."
if [ ! -f /swapfile ]; then
    sudo fallocate -l 4G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    sudo cp /etc/fstab /etc/fstab.bak
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
else
    echo "Swapfile already exists, skipping..."
fi
)

(
# install codecs
echo "Installing media codecs..."
sudo DEBIAN_FRONTEND=noninteractive \
    apt install -y ubuntu-restricted-extras
)

(
# set rdp session to use cinnamon
echo "Configuring desktop session..."
echo "cinnamon-session" > ~/.xsession 
D=/usr/share/gnome:/usr/share/cinnamon:/usr/local/share:/usr/share
D=${D}:/var/lib/snapd/desktop 
C=/etc/xdg/xdg-cinnamon:/etc/xdg

cat <<EOF > ~/.xsessionrc
export XDG_SESSION_DESKTOP=cinnamon
export XDG_DATA_DIRS=${D}
export XDG_CONFIG_DIRS=${C}
export CINNAMON_2D=1
dconf write /org/cinnamon/desktop/interface/icon-theme "'Humanity'"
dconf write /org/cinnamon/desktop/session/idle-delay "uint32 0"
dconf write /org/cinnamon/settings-daemon/plugins/power/sleep-display-ac 0
EOF
)

(
# Create basic OBS configuration
echo "Setting up OBS configuration..."
mkdir -p ~/.config/obs-studio/basic/profiles/Streaming/

cat <<EOF > ~/.config/obs-studio/basic/profiles/Streaming/basic.ini
[Output]
Mode=Simple

[SimpleOutput]
VBitrate=2500
StreamEncoder=x264
ABitrate=160
UseAdvanced=false
Preset=fast
RecQuality=Stream
RecEncoder=x264

[Video]
BaseCX=1920
BaseCY=1080
OutputCX=1920
OutputCY=1080
FPSType=0
FPSCommon=30

[Audio]
SampleRate=44100
ChannelSetup=Stereo
EOF

echo "OBS streaming profile created with optimized settings"
)

(
# Configure firewall (safely)
echo "Configuring firewall..."
# Check if ufw is active to avoid breaking existing setup
if sudo ufw status | grep -q "Status: active"; then
    echo "UFW is active, adding RDP rule..."
    sudo ufw allow 3389/tcp
else
    echo "Setting up UFW with SSH and RDP access..."
    sudo ufw allow 22/tcp    # SSH
    sudo ufw allow 3389/tcp  # RDP
    sudo ufw --force enable
fi
)

echo "Setup complete! Rebooting in 10 seconds..."
echo "After reboot:"
echo "1. Connect via RDP to manage your streams"
echo "2. OBS is configured with CPU encoding (x264) on 'fast' preset"  
echo "3. Default stream settings: 1080p30, 2500kbps bitrate"
echo "4. Use pavucontrol for audio management"
echo "5. External IP: $(curl -s ifconfig.me)"

sleep 10
sudo reboot
