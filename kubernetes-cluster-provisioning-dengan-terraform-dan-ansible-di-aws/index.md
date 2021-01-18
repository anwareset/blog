# Kubernetes Cluster Provisioning dengan Terraform dan Ansible


Kubernetes Vanilla atau versi Kubernetes orisinil merupakan aplikasi opensource untuk _Container Orchestration_ dengan berbagai fleksibilitas yang dapat diintegrasikan lebih lanjut bersama komponen atau tools lain dari pihak ketiga. Berbeda dengan distribusi Kubernetes (seperti OpenShift dan OKD) yang menawarkan kemudahan dalam instalasi, Kubernetes (atau sering disebut k8s) seringkali membutuhkan _effort_ yang lebih besar dalam membangun klaster-nya jika dilakukan secara manual tanpa layanan dari vendor atau _cloud provider_ (seperti EKS atau GKS). Namun sebenarnya kita dapat melakukannya dengan lebih _effortless_ menggunakan tools untuk _provisioning_ infrastruktur maupun konfigurasi, seperti Terraform dan Ansible.

<!--more-->

## Struktur Klaster
Pada artikel ini saya akan membangun klaster Kubernetes yang terdiri dari satu Control Plane atau Master Node, serta dua Worker Node. Untuk pengelolaan container di dalam klaster saya akan menggunakan Docker. Infrastruktur akan didirikan di _cloud provider_ AWS (Amazon Web Service).

![Klaster Kubernetes](klaster-kubernetes.png "Klaster Kubernetes")

### Provisioning
**Provisioning** dalam urusan _cloud native_ sering dianggap sebagai proses menyediakan suatu layanan. Segala proses yang terjadi dalam menyediakan layanan tersebut dapat dilakukan secara manual, namun seiring berkembangnya zaman tentu saja kini banyak sekali tools yang bermunculan dan menawarkan kemampuan untuk membuat _process automation_. Misalnya pada artikel ini, [**Terraform**](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) akan digunakan untuk melakukan provisioning infrastruktur pada AWS. Sementara dalam melakukan konfigurasi atau _setup_ setiap instance, [**Ansible Playbook**](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html) adalah tools yang akan bertanggung jawab untuk melakukannya. Ansible Playbook akan melakukan beberapa proses mulai dari update sistem operasi, setup Docker, memasang `kubelet`, `etcd`, membuat token bagi worker node hingga berbagai keperluan _setup_ k8s lainnya sampai selesai.


---


## Prerequisites
Pastikan beberapa prasyarat berikut ini sudah terpenuhi sebelum melakukan tahap lebih lanjut.

