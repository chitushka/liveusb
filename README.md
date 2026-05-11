# Архитектура флешки:

```
/dev/sdb

├── sdb1  EFI FAT32           512MB
├── sdb2  LIVE SYSTEM         5-6GB
└── sdb3  LUKS2 PERSISTENCE   остальное
```


## Установка инструментов:

```bash

sudo apt update

sudo apt install -y \
    debootstrap \
    squashfs-tools \
    xorriso \
    grub-pc-bin \
    grub-efi-amd64-bin \
    dosfstools \
    gdisk \
    cryptsetup \
    rsync \
    mtools
```

## Подготовка workspace:

```bash

mkdir -p ~/liveusb
cd ~/liveusb

mkdir rootfs
mkdir iso
mkdir mnt
```

## Создание minimal Ubuntu rootfs:

```bash

sudo debootstrap \
  --arch=amd64 \
  bionic \
  rootfs \
  http://archive.ubuntu.com/ubuntu
```

В rootfs/ теперь полноценная минимальная Ubuntu.


## Mount pseudo filesystems:
```bash

sudo mount --bind /dev rootfs/dev
sudo mount --bind /run rootfs/run
sudo mount -t proc /proc rootfs/proc
sudo mount -t sysfs /sys rootfs/sys
 
```

## Chroot:
```bash

sudo chroot rootfs
```

Теперь ты внутри будущей Live OS.

```bash
sudo cp /etc/resolv.conf rootfs/etc/
```

## Настройка repositories:
```bash

cat > /etc/apt/sources.list <<EOF
deb http://archive.ubuntu.com/ubuntu bionic main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu bionic-updates main restricted universe multiverse
deb http://security.ubuntu.com/ubuntu bionic-security main restricted universe multiverse
EOF

apt update
```

## Установка ядра и boot stack:
```bash

apt install -y \
    linux-image-generic \
    linux-firmware \
    casper \
    systemd-sysv \
    network-manager
```

Минимальный XFCE:
```bash

apt install -y \
    xfce4 \
    xfce4-terminal \
    lightdm \
    lightdm-gtk-greeter \
    thunar \
    pavucontrol \
    gvfs \
    gvfs-backends \
    file-roller
```

Base tools:
```bash

apt install -y \
    vim \
    curl \
    wget \
    jq \
    git \
    htop \
    tree \
    unzip \
    zip \
    tmux \
    ripgrep \
    fd-find \
    build-essential \
    make \
    gcc \
    g++


apt install -y \
    openssh-client \
    openssh-server

apt install -y \
    php-cli \
    php-curl \
    php-mbstring \
    php-xml \
    composer

apt install -y firefox

cd /tmp

wget https://go.dev/dl/go1.25.10.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.25.10.linux-amd64.tar.gz

mkdir -p /tmp/go-cache
```

## Создание пользователя:
```bash

useradd -m -s /bin/bash dev
passwd dev

usermod -aG sudo dev
```

## Отключение автообновления:
```bash

sudo apt remove -y unattended-upgrades
sudo systemctl disable apt-daily.timer
sudo systemctl disable apt-daily-upgrade.timer
sudo systemctl stop apt-daily.timer
sudo systemctl stop apt-daily-upgrade.timer

sudo apt remove -y update-notifier

cat >> /etc/apt/apt.conf.d/99-no-updates <<EOF

APT::Periodic::Enable "0";
APT::Periodic::Update-Package-Lists "0";
APT::Periodic::Download-Upgradeable-Packages "0";
APT::Periodic::AutocleanInterval "0";
APT::Periodic::Unattended-Upgrade "0";

EOF
```

## Настройка autologin:

```bash

mkdir -p /etc/lightdm/lightdm.conf.d

cat > /etc/lightdm/lightdm.conf.d/50-autologin.conf <<EOF
[Seat:*]
autologin-user=dev
autologin-user-timeout=0
EOF

cat >> /home/dev/.bashrc <<EOF

alias ll='ls -lah'

export GOPATH=\$HOME/go
export PATH=\$PATH:/usr/local/go/bin:\$GOPATH/bin
export GOCACHE=/tmp/go-cache
export COMPOSER_CACHE_DIR=/tmp/composer-cache

EOF

chown dev:dev /home/dev/.bashrc
```

## Flash optimizations:
```bash

mkdir -p /etc/systemd/journald.conf.d

cat > /etc/systemd/journald.conf.d/volatile.conf <<EOF
[Journal]
Storage=volatile
RuntimeMaxUse=100M
EOF

mkdir -p /home/dev/.cache

cat >> /etc/fstab <<EOF

tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0
tmpfs /var/tmp tmpfs defaults,noatime,mode=1777 0 0
tmpfs /var/cache tmpfs rw,noatime,nodev,nosuid,size=512M 0 0
tmpfs /home/dev/.cache tmpfs rw,noatime,nodev,nosuid,size=1G 0 0

EOF

apt install -y zram-tools
```

## Machine cleanup:
```bash

apt clean
rm -rf /tmp/*
rm -rf /var/tmp/*

truncate -s 0 /etc/machine-id

exit
```

## Unmount pseudo FS:
```bash

sudo umount rootfs/dev
sudo umount rootfs/run
sudo umount rootfs/proc
sudo umount rootfs/sys
```

## Create SquashFS:
```bash

sudo mksquashfs \
  rootfs \
  iso/filesystem.squashfs \
  -comp xz \
  -b 1M
```

## Подготовка флешки:

Найти устройство:
```bash

lsblk
```

Допустим это: `/dev/sdb`

Удалить таблицу:
```bash

sudo wipefs -a /dev/sdb
```

Создать GPT:
```bash

sudo gdisk /dev/sdb
```

Создать разделы:

`sdb1 EFI: 512MB, type EF00`

`sdb2 LIVE: 6GB, type 8300`

`sdb3 PERSISTENCE: remaining space, type 8300`


## Форматирование:
```bash

sudo mkfs.vfat -F32 /dev/sdb1
sudo mkfs.ext4 /dev/sdb2
sudo cryptsetup luksFormat \
  --type luks2 \
  /dev/sdb3
sudo cryptsetup open /dev/sdb3 persistence
sudo mkfs.ext4 /dev/mapper/persistence
sudo e2label /dev/mapper/persistence writable
sudo mount /dev/mapper/persistence /mnt

echo "/home union" | sudo tee /mnt/persistence.conf

sudo umount /mnt
sudo cryptsetup close persistence
```

## Copy live system:
```bash

sudo mount /dev/sdb2 /mnt
sudo mkdir -p /mnt/casper
sudo cp iso/filesystem.squashfs /mnt/casper/
sudo cp rootfs/boot/vmlinuz-* /mnt/vmlinuz
sudo cp rootfs/boot/initrd.img-* /mnt/initrd
sudo mkdir -p /mnt/boot/efi
sudo mount /dev/sdb1 /mnt/boot/efi
sudo grub-install \
  --target=x86_64-efi \
  --efi-directory=/mnt/boot/efi \
  --boot-directory=/mnt/boot \
  --removable \
  --recheck

sudo mkdir -p /mnt/boot/grub

cat <<EOF | sudo tee /mnt/boot/grub/grub.cfg

set timeout=5

menuentry "Ubuntu Live Dev" {
    linux /vmlinuz boot=casper persistent quiet splash
    initrd /initrd
}

EOF

sudo umount /mnt/boot/efi
sudo umount /mnt
```
