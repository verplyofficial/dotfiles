# Полная установка Arch Linux + Hyprland + Caelestia (с shell и дотами)

### 1. Подключение к WiFi (необязательно)
iwctl
device list
station устройство scan
station устройство get-networks
station устройство connect SSID
ping google.com

### 2. Установка крупного шрифта (необязательно)
pacman -S terminus-font
cd /usr/share/kbd/consolefonts
setfont ter-u32b.psf.gz

### 3. Разметка диска под UEFI GPT с шифрованием
Пример для SSD:
* /dev/nvme0n1p1 → EFI
* /dev/nvme0n1p2 → шифрованный root

parted /dev/sda
mklabel gpt
mkpart ESP fat32 1Mib 512Mib
set 1 boot on
mkpart primary
# file system (ENTER)
# start: 513Mib
# end: 100%
quit

### 4. Шифрование раздела
cryptsetup luksFormat /dev/sda2
cryptsetup open /dev/sda2 luks
pvcreate /dev/mapper/luks
vgcreate main /dev/mapper/luks
lvcreate -l 100%FREE main -n root
lvs

### 5. Подготовка разделов и монтирование
mkfs.ext4 /dev/mapper/main-root
mkfs.fat -F 32 /dev/sda1
mount /dev/mapper/main-root /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot

### 6. Установка системы
pacstrap -K /mnt base linux linux-firmware base-devel lvm2 \
dhcpcd net-tools iproute2 networkmanager vim micro efibootmgr iwd

### 7. Настройка системы
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt

micro /etc/locale.gen
# раскомментируйте en_US.UTF-8 и ru_RU.UTF-8
locale-gen
ln -sf /usr/share/zoneinfo/Europe/Kiev /etc/localtime
hwclock --systohc
echo "arch" > /etc/hostname
passwd
useradd -m -G wheel,users,video -s /bin/bash user
passwd user
systemctl enable dhcpcd
systemctl enable iwd.service

### 8. mkinitcpio
micro /etc/mkinitcpio.conf
# HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block filesystems encrypt lvm2 fsck)
mkinitcpio -p linux

### 9. Установка загрузчика
bootctl install --path=/boot
cd /boot/loader
micro loader.conf
# Вставляем:
timeout 3
default arch
# Создаем /boot/loader/entries/arch.conf:
title Arch Linux Hyprland
linux /vmlinuz-linux
initrd /initramfs-linux.img
options rw cryptdevice=UUID=uuid_от_/dev/sda2:main root=/dev/mapper/main-root
EDITOR=micro visudo
# раскомментировать %wheel ALL=(ALL:ALL) ALL

Ctrl+D
umount -R /mnt
reboot

---

## 10. Установка Hyprland окружения

### 10.1. Базовые пакеты
sudo pacman -Syu
sudo pacman -S git fish

### 10.2. Установка AUR-хелпера paru
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si

### 10.3. Установка Hyprland и зависимостей
paru -S hyprland xdg-desktop-portal-hyprland xdg-desktop-portal-gtk \
hyprpicker hypridle wl-clipboard cliphist bluez-utils inotify-tools \
app2unit wireplumber trash-cli foot fastfetch starship btop jq socat \
imagemagick curl adw-gtk-theme papirus-icon-theme qt5ct qt6ct ttf-jetbrains-mono-nerd

# Либо одним пакетом:
paru -S caelestia-meta

---

## 11. Установка caelestia dotfiles
git clone https://github.com/caelestia-dots/caelestia.git ~/.local/share/caelestia
cd ~/.local/share/caelestia
fish install.fish --paru --noconfirm --vscode=codium --spotify --discord --zen

# ⚠️ Не удаляй папку ~/.local/share/caelestia — там все symlink’и

---

## 12. Установка логин-менеджера
paru -S greetd tuigreet
sudo micro /etc/greetd/config.toml

# Пример:
[terminal]
vt = 1

[default_session]
command = "tuigreet --time --cmd Hyprland"
user = "user"

sudo systemctl enable greetd

---

## 13. Установка дополнительных зависимостей shell (manual)
paru -S caelestia-cli quickshell-git ddcutil brightnessctl cava \
networkmanager lm-sensors aubio libpipewire glibc qt6-declarative \
gcc-libs material-symbols caskaydia-cove-nerd swappy libqalculate \
bash qt6-base qt6-declarative app2unit fish cmake ninja

mkdir -p ~/.config/quickshell
cd ~/.config/quickshell
git clone https://github.com/caelestia-dots/shell.git caelestia
cd caelestia
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/
cmake --build build
sudo cmake --install build
sudo chown -R $USER ~/.config/quickshell/caelestia

# Для локальной сборки:
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
-DCMAKE_INSTALL_PREFIX=/ \
-DINSTALL_QSCONFDIR=~/.config/quickshell/caelestia
cmake --build build
sudo cmake --install build
sudo chown -R $USER ~/.config/quickshell/caelestia

---

## 14. Настройка shell и обоев
# Создаём ~/.config/caelestia/shell.json
# Здесь можно добавить рекомендуемые опции shell

# Wallpapers по умолчанию: ~/Pictures/Wallpapers
# Чтобы установить обои:
caelestia wallpaper -f ~/Pictures/Wallpapers/<file>
# Для динамической цветовой схемы:
caelestia scheme set -n dynamic

# PFP/dashboard: ~/.face

---

## 15. Автостарт shell
Если установлены все caelestia dots, shell запускается автоматически через Hyprland `exec-once`.
Иначе вручную:
caelestia shell -d
# или
qs -c caelestia

---

## 16. Исправления и кастомизация Hyprland
# Отключение мерцания (VRR):
nano ~/.config/caelestia/hypr-user.conf
misc {
    vrr = 0
}

# Добавление своих конфигов Hyprland:
# ~/.config/caelestia/hypr-user.conf
