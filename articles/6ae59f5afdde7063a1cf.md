---
title: "ラズパイでプログラムを自動実行したりログローテーションしたり自動更新したり"
emoji: "🍓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [raspberrypi]
published: true
---

# はじめに
ラズパイでプログラムを自動実行したりログローテーションしたり自動更新したりするときのメモ。
いい感じに勝手に動いて欲しいときの設定。
ラズパイというより、debian系のディストリビューションの設定なんやと思う。

# 自動実行
`systemd`っての使うのが王道っぽい。
こんな感じで、設定ファイルを書く。
プログラムが死んでもいい感じにリスタートしてくれる。
詳細は適当に検索プリーズ。

```
$ cat /etc/systemd/system/hoge.service
[Unit]
Description=Hoge!
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=hoge
EnvironmentFile=/etc/default/env
WorkingDirectory=/home/hoge/
ExecStart=/bin/bash -c '/home/hoge/hoge >> /home/hoge/hoge.log 2>&1'
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

環境変数はここらに置いとくのが良さげ。
`.bashrc`とかじゃないっぽい。
```
$ cat /etc/default/enb
HOGE=fuga
```

スタートしたりするのに、ここらのコマンド使う。
```
sudo systemctl start hoge.service
sudo systemctl enable hoge.service
sudo systemctl stop hoge.service
sudo systemctl status hoge.service
sudo systemctl restart hoge.service
systemctl list-unit-files --type=service
sudo systemctl daemon-reload
```

# ログローテーション
こんな感じでログファイルのローテションの設定をする。
systemdで実行して出てきたファイルをローテーションさせる前提。
compressとかしてサービスをリスタートする感じ。

```
$ cat /etc/logrotate.d/hoge
/home/hoge/hoge.log
{
        rotate 10
        size 1M
        missingok
        notifempty
        delaycompress
        compress
        postrotate
            sudo systemctl restart hoge.service
        endscript
}

```

# 自動アップデート
`unattended-upgrades`っての使っていい感じにする。
途中はしょってるから、適当に検索プリーズ。

```
$ sudo apt-get install unattended-upgrades
```

こんな感じにするとダウンロードする時間を設定できる。

```txt:/lib/systemd/system/apt-daily.timer
[Unit]
Description=Daily apt download activities

[Timer]
OnCalendar=*-*-* 1:00
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target

```

こんな感じにするとインストールする時間を設定できる。
```txt:/lib/systemd/system/apt-daily-upgrade.timer
[Unit]
Description=Daily apt upgrade and clean activities
After=apt-daily.timer

[Timer]
OnCalendar=*-*-* 2:00
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target

```

こんな感じにしとくと、パッケージリストのこうしんとアップグレードをしてくれる。
`apt update`と`apt upgrade`に対応するはず。
```
$ cat /etc/apt/apt.conf.d/20auto-upgrades 
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

あと、`/etc/apt/apt.conf.d/50unattended-upgrades`ってファイルをいじると、アップグレードしたとき自動再起動もしてくれる。

# おわりに
もっと他にも設定しとくべきことありそうやけど、気づいたら追記したい。
