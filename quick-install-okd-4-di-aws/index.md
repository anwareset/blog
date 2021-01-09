# Quick Install OKD 4 di AWS


OKD 4 memiliki _installer_ yang menawarkan fleksibilitas dan kemudahan bagi pengguna. Proses instalasi hanya akan memakan waktu sekitar 40 menit hingga cluster OKD sudah dapat berjalan dan siap untuk dikelola.

<!--more-->

## OKD 4
**OKD** adalah distribusi dari **Kubernetes** yang menjadi _upstream_ dari pengembangan **OpenShift**. OKD merupakan versi _community_ dari OpenShift, yang dapat digunakan tanpa biaya _subscription_. Kode sumbernya dipublikasikan pada [repositori Github](https://github.com/openshift/okd), dan semua orang dapat berkontribusi untuk mengembangkannya. Silahkan lihat perbandingan antara OKD 4 dengan OKD 3 pada [dokumentasi resmi OKD](https://docs.okd.io/latest/migration/migrating_3_4/planning-migration-3-to-4.html#migration-comparing-ocp-3-4).

### Jenis Instalasi
Pada umumnya terdapat 2 macam instalasi OKD 4 seperti berikut.
1. **IPI (Installer-Provisioned Infrastructure)**. 
Dengan metode IPI, installer akan bertindak sepenuhnya dalam proses instalasi cluster. Setiap _machine_ harus mampu mengakses internet untuk menarik _images_ yang diperlukan untuk inisialisasi ketika _deployment_ cluster. IPI merupakan cara paling cepat untuk membangun cluster OKD yang dapat kita implementasikan pada [AWS (Amazon Web Service)](https://aws.amazon.com/id/), [GCP (Google Cloud Platform)](https://cloud.google.com/), [Azure](https://azure.microsoft.com/en-us/), hingga _baremetal_. Dalam artikel ini saya akan menggunakan IPI sebagai metode instalasi.

2. **UPI (User-Provisioned Infrastructure)**. 
Mirip dengan IPI, namun lebih _advanced_, di mana kita dapat menggunakan metode UPI jika benar-benar _concern_ terhadap keamanan. Biasanya cluster akan membutuhkan sebuah _proxy_ dan akses ke internet akan lebih dibatasi. Karena akses ke internet yang lebih ketat, maka cluster juga memerlukan _mirror registry_ yang lebih spesifik. UPI cenderung lebih rumit, namun memiliki pilihan konfigurasi yang sangat beragam dibandingkan metode IPI, misalnya seperti mengganti sistem operasi _machine_ dengan RHCOS (Red Hat Enterprise Linux CoreOS).


---


## Installation Workflow
OKD menggunakan sebuah _bootstrap machine_ temporer pada saat inisialisasi konfigurasi guna menyediakan informasi yang diperlukan bagi _control plane_. _Bootstrap machine_ dihidupkan dengan sebuah file konfigurasi bernama _Ignition_ yang mendeskripsikan bagaimana proses pembangunan cluster akan dilakukan. _Bootstrap machine_ akan membuat sebuah _master machine_ yang digunakan sebagai _control plane_. Dan _control plane_ kemudian akan membuat _compute machine_ atau yang biasa disebut sebagai _worker machine_.

![Proses Bootstraping](proses-bootstraping.png "Proses Bootstraping")

Setelah semua _machine_ pada cluster di-inisialisasi, maka _bootstrap machine_ akan di-_destroy_. Jika menggunakan metode UPI, kita perlu melakukan beberapa proses di atas secara manual.


---


## Minimum Requirements
OKD 4 memiliki syarat minimum spesifikasi sistem yang cenderung mirip dengan OKD 3. Berikut ini adalah spesifikasi singkat sebagai gambarannya.

| Host | Specs|
|:-----:|:----:|
| Masters   | 4 vCPUs dan 16 GB RAM    |
| Nodes     | 1 vCPUs dan 8 GB RAM     |

Jika menggunakan AWS sebagai _host_, maka _by-default_ OKD installer akan membuat instance dengan tipe `m5.xlarge` sebagai master dan tipe `m5.large` sebagai worker.
Untuk lebih lengkapnya silahkan baca spesifikasi minimum di [dokumentas resmi OKD](https://docs.okd.io/3.11/install/prerequisites.html#hardware) dan jenis-jenis dari [instance AWS EC2](https://aws.amazon.com/id/ec2/instance-types/).


---


## Prerequisites
Sebelum melakukan instalasi lebih lanjut, pastikan beberapa hal berikut sudah terpenuhi.

### Akses AWS
Pasang paket `aws-cli` pada komputer lokal.

```bash
$ sudo pacman -Sy aws-cli
```

{{< admonition warning "Perhatian" >}}
Dalam artikel ini saya akan menggunakan **AWS Programmatic Access** sebagai _credentials_ untuk mengelola AWS menggunakan `aws-cli`. Untuk informasi lebih lanjut silahkan baca [dokumentasi AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html).
{{</ admonition >}}

Saya memiliki AWS Programmatic Access  yang terdiri dari `aws_access_key_id` dan `aws_secret_access_key`. Masukkan kedua key tersebut ke dalam konfigurasi _aws cli_.

```bash
$ mkdir -p ~/.aws
$ vim ~/.aws/credentials
```

Isi file `credentials` dengan kedua key yang kita meliki, sehingga kurang lebih akan manjadi seperti berikut.
```cfg
[default]
aws_access_key_id=AKIA3ANXUSNAUAXXXXXX
aws_secret_access_key=+JDR1HWw4vrcFrLqbf+ewv/nJL/L6TlVAhXXXXXX
```

### SSH Private Key
Kita perlu menambahkan SSH key ke _agent_ dan installer untuk menampilkan informasi ketika proses instalasi berjalan, atau untuk melakukan pengelolaan lebih lanjut terhadap cluster menggunakan installer. Di sini saya sudah memiliki SSH key seperti berikut.

```cfg
/home/pwn3r/.ssh
├── id_rsa
├── id_rsa.pub
└── known_hosts
```

Jika belum memiliki SSH key kita dapat membuatnya dengan `ssh-keygen`.

```bash
$ ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa 
```

Kemudian jalankan `ssh-agent` di _background_.

```bash
$ eval "$(ssh-agent -s)"
```

Lalu tambahkan SSH key ke `ssh-agent`.

```bash
$ ssh-add ~/.ssh/id_rsa
```

### Unduh Installer
Program installer dapat diunduh dari [**https://github.com/openshift/okd/releases**](https://github.com/openshift/okd/releases).

![Installer OKD 4](openshift-installer.png "Installer OKD 4")

Untuk lebih mudahnya saya akan membuat sebuah direktori baru bernama `okd` pada _home directory_ dan mengunduh installer ke dalam direktori tersebut.

```bash
$ mkdir ~/okd
$ cd ~/okd
$ wget https://github.com/openshift/okd/releases/download/4.6.0-0.okd-2020-12-12-135354/openshift-install-linux-4.6.0-0.okd-2020-12-12-135354.tar.gz
```

{{< admonition warning "Perhatian" >}}
Pastikan URL installer untuk perintah `wget` di atas sesuai dengan versi yang benar. Pada saat artikel ini diterbitkan, OKD terbaru berada pada versi 4.6.
{{</ admonition >}}

Ekstrak file `tar.gz` dengan perintah berikut.

```bash
$ tar -xvf openshift-install*.tar.gz
```

Kita akan memiliki sebuah file bernama `openshift-install`, jadikan file tersebut menjadi _executable_.

```bash
$ chmod +x openshift-install
```


---


## Instalasi
Untuk melakukan instalasi kita cukup menjalankan satu perintah seperti berikut.

```bash
$ ./openshift-install create cluster --log-level=info 
```

Selanjutnya installer akan meminta beberapa informasi yang dibutuhkan, silahkan pilih atau isi sesuai dengan kebutuhan. **Pull Secret** bisa diperoleh dari situs [Red Hat OpenShift Cluster Manager](https://cloud.redhat.com/openshift/install/pull-secret). Cukup tekan _Copy pull secret_ dan paste pada terminal ketika installer memintanya.

![Pull Secret](pull-secret.png "Pull Secret")

### Hasil
Proses akan berlangsung sedemikian rupa secara otomatis. Pada umumnya intalasi cluster OKD dengan konfigurasi _default_ akan berlangsung sekitar 40 menit. Jika berhasil kurang lebih akan muncul tampilan seperti berikut.

![Proses Instalasi](proses-instalasi.png "Proses Instalasi")


---


## Pengujian
Kita akan melakukan beberapa pemeriksaan terhadap cluster OKD 4. Selain lewat web browser, kita dapat menggunakan `oc` atau `origin-client`. Jika belum memilikinya maka lakukan pemasangan paket pada komputer lokal seperti berikut.

```bash
$ yay -Sy origin-client
```

{{< admonition info "Info" >}} 
Pada distro Arch-based, `origin-client` tersedia pada **AUR** (Arch User Repository). Untuk melakukan instalasi paket dari AUR, saya menggunakan _AUR Helper_ bernama `yay`.
{{</ admonition >}}

### Login sebagai System Admin
Untuk bertindak sebagai sistem admin melalui perintah `oc`, kita me-_export_ variabel _environment_ seperti berikut.

```bash
$ export KUBECONFIG=/home/pwn3r/okd/auth/kubeconfig
```

Kemudian jalankan periksa user yang sedang berjalan di sesi `oc` saat ini.

```bash
$ oc whoami
```

Harusnya akan muncuk tampilan seperti berikut.

```cfg
system:admin
```

### Periksa Nodes
Coba periksa apakah semua node sudah dalam keadaan berjalan dengan `oc`.

```bash
$ oc get nodes
```

Kurang lebih akan muncul tampilan seperti berikut.

```cfg
NAME                                         STATUS   ROLES    AGE     VERSION
ip-10-0-130-52.us-west-1.compute.internal    Ready    master   5h41m   v1.19.2+7070803-1008
ip-10-0-132-253.us-west-1.compute.internal   Ready    master   5h41m   v1.19.2+7070803-1008
ip-10-0-155-103.us-west-1.compute.internal   Ready    worker   5h31m   v1.19.2+7070803-1008
ip-10-0-166-37.us-west-1.compute.internal    Ready    worker   5h32m   v1.19.2+7070803-1008
ip-10-0-225-80.us-west-1.compute.internal    Ready    master   5h41m   v1.19.2+7070803-1008
ip-10-0-240-130.us-west-1.compute.internal   Ready    worker   5h32m   v1.19.2+7070803-1008
```

Terlihat bahwa terdapat 3 node yang bertindak sebagai _master_, dan 3 node lain sebagai _worker_.

### Scaling MachineSet
Periksa _machineset_ yang ada dengan perintah berikut.

```bash
$ oc get machineset -n openshift-machine-api
```

Maka akan tampil _machineset_ yang ada pada OKD.

```cfg
NAME                             DESIRED   CURRENT   READY   AVAILABLE   AGE
itglab-f2l75-worker-us-west-1a   2         2         2       2           5h48m
itglab-f2l75-worker-us-west-1b   1         1         1       1           5h48m
```

Artinya _machineset_ `itglab-f2l75-worker-us-west-1a` memiliki replika sebanyak 2 buah. Sedangkan `itglab-f2l75-worker-us-west-1b` hanya memiliki 1 replika. Sekarang coba lakukan _scaling_ supaya _machineset_ memiliki 2 buah replikasi.

```bash
$ oc scale --replicas=2 machinesets itglab-f2l75-worker-us-west-1b -n openshift-machine-api
```

Biasanya kita perlu menunggu beberapa saat supaya _machine_ berjalan dan kemudian bergabung pada _machineset_ `itglab-f2l75-worker-us-west-1b`. Lama durasi tergantung pada spesifikasi sistem yang kita gunakan di AWS. Periksa apakah _scaling_ berhasil dilakukan dengan menjalankan kembali perintah berikut.

```bash
$ oc get machineset -n openshift-machine-api
```

Jika berhasil maka semua _machineset_ akan menjadi seperti berikut.

```cfg
NAME                             DESIRED   CURRENT   READY   AVAILABLE   AGE
itglab-f2l75-worker-us-west-1a   2         2         2       2           5h58m
itglab-f2l75-worker-us-west-1b   2         2         2       2           5h58m
```

### Web Console OKD
Sekarang coba akses OKD Console melalui web browser dengan alamat dan kata sandi seperti yang terlihat [sebelumnya](#hasil) menggunakan username `kubeadmin`.

![OKD Web Console](okd-web-console.png "OKD Web Console")

### Destroy Cluster
Installer OKD 4 juga memiliki fitur untuk menghapus semua resource yang telah dibuat sehingga saya tidak perlu repot-repot membersihkan satu region di AWS Console. _Destroy cluster_ dapat dijalankan dengan perintah seperti berikut.

```bash
$ ./openshift-install destroy cluster --log-level=info
```


---


## Kesimpulan
OKD 4 memiliki installer yang tergolong mudah untuk diimplementasikan (pada AWS). Setiap node OKD 4 berjalan dengan sistem operasi FCOS (Fedora Core OS), namun jika kita menggunakan OpenShift maka yang digunakan adalah RHCOS (Red Hat Enterprise Linux CoreOS). OKD 4 menggunakan [Prometheus](https://prometheus.io/) untuk mendapatkan _metric_ yang berguna untuk monitoring. Container Runtime yang dipakai oleh OKD 4 adalah [CRI-O](http://cri-o.io/).


--- 


## Referensi
- [docs.okd.io/latest/architecture/architecture-installation.html](https://docs.okd.io/latest/architecture/architecture-installation.html)
- [docs.okd.io/latest/installing/installing_aws/installing-aws-default.html](https://docs.okd.io/latest/installing/installing_aws/installing-aws-default.html)
- [wiki.archlinux.org/index.php/OpenShift](https://wiki.archlinux.org/index.php/OpenShift)
- [github.com/openshift/okd#getting-started](https://github.com/openshift/okd#getting-started)
- [github.com/openshift/okd/blob/master/FAQ.md](https://github.com/openshift/okd/blob/master/FAQ.md)
- [github.com/openshift/okd/releases](https://github.com/openshift/okd/releases)
- [docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)
- [aws.amazon.com/ec2/instance-types/](https://aws.amazon.com/ec2/instance-types/)

