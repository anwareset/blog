# Plug and Play USB Devices dengan Udev Rules


Perangkat USB yang terpasang ke komputer dengan sistem operasi GNU/Linux akan dimonitor oleh daemon dari Udev. Saya baru saja membeli sebuah USB Mouse _hybrid_ yaitu Rexus Arka RX107 yang mendukung _dual-mode_ di mana kita dapat menggunakannya secara nirkabel maupun berkabel. Dalam kasus ini mouse dapat terdeteksi dan digunakan secara langsung, namun ada beberapa hal yang saya butuhkan, seperti mengatur beberapa _properties_ pada mouse. Yang saya maksud misalnya akselerasi kursor, kecepatan _double-click_, hingga kecepatan _scroll_. Konfigurasi semacam itu terkadang perlu kita _adjust_ secara mandiri dan tidak dapat serta merta dilakukan begitu saja dari System Settings (contohnya `xfce4-mouse-settings`) di DE (Desktop Environment) XFCE. Terlebih saya tidak menggunakan DE, hanya Window Manager saja.

<!--more-->

## Xinput
Biasanya saya menggunakan `xinput` untuk melakukan konfigurasi pada properties milik mouse.

### Listing
Kita dapat melihat perangkat apa saja yang terhubung seperti berikut.

```shell
xinput list
```
Dan di sini saya mempunyai beberapa perangkat input, kita akan berfokus pada perangkat yang namanya mengandung kata _Wireless_ dan _pointer_ karena kita membahas tentang akselerasi mouse.
```
⎡ Virtual core pointer                    	id=2	[master pointer  (3)]
⎜   ↳ LXDDZ 2.4G Wireless Mouse               	id=19	[slave  pointer  (2)]
⎜   ↳ LXDDZ 2.4G Wireless Mouse Consumer Control	id=12	[slave  pointer  (2)]
...
```

### Properties
Untuk melihat properties apa saja yang mungkin dapat kita _set_, jalankan perintah berikut.
```shell
xinput list-props 19
```
Angka **19** adalah **id** dari pointer **LXDDZ 2.4G Wireless Mouse**. Kurang lebih outputnya seperti dibawah ini, cukup cari kata yang berhubungan dengan _Speed_ atau _Accel_.
```log
Device 'LXDDZ 2.4G Wireless Mouse':
...
	libinput Accel Speed (321):	0.000000
	libinput Accel Speed Default (322):	0.000000
...
```
Dari sini kita akan mendapatkan **id** dari properties **Accel Speed** adalah **321**. Kita akan membutuhkannya untuk melakukan _adjustment_ pada mouse.

### Set Mouse Properties
Untuk memberikan value baru pada properties di atas, kita dapat menggunakan perintah berikut.
```shell
xinput set-prop 19 321 --type=float -0.575
```
Saya memberikan value sebesar **-0.575** bertipe _float_ atau pecahan, untuk menurunkan kecepatan akselerasi pada mouse. Dari sini seharusnya mouse akan terasa lebih lambat untuk digerakkan, sesuaikan dengan preferensi kalian. Kita perlu mengatur properties tersebut tiap kali mouse _reconnected_, tentu saja sangat melelahkan. Maka dari itu, mari membuatnya menjadi otomatis.


---

## Deteksi USB Mouse
Saya mencoba membaca perangkat USB yang terhubung menggunakan perintah `lsusb`, beberapa informasi yang kita butuhkan adalah **Device ID** dan **Vendor ID**.

### lsusb
Informasi terkait dapat kita dapatkan dengan perintah berikut.
```shell
sudo lsusb -v | grep -iE 'idproduct|idvendor'
```

Berikut ini adalah sebelum Mouse USB terpasang.
```log
...
  idVendor           0x1d6b Linux Foundation
  idProduct          0x0003 3.0 root hub
  idVendor           0x138a Validity Sensors, Inc.
  idProduct          0x0017 VFS 5011 fingerprint sensor
  idVendor           0x04ca Lite-On Technology Corp.
  idProduct          0x7035 
...
```

Kemudian saya menghubungkan Mouse USB, dan inilah outputnya.
```log
...
  idVendor           0x1d6b Linux Foundation
  idProduct          0x0003 3.0 root hub
  idVendor           0x138a Validity Sensors, Inc.
  idProduct          0x0017 VFS 5011 fingerprint sensor
  idVendor           0x1d57 Xenta
  idProduct          0xfa60 
  idVendor           0x04ca Lite-On Technology Corp.
  idProduct          0x7035 
...
```

