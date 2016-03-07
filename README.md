# raspbian-on-qemu

WindowsでRaspbianを動作させるため、Raspbian+QEMUのパッケージを提供します。

自宅でRaspberry Pi Model B+が元気に動作していますが、これに変更を加える前に動作確認する環境が欲しいです。一般的なOSであれば、普通にVagrant+VirtualBoxやDockerを使用しますが、ARMプロセッサが前提のRaspbianでは簡単にはできません。当記事は、その環境を作るための作業手順です。ちなみに、この作業手順は以下のページを参考にしました。

[Emulating Jessie image with 4.1.x kernel · dhruvvyas90/qemu-rpi-kernel Wiki](https://github.com/dhruvvyas90/qemu-rpi-kernel/wiki/Emulating-Jessie-image-with-4.1.x-kernel)

## ダウンロード

この作業で作成したパッケージは、[GitHubで公開](https://github.com/u6k/raspbian-on-qemu/releases)しています。

## 作業環境

* Surface Pro 4 + Windows 10
    * `qemu-system-arm`が動作すれば、この環境でなくても問題無いはずです。
* Vagrant 1.8.1 + VirtualBox
    * Raspbianイメージをマウントするために必要ですが、`.img`内を編集できるのであれば不要です。
* Qemu 2.4.1 on Windows
* Raspbian Jessie 2016/2/26版

## 作業手順

### Qemu、Raspbian、カーネルイメージをダウンロード

Qemuは、Windows用にビルドされたアプリをダウンロードします。記事執筆時点では`Qemu-2.4.1-windows.7z`が最新です。

[Qemu On Windows](http://lassauge.free.fr/qemu/)

Raspbianをダウンロードします。記事執筆時点では`2016-02-26-raspbian-jessie.zip`が最新です。

[Download Raspbian for Raspberry Pi](https://www.raspberrypi.org/downloads/raspbian/)

> *NOTE:* 旧バージョンのRaspbianがページに表示されていませんが、 https://downloads.raspberrypi.org/ から旧バージョンをダウンロードできるっぽいです。

RaspbianをQemuで動作させるために必要なカーネルイメージをダウンロードします。Jessie用をダウンロードしてください。記事執筆時点では`kernel-qemu-4.1.13-jessie`が最新です。

[dhruvvyas90/qemu-rpi-kernel: Qemu kernel for emulating Rpi on QEMU](https://github.com/dhruvvyas90/qemu-rpi-kernel)

ダウンロードしたファイルは、全て`C:\raspbian-jessie-on-qemu\`に置きます。

### ダウンロードしたファイルを展開

ダウンロードしたファイルを展開します。フォルダ・パスを決め打ちにしていますが、別フォルダでも問題ありません。

`Qemu-2.4.1-windows.7z`を`C:\raspbian-jessie-on-qemu\qemu\`に展開します。`C:\raspbian-jessie-on-qemu\qemu\`以下にQemuのファイル群が展開されている状態になります。

`2016-02-26-raspbian-jessie.zip`を`C:\raspbian-jessie-on-qemu\raspbian\`に展開します。

`kernel-qemu-4.1.13-jessie`は`C:\raspbian-jessie-on-qemu\raspbian\`に移動します。

これで、以下のようなファイル構成になります。

```
C:\
+---raspbian-jessie-on-qemu\
    +---qemu\
    |   +---qemu-system-arm.exe などQemuのファイル群
    +---raspbian\
        +---2016-02-26-raspbian-jessie.img
        +---kernel-qemu-4.1.13-jessie
```

### Raspbianイメージを変更

Raspbianはこのままでは起動できません。起動できるように内部を編集します。この作業でVagrant+VirtualBoxを使用します。Raspbianイメージをマウントできて内部を編集できるのであれば、別の手段で問題ありません。

コマンドプロンプトを開き、`C:\raspbian-jessie-on-qemu\raspbian\`まで移動して、Vagrantを起動します。

```
cd C:\raspbian-jessie-on-qemu\raspbian\
vagrant init ubuntu/trusty64
vagrant up
vagrant ssh
```

Vagrantは、デフォルトで`Vagrantfile`があるフォルダを`/vagrant/`にマウントします。つまり、`/vagrant/2016-02-26-raspbian-jessie.img`にRaspbianイメージがあります。このRaspbianイメージをマウントします。

```
$ sudo mount -v -o offset=67108864 -t ext4 /vagrant/2016-02-26-raspbian-jessie.img /mnt/
```

`/mnt/etc/ld.so.preload`を開きます。

```
$ sudo vi /mnt/etc/ld.so.preload
```

1行目をコメントアウトします。

```
/usr/lib/arm-linux-gnueabihf/libarmmem.so
↓
#/usr/lib/arm-linux-gnueabihf/libarmmem.so
```

`/mnt/etc/fstab`を開きます。

```
sudo vi /mnt/etc/fstab
```

`/dev/mmcblk`を含む行をコメントアウトします。

```
/dev/mmcblk0p1  /boot           vfat    defaults          0       2
/dev/mmcblk0p2  /               ext4    defaults,noatime  0       1
↓
#/dev/mmcblk0p1  /boot           vfat    defaults          0       2
#/dev/mmcblk0p2  /               ext4    defaults,noatime  0       1
```

`exit`して、Vagrantを破棄します。

```
vagrant destroy -f
rm Vagrantfile
rmdir /S /Q .vagrant
```

これで、Raspbianイメージの変更は完了です。

### Raspbianを起動

QemuでRaspbianを起動します。コマンドが長いので、起動バッチ・ファイルを作成します。`C:\raspbian-jessie-on-qemu\run.bat`を以下のように作成します。

```
qemu\qemu-system-arm.exe -kernel raspbian\kernel-qemu-4.1.13-jessie -cpu arm1176 -m 256 -M versatilepb -serial stdio -append "root=/dev/sda2 rootfstype=ext4 rw" -hda raspbian\2016-02-26-raspbian-jessie.img -redir tcp:50022::22
```

作成した`run.bat`を実行します。問題が無ければ、以下のようにRaspbianが起動します。

![raspbian-startup](https://raw.githubusercontent.com/u6k/raspbian-on-qemu/master/build-package/doc/img/raspbian-startup.png)

初期ユーザーは`pi`、パスワードは`raspberry`です。

上記のバッチ・ファイルに`-redir tcp:50022::22`というオプションが指定されているため、50022番ポートでssh接続ができます。

### Raspbian Liteの場合

以上の作業をRaspbian Liteで行うと、以下のように起動します。

![raspbian-lite-startup](https://raw.githubusercontent.com/u6k/raspbian-on-qemu/master/build-package/doc/img/raspbian-lite-startup.png)

## ライセンス

[RaspbianはDFSGで提供](https://www.debian.org/legal/licenses/)されています。

[QEMUはGPL v2で提供](http://wiki.qemu.org/License)されています。

この文章の作業手順で作成してGitHubで公開しているパッケージは、RaspbianとQEMUのライセンスに準拠します。

[![クリエイティブ・コモンズ・ライセンス](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-sa/4.0/)この文章は[クリエイティブ・コモンズ 表示 - 継承 4.0 国際 ライセンスの下に提供されています。](http://creativecommons.org/licenses/by-sa/4.0/)この文章に書いてある内容は無保証であり、自己責任でご利用ください。
