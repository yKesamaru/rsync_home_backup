## はじめに
自動化されている日々のバックアップは様々な理由から`rsync`を使っています。

`rsync`では`--exclude`オプションにより「バックアップに含めたくないフォルダ・ファイル」を指定することが出来ます。

特に気にしなくてもすべてバックアップしてくれるので良いのですが、ハードディスクの音が気になり始めました。故障の前触れとかではなく、要らない無数のファイルまで幾時間ごとに書き込み・更新するのはSSD・HDDともに負担になるかも知れません。

新しいHDDが欲しいのですが、為替の影響なのか、一昨年買った8TBのHDDの値段が倍近くになっています。アマゾンブラックフライデーで4000円近く値下げされていて、あやうくカートにぶち込むところでした。これでもまだ以前の購入金額を3000円オーバーです。

今あるHDDを大事に使うために、バックアップスクリプトを修正します。

![](assets/eye-catch.png)

## 環境
```bash
$ inxi -S --filter
System:
  Kernel: 6.8.0-49-generic x86_64 bits: 64 Desktop: GNOME 42.9
    Distro: Ubuntu 22.04.5 LTS (Jammy Jellyfish)
```

## サービスとタイマー
自動化スクリプトはユーザーの`systemd`によって管理しています。

1. `/home/user/.config/systemd/user/rsync.service`
   ```bash: /home/user/.config/systemd/user/rsync.service
   [Unit]
   Description=2022年5月8日

   [Service]
   Type=oneshot
   ExecStart=/home/user/bin/rsync_backup.sh

   [Install]
   WantedBy=default.target
   ```

2. `/home/user/.config/systemd/user/rsync.timer`
   ```bash: /home/user/.config/systemd/user/rsync.timer
   [Unit]
   Description=2022年5月8日

   [Timer]
   # ブート後、この時間後にこのタイマーを開始する
   OnBootSec=10min
   # 実行する時間の間隔
   OnUnitActiveSec=4h
   Unit=rsync.service
   Persistent=true

   [Install]
   WantedBy=timers.target
   ```

- システム起動後10分で最初のバックアップ。
- その後、4時間ごとにバックアップ処理を実行。
- シャットダウン中に実行予定だったバックアップも、次回起動時に補填される。

## 修正前のバックアップスクリプト
`/home/user/bin/rsync_backup.sh`

いくつかの指定と、除外するフォルダ等は`exclude_patterns.txt`にまとめてあります。
とはいえ、スクリプト自体は野暮ったいです。

```bash
#!/usr/bin/env bash
set -eu

function my_command() {
    sleep 500
    pactl set-sink-volume @DEFAULT_SINK@ 20%  # システムの音量を20%にする
    play -v 1 -q /home/user/.config/systemd/user/voice/003同期バックアップス….wav

    gio mount -d <UUID> && sleep 2

    rsync \
    -r \
    -t \
    -u \
    -l \
    -H \
    -s \
    --exclude-from=/home/user/bin/exclude_patterns.txt \
    /home/user/ \
    /media/user/1TB_HDD/homeフォルダバックアップ/

    # /home/binディレクトリのrsyncを行う
    rsync \
    -r \
    -t \
    -u \
    -l \
    -H \
    -s \
    --delete \
    /home/user/bin/ \
    /media/user/1TB_HDD/homeフォルダバックアップ/bin/

    play -v 1 -q /home/user/.config/systemd/user/voice/001同期バックアップス….wav
    return 0
}

function my_error() {
    zenity --error --text=" \
    '1TB_HDD/homeフォルダバックアップ'が処理失敗しました \
    "
    pactl set-sink-volume @DEFAULT_SINK@ 20%
    play -v 1 -q /home/user/.config/systemd/user/voice/002なんか良く分かんな….wav
    exit 1
}

my_command || my_error
```

### exclude_patterns.txt
除外するディレクトリやファイルを記しておくテキストファイルです。
```bash
.bash_history
.cache
.cupy
.dbus
.docker
.dotnet
.dvdcss
.fonts
.gnome
.ipython
.keras
.mongodb
.mozilla
.nv
.pki
.pyenv
.ssr
.synaptic
.thumbnails
.vscode
.xsession_errors
*.mypy*
*/.git/*
*/site-packages/*
arrow
bin
Code
Dropbox
google-chrome
snap
tmp
Trash
```

## 修正案
`exclude_patterns.txt`は廃止しました。
```bash
#!/usr/bin/env bash
set -eu

function my_command() {
    pactl set-sink-volume @DEFAULT_SINK@ 20%  # システムの音量を20%にする
    play -v 1 -q /home/user/.config/systemd/user/voice/003同期バックアップス….wav

    gio mount -d <UUID> && sleep 2

    rsync -au \
    --include='.cache/Microsoft' \
    --exclude='*.mypy*' \
    --exclude='*/.git/*' \
    --exclude='*/site-packages/*' \
    --exclude='.bash_history' \
    --exclude='.cache' \
    --exclude='.config/google-chrome' \
    --exclude='.cupy' \
    --exclude='.dbus' \
    --exclude='.docker' \
    --exclude='.dotnet' \
    --exclude='.dropbox' \
    --exclude='.dropbox-dist' \
    --exclude='.dvdcss' \
    --exclude='.fonts' \
    --exclude='.gconf' \
    --exclude='.gnome' \
    --exclude='.gnupg' \
    --exclude='.gvfs' \
    --exclude='.idea' \
    --exclude='.ipython' \
    --exclude='.keras' \
    --exclude='.local/share/Trash' \
    --exclude='.local/share/containers' \
    --exclude='.local/share/flatpak' \
    --exclude='.local/share/gnome-shell/extensions' \
    --exclude='.local/share/gvfs-metadata' \
    --exclude='.local/share/lxc' \
    --exclude='.mongodb' \
    --exclude='.mozilla' \
    --exclude='.npm' \
    --exclude='.nv' \
    --exclude='.pki' \
    --exclude='.pyenv' \
    --exclude='.ssr' \
    --exclude='.synaptic' \
    --exclude='.thumbnails' \
    --exclude='.venv' \
    --exclude='.vscode' \
    --exclude='.xsession_errors' \
    --exclude='Code' \
    --exclude='Dropbox' \
    --exclude='Trash' \
    --exclude='trash' \
    --exclude='tracker*' \
    --exclude='bin' \
    --exclude='build' \
    --exclude='crash_report' \
    --exclude='google-chrome*' \
    --exclude='include' \
    --exclude='*pyvenv*' \
    --exclude='snap' \
    --exclude='venv' \
    --exclude='lib' \
    /home/user/ \
    /media/user/1TB_HDD/homeフォルダバックアップ/

    rsync -au --delete \
    /home/user/bin/ \
    /media/user/1TB_HDD/homeフォルダバックアップ/bin/

    play -v 1 -q /home/user/.config/systemd/user/voice/001同期バックアップス….wav && zenity --info --title="バックアップ完了" --text="homeディレクトリのバックアップが終了しました。"
    return 0
}

function my_error() {
    zenity --error --text=" \
    '1TB_HDD/homeフォルダバックアップ'が処理失敗しました \
    "
    pactl set-sink-volume @DEFAULT_SINK@ 20%
    play -v 1 -q /home/user/.config/systemd/user/voice/002なんか良く分かんな….wav
    exit 1
}

my_command || my_error
```

## さいごに
まだまだ保存しなくても良いファイルはありそうですが、テストしてみたところ、以前よりかなり実行時間が短くなり、これならHDDにも優しいと思えるレベルになりました。

以上です。ありがとうございました。

