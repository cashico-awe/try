# 概要

外付けSSDにArchLinuxインストールしたので備忘録。情報が散らかってしまったのでまとめる。
https://wiki.archlinux.jp/index.php/%E3%83%AA%E3%83%A0%E3%83%BC%E3%83%90%E3%83%96%E3%83%AB%E3%83%A1%E3%83%87%E3%82%A3%E3%82%A2%E3%81%AB_Arch_Linux_%E3%82%92%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB

### 完成形

- マルチブートUSBドライブを作成する
- 外付けSSDにArchLinuxをインストールする
	- amdおよびintel系で起動でき、別のPCでも接続するだけで起動できるようにする
	- UEFIのみ、BIOSでの起動には対応しない
	- 最低限のセットアップ
		- yayを導入する
		- 非特権ユーザ
		- Wifi使用可能
		- デスクトップ環境には対応しない
		- 日本語には対応しない(en_US.UTF-8)
tres

# 1. ArchLinuxインストーラの起動

### 1.1 マルチブートUSBドライブの作成
https://wiki.archlinux.jp/index.php/%E3%83%9E%E3%83%AB%E3%83%81%E3%83%96%E3%83%BC%E3%83%88_USB_%E3%83%89%E3%83%A9%E3%82%A4%E3%83%96

```bash
// USBドライブの先頭パーティションの確認とフォーマット
# lsblk
# mkfs.fat -F 32 /dev/sdX1

// GRUBのインストール
# mount /dev/sdX1 /mnt
# mkdir /mnt/boot
# grub-install --target=i386-pc --removable --boot-directory=/mnt/boot /dev/sdX1
```

grub-installのオプション **--removable** を省略しないこと。
grubはデフォルトでNVRAMを書き換えるので、予想外の影響が出る可能性がある。

### 1.2 インストーラの配置と起動

ArchLinuxインストーラのisoファイルをダウンロードしておく。
https://www.archlinux.jp/download/
```bash
// 後で打ち込むとき面倒なので、isoファイルの名前を変更する
# mv [インストールしたisoファイルのパス] ~/arch.iso

// iso配置
# mkdir /mnt/boot-isos
# cp ~/arch.iso /mnt/boot-isos
```
1. パソコンを再起動する
2. F12を連打してBIOS boot optionに入る(Dellの場合。UEFIなのにBIOSと表示される)
3. USBドライブの名前を選択してエンター。
4. GRUBのシェルが表示される。
以下、GRUBシェル
```
grub> ls /
boot boot-isos ...
grub> loopback loop /boot-isos/arch.iso
grub> set iso_path=/boot-isos/arch.iso
grub> export iso_path
grub> root=(loop)
grub> configfile /boot/grub/loopback.cfg
```
ArchLinuxインストーラが起動してシェルに入れたら次の手順に進む。
起動しなかった場合、次のコマンドでGRUBシェルから抜け出せる。
```
grub> halt
```

# 2. ArchLinuxを外付けSSDにインストール

以降、上で起動したArchLinuxインストーラ内で作業する。

### 2.1　ArchLinuxインストーラの環境構築

```bash
# loadkeys jp106
# timedatectl status
```
有線LAN接続してる場合、すでにインターネットに接続できていると思われる。以下は無線接続の手順。
```bash
# iwctl device list
# iwctl station wlan0 connect SSID
// password入力

// 次のコマンドでconnectedになってたら成功
# iwctl station wlan0 show

// pingが通るのがわかったら ctrl+c
# ping archlinux.jp
```

### 2.2 パーティションの作成とフォーマット

次のような構成にする。swapパーティションは作らない。
- /dev/sdY
	- SSD
- /dev/sdY1
	- EFI Sytem Partition
	- fat32 ラベル：ESP
	- 5GB
- /dev/sdY2
	- Linux Root Partition
	- ext4 ラベル：ARCH
	- 100GB

:::message alert
SSD内のデータは、次の手順で、w(書き込み)を入力し、決定した時点ですべて削除される。変更せずに終了したい場合は、qを入力する。
:::

```bash
# fdisk /dev/sdY

// 以下、fdiskシェル
command: g

command: n
partition number: (enter でデフォルトの1になる)
first sector: (enter でデフォルトを選択)
last sector: +5G
command: t
target: 1
partition type: 1(EFI System Partition)

command: n
partition number: (enter でデフォルトの2になる)
first sector: (enter でデフォルトを選択)
last sector: +100G
command: t
target: 2
partition type: 23(Linux Root Partition x86_64)

command: w
```
フォーマットする。
```bash
# mkfs.fat -F 32 -n ESP /dev/sdY1
# mkfs.ext4 -L ROOT /dev/sdY2
```