Berdasarkan output diatas, dapat kita ambil value yang baru muncul (bernama **Xenta**).
| Key | Value |
| :---: | :---: |
| idVendor | 0x1d57 |
| idProduct | 0xfa60 |

### udevadm
Sekarang mari kita monitor _path_ dari pertangkat USB yang terhubung. Tapi sebelumnya saya akan mencabut mouse terlebih dahulu, barulah menjalankan perintah berikut.
```shell
udevadm monitor
```

Setelah mouse kembali dihubungkan ke komputer, saya mendapatkan output seperti berikut.
```log
monitor will print the received events for:
UDEV - the event which udev sends out after rule processing
KERNEL - the kernel uevent

KERNEL[2977.046735] add      /devices/pci0000:00/0000:00:14.0/usb3/3-3 (usb)
KERNEL[2977.049111] change   /devices/pci0000:00/0000:00:14.0/usb3/3-3 (usb)
KERNEL[2977.049159] add      /devices/pci0000:00/0000:00:14.0/usb3/3-3/3-3:1.0 (usb)
KERNEL[2977.050130] add      /devices/pci0000:00/0000:00:14.0/usb3/3-3/3-3:1.0/0003:1D57:FA60.0011 (hid)
KERNEL[2977.050251] add      /devices/pci0000:00/0000:00:14.0/usb3/3-3/wakeup/wakeup25 (wakeup)
KERNEL[2977.050369] add      /devices/pci0000:00/0000:00:14.0/usb3/3-3/3-3:1.0/0003:1D57:FA60.0011/input/input44 (input)
```

Kita hanya perlu mencari _path_ yang berhubungan dengan **idVendor** dan **idProduct** yang telah kita dapatkan sebelumnya. Maka kita akan menggunakan baris berikut.
```log
KERNEL[2977.049159] add      /devices/pci0000:00/0000:00:14.0/usb3/3-3/3-3:1.0 (usb)
KERNEL[2977.050130] add      /devices/pci0000:00/0000:00:14.0/usb3/3-3/3-3:1.0/0003:1D57:FA60.0011 (hid)
```

Lalu kita cari informasi lebih lanjut tentang _path_ tersebut menggunakan perintah berikut.
```shell
udevadm info -a -p /devices/pci0000:00/0000:00:14.0/usb3/3-3/3-3:1.0
```

Maka kita akan mendapatkan banyak sekali output seperti berikut.
```log
  looking at parent device '/devices/pci0000:00/0000:00:14.0/usb3/3-3':
    KERNELS=="3-3"
    SUBSYSTEMS=="usb"
    DRIVERS=="usb"
    ATTRS{authorized}=="1"
    ATTRS{avoid_reset_quirk}=="0"
    ATTRS{bConfigurationValue}=="1"
    ATTRS{bDeviceClass}=="00"
    ATTRS{bDeviceProtocol}=="00"
    ATTRS{bDeviceSubClass}=="00"
    ATTRS{bMaxPacketSize0}=="64"
    ATTRS{bMaxPower}=="100mA"
    ATTRS{bNumConfigurations}=="1"
    ATTRS{bNumInterfaces}==" 4"
    ATTRS{bcdDevice}=="2003"
    ATTRS{bmAttributes}=="a0"
    ATTRS{busnum}=="3"
    ATTRS{configuration}==""
    ATTRS{devnum}=="9"
    ATTRS{devpath}=="3"
    ATTRS{idProduct}=="fa60"
    ATTRS{idVendor}=="1d57"
    ATTRS{ltm_capable}=="no"
    ATTRS{manufacturer}=="LXDDZ"
    ATTRS{maxchild}=="0"
    ATTRS{power/active_duration}=="553324"
    ATTRS{power/autosuspend}=="2"
    ATTRS{power/autosuspend_delay_ms}=="2000"
    ATTRS{power/connected_duration}=="553324"
    ATTRS{power/control}=="on"
    ATTRS{power/level}=="on"
    ATTRS{power/persist}=="1"
    ATTRS{power/runtime_active_time}=="553060"
    ATTRS{power/runtime_status}=="active"
    ATTRS{power/runtime_suspended_time}=="0"
    ATTRS{power/wakeup}=="enabled"
    ATTRS{power/wakeup_abort_count}=="0"
    ATTRS{power/wakeup_active}=="0"
    ATTRS{power/wakeup_active_count}=="0"
    ATTRS{power/wakeup_count}=="0"
    ATTRS{power/wakeup_expire_count}=="0"
    ATTRS{power/wakeup_last_time_ms}=="0"
    ATTRS{power/wakeup_max_time_ms}=="0"
    ATTRS{power/wakeup_total_time_ms}=="0"
    ATTRS{product}=="2.4G Wireless Mouse"
    ATTRS{quirks}=="0x0"
    ATTRS{removable}=="removable"
    ATTRS{remove}=="(write-only)"
    ATTRS{rx_lanes}=="1"
    ATTRS{speed}=="12"
    ATTRS{tx_lanes}=="1"
    ATTRS{urbnum}=="22306"
    ATTRS{version}==" 1.10"
```

