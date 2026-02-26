# android-full-hardware-access-without-root `In Termux?`
How to get full access of hardware connected to your android device via USB port? Here is how to:

### Termux native 1st terminal
update everything
```
pkg update && yes | pkg upgrade
```
```
pkg i termux-api git python ninja pkg-config glib qemu-system-aarch64-headless qemu-utils wget libusb xorriso 
```
```
pip install meson
```
```
git clone --depth 1 https://gitlab.freedesktop.org/spice/usbredir.git
cd usbredir
meson setup build
ninja -C build
```
`connect your scanner/printer/hardware`
see listed device.
(install termux-api addon, make sure both termux and the termux-api is signed by either github or foss, we don't want any mismatches)
```
termux-usb -l
```
request access and press `OK`
```
termux-usb -r "/dev/bus/usb/001/002"
```
manually edit the script according to the number `/dev/bus/usb/XXX/YYY`.
```
cd ~
echo '#!/data/data/com.termux/files/usr/bin/bash'                                                               > run_scanner.sh
echo '/data/data/com.termux/files/home/usbredir/build/tools/usbredirect --device 001-002 --as 127.0.0.1:23456' >> run_scanner.sh
chmod +x run_scanner.sh
```
Kepp it running !
```
termux-usb -r -E -e ./run_scanner.sh /dev/bus/usb/001/002
```
download the cloud ubuntu image around 550MB, `(alpine didn't work for some cases where proprietary drivers compiled for glibc and alpine use musl)`
also If you wanna save time you can forge this img in github actions with the libs/software/drivers you want it will save your time in termux you just boot via qemu and start using your hardware.
```
wget https://cloud-images.ubuntu.com/releases/noble/release/ubuntu-24.04-server-cloudimg-arm64.img
```
at least 10GB of storage.
```
qemu-img resize ubuntu-24.04-server-cloudimg-arm64.img 10G
```
(Cloud images do not have passwords. They are built for servers like AWS or Google Cloud. When they boot, they expect a "Cloud Controller" to magically inject a password or an SSH key into them. Since you are booting it manually on a phone, it has no password, meaning it rejects everything and locks you out).

It takes 10 seconds, and it will permanently unlock the VM.
```
echo "instance-id: termux-vm" > meta-data

cat << 'EOF' > user-data
#cloud-config
password: root
chpasswd: { expire: False }
EOF

xorriso -as mkisofs -V CIDATA -J -r -o seed.img user-data meta-data
```

---
### Termux (qemu aarch64) 2nd terminal (painfully slow but works)
```
qemu-system-aarch64 \
  -machine virt \
  -cpu max \
  -m 4G \
  -smp 4 \
  -bios $PREFIX/share/qemu/edk2-aarch64-code.fd \
  -nographic \
  -drive file=ubuntu-24.04-server-cloudimg-arm64.img,if=virtio \
  -drive file=seed.img,format=raw,if=virtio \
  -netdev user,id=net0 -device virtio-net-pci,netdev=net0 \
  -device qemu-xhci,id=xhci \
  -chardev socket,host=127.0.0.1,port=23456,id=c1,server=off \
  -device usb-redir,chardev=c1,id=u1,bus=xhci.0
```
use this user name when asked for login:
```
ubuntu
```
use this passwd when asked for login in username "ubuntu"
```
root
```
varify your device
```
lsusb
```
use run this first otherwise system can see but driver will can't see the printer/scanner/other device
```
sudo su -
```
(skip) In my case i am using a scanner/printer (all in one) hardware so for me it's 
```
apt update
apt upgrade
```
```
apt install hplip
```
```
hp-plugin -i
```

---
## pulling our the scan or data

* 1 In termux (use same file name)
```
nc -l -p 8080 > hpscan001.png
```
* 2 From ubuntu
```
nc 10.0.2.2 8080 < hpscan001.png
```
veryfy in other 3rd terminal or file exploer for scanned image like FX file exploler.

---

### some notes:
* only tested in "HP_LaserJet_Professional_M1136_MFP" it have both printer and scanner, only scanner was tested as there was a proot method to print doc easily (https://github.com/Kuldeep-Dilliwar/android-printer).
* NOTE- Very slow scanning the motor in hardwere moves fast but the data through usb and qemu didn't the blub might stop and run, and might result in choopy scan but the quality were very good.
* It is the only way I found to use scanner in Termux without root. 
* The method might be used to bridge other hardware with using natively compiled usb `usbredirect` then must specify the device type in qemu arguments `-chardev socket,host=127.0.0.1,port=23456,id=c1,server=off ` .
Lack of KVM (Kernel Virtual Machine): Because your phone is not rooted, QEMU cannot access the Android kernel's hypervisor. It is forced to run in TCG (Tiny Code Generator) mode, which is pure software emulation. Even though you are running an ARM64 image on an ARM64 phone, every instruction is being translated in user-space.
---
USB-over-TCP Overhead: USB 2.0/3.0 relies on very precise timing and fast polling. Routing that raw data through a local TCP socket (127.0.0.1:23456), then through QEMU's emulated XHCI controller, and finally into an emulated CPU causes latency spikes. When the scanner's buffer empties faster than the VM can digest the data, the physical motor pauses to wait for the system to catch up.

Note: You cannot easily fix this without root, but knowing why it happens helps set expectations. The resulting image quality remains high (few white line where bulb had stoped) because the data packets themselves arrive intact, just delayed.

---