### 2.3 ArchLinuxインストール

```bash
# mount /dev/sdY2 /mnt
# mount --mkdir /dev/sdY1 /mnt/boot
# refrector --country Japan --sort rate --save /etc/pacman.d/mirrorlist
# pacstrap -K /mnt base linux linux-firmware
```

### 2.4 ArchLinuxの最低限のセットアップ

```bash
# genfstab -U /mnt > /mnt/etc/fstab
# arch-chroot /mnt
//以下、chroot内

root# pacman -S vim
root# pacman -Syyu

// en_US.UTF-8 UTF-8をアンコメント
// 必要に応じてほかのロケールもアンコメント
root# vim /etc/locale.gen
root# locale-gen

root# ln -fs /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
root# hwclock --systohc
root# echo "LANG=en_US.UTF-8" > /etc/lcoale.conf
root# echo "KEYMAP=jp106" > /etc/vconsole.conf
root# echo "myhostname" > /etc/hostname

root# vim /etc/mkinitcpio
root# mkinitcpio -P
root# pacman -S amd-ucode intel-ucode
root# bootctl --path=/boot install
```

ブートローダの設定
```bash
// 新しくarch.confを追加しエントリを作成。
root# vim /boot/loader/entries/arch.conf
```
arch.confを以下のように編集。
```conf:arch.conf
title archlinux
linux /vmlinuz-linux
initrd /amd-ucode.img
initrd /intel-ucode.img
initrd /initramfs-linux.img
option root="LABEL=ARCH" rw
```

セットアップ終了。chrootおよびインストーラから抜ける。
```bash
root# exit
# shutdown
```

シャットダウンをキャンセルしたりしたい場合は、以下を参照。
```bash
// シャットダウンのキャンセル
# shutdown -c

// 即シャットダウン
# shutdown -h 0
```

# 3. インストール後のセットアップ

### 3.1 非特権ユーザ作成、パスワード設定

```bash
// rootのパスワード設定
# passwd
newpassword: [好きなパスワード]
retype password: [好きなパスワード]

// 非特権(sudo可能)ユーザ作成
# useradd [ユーザ名] -m -g wheel
# passwd [ユーザ名]
newpassword: [好きなパスワード]
retype password: [好きなパスワード]

// sudoerの"%wheel: ALL(ALL:ALL)"をコメントアウト
# vim /etc/sudoer
```

### 3.2 Wifi環境

```bash
# systemctl iwd start
# iwctl device list
# iwctl station wlan0 connect [SSID]
// password入力
# iwctl station wlan0 show
# systemctl enable iwd
```

### 3.3 yay導入

```bash
# pacman -S --needed git base-devel
# git clone https://aur.archlinux.org/yay-bin.git
# cd yay-bin
# chmod 777 .
# su [非特権ユーザ名]

// 以下、非特権ユーザとして実行
$ makepkg -si
// yayの確認
$ yay
```

# 補足

### ファームウェアとUEFI

https://wiki.archlinux.jp/index.php/%E3%83%91%E3%83%BC%E3%83%86%E3%82%A3%E3%82%B7%E3%83%A7%E3%83%8B%E3%83%B3%E3%82%B0

ファームウェアとは、PCの起動プロセスで最も最初に起動するプログラムのこと。マザーボードに各メーカーが実装している。

- UEFI
	- 新しいほう。最近のPCは大体これ。
	- 仕様が定義されており、ネット上で公開されている。
	- GUID Partiton Table (GPT)
	- EFI System Partition
		- システムの起動に必要なものを配置するパーティション
			- ブートローダ
			- カーネルイメージ
			- 初期RAMファイル
- BIOS
	- 古いほう
	- 単なるデファクトスタンダードだったもので、明確な仕様が定義されていない
	- Master Boot Record (MBR)
	- ブートローダとかは、先頭512byteに存在する必要がある

マルチブートUSBドライブの作成で行ったように、ブートローダは必ずしもESPに存在する必要はない。ESPかどうかを定義するのはパーティションのGUIDであり、ファームウェアがパーティションを見つけやすくする役割しかない。GPTのパーティションタイプ全般に同様のことがいえる。
https://www.happyassassin.net/posts/2014/01/25/uefi-boot-how-does-that-actually-work-then/

### ArchLinuxインストーラ