Kita dapat melihat lebih detail informasi yang tentang perangkat USB Mouse yang baru saja saya beli, untuk membuat Udev rules saya membutuhkan value dari **SUBSYSTEMS** pada output di atas.
```log
...
  looking at parent device '/devices/pci0000:00/0000:00:14.0/usb3/3-3':
    KERNELS=="3-3"
    SUBSYSTEMS=="usb"
    DRIVERS=="usb"
...
```
Value dari **SUBSYSTEMS** adalah **usb**.

### Symlink
Mari kita buktikan dengan langsung membuat sebuah Udev rules yang akan menciptakan sebuah _symlink path_ pada `/dev/` ketika perangkat terhubung, dan hilang saat perangkat tercabut.
```shell
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="1d57", ATTRS{idProduct}=="fa61", SYMLINK+="usb_mouse_rexus_wired"' | sudo tee -a /etc/udev/rules.d/83-hybrid-wireless-mouse-rexus-rx107.rules
```

Dengan perintah di atas, saya membuat file baru di direktori `/etc/udev/rules.d` yang bernama `83-hybrid-wireless-mouse-rexus-rx107.rules`. Untuk format nama file pada Udev selalu diawali dengan angka yang akan menjadi nomor urut dari rules tersebut untuk dieksekusi.

Kemudian jalankan ujicoba pada _path_ yang kita dapatkan sebelumnya.
```shell
sudo udevadm test -a /devices/pci0000:00/0000:00:14.0/usb3/3-3
```
Jika Udev rules berhasil, maka kita akan mendapatkan output seperti berikut.
```log
...
Reading rules file: /etc/udev/rules.d/83-hybrid-wireless-mouse-rexus-rx107.rules
...
3-3: /etc/udev/rules.d/83-hybrid-wireless-mouse-rexus-rx107.rules:6 LINK 'usb_mouse_rexus_wireless'
...
ID_VENDOR_ID=1d57
ID_MODEL_ID=fa60
DEVLINKS=/dev/usb_mouse_rexus_wireless
...
```
Periksa juga apakah _symlink_ telah terbuat di direktori `/dev/`.
```log
$ ls -l /dev/usb_mouse_rexus_wireless
lrwxrwxrwx 1 root root 15 Jan 21 20:24 /dev/usb_mouse_rexus_wireless -> bus/usb/003/009
```
Dan saat mouse dicabut, _symlink_ tersebut akan hilang.
```log
ls -l /dev/usb_mouse_rexus_wireless
ls: cannot access '/dev/usb_mouse_rexus_wireless': No such file or directory
```

---


## Plug And Play
Sebenarnya menggunakan istilah _PnP_ atau Plug and Play kurang tepat, karena mouse langsung dapat digunakan tanpa perlu memasang _driver_ dan sebagainya. Hanya saja, seperti yang saya tulis sebelumnya bahwa konfigurasi properties pada mouse perlu tidaklah permanen. Artinya saya perlu melakukannya setiap kali _reconnecting_ mouse, namun terimakasih kepada `udev` dan shell script sederhana yang akan melakukannya secara otomatis.

### Udev rules
Sebelumnya kita telah mencoba membuat rules untuk _path symlink_, kali ini kita akan menambahkan beberapa baris rules lagi untuk mengeksekusi sebuah shell script yang berisi perintah `xinput` untuk konfigurasi properties pada mouse.
```config
# Wired
SUBSYSTEM=="usb", ATTRS{idVendor}=="1d57", ATTRS{idProduct}=="fa61", SYMLINK+="usb_mouse_rexus_wired"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="1d57", ATTRS{idProduct}=="fa61", ENV{DISPLAY}=":0.0", ENV{XAUTHORITY}="/home/pwn3r/.Xauthority", RUN+="/usr/local/bin/set-prop-rexus-mouse"

# Wireless
SUBSYSTEM=="usb", ATTRS{idVendor}=="1d57", ATTRS{idProduct}=="fa60", SYMLINK+="usb_mouse_rexus_wireless"
ACTION=="add", SUBSYSTEM=="usb", ATTRS{idVendor}=="1d57", ATTRS{idProduct}=="fa60", ENV{DISPLAY}=":0.0", ENV{XAUTHORITY}="/home/pwn3r/.Xauthority", RUN+="/usr/local/bin/set-prop-rexus-mouse"
```