### Access Key
Untuk mendapatkan access key AWS, kita harus memiliki akun AWS. Access Key ini sama dengan yang biasanya digunakan sebagai credentials `aws-cli`, dan pada artikel ini access key tersebut akan dipakai oleh Terraform untuk mengeksekusi perintah sesuai dengan script yang kita buat. Untuk lebih lengkapnya silahkan lihat [dokumentasi resmi AWS](https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html#get-access-keys). Simpan access key tersebut ke dalam sebuah file, misalnya `credentials` seperti berikut ini.

```credentials
[default]
aws_access_key_id=AKIA3ANXUSNAUATXXXXX
aws_secret_access_key=+JDR1HWw4vrcFrLqbf+ewv/nJL/L6TlVAhwXXXXX
```

### VPC
Pada umumnya _by-default_ AWS sudah memiliki VPC (Virtual Private Cloud), namun jika belum kita dapat membuatnya sendiri dengan perintah berikut.
```bash
$ aws ec2 create-default-vpc --availability-zone us-east-1
```

Saya akan menggunakan default VPC Subnet yang sudah ada pada AWS. Namun jika belum ada maka kita dapat membuatnya sendiri seperti berikut.
```bash
$ aws ec2 create-default-subnet --availability-zone us-east-1a 
```

Kita akan membutuhkan **id** dari Subnet VPC yang telah kita buat untuk tahap selanjutnya.
```bash
$ aws ec2 describe-subnets --region us-east-1
```

Catat dan simpan **id** dari Subnet VPC yang telah kita buat.


### SSH Key
Buat sebuah SSH Key bagi instance AWS yang akan kita dirikan. Untuk membuatnya saya menggunakan `aws-cli` dengan perintah berikut.
```bash
$ aws ec2 create-key-pair --region us-east-1 --key-name infra-k8s --query 'KeyMaterial' --output text > infra-k8s.pem
```

Nantinya perintah tersebut akan menghasilkan sebuah file SSH Key bernama `infra-k8s.pem`. Selanjutnya kita perlu merubah permissionnya supaya dapat digunakan untuk mengakses instance.
```bash
$ chmod 400 infra-k8s.pem
```

{{< admonition "info" "Info" >}}
Dalam artikel ini saya menggunakan region `us-east-1` atau North Virginia. Jika ingin menggunakan regional lain, silahkan lihat [dokumentasi resmi AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html).
{{</ admonition >}}


---

 
## Infrastructure Provisioning
### Pasang Paket Terraform
Pasang paket `terraform` pada komputer lokal seperti berikut.
```bash
$ sudo pacman -Sy terraform
```
{{< admonition "info" "Info" >}}
Saya menggunakan sistem operasi GNU/Linux dengan distro Arch-based yang memiliki package manager `pacman`. Silahkan sesuaikan dengan perintah package manager pada distro masing-masing.
{{</ admonition >}}

### Script Terraform
Selanjutnya silahkan buat file baru untuk menulis script terraform. Misalnya saya membuat file `infra.tf` yang akan berisi script Terraform seperti berikut ini.

```terraform
## AWS Provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

## Init Credentials Profile and Region
provider "aws" {
  profile = "default"
  region  = "us-east-1"
  shared_credentials_file = "credentials"
}

## Public Security Group
resource "aws_security_group" "public-sec" {
  name        = "public"
  description = "Public Server Security Group"

## SSH, HTTP, HTTPS Access
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
## Kubernetes Cluster  
  ingress {
    from_port   = 6443
    to_port     = 6443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 2379
    to_port     = 2380
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 10250
    to_port     = 10250
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 10251
    to_port     = 10251
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 10252
    to_port     = 10252
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Name        = "public"
    Description = "Public Server Security Group"
  }
}

## Elastic IP for Public Instance
resource "aws_eip" "lb" {
  instance = aws_instance.public.id
}

## Create Public Instance
resource "aws_instance" "public" {
  ami               = "ami-00ddb0e5626798373"
  instance_type     = "t2.medium"
  source_dest_check = false
  key_name          = "infra-k8s"
  subnet_id         = "subnet-0f5a08d3063a6b742"
  private_ip        = "172.31.16.20"
  vpc_security_group_ids   = aws_security_group.public-sec.*.id
  tags = {
    Name = "public"
  }

## Disk Space
  root_block_device {
    delete_on_termination = true
    encrypted             = false
    iops                  = 100
    volume_size           = 10
  }
}

## Node Security Group
resource "aws_security_group" "node-sec" {
  name        = "node"
  description = "Node/Worker Security Group"

## SSH Access
  ingress {
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"

## Close all traffic IP after setup!
    #cidr_blocks     = ["172.31.16.20/32"]
    cidr_blocks    = ["0.0.0.0/0"]
  }
## Kubernetes Cluster  
  ingress {
    from_port   = 10250
    to_port     = 10250
    protocol    = "tcp"
    cidr_blocks = ["172.31.16.20/32"]
  }
  ingress {
    from_port   = 10255
    to_port     = 10255
    protocol    = "tcp"
    cidr_blocks = ["172.31.16.20/32"]
  }
  ingress {
    from_port   = 30000
    to_port     = 32767
    protocol    = "tcp"
    cidr_blocks = ["172.31.16.20/32"]
  }
  ingress {
    from_port   = 3000
    to_port     = 3000
    protocol    = "tcp"
    cidr_blocks = ["172.31.16.20/32"]
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Name        = "node"
    Description = "Node/Worker Security Group"
  }
}

## Elastic IP for Node Server (Temporary)
resource "aws_eip" "lb-node" {
  count = 2
  instance = aws_instance.node[count.index].id
}

## Create Instance Node
resource "aws_instance" "node" {
  ami                         = "ami-00ddb0e5626798373"
  instance_type               = "t2.small"
  associate_public_ip_address = false
  source_dest_check           = false
  key_name                    = "infra-k8s"
  subnet_id                   = "subnet-0f5a08d3063a6b742"
  vpc_security_group_ids      = aws_security_group.node-sec.*.id
  count                       = 2
  tags = {
    Name = "node-${count.index + 1}"
  }

## Disk Space
    root_block_device {
    delete_on_termination = true
    encrypted             = false
    iops                  = 100
    volume_size           = 10
  }
}

```

{{< admonition "warning" "Perhatian" >}}
Sesuaikan variabel `subnet_id` dengan **id** Subnet VPC yang telah kita buat [sebelumnya](#vpc). Sedangkan variabel `ami` merupakan **id** dari Amazon Machine Image yang value nya tergantung dari distro apa yang akan kita jalankan pada setiap instance EC2 untuk informasi lebih lanjut kita dapat melihatnya dari [dokumentasi resmi AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami.html) karena setiap region dan distro hingga sistem operasi memiliki AMI ID yang berbeda-beda. Terlihat bahwa saya membuka seluruh traffic dari _inbound_ dan _outbound_ yang tentu saja tidaklah aman, sesuaikan dengan kebutuhan [port Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports) pada umumnya.
{{</ admonition >}}

Inisialiasi terraform untuk mengunduh dependency yang diperlukan dengan perintah berikut.

```bash
$ terraform init
```

### Eksekusi Terraform
Sekarang jalankan provisioning menggunakan terraform sesuai dengan script yang telah dibuat sebelumnya.

```bash
$ terraform apply
```


---


## Configuration Provisioning
### Pasang Paket Ansible
Pasang `ansible` pada komputer lokal menggunakan perintah berikut.
```bash
$ sudo pacman -Sy ansible
```

### Inventory Ansible Playbook
Jalankan perintah berikut untuk mendapatkan IP Publik dari Master Node.
```bash
$ aws ec2 describe-instances  \
    --region us-east-1 \
    --filters "Name=tag-value,Values=public"
```

Jalankan perintah berikut untuk mendapatkan IP Publik dari Worker Node pertama.
```bash
$ aws ec2 describe-instances  \
    --region us-east-1 \
    --filters "Name=tag-value,Values=node-1"
```

Jalankan perintah berikut untuk mendapatkan IP Publik dari Worker Node pertama.
```bash
$ aws ec2 describe-instances  \
    --region us-east-1 \
    --filters "Name=tag-value,Values=node-2"
```

Kemudian masukan ketiga IP tersebut ke dalam file bernama `inventory` sehingga kurang lebih isinya akan menjadi seperti berikut.
```inventory
[public]
18.233.242.19

[nodes]
3.224.63.65
54.198.25.43
```

### Docker
Selanjutnya saya membuat file baru bernama `docker.yml` yang berisi script untuk melakukan update sistem operasi dan _setup_ Docker.
```yaml
- hosts: all
  gather_facts: false
  tasks:
    - name: Update & Upgrade
      become: yes
      apt:
        upgrade: dist
        update_cache: yes

    - name: Install Docker Requiremts
      become: yes
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg-agent
          - python3-pip
          - software-properties-common

    - name: GPG Key Docker
      become: yes
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg

    - name: Repo Docker
      become: yes
      apt_repository:
        repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable
        state: present
        update_cache: yes

    - name: Install Docker
      become: yes
      apt:
        force: True
        name:
          - docker-ce

    - name: Hold Docker
      become: yes
      dpkg_selections:
        name: docker-ce
        selection: hold

    - name: Install Docker Py as root
      become: yes
      command: pip3 install docker-py

    - name: Install Docker Py as normal user
      command: pip3 install docker-py
    
    - name: Enable service docker, and enable persistently
      become: yes
      service:
        name: docker
        enabled: yes

    - name: Add the user 'ubuntu' to docker group
      become: yes
      user:
        name: ubuntu
        group: docker
```

Terapkan script yang telah kita buat dengan Ansible Playbook seperti berikut.
```bash
$ ansible docker.yml
```

Jika berhasil maka akan muncul tampilan seperti berikut ini.
```ansible
PLAY RECAP *******************************************************************************************************
18.233.242.19              : ok=10   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
3.224.63.65                : ok=10   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
54.198.25.43               : ok=10   changed=9    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

### Bootstraping Kubernetes
Sekarang buat file baru bernama `k8s.yml` yang berisi script untuk melakukan _setup_ Kubernetes.
```yaml
- hosts: all
  gather_facts: false
  tasks:
    - name: Add Google official GPG key
      become: yes
      apt_key:
        url: https://packages.cloud.google.com/apt/doc/apt-key.gpg
        state: present

    - name: Add Kubernetes Repository
      become: yes
      apt_repository:
        repo: deb http://apt.kubernetes.io/ kubernetes-xenial main
        state: present
        filename: kubernetes
        mode: 0600

    - name: Installing Kubernetes Cluster Packages
      become: yes
      apt:
        force: True
        name:
          - kubeadm
          - kubectl
          - kubelet
        state: present

    - name: Hold kubeadm
      become: yes
      dpkg_selections:
        name: kubeadm
        selection: hold

    - name: Hold kubectl
      become: yes
      dpkg_selections:
        name: kubectl
        selection: hold

    - name: Hold kubelet
      become: yes
      dpkg_selections:
        name: kubelet
        selection: hold

    - name: Add line Net Bridge on sysctl.conf
      become: yes
      lineinfile:
        path: /etc/sysctl.conf
        line: net.bridge.bridge-nf-call-iptables=1

    - name: Add line 'max_map_count' on sysctl.conf
      become: yes
      lineinfile:
        path: /etc/sysctl.conf
        line: vm.max_map_count=262144

    - name: Apply change on sysctl
      become: yes
      become_method: sudo
      shell: sysctl -p

- hosts: public
  gather_facts: false
  tasks:
    - name: Initialize k8s cluster
      become: yes
      become_method: sudo
      shell: kubeadm reset -f
    
    - name: Start k8s init
      become: yes
      become_method: sudo
      shell: kubeadm init --pod-network-cidr=10.244.0.0/16
      register: kubeadm_result
    - debug:
        var: kubeadm_result.stdout_lines

    - name: Make configuration folder for k8s
      become: yes
      become_user: ubuntu
      file:
        state: directory
        path: /home/ubuntu/.kube
        mode: 0755

    - name: Copy configuration k8s into home
      become: yes
      copy:
        src: /etc/kubernetes/admin.conf
        dest: /home/ubuntu/.kube/config
        remote_src: yes
        owner: ubuntu

    - name: apply network plugin flannel
      become: yes
      become_user: ubuntu
      shell: kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

- hosts: nodes
  gather_facts: false
  vars_prompt:
    - name: "kubeadm"
      prompt: "kubeadm Command "
      private: no
  tasks:
    - name: Reset kubeadm
      become: yes
      become_method: sudo
      shell: kubeadm reset -f

    - name: Prune docker images
      become: yes
      become_method: sudo
      shell: sudo docker image prune -af

    - name: Retriving input command kubeadm
      become: yes
      become_method: sudo
      shell: "{{ kubeadm }}"
```

Kemudian terapkan script tersebut dengan Ansible Playbook seperti berikut.
```bash
$ ansible k8s.yml
```

### Joining Cluster
Di tengah proses, kita akan diberikan sebuah token yang harus dimasukkan secara manual. Token tersebut digunakan oleh Worker Node untuk bergabung ke dalam klaster.
![Join Token](join-token.png "Join Token")

Jika berhasil maka akan muncul tampilan seperti berikut ini.
```bash
PLAY RECAP *******************************************************************************************************
18.233.242.19              : ok=15   changed=14   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
3.224.63.65                : ok=12   changed=12   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
54.198.25.43               : ok=12   changed=12   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
```

---


## Post Installation
### Menghapus Worker Node Elastic IP
Karena Worker Node sudah tidak lagi membutuhkan IP Publik, maka kita akan menghapusnya dengan cara merubah script file `infra.tf` yang sebelumnya kita buat. Jadikan beberapa baris konfigurasi menjadi _comment_ seperti berikut ini.
```ansible
## Elastic IP for Node Server (Temporary)
#resource "aws_eip" "lb-node" {
#  count = 2
#  instance = aws_instance.node[count.index].id
#}
```
Dan tambahkan _comment_ juga pada baris konfigurasi variabel `associate_public_ip_address`.
```ansible
#associate_public_ip_address = false
```

Kemudian jalankan kembali Terraform seperti berikut.
```bash
$ terraform apply
```

Periksa dan jika sudah benar maka ketik **`yes`** dan tekan **Enter**. Apabila berhasil maka akan muncul tampilan seperti berikut.
```ansible
aws_eip.lb-node[1]: Destroying... [id=eipalloc-0adb19f75c6648130]
aws_eip.lb-node[0]: Destroying... [id=eipalloc-07464709ac36cafe4]
aws_eip.lb-node[0]: Destruction complete after 5s
aws_eip.lb-node[1]: Destruction complete after 5s

Apply complete! Resources: 0 added, 0 changed, 2 destroyed.
```

### Ubah Label Worker Node
Sekarang coba akses Master Node lewat SSH menggunakan SSH Key yang sebelumnya telah kita miliki.
```bash
$ ssh -i infra-k8s.pem ubuntu@18.233.242.19
```

{{< admonition "warning" "Perhatian" >}}
Sesuaikan dengan alamat IP Publik yang dimiliki oleh Master Node atau Control Plane.
{{</ admonition >}}

Setelah berhasil masuk, sekarang jalankan perintah berikut.
```bash
$ kubectl get nodes
```
Maka akan muncul daftar node yang terhubung pada klaster.
```bash
NAME               STATUS   ROLES                  AGE     VERSION
ip-172-31-16-20    Ready    control-plane,master   8m50s   v1.20.2
ip-172-31-25-81    Ready    <none>                 7m41s   v1.20.2
ip-172-31-29-192   Ready    <none>                 7m40s   v1.20.2
```

Ubah **ROLES** Worker Node dengan menambahkan label.
```bash
$ kubectl label node ip-172-31-25-81 node-role.kubernetes.io/worker=worker1
$ kubectl label node ip-172-31-29-192 node-role.kubernetes.io/worker=worker2
```

{{< admonition "warning" "Perhatian" >}}
Sesuaikan nama node dengan **NAME** seperti yang muncul dari hasil pemeriksaan daftar node.
{{</ admonition >}}


---


## Pengujian
Sekarang coba periksa lagi daftar node yang ada.
```bash
$ kubectl get nodes
```

Jika berhasil hasilnya akan menjadi seperti berikut.

```cfg
NAME               STATUS   ROLES                  AGE     VERSION
ip-172-31-16-20    Ready    control-plane,master   8m50s   v1.20.2
ip-172-31-25-81    Ready    worker                 7m41s   v1.20.2
ip-172-31-29-192   Ready    worker                 7m40s   v1.20.2
```

Periksa pod yang berjalan pada semua namespace.
```bash
$ kubectl get pods --all-namespaces
```

Hasilnya kurang lebih akan seperti berikut.
```cfg
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   coredns-74ff55c5b-kfzvr                   1/1     Running   0          4h56m
kube-system   coredns-74ff55c5b-qrz2j                   1/1     Running   0          4h56m
kube-system   etcd-ip-172-31-16-20                      1/1     Running   0          4h56m
kube-system   kube-apiserver-ip-172-31-16-20            1/1     Running   0          4h56m
kube-system   kube-controller-manager-ip-172-31-16-20   1/1     Running   0          4h56m
kube-system   kube-flannel-ds-hltkg                     1/1     Running   0          4h55m
kube-system   kube-flannel-ds-jvvbv                     1/1     Running   0          4h56m
kube-system   kube-flannel-ds-xfdq8                     1/1     Running   0          4h55m
kube-system   kube-proxy-67kgp                          1/1     Running   0          4h55m
kube-system   kube-proxy-bd7cj                          1/1     Running   0          4h56m
kube-system   kube-proxy-gjkr6                          1/1     Running   0          4h55m
kube-system   kube-scheduler-ip-172-31-16-20            1/1     Running   0          4h56m
```

---


## Kesimpulan
Dengan menggunakan Terraform dan Ansible, kita dapat membangun klaster Kubernetes sendiri pada AWS. Salah satu keunggulan yang saya rasakan adalah fleksibilitas yang tinggi, dimana saya dapat menggunakan spesifikasi instance yang terbilang rendah untuk mencoba membangun klaster Kubernetes jika dibandingkan dengan menggunakan distribusi Kubernetes (seperti OKD atau OpenShift). Jika semua script telah tersedia, maka proses instalasi klaster bisa dikatakan sangat cepat.


---


## Referensi
* [registry.terraform.io/providers/hashicorp/aws/latest/docs](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
* [docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html](https://docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html#playbook-keywords)
* [kubernetes.io/docs/setup/production-environment/container-runtimes/#docker](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker)
* [kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
* [docs.aws.amazon.com/cli/latest/reference/ec2/](https://docs.aws.amazon.com/cli/latest/reference/ec2/)
* [docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2-keypairs.html](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2-keypairs.html)
* [docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2-instances.html](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2-instances.html)
* [docs.aws.amazon.com/cli/latest/reference/ec2/describe-vpcs.html](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-vpcs.html)
* [blog.leonprasetya.my.id](https://blog.leonprasetya.my.id/p/setup-kubernetes-cluster-pada-aws-educate-mulai-dari-penyiapan-server-sampai-selesai/)