軽量のArchLinux。pacstrapやarch-install-scriptsなどインストール用のツールが入っている。
通常のOSインストーラと同様にisoファイルとして提供されており、ddやRufusなどでUSBに焼いたり、VirtualBoxなどの仮想マシンの機能を使って起動する必要がある。

### isoファイル

ディスクを単にビット列として扱ってファイルとして保存したもの。ループバックデバイスやディスクイメージと呼ばれる。
完全なコピーといえるので、isoファイルをマウントした場合はパーティションも再現される。

### ブートローダ

OSをロードするためのEFI実行可能ファイル。Linuxの場合、vmlinuzの展開とinitrdの読み込みが主な役割。GRUB Legacy、GRUB2、systemd-bootなどがある。

UEFIやBIOSもOSをロードするためのアプリケーション(ファームウェア)なので、UEFIやBIOSを一次ブートローダ、あとから実行されるものを二次ブートローダと呼んだりする。

EFISTUBという、二次ブートローダを使用せずにロードする方法もある。

### チェインロード

ブートローダからブートローダを呼び出すこと。単にEFI実行可能ファイルを指定するだけ。呼び出し元のブートローダは、自身がUEFIのふりをすることで、ブートローダを円滑に起動する。

### マルチブートUSBドライブ

かなり広義な名前だが、あくまでisoファイルごと起動することができるUSBドライブのこと。ddやRufusを使う必要がなく、複数のisoを持ち歩けるので好き。

isoファイルからデータを取り出して起動する、**ブートローダの機能**である。GRUBやSyslinuxは対応しているが、systemd-bootは対応していない。isoファイルをマウントしているわけではないので、チェインロードには対応していない。

GRUBの場合、カーネルパラメータの指定やチェインロードもどきを使用するために**loopback.cfg**というファイルをconfigfileに指定して起動できる。各OSがloopback.cfgを用意していたら、マルチブートは簡単にできる。
https://supergrubdisk.org/wiki/Loopback.cfg

```grub
grub> set iso_path=[起動したいisoのpath。/boot-isos以下に配置しておく。]
grub> export iso_path
grub> loopback loop $iso_path
grub> root=(loop)
grub> configfile /boot/grub/loopback.cfg
```

loopback.cfgが存在しない場合、grub.cfgの内容を読んでエントリを再現したらうまくいくかもしれない。

### grub.cfg

GRUBシェルで手動で入力するコマンドを/boot/grub/grub.cfgに記述することで、エントリメニューから簡単に実行できる。

```cfg:grub.cfg
menuentry '好きなエントリ名' {
	set iso_path='/boot-isos/arch.iso'
	export iso_path
	search --set=root --file "$iso_path"
	loopback loop "$iso_path"
	root=(loop)
	configfile /boot/grub/loopback.cfg
	loopback --delete loop
}
```

エントリメニュー内では、cを入力するとGRUBシェルに入り、eを入力するとエントリを編集できる(grub.cfgは変更されない)ので、簡単にテストできる。

### フォールバックルートパス

https://wiki.archlinux.jp/index.php/GRUB/%E3%83%92%E3%83%B3%E3%83%88%E3%81%A8%E3%83%86%E3%82%AF%E3%83%8B%E3%83%83%E3%82%AF#USB_.E3.82.B9.E3.83.86.E3.82.A3.E3.83.83.E3.82.AF.E3.81.AB.E3.82.A4.E3.83.B3.E3.82.B9.E3.83.88.E3.83.BC.E3.83.AB

**/BOOT/EFI/BOOTX64.EFI**のこと。UEFIが勝手にブートエントリとして認識してくれる。

GRUBは、実体であるEFI実行可能ファイルを**フォールバックルートパスではない**場所に配置し、インストール時に使ったPCの**NVRAMを変更する**ことで起動できるようにしている。そのため別のPCでは、インストールしたドライブを接続しただけでは起動できない。--removableオプションを指定することでフォールバックルートパスに配置するか、別のPCに移る毎にUEFIの設定画面で直接EFI実行可能ファイルを指定する。

systemd-bootを採用した理由は、GRUBの挙動を把握しきれなかったのと、エントリの設定が楽そうだったから。

### vmlinuz

https://ja.wikipedia.org/wiki/Vmlinux
https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html

Linuxのカーネルが圧縮されたもの。vmlinuxがzipされたからvmlinuzらしい。

### 初期RAMファイル (initramfs)



### マイクロコード
### sudo
### timedatectl
### fdisk
### chroot
### reflector
### pacstrap
### sudo
### セットアップ後でも可能な操作
### Wifi環境
### AURおよびyay