Kenapa saya membuat dua _symlink_ dan terdapat dua **idProduct** yang berbeda? Hal ini karena ketika mouse mempunyai _dual-mode_, yang ternyata memiliki **idProduct** berbeda saat dioperasikan secara nirkabel ataupun berkabel.

Lalu kita dapat melihat `ACTION=="add"` yang artinya rules akan mulai menjalankan `RUN` saat mendeteksi _action_ berupa **add** pada atribut yang sesuai dengan **SUBSYSTEM**, **idVendor** dan **idProduct**. Sementara `ENV` adalah _environment variable_ yang akan dilewatkan ketika `RUN` dieksekusi, hal ini diperlukan karena perintah `xinput` pada shell script akan dieksekusi oleh sistem (root). Maka `udev` perlu tahu mana user yang dimaksud, dan mana letak sesi display manager user tersebut.

Selanjutnya muat ulang Udev rules yang telah kita buat supaya _daemon_ akan menerapkannya.
```shell
sudo udevadm control --reload
```

### Response
Dari sini kita akan melakukan sedikit shell scripting, kurang lebih seperti berikut.
```shell
#!/usr/bin/bash

export DISPLAY=${DISPLAY}
export HOME=/home/pwn3r
export XAUTHORITY=$HOME/.Xauthority

/usr/local/bin/set-prop-rexus-mouse-worker &
```

Namun file tersebut tidak akan menjalankan perintah `xinput`. Karena mouse belum terdeteksi pada `xinput list` ketika Udev rules pertama kali dieksekusi. Maka kita akan memanggil shell script lain di _background_ seperti berikut.

```shell
#!/usr/bin/bash

sleep 2

for i in $(xinput list | grep -i "LXDDZ 2.4G Wireless Mouse" | grep -iEv "Consumer|Control|keyboard" | sed -e 's/^.*id=\([0-9]*.\).*$/\1/')
do 
    xinput set-prop $i 321 --type=float -0.6
done
```

{{< admonition warning "Perhatian" >}} 
Perhatikan argumen dari perintah `grep` di atas, kita perlu menyesuaikannya untuk mendapatkan **id** dari perangkat yang terdeteksi oleh `xinput list`. Artinya setiap perangkat memiliki karakteristik yang berbeda, dan silahkan bermain-main untuk mencari value yang sesuai dengan perangkat kalian.
{{< /admonition >}} 

Saya menyimpan kedua file shell script tersebut ke direktori `/usr/local/bin` dan jangan lupa tambahkan executable permission.
```shell
sudo chmod +x /usr/local/bin/set-prop-rexus-mouse.sh
sudo chmod +x /usr/local/bin/set-prop-rexus-mouse-worker.sh
```

Dari sini kita dapat memicu Udev rules dengan perintah berikut. Atau sederhana saja, cukup cabut dan hubungkan kembali mouse. Maka kedua shell script tersebut akan dieksekusi oleh Udev rules.
```shell
sudo udevadm trigger -c add
```


---


## Kesimpulan
Sebenarnya `udev` tidak terlalu rumit untuk digunakan, hanya sedikit susah dalam hal _debug_ perangkat. Faktanya `udev` juga dapat digunakan untuk berbagai macam perangkat dengan interface lain seperti SCSI, SATA/eSATA, PCI, hingga VGA dan HDMI. Jadi tidak untuk USB saja, `udev` dipakai oleh sistem operasi untuk _handling_ berbagai _peripheral_ yang terhubung ke komputer.


---


## Referensi
- [granjow.net/udev-rules.html](http://granjow.net/udev-rules.html)
- [man7.org/linux/man-pages/man8/udevadm.8.html](https://man7.org/linux/man-pages/man8/udevadm.8.html)
- [wiki.archlinux.org/title/udev](https://wiki.archlinux.org/title/udev)
- [bootlin.com/doc/legacy/udev/udev.pdf](https://bootlin.com/doc/legacy/udev/udev.pdf)
- [wiki.archlinux.org/title/xinput](https://wiki.archlinux.org/title/xinput)

