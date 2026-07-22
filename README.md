# The Arch Linux Experience

Insert meme

## Installation

### Getting the ISO

Downloading via the the torrent from the [arch download page](https://archlinux.org/download/)

### Creatting the installation medium

#### From Windows :

Using Rufus as described in the [arch wiki]( https://wiki.archlinux.org/title/USB_flash_installation_medium#Rufus)

> Making it usable for both BIOS and UEFI by selecting `MBR` and `BIOS or UEFI`
>
> Allowing the download of the missing `sys` files by Rufus

#### From Linux :

[dd command](https://wiki.archlinux.org/title/USB_flash_installation_medium#In_GNU/Linux_2)

### Booting the live environment

Spam `F12` to access boot menu and choose the usb drive

If a `Secure Boot Violation` message appears: go to BIOS by spamming `F2` and going in the `Security` tab.

#### Setting the keyboard layout to bépo

> loadkeys fr-bepo

#### Setting the font

If font is to small use:

> setfont ter-132b
>
>> suggested large font for HiDPI screens by the [arch wiki]( https://wiki.archlinux.org/title/Installation_guide#Set_the_console_keyboard_layout_and_font)
>
>> alternatively, `setfont -d` or `setfont --double` to double current font size

#### Setting up the internet connection

Making sure the Wi-Fi network adaptor is not hard or soft blocked (try again after flipping the switch, if needed)

> rfkill

> rfkill unblock wifi
>
>> if needed

Checking the network interfaces

> ip link

Entering iwd interactive prompt (to avoid having to ad `iwctl` in front of every iwd command:

> iwctl

Checking the device name

> `[iwd]#` device list

If powered off:

> `[iwd]#`device `name` set-property Powered on

Scan for networks with the device:

> `[iwd]#` station `name` scan

Then list available networks:

> `[iwd]#` station `name` get-networks

Connect to desired network:

> `[iwd]#` station `name` connect `SSID`
>
>> if network hidden: `[iwd]#` station `name` connect-hidden `SSID`

And enter passphrase (all `-` in the default ISP passphrase must be typed in)

Making sure the connection was successful (no `Poeration failed` message):

> `[iwd]#` station `name` show

#### Update system clock:

> timedatectl

### Install preparation

#### On work laptop - ASUS ExpertBook P3 PM3406CKA
14″ 1920x1200@60 TN
1TiB WD M2 PCIe4
32GB (Max 64)
AMD Ryzen AI 7 350 (4+4/8+8@2-5GHz+2-3,5GHz; 16Mo cache ; Max DDR5@256GB bi-channel)
AMD Radeon™ 860M (3000MHz FreeSync; Max 4 displays)

Only one drive: NVMe. Requiered to by encrypted.

1.	Sanitizing the NVMe to ensure proper removal of factory OS (https://wiki.archlinux.org/title/Solid_state_drive/Memory_cell_clearing#NVMe_drive)
2.	Changing the NVMe sector size to 4096 (if relevant) as recommended for dm-crypt by [arch wiki]( https://wiki.archlinux.org/title/Advanced_Format#dm-crypt)

> will wipe the disk (full of zeros) but for stronger encryption (https://wiki.archlinux.org/title/Solid_state_drive#dm-crypt) (https://wiki.archlinux.org/title/Securely_wipe_disk#Preparations_for_block_device_encryption) and [plausible deniability](https://wiki.archlinux.org/title/Dm-crypt/Specialties#Discard/TRIM_support_for_solid_state_drives_(SSD)) the disk must then be filled with RNG

3.	No TRIM, Additional Drive preparation with RNG wipe (will forbid the use of ssd TRIM which may or may not impact performances and is thus disabled by default): 
https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation 

> Hynix PC601, Lenovo and Dell laptops, Samsung and WD SN750 drive may not `nvme format` command and require a 

4.	plain dm-crypt on LVM: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Plain_dm-crypt 
OR: LUKS on LVM https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#LUKS_on_LVM 

Given the threat model the company faces, the “weaker” encryption / “better” performances model of LUKS on LVM and enabled TRIM is adopted.

Swap is encrypted at each reboot but /boot remains unencrypted.
Add UEFI secure boot ?

**NVMe sanitization**

`fdisk -l` to verify devices, sector size, partitions

`nvme sanitize /dev/nvme0 -a start-block-erase` to start wipe

`nvme sanitize-log /dev/nvme0` to follow progress

> `Sanitize Status : 0x101` is successful complete
>
> `(No time period reported)` is several hours…

**NVMe sector size optimization**

`nvme id-ns -H /dev/nvme0n1 | grep "Relative Performance"` to check 4KiB sector size compatibility

`nvme format --lbaf=1 /dev/nvme0n1` to make the change

> `lbaf` value is given in the previous command’s output

##### LUKS on LVM

**Partitioning**

`cfdisk /dev/nvme0n1`, to create partitions with automatic partition alignment in a TUI

>Select `gpt`
>
> Select `[ New ]` on `Free space` and set `Partition size` to `1G`
>
> Select `[Type]` and then `EFI System`
>
> Again on `Free space`, select `[ New ]` and leave partition size as is
>
> Then select `[ Write ]` and enter `yes`
>
> Finally select `[ Quit ]`

**Preparing the logical volumes**

`pvcreate /dev/nvme0n1p2`

`vgcreate MyVolGroup /dev/nvme0n1p2`

`lvcreate -L 32G -n cryptswap MyVolGroup`

`lvcreate -L 32G -n cryptroot MyVolGroup`

`lvcreate -l 100%FREE -n crypthome MyVolGroup`

`cryptsetup luksFormat /dev/MyVolGroup/cryptroot`

> type `YES`
>
> enter passphrase and verify

`cryptsetup --allow-discards --persistent open /dev/MyVolGroup/cryptroot root`

> enter passphrase

See https://wiki.archlinux.org/title/Dm-crypt/Specialties#LUKS2 

**Formatting**

`mkfs.ext4 /dev/mapper/root`

`mkfs.fat -F32 /dev/nvme0n1p1`

**Mounting**

`mount /dev/mapper/root /mnt`

`mkdir /mnt/boot`

`mount /dev/nvem0n1p1 /mnt/boot`

**Configuring mkinitcpio**

`pacman -S lvm2` to make sure it’s installed

For system-based initramfs edit `/etc/mkinitcpio.conf`

> Add `sd-encrypt` and `lvm2` to the already uncommented line in the following order:
>
> `HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)`
>
>> `keyboard` and `sd-console` might also need to be added

`mkinitcpio -P` to regenerate the initramfs image

> The `systemd` hook already provides a resume mechanism https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Configure_the_initramfs 

**Configurating the boot loader**
`lsblk -f` to check the UUID of the LUKS superblock: `/dev/MyVolGroup/cryptroot`

> 472cb548-801b-408d-8845-ce93b0b7e649

Edit `/etc/default/grub` by adding the following to the `GRUB_CMDLINE_LINUX_DEFAULT=`:

`rd.luks.name=<cryptroot_UUID>=root root=/dev/mapper/root resume=/dev/mapper/swap`

**Configuring fstab and crypttab to re-encrypt /swap on each reboot**

> plain dm-crypt, see: https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Preparing_the_disk_4 

Uncomment and edit the `etc/crypttab` line starting with `swap` to unlock it (add option `discard` as fourth field):

> `swap	/dev/MyVolGroup/cryptswap	/dev/urandom	swap,cipher=aes-xts-plain64,size=256,sector-size=4096	discard`

`lsblk -f` to check the UUID of `nvme0n1p1`

> 45BB-673F

Edit `etc/fstab` and add the following lines to mount it:

> /dev/mapper/root	/	ext4	defaults	0	1
>
> UUID=<nvme0p1n1_UUID>	/boot	vfat	defaults	0	2
>
> /dev/mapper/swap	none	swap	discard	0	0

See https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption#systemd-based_initramfs and https://wiki.archlinux.org/title/Dm-crypt/Specialties#LUKS1_and_plain_dm-crypt 


**Encrypting /home**

To be done after rebooting the installed OS https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Encrypting_logical_volume_/home 

#### On personal laptop (Broadcom wireless controller BCM4311)

SSD + HDD

Swap with hibernate and no encryption; enabled TRIM

> See https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation


Unsupported device for linux, including by the mainline driver and the modern reversed engineered driver: must use the b43-firmware-classicAUR package.

> Installation must be done via Ethernet or wi-fi usb dongle. See [Download desired packages]( #### Download-desired-packages)

Needs revisioning:
**Partitioning discs**

`lsblk` to list all block devices and confirm discs names

`cfdisk /dev/<name>` to partition with a curser-based UI (TUI) of `fdisk`

> new discs may prompt for a label type (gpt, dos, sgi, or sun)
>
>> choose gpt (GUID Partition Table) for a UEFI install

> For single-OS boot delete all existing partitions 
>
> For multi-OS boot resize largest

-	If booting in UEFI and EFI system partition not already existing (in case of multi-boot)
`/boot`: 1GB
-	If desired
`[swap]`: at least 4GB
-	`/`: remainder of the device

**Formatting**

For the EFI system partition (if newly created, [see Partitioning discs](url#Partitioning-discs):

`mkfs.fat -F 32 /dev/<efi_partition>`

> Must unavoidably be FAT32

For the root partition :

For the swap

`mkfs.ext4 /dev/

**Mounting**

Mount root partition first:

Make the `/boot` directory to mount the boot partition on:

`mkdir /mnt/boot`

And mount it on:

`mount /dev/boot_partition /mnt/boot`

`mount /dev/swap_partition /mnt/swap`

`swapon --discard /dev/swap_partition` to enable swap volume

setup hibernation https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Configure_the_initramfs 


#### On personal desktop

SSDs + HDDs

Swap with hibernate and no encryption; enabled TRIM


### Installing the OS

#### Selecting the mirrors

Edit `/etc/pacman.d/mirrorlist` by moving the geographically closest mirrors to the top of the list

In nano:

> Move to a desired block of text with `Ctrl` + `Down_arrow`
>
> Set a mark with `Alt` + `A`
>
> Move to the start of the next block with `Ctrl` + `Down_arrow`
>
> Cut the block with `Ctrl` + `K`
>
> Move to the top of the file with `Alt` + `\` or `Ctrl` + `Up_arrow`
>
> Paste the block in the desired position with `Ctrl` + `U`

#### Allowing for parallel download

For fast arch installation and faster packet downloads in general, edit `etc/pacman.conf`:

> `#ParallelDownloads = 5`
> 
>> Uncomment the line by deleting the `#` and change the value to the number of threads the CPU has? Or set between 3 and 5

This will have to be done again in the [base configuration step](#base-configuration) as no configuration (except for /etc/pacman.d/mirrorlist) gets carried over from the live environment to the installed system.

#### Download desired packages

`pacstrap -K /mnt base base-devel linux linux-firmware amd-ucode vi vim nano zsh man-db man-pages networkmanager iwd lvm2 cryptsetup grub efibootmgr`

> add terminus-font and vi
>
> add timesyncd?
>
> remove iwd?

#### Fstab

`genfstab -U /mnt >> /mnt/etc/fstab`

> Check `/mnt/etc/fstab` and edit it in case of errors
>
>> fab8e35b-2d27-42ce-b987-f6f926c102e6

#### Chroot

Chroot with active dbus connection:

`arch-chroot -S /mnt`

#### Time

`ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime`

Generate `/etc/adjtime`:

`hwclock --systohc`

Set up time synchronization

`system-timesyncd`

Enable NTP:
sudo timedatectl set-ntp true

#### Localization

Edit `/etc/locale.gen` by uncommenting needed UTF-8 locales:

> `en_US.UTF-8 UTF-8`
>
> …
>
> `fr_FR.UTF-8 UTF-8`
>
>> `en_US.UTF-8 UTF-8` is recommended as a fallback for various tools

Then generate locales with:

`locale-gen`

Create `/etc/locale.conf`:

> LANG=fr_FR.UTF-8
>
> LANGUAGE=en_US:en
>
>> Fallbacks separated by `:` (`:C:` if adding other languages after English)
>
> LC_MESSAGES=en_US.UTF-8

Console fonts are located in `/usr/share/kbd/consolefonts/`

> `ls /usr/share/kbd/consolefonts/ 2>&1| less` to scroll through the list
>
>> `q` to quit `less`

Set Latin-9 (8859-9), a latin-1 with full french coverage:

`setfont --double lat9w-16`

To set with persistence, edit `/etc/vconsole.conf`:

> `KEYMAP=bepo-fr`
>
> `FONT=lat9w-16`
>
>> Or install `terminus-fonts` and use `ter-132b` (bold) or `ter-132n` (normal)

#### Network

Edit `/etc/hostname`

> <hostname>

-	arch
-	btw
-	…

Networkmanager

`systemctl enable NetworkManager`

Enabling iwd’s network configuration feature (once installed):

> echo -e “[General]\nEnableNetworkingConfiguration=true” > /etc/iwd/main.conf
>
>> `-e` instructs to interpret the newline character: `\n`

See [iwd]( https://wiki.archlinux.org/title/Iwd#Optional_configuration) of the arch wiki

##### On personal laptop (Broadcom wireless controller BCM4311)

Unsupported device for linux, including by the mainline driver and the modern reversed engineered driver: must use the b43-firmware-classicAUR package.

> To download it and it’s dependencies, connect via Ethernet or wi-fi usb dongle.

Installing yay dependencies

Installing yay

Installing b43 dependency

Installing b43 classic

[Blacklisting]( https://wiki.archlinux.org/title/Kernel_module#Blacklisting) the erroneous kernel driver as advised by the [arch wiki]( https://wiki.archlinux.org/title/Broadcom_wireless#b43):
Some modules are loaded as part of the initramfs. mkinitcpio -M will print out all automatically detected modules: to prevent the initramfs from loading some of those modules, blacklist them in a .conf file under /etc/modprobe.d and it shall be added in by the modconf hook during image generation. Running mkinitcpio -v will list all modules pulled in by the various hooks (e.g. filesystems hook, block hook, etc.). Remember to add that .conf file to the FILES array in /etc/mkinitcpio.conf if you do not have the modconf hook in your HOOKS array (e.g. you have deviated from the default configuration), and once you have blacklisted the modules regenerate the initramfs, and reboot afterwards.

Then resume connecting to wifi or proceed with alternative internet connection and deal with wlan0 later.

#### Initramfs

For system-based initramfs edit `/etc/mkinitcpio.conf`:

> `HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)`
>
>> add `sd-encrypt` and `lvm2`

Regenerate the initramfs image:

`mkinitcpio -P`

#### Root password

Set to none and lock root.

First create user

`useradd -m -G wheel,users <user>`

-	admin
-	sysadmin
-	god
-	godmod
-	iddqd
-	godlovesecretsex
-	arch
-	raven
-	corvus
-	simon

@:
-	enroute
-	raven
-	arch
-	btw

Set password

`passwd <user>`

Install sudo (and visudo) package(s) (add to pacstap ?):

`pacman -S vi`

Grant wheel group all access:

`visudo /etc/sudoers.d/10-wheel`

> `%wheel	ALL=(ALL:ALL) ALL`

Log as the user

`su <user>`

> enter password

Lock root account

`sudo passwd -l root`

> if already existing delete and lock:
>
> `sudo passwd -dl root`

#### Boot loader

Install grub

`grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB`

> required grub and efibootmgr packages already downloaded by previous `pacstrap` command

https://wiki.archlinux.org/title/GRUB 

Generate `grub.cfg`:

`grub-mkconfig -o /boot/grub/grub.cfg`

https://wiki.archlinux.org/title/GRUB#Generated_grub.cfg

> If `grub-probe: error: failed to get canonical path of /dev/<device>`, run:
>
>> `arch-chroot -S /mnt grub-mkconfig -o /boot/grub/grub.cfg `

#### Reboot

Exit chroot environment:

`exit`

Unmount all partitions:

`umount -R /mnt`

> use `fuser` in case of busy partitions

`reboot now` and remove installation medium

Login with created user

## Post-installation

### Encrypting logical volume /home on work laptop

Creating keyfile to avoid entering a second passphrase at boot:

`dd bs=512 count=4 if=/dev/random iflag=fullblock | install -m 0600 /dev/stdin /etc/cryptsetup-keys.d/home.key`

Encrypting:

`cryptsetup luksFormat -v /dev/MyVolGroup/crypthome /etc/cryptsetup-keys.d/home.key`

`cryptsetup -d /etc/cryptsetup-keys.d/home.key --allow-discards --persistent open /dev/MyVolGroup/crypthome home`

Mounting:

`mkfs.ext4 /dev/mapper/home`

`mount /dev/mapper/home /home`

Editing `/etc/crypttab`:

`home	/dev/MyVolGroup/crypthome	none`

Editing `/etc/fstab`:

`/dev/mapper/home	/home	ext4	defaults	0	2`

### Hardening

#### Users

https://wiki.archlinux.org/title/Sudo#Harden_with_sudo_example


## Customisation

### Buffir API

GBM : supports all wayland compositors for all all GPU drivers except NVIDIA < 495 (GTX 1060 at 560)

Run `journalctl -b 0 --grep "renderer for"` to check it

If needed force GBM as a backend by setting the following environment variables in `~/.config/environment.d/envvars.conf` [wayland session EV]( https://wiki.archlinux.org/title/Environment_variables#Per_Wayland_session):

> GBM_BACKEND=nvidia-drm
>
>__GLX_VENDOR_LIBRARY_NAME=nvidia



### Wayland compositor

-	Hyprland (the Only with HDR support and good squircles support ; not AUR)
-	Mango (dwl-based so dwm+wlroots, with eye-candy ; AUR…)
-	Sway (i3 compatible tilling based on wlroots ; not AUR)
-	SwayFX (Sway with eye-candy ; AUR)
-	Niri (scrollable tilling ; not AUR)
-	

	Hyprland pre-configs
-	HyDE
-	Zenities

### DE-like utilities

-	Login manager
o	greetd (minimal and flexible login deamon that runs on way, X and tty)
o	tuigreet
o	lemurs (TUI display manager in Rust)
o	ly (TUI display manager in Zig)
o	emptty (simple CLI display manger on tty)
o	lidm (fully colorful customizable TUI display manager in C from AUR)

> Check `/usr/share/wayland-sessions/compositor.desktop` to see display manger support.

See [arch wiki]( https://wiki.archlinux.org/title/Display_manager)

-	Locking and Idling
o	hyprlock+hypridle
-	screen capture
-	Glass magnifier zoom
o	https://wiki.hypr.land/Configuring/Advanced-and-Cool/Uncommon-tips-and-tricks/#glass-magnifier-zoom
-	Color picker
o	https://wiki.hypr.land/Useful-Utilities/Color-Pickers/#hyprpicker
-	

See :
-	wayland tips and tricks from the [arch wiki]( https://wiki.archlinux.org/title/Wayland#Tips_and_tricks).
-	Hypr [ecosystem](https://wiki.hypr.land/Hypr-Ecosystem/)
-	Hypr [utilities]( https://wiki.hypr.land/Useful-Utilities/)

### GUI libraries
-	Electon
-	GTK
-	Gt
… See [arch wiki](https://wiki.archlinux.org/title/Wayland#GUI_libraries)

### Layouts

-	Spiraling (newest in the main panel and the oldest spiraling)

Per workspace layouts
https://wiki.hypr.land/Configuring/Advanced-and-Cool/Uncommon-tips-and-tricks/#per-workspace-layouts

Display specific workspaces

	Other functionalities
-	sticky tile: follows in all workspaces until toggled off and goes back to original WS

### Compositing (animations, transparency, blur…)

Keybind to toggle compositing 
https://wiki.hypr.land/Configuring/Advanced-and-Cool/Uncommon-tips-and-tricks/#toggle-animationsbluretc-hotkey

### Terminal emulator

-	kitty (fast, GPU accelerated, themable)
-	alacrity (rust, cross-platform, no image or ligatures)
-	ghostty (zig coded and kitty based for image rendering)
-	wezterm

### Terminal utilities

-	Text editors
o	Neovim
-	Shells
o	zsh
-	File explorer
o	ranger (vim inspired image preview and metadata)
o	If (ranger inspired, Go)
o	LazyVim (neovim plugin, custom title page)
o	Yazi (file prieview, blazing fast, Rust)
o	superfile (fancy and modern)
o	nnn (unorthodox)
-	MD renderer (neovim plugin)
-	PDF renderer
-	Image viewer
o	feh (X11 only?)
o	xwgv (X11 only?)
-	Ressource manager
o	htop (base)
o	btop
o	btm
-	Specs flaunting (better fetch commands)
o	neofetch (great support for images)
o	fastfetch (neofetch in C instead of bash) = ff
o	pfetch (smaller fastfetch)
o	bunnyfetch (cute AF)
o	nitch (fetch in nim)
o	rxfetch
o	tmux (terminal multiplexer ?)
o	ufetch
o	fm6000
o	uwufetch
-	Clock
o	tty-clock
-	Calendar and task manager
o	calcure (vim keybinds)
o	calcures (calcure predecessor)
-	Music player
o	ncmpp (built in visualizer
o	ncspot (tracks, albums, artists, playlists, podcast, browse, visualizer…)
o	termusic (album art, poor customization, bad resizing)
o	 rmpc (mpd client, high customization, top bar with find and playlists and other customizable tabs, customizable header bar with repeat and randomize, ranger style tri-panel navigation, pywall integrable, vim keybinds by default, synchronized lyrics support)
o	yt-x (youtube)

### Terminal decorations

-	cmatrix
-	tmatrix
-	cava (music visualizer)
-	asciiquarium
-	pipes.sh
-	cbonsai
-	k-vernooy/tetris(?)
-	rose(?)
-	tux
-	pokeget
-	lava lamp?
-	terminal-rain
-	weathr

Can be auto-exec on each launch

### Launcher

Possible to have multiple style and also to browse installed styles on each launch

-	dmenu
-	wmenu?
-	rofi
-	wofi
-	tofi
-	

### Graphical File Managers

-	dolphin: File manager by KDE.
-	nautilus: File manager by Gnome.
o	nautilus-admin-gtk4: Open files with elevated privileges.
o	nautilus-image-converter: Resize and rotate images.
o	nautilus-open-any-terminal: Open terminals in selected directory.
-	nemo: File manager by Cinnamon.
o	nemo-fileroller: File archiver extension.
o	nemo-terminal: Embedded terminal window.
-	pcmanfm-qt: File manager by LXQt.
-	pcmanfm: File manager by LXDE.
-	thunar: File manager by XFCE.

### Color Themes

-	catppuccin mocha
-	nordtheme
-	biscuit
-	gruvbox-material
-	swamp
-	decay
-	everblush
-	javacafe ghost

### Wallpaper

hypaper

mpvpaper: moving wallpaper?

Dynamic theming from wallpaper colors
$ pip3 install pywal
$ wal -i /path/to/wallpaper.png
OR wallust

Keybind wallpaper selector script (including cycling and/or random toggle modes)
	See hayyaoe/zenities

### Icons

-	papirus (graphical icons for apps)
-	phingers (cursors)

### Cursors

### Status bar

-	yasb
-	waybar
-	polybar (for X11)
-	ewwbar
-	Hyprpanel

Elements
-	workspaces with thematic icons and highlight
-	selection name
-	keymaps
-	clock, date
-	resources
-	sound
-	music player
-	network
-	weather
-	(battery)
-	apps (running: “traybar”, “taskbar”, “toolbar”… and/or short-cut launch: “favorite apps”…)
-	menu (widgets and power options… animations, wallpaper, theme, waybar, worflows? lockscreen, shaders? keybinds…)


### Notifications

-	hyprdots

Deamons
-	Dunst
-	Fnott
-	mako
-	swaync

### Widgets

-	AGS/Astal (JS+CSS)
-	Eww
-	Quickshell (QML… a bit complicated…)

Can be toggled shown/hidden with a bindkey

Functions
-	sound volume
-	brightness cursor
-	networks
-	Bluetooth
-	toggle idle mode
-	screen capture
-	resources monitor
-	music player with cover art
-	calendar => terminal? (calcure)
-	power/lock options (keybind)

### Easter eggs

-	Welcome to the rice fields motherfucker
-	singing/bango/banging… cat
-	dancing gif (get recked, etc.)
-	NinjasSays(JDG haiku)
-	Lokhir
-	Food cats
-	Kay Linit
-	Automatic gif or ascii-says in special tile with running custom program when running or opening specific files in other tiles?

## GUI Apps

-	Browsers
o	Firefox
o	Firefox Dev
o	ZenBrowser
o	Mullvad
o	Tor
-	File download
o	Deluge (Torrent)
o	FileZilla?
o	Video Download?
-	Social
o	Mail client: Thunderbird (+Proton?)
o	Session
o	Discord alternative
o	Libreddit
-	Office productivity
o	Writer (LibreOffice word processor)
o	Impress (LibreOffice presentation)
o	Calc (LibreOffice spreadsheet)
o	Calculator (Qalculate! or KCalc)
o	Geogebra (Math editor)
o	Obsidian (notetaking)
o	PDF editors (Apache PDFBox, PDF Arranger, PDFedit, Pdftk…)
o	Inkscape (vector graphics)
o	GIMP or Krita (raster graphics editor)
o	Image viewer (Eye of GNOME, Geeqie, 
o	Blender (3d object editor)
o	Audacity (Digital audio editor)
o	Music production (LMMS, Rosegarden, MusE Sequencer…)
o	OBS (screencasting and live streaming)
o	OpenShot Video Editor
o	Game engines (Godot…)
o	Databases managers
-	Gaming
o	Steam
o	Emulators
	Dolphin
	PSX2
o	Solo installs
-	Virtualization
o	Waydroid? (Android in lxc)
o	VirtualBox
-	Other
o	iCUE or OpenRGB
o	Logitech G Hub

## Compatibility

### Keyboard

Device it self (Corsair K70 PRO MINI WL) : OOTB (RF)
Layout : QWERTY by default, fr-bepo available OOTB

> loadkeys fr-bepo

iCUE

### Headset

### Mouse

### Displays

### Games

Emulators
Steam

### 

## Other

### Boot loader?

-	GRUB
-	Limine
-	Plymouth
-	rEFInd

### File system
-	ZFS (goated?)
-	btrfs?

### Init system/Initialization manager

-	systemd (only officially supported by arch ; BSD, Solaris, Alpine, Android, Gentoo, Slackware, Void… are systemd-less)
-	openRC
-	busybox

