### Подключаемся к WiFi (необязательно)
```bash
iwctl
device list
station устройство scan
station устройство get-networks
station устройство connect SSID
ping google.com
```

### Установка крупного шрифта (необязательно)
```bash
pacman -S terminus-font
cd /usr/share/kbd/consolefonts
setfont ter-u32b.psf.gz
```

### Разметка диска под UEFI GPT с шифрованием
```bash
parted /dev/nvme0n1
mklabel gpt
mkpart ESP fat32 1Mib 512Mib
set 1 boot on

mkpart primary
# file system (нажимаем ENTER)
# start: 513Mib
# end: 100%

quit
```

### Шифруем раздел который подготавливался ранее
```bash
cryptsetup luksFormat /dev/nvme0n1p2
# nvme0n1p2 – раздел с шифрованием
# вводим YES большими буквами
# вводим пароль 2 раза

# Открываем зашифрованный раздел
cryptsetup open /dev/nvme0n1p2 luks

# Проверяем разделы
ls /dev/mapper/*

# Создаем логические разделы внутри зашифрованного раздела
pvcreate /dev/mapper/luks
vgcreate main /dev/mapper/luks

# 100% зашифрованного раздела помещаем в логический раздел root
lvcreate -l 100%FREE main -n root

# Посмотреть все логические разделы
lvs
```

### Подготовка разделов и монтирование
```bash
# Форматируем раздел под ext4
mkfs.ext4 /dev/mapper/main-root

# Форматируем boot раздел под Fat32, на физ.разделе /dev/nvme0n1p1 лежит boot
mkfs.fat -F 32 /dev/nvme0n1p1

# Монтируем разделы для установки системы
mount /dev/mapper/main-root /mnt
mkdir /mnt/boot

# Монтируем раздел с boot в текущую рабочую папку
mount /dev/nvme0n1p1 /mnt/boot
```
### Сборка ядра и базовых софтов
```bash
# Устанавливаем базовые софты
pacstrap -K /mnt base linux linux-firmware base-devel lvm2
dhcpcd net-tools iproute2 networkmanager vim micro efibootmgr iwd

# Генерируем fstab
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab

# Настройка системы
arch-chroot /mnt

# Нужно раскомментировать ru_RU и en_US в этом файле
micro /etc/locale.gen

# Генерируем локали
locale-gen

# Настраиваем время
ln -sf /usr/share/zoneinfo/Europe/Kiev /etc/localtime
hwclock --systohc

# Указать имя хоста
echo "arch" > /etc/hostname

# Укажите пароль для root пользователя
passwd

# Добавляем нового пользователя и настраиваем права
useradd -m -G wheel,users,video -s /bin/bash user
passwd user
systemctl enable dhcpcd
systemctl enable iwd.service

micro /etc/mkinitcpio.conf
# Пересборка ядра. Найдите строку HOOKS=(base udev autodetect modconf kms
# keyboard keymap consolefont block filesystems fsck)

# и замените на:

# HOOKS=(base udev autodetect modconf kms keyboard keymap consolefont block filesystems encrypt lvm2 fsck)

# Запустить процесс пересборки ядра
mkinitcpio -p linux
```

### Установка загрузчика
```bash
bootctl install --path=/boot
cd /boot/loader
micro loader.conf

# Вставляем в loader.conf следующий конфиг:
timeout 3
default arch

# Создаем конфигурацию для запуска
cd /boot/loader/entries
micro arch.conf

# Вставляем в arch.conf следующее:
# UUID можно узнать командой blkid
title Arch Linux by ZProger
linux /vmlinuz-linux
initrd /initramfs-linux.img
options rw cryptdevice=UUID=uuid_от_/dev/nvme0n1p2:main root=/dev/mapper/main-root

# Выдаем права на sudo
sudo EDITOR=micro visudo
# После открытия раскомментируйте %wheel ALL=(ALL:ALL) ALL

# Выходим из системы и перезагружаемся
Ctrl+D
umount -R /mnt
reboot
```

### Установка Hyprland окружения
```bash
# Базовые пакеты
sudo pacman -Syu
sudo pacman -S git fish

# Установка AUR-хелпера paru
git clone https://aur.archlinux.org/paru.git

cd paru

makepkg -si

# Установка Hyprland и зависимостей
paru -S hyprland xdg-desktop-portal-hyprland xdg-desktop-portal-gtk \
hyprpicker hypridle wl-clipboard cliphist bluez-utils inotify-tools \
app2unit wireplumber trash-cli foot fastfetch starship btop jq socat \
imagemagick curl adw-gtk-theme papirus-icon-theme qt5ct qt6ct ttf-jetbrains-mono-nerd
```

### Установка caelestia dotfiles
```bash
git clone https://github.com/caelestia-dots/caelestia.git ~/.local/share/caelestia

cd ~/.local/share/caelestia

fish install.fish --paru --noconfirm --vscode=codium --spotify --discord --zen

# ⚠️ Не удаляй папку ~/.local/share/caelestia — там все symlink’и
```


### Установка логин-менеджера
```bash
paru -S greetd tuigreet
sudo micro /etc/greetd/config.toml

# Пример:
[terminal]
vt = 1

[default_session]
command = "tuigreet --time --cmd Hyprland"
user = "user"

sudo systemctl enable greetd
```

### Установка дополнительных зависимостей shell (manual)
```bash
paru -S caelestia-cli quickshell-git ddcutil brightnessctl cava networkmanager lm-sensors aubio libpipewire glibc qt6-declarative gcc-libs material-symbols caskaydia-cove-nerd swappy libqalculate bash qt6-base qt6-declarative app2unit fish cmake ninja

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
```

### Настройка shell и обоев
```bash
# Создаём ~/.config/caelestia/shell.json
# Здесь можно добавить рекомендуемые опции shell

# Wallpapers по умолчанию: ~/Pictures/Wallpapers
# Чтобы установить обои:
caelestia wallpaper -f ~/Pictures/Wallpapers/<file>
# Для динамической цветовой схемы:
caelestia scheme set -n dynamic

# PFP/dashboard: ~/.face
```
### Автостарт shell
```bash
Если установлены все caelestia dots, shell запускается автоматически через Hyprland `exec-once`.
Иначе вручную:
caelestia shell -d
# или
qs -c caelestia
```

### Исправления и кастомизация Hyprland
```bash
# Отключение мерцания (VRR):
nano ~/.config/caelestia/hypr-user.conf
misc {
    vrr = 0
}

# Добавление своих конфигов Hyprland:
# ~/.config/caelestia/hypr-user.conf
```
