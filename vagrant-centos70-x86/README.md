Vagrantfile for CentOS 7.0
==========================

VirtualboxでCentOS7.0を動かして仕組みをお勉強しよう

## Environment

* Virtualbox 4.3.14
* Vagrant 1.6.5
    * Box file of CentOS 7.0 is https://f0fff3908f081cb6461b407be80daf97f07ac418.googledrive.com/host/0BwtuV7VyVTSkUG1PM3pCeDJ4dVE/centos7.box


## Usage

* Install vagrant-vbguest plugin

```bash
$ vagrant plugin install vagrant-vbguest
```

* vagrant up

```bash
$ vagrant up
```

* この際に共有フォルダのマウントが失敗する可能性があるので、その場合は下記手順を行う

```
# 発生するエラー
Failed to mount folders in Linux guest. This is usually beacuse
the "vboxsf" file system is not available. Please verify that
the guest additions are properly installed in the guest and
can work properly. The command attempted was:


# GuestAdditionsのrebuildを行って再起動
$ vagrant vbguest --do rebuild
$ vagrant halt
$ vagrant up
```

* vagrant ssh => 動作確認

```bash
$ vagrant ssh
```


## Initial Settings


### By Vagrantfile

* （おそらく）非・仮想環境にCentOS7をセットアップしたい際も初期設定として行っておきたい処理はVagrantfileに書いておく


### terminfoが足りない

* vimでファイル開こうとすると警告発生
    * `$TERM`が`screen-256color-bce`なんだけど、terminfoが無い
    * ncurses-term パッケージを入れる

```bash
[vagrant@vmcn7 ~]$ vi .bashrc
E437: 端末に "cm" 機能が必要です

[vagrant@vmcn7 ~]$ echo $TERM
screen-256color-bce

# 確かに無い
[vagrant@vmcn7 ~]$ ll /usr/share/terminfo/s/screen*
-rw-r--r--. 1 root root 1571  6月 10 23:11 /usr/share/terminfo/s/screen
-rw-r--r--. 1 root root 2009  6月 10 23:11 /usr/share/terminfo/s/screen-16color
-rw-r--r--. 1 root root 1847  6月 10 23:11 /usr/share/terminfo/s/screen-256color
:
:
-rw-r--r--. 1 root root 1503  6月 10 23:11 /usr/share/terminfo/s/screen.xterm-r6

# ncurses-base しか入ってない
[root@vmcn7 system]# rpm -ql ncurses-base
/usr/share/terminfo
/usr/share/terminfo/A
/usr/share/terminfo/A/Apple_Terminal
:
:

# ncurses-term 入れたら解決するかも
[vagrant@vmcn7 ~]$ yum -y install ncurses-term

# 入った
[vagrant@vmcn7 ~]$ ll /usr/share/terminfo/s/screen*
-rw-r--r--. 1 root root 1571  6月 10 23:11 /usr/share/terminfo/s/screen
-rw-r--r--. 1 root root  474  6月 10 23:11 /usr/share/terminfo/s/screen+fkeys
-rw-r--r--. 1 root root 2009  6月 10 23:11 /usr/share/terminfo/s/screen-16color
-rw-r--r--. 1 root root 2021  6月 10 23:11 /usr/share/terminfo/s/screen-16color-bce
-rw-r--r--. 1 root root 2049  6月 10 23:11 /usr/share/terminfo/s/screen-16color-bce-s
-rw-r--r--. 1 root root 2039  6月 10 23:11 /usr/share/terminfo/s/screen-16color-s
-rw-r--r--. 1 root root 1847  6月 10 23:11 /usr/share/terminfo/s/screen-256color
-rw-r--r--. 1 root root 1859  6月 10 23:11 /usr/share/terminfo/s/screen-256color-bce
-rw-r--r--. 1 root root 1885  6月 10 23:11 /usr/share/terminfo/s/screen-256color-bce-s
-rw-r--r--. 1 root root 1875  6月 10 23:11 /usr/share/terminfo/s/screen-256color-s
:
:
-rw-r--r--. 1 root root  630  6月 10 23:11 /usr/share/terminfo/s/screen3
```


## Survey

### systemd

* アーキテクチャの概要図 - https://wiki.tizen.org/w/images/thumb/9/99/Systemd_arch.PNG/700px-Systemd_arch.PNG
* `sysvinit`とか`upstart`と比較して何が変わってるのか（何が同じなのか）？
    * 当然だが、サーバ起動時の起動プロセスが変わる
        * sysvinitやupstartの場合は`/sbin/init`が最初に実行される, これがsysvinitやupstartの本体
            * sysvinitの起動の流れ - `/etc/rc.d/rc.sysinit` => `/etc/rc.d/rc` (ここで`/etc/init.d/SERVICE_NAME start`で自動起動すべき各サービス起動) => `getty(mingetty)`(コンソールログイン受付プロセス) の順に起動する
        * systemdは`/usr/bin/systemd`が最初に実行される
    * サーバの起動時に初期設定やサービス起動をおこなうことにとどまらず、プロセスやリソースなど様々な管理を行う
        * 従来は`ulimit`でリソース制限
            * プロセス単位なのでforkすると子プロセスにはその制限が効かない
        * systemdでは __cgroups による"サービス単位での"リソースの分離__ を行う
    * PIDではなくcgroupsを使ってサービスプロセス群を監視する
        * デーモンが2回forkしてもsystemdから逃れることはできない
        * 何が幸せか？ __プロセスを派生プロセス含めて綺麗に停止__ 出来る
            * 従来はサービス起動スクリプトに対象プロセスのIDや名前を指定して個別に停止させていた
    * シェルスクリプトに依存した起動処理を完全に排除
    * __"Unit"__ という単位で処理を管理
        * service
        * target
        * mount
        * device
        * etc...
    * __systemdはPIDが常に`1`__

        ```bash
        USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
        root         1  0.0  0.6  50016  6212 ?        Ss   00:44   0:01 /usr/lib/systemd/systemd --system --deserialize 18
        ```

        * `default.target`というUnitを頂点とする依存関係のツリーを構築した後、依存するUnitを起動
            * __依存関係とは別に起動順の設定も与えられる__
                * 起動順の指定が無いと依存関係の有るUnitでも並列に起動
        * upstartではイベントの連鎖によって依存する処理が順次実行されるので、処理の並列化が出来ない
    * Unitには「特定のデバイスが接続された」「D-Bus経由でサービスが要求された」など、 __システム稼働中に発生するイベントをトリガーとして、オンデマンドに起動する__ ものもある
        * 例としては、 __「パケットが届いたら起動」とか「ファイルが作成されたら起動」とか__
        * 必ずしもシステム起動時に立ち上げる必要の無いサービスを起動処理から除外することが可能
* `systemctl`は、そもそも`service`とか`chkconfig`と比較して何が変わってるのか（何が同じなのか）？


#### cgroupsとの関係

* __リソースの分離__


#### Unit

* Unitの構成はファイルで定義されている
    * `/usr/lib/systemd/system/`以下の設定ファイル - システムデフォルトの設定
    * `/etc/systemd/system/`以下 - ユーザー設定
* ファイルの種類
    * `.service`ファイル - 従来のサービス/デーモンに相当するUnitのファイル、指定したバイナリを実行/デーモンを起動する
    * `.target`ファイル - 複数のUnitをグループ化してまとめる, 依存関係や順序関係を定義
    * `.target.wants`ディレクトリ - 前提となるUnitを記載する（前提Unitの設定ファイルへのシンボリックリンクを作成する）為のディレクトリ
        * 設定ファイル内の[Unit]セクションにおいて、「Wants=」オプション、もしくは「Requires=」オプションで指定する方法もある
            * システム的に必須の設定はこの方法で定義することが一般的
* systemdが起動してまず初めに起動されるtargetは `default.target`
    * ユーザー設定を想定して`/etc/systemd/system`にファイルがある

```bash
[root@vmcn7 ~]# ll /etc/systemd/system/default.target
lrwxrwxrwx. 1 root root 37 Aug  1 08:35 /etc/systemd/system/default.target -> /lib/systemd/system/multi-user.target


[root@vmcn7 ~]# cat /etc/systemd/system/default.target
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.

[Unit]
Description=Multi-User System
Documentation=man:systemd.special(7)
Requires=basic.target  # targetファイルに前提となる依存targetを記述
Conflicts=rescue.service rescue.target
After=basic.target rescue.service rescue.target
AllowIsolate=yes

[Install]
Alias=default.target


[root@vmcn7 system]# ll /etc/systemd/system/default.target.wants/
total 0
lrwxrwxrwx. 1 root root 57 Aug  1 08:33 systemd-readahead-collect.service -> /usr/lib/systemd/system/systemd-readahead-collect.service
lrwxrwxrwx. 1 root root 56 Aug  1 08:33 systemd-readahead-replay.service -> /usr/lib/systemd/system/systemd-readahead-replay.service
```


#### パッケージインストール時の挙動

* systemd対応パッケージをインストールした場合、 __/usr/lib/systemd/system に .service ファイルが用意され、これがsystemctlコマンド実行時に使用される__

```bash
[vagrant@vmcn7 ~]$ ll /usr/lib/systemd/system | more
合計 752
-rw-r--r--. 1 root root  403  7月 30 11:21 -.slice
-rw-r--r--. 1 root root  225  7月 10 04:22 NetworkManager-dispatcher.service
-rw-r--r--. 1 root root  281  7月 10 04:22 NetworkManager-wait-online.service
-rw-r--r--. 1 root root  400  7月 10 04:22 NetworkManager.service
-rw-r-----. 1 root root  602  6月  9 23:31 auditd.service
lrwxrwxrwx. 1 root root   14  8月  1 08:41 autovt@.service -> getty@.service
-rw-r--r--. 1 root root 1044  6月 10 01:26 avahi-daemon.service
-rw-r--r--. 1 root root  874  6月 10 01:26 avahi-daemon.socket
-rw-r--r--. 1 root root  546  7月 30 11:21 basic.target
drwxr-xr-x. 2 root root 4096  9月  4 23:17 basic.target.wants
-r--r--r--. 1 root root  341  6月  9 15:38 blk-availability.service
-rw-r--r--. 1 root root  379  7月 30 11:21 bluetooth.target
-rw-r--r--. 1 root root  160  4月  2 11:30 brandbot.path
-rw-r--r--. 1 root root  101  4月  2 11:30 brandbot.service
```

* systemd非対応パッケージの場合、`/etc/init.d`以下のスクリプトしか用意されないが、このスクリプト内でsystemdが利用できるよう工夫が施されている


#### サービスの起動・停止・確認など

* [SysVinit to Systemd Cheatsheet - FedoraProject](https://fedoraproject.org/wiki/SysVinit_to_Systemd_Cheatsheet)
* 基本的な起動・停止操作系
    * sysvinit/upstart : `serivice SERVICE_NAME {start, stop, status}` `/etc/init.d/SERVICE_NAME {start, stop, status}`
    * __systemd : `service SERVICE_NAME {start, stop, status}`__ `systemctl {start, stop, status} SERVICE_NAME`
* 強制終了, cgroupsの同じグループに属するプロセスにまとめてシグナルを送信
    * sysvinit/upstart : `kill -9 PID`
    * __systemd : `systemctl kill -s 9 SERVICE_NAME`__
* 自動起動の有効・無効化
    * sysvinit/upstart : `chkconfig SERVICE_NAME {on, off}`
    * __systemd : `systemctl {enable, disable} SERVICE_NAME`__
* 自動起動の状態確認
    * sysvinit/upstart : `chkconfig --list SERVICE_NAME`
    * __systemd : `systemctl is-enabled SERVICE_NAME`__
* サービス一覧表示
    * sysvinit/upstart : `ls /etc/init.d`
    * __systemd : `systemctl --type service`__
* 稼働しているUnitの確認(serviceの場合)
    * __systemd : `systemctl list-units --type service`__
* 全てUnitの確認(serviceの場合)
    * __systemd : `systemctl list-unit-files --type service`__
* サービスの依存関係をtree形式で表示
    * __systemd : `systemctl list-dependencies SERVICE_NAME`__
* サービスの最新情報をsystemdに知らせる（再読込する）
    * __systemd : `systemctl daemon-reload`
    * 手動で .service ファイルを配置してサービス追加した際等に実行する

#### サービスのログ調査・管理

* sysvinit/upstart : `/var/log`あたりを探す
* __systemd : `journalctl {-u, -f, -k, -o JSON} SERVICE_NAME`__
    * `-u` : 特定のサービスのログを確認
    * `-f` : tail -f
    * `-k` : `--dmesg`のエイリアス
    * `-o JSON` : ログをJSONで取得


#### systemd-* コマンド

```bash
[root@vmcn7 system]# ll /bin/systemd-*
-rwxr-xr-x. 1 root root  82376 Jul 30 11:22 /bin/systemd-analyze
-rwxr-xr-x. 1 root root  44872 Jul 30 11:22 /bin/systemd-ask-password
-rwxr-xr-x. 1 root root  32296 Jul 30 11:22 /bin/systemd-cat
-rwxr-xr-x. 1 root root  57648 Jul 30 11:22 /bin/systemd-cgls
-rwxr-xr-x. 1 root root  61800 Jul 30 11:22 /bin/systemd-cgtop
-rwxr-xr-x. 1 root root 128616 Jul 30 11:22 /bin/systemd-coredumpctl
-rwxr-xr-x. 1 root root  49304 Jul 30 11:22 /bin/systemd-delta
-rwxr-xr-x. 1 root root  32536 Jul 30 11:22 /bin/systemd-detect-virt
-rwxr-xr-x. 1 root root  44944 Jul 30 11:22 /bin/systemd-inhibit
lrwxrwxrwx. 1 root root      8 Aug  1 08:41 /bin/systemd-loginctl -> loginctl
-rwxr-xr-x. 1 root root  40704 Jul 30 11:22 /bin/systemd-machine-id-setup
-rwxr-xr-x. 1 root root  32360 Jul 30 11:22 /bin/systemd-notify
-rwxr-xr-x. 1 root root 187352 Jul 30 11:22 /bin/systemd-nspawn
-rwxr-xr-x. 1 root root 137464 Jul 30 11:22 /bin/systemd-run
-rwxr-xr-x. 1 root root 133216 Jul 30 11:22 /bin/systemd-stdio-bridge
-rwxr-xr-x. 1 root root   3979 Jul 30 11:21 /bin/systemd-sysv-convert
-rwxr-xr-x. 1 root root  82816 Jul 30 11:22 /bin/systemd-tmpfiles
-rwxr-xr-x. 1 root root  69944 Jul 30 11:22 /bin/systemd-tty-ask-password-agent
```

* `systemd-run CMD` - systemdを経由して __systemdの各種恩恵を受けつつコマンドを実行__
    * リソース管理
    * 状態監視
    * ログ出力
* `systemd-cgls` - cgroupsによる分類をツリー表示する
* `systemd-cgtop` - cgroupsによるグループ毎のリソース使用状況を表示


### ネットワークアダプタ

* アダプタの名前が従来と違う

```bash
[root@vmcn7 ~]# ll /etc/sysconfig/network-scripts/ifcfg*
-rw-r--r--. 1 root root 289 Aug  1 08:35 /etc/sysconfig/network-scripts/ifcfg-enp0s3
-rw-r--r--. 1 root root 218 Sep  5 00:45 /etc/sysconfig/network-scripts/ifcfg-enp0s8
-rw-r--r--. 1 root root 254 Apr  2 11:30 /etc/sysconfig/network-scripts/ifcfg-lo
```

* net-toolsをインストールして`ifconfig`叩いて確認する
    * vagrantで設定されたアダプタは`enp0s8`

```bash
$ yum -y install net-tools

$ ifconfig
enp0s3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet [IPv4アドレス]  netmask 255.255.255.0  broadcast [IPv4アドレス]
        inet6 [IPv6アドレス]  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:ea:9b:b5  txqueuelen 1000  (Ethernet)
        RX packets 116645  bytes 115939292 (110.5 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 29683  bytes 1993820 (1.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.120  netmask 255.255.255.0  broadcast 192.168.56.255
        inet6 fe80::a00:27ff:feb7:f3db  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:b7:f3:db  txqueuelen 1000  (Ethernet)
        RX packets 722  bytes 58323 (56.9 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 36  bytes 5963 (5.8 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```


### Dockerを入れる

* `yum install docker`でインストールできるがバージョンが古い
* dockerの公式から最新版のdocker（のバイナリ）の直リンクがあるのでダウンロード可能だが、折角なんで`yum install`したら得られる恩恵＝systemdによるサービス管理は享受したい
* [docker/centos.md at master · docker/docker](https://github.com/docker/docker/blob/master/docs/sources/installation/centos.md) で紹介されてる手順を踏襲する

> Manual installation of latest version
> While using a package is the recommended way of installing Docker, the above package might not be the latest version. If you need the latest version, you can install the binary directly.
> When installing the binary without a package, you may want to integrate Docker with systemd. For this, simply install the two unit files (service and socket) from the github repository to /etc/systemd/system.)

```
# cd /usr/bin
# wget https://get.docker.io/builds/Linux/x86_64/docker-latest -O docker
# chmod +x docker

# cd /etc/systemd/system
# wget https://raw.githubusercontent.com/docker/docker/master/contrib/init/systemd/docker.service
# wget https://raw.githubusercontent.com/docker/docker/master/contrib/init/systemd/docker.socket
```

#### デフォルトのdocker.sock以外へのsocket割り当て

* [Bind Docker to another host/port or a Unix socket](http://docs.docker.com/articles/basics/#bind-docker-to-another-hostport-or-a-unix-socket)
* unix domain socket以外にtcp経由でのアクセスを可能にする = `-H tcp://host:port`オプション付きで起動
* 対象の一般ユーザーの環境変数に`DOCKER_HOST=<起動時に指定したhost>`を設定
* Docker Remote APIにHTTPでアクセス可能になる

```
# cd /etc/systemd/system/
# cp docker.service docker.service.org
# vi docker.service
:
:
# diff -u docker.service.org docker.service
```
```diff
@@ -5,7 +5,9 @@
 Requires=docker.socket

 [Service]
-ExecStart=/usr/bin/docker -d -H fd://
+#ExecStart=/usr/bin/docker -d -H fd://
+Environment="DOCKER_HOST=tcp://0.0.0.0:2375"
+ExecStart=/usr/bin/docker -d -D -H fd:// -H $DOCKER_HOST
 LimitNOFILE=1048576
 LimitNPROC=1048576
```
```
# systemctl daemon-reload
# systemctl start docker

# systemctl status docker
docker.service - Docker Application Container Engine
   Loaded: loaded (/etc/systemd/system/docker.service; static)
   Active: active (running) since 水 2014-10-08 15:54:41 JST; 1min 24s ago
     Docs: http://docs.docker.com
 Main PID: 3688 (docker)
   CGroup: /system.slice/docker.service
           └─3688 /usr/bin/docker -d -D -H fd://* -H tcp://0.0.0.0:2375
```
```
$ whoami
vagrant
$ export DOCKER_HOST="tcp://0.0.0.0:2375"
$ docker info
: (エラーにならないことを確認)
```

* VMのfirewalldを止めて、VM内＆VM稼働マシン(Mac想定)からdocker remote APIへのアクセス確認

```
# systemctl status firewalld
firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled)
   Active: active (running) since 水 2014-10-08 15:58:42 JST; 59min ago
 Main PID: 536 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─536 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

10月 08 15:58:42 vmcn7 systemd[1]: Started firewalld - dynamic firewall daemon.
10月 08 16:01:45 vmcn7 firewalld[536]: 2014-10-08 16:01:45 ERROR: '/sbin/iptables -I FORWARD_OUT_ZONES 1 -t filter -o docker0 -g FWDO_public' failed: Another app is currently...e -w option?
10月 08 16:01:45 vmcn7 firewalld[536]: 2014-10-08 16:01:45 ERROR: '/sbin/iptables -D INPUT_ZONES 1 -t filter -i docker0 -g IN_public' failed: iptables v1.4.21: Illegal option...this command

                                        Try `iptables -h' or 'iptables --help' for more information....
10月 08 16:01:45 vmcn7 firewalld[536]: 2014-10-08 16:01:45 ERROR: COMMAND_FAILED: '/sbin/iptables -I FORWARD_OUT_ZONES 1 -t filter -o docker0 -g FWDO_public' failed: Another ...e -w option?
Hint: Some lines were ellipsized, use -l to show in full.


# systemctl stop firewalld
```
```bash
# from vm
$ curl -XGET "http://localhost:2375/containers/json?all=1"
(OK)
```
```bash
# from host machine
$ curl -XGET "http://localhost:<forwarded port>/containers/json?all=1"
(OK)
```


### Firewalld

* [FirewallD/jp - FedoraProject](https://fedoraproject.org/wiki/FirewallD/jp)
* iptables の代替
    * とはいえ、内部的にはiptablesを使用している
* __NICポートごとに仮想的なファイアウォールを設定__ する
* "一時的・永続的な設定オプションを分けて保持します"
* "ファイアウォールを動的に管理し、設定を反映するのにファイアウォール全体の再起動を必要としません"
    * "system-config-firewall/lokkitによる従来のファイアウォールモデルは静的で、いかなる変更でもファイアウォールの完全な再起動が必要でした"
* "デーモンは有効なファイアウォールの設定の情報をD-BUS経由で提供し、PolicyKit認証の仕組みを利用してD-BUS経由で変更を受け付けます"
    * "アプリケーションとデーモン、ユーザはD-BUSを通じてファイアウォールの機能を有効に出来ます。"

```bash
[root@vmcn7 ~]# systemctl status firewalld
firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled)
   Active: active (running) since Thu 2014-09-04 23:14:40 EDT; 38min ago
 Main PID: 539 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─539 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

Sep 04 23:14:38 localhost.localdomain systemd[1]: Starting firewalld - dynamic firewall daemon...
Sep 04 23:14:40 localhost.localdomain systemd[1]: Started firewalld - dynamic firewall daemon.
```

* ゾーン (zone)
    * コネクションについての通信の許可不許可を定義したもの
    * "ネットワークゾーンはネットワークコネクションの信頼度を定義します。これは一対多の関係で、あるコネクションはあるゾーンの単なる部分であることを意味しますが、あるゾーンが多数のネットワークコネクションに利用されることもあり得ます。"
    * ゾーンごとに何を設定するか？
        * 許可する「サービス」
        * 受信を禁止するICMPタイプ
        * IPマスカレードの有無,DNA(P)Tの設定
        * その他(ポート番号を明示した設定、送信元/先でのフィルタリング etc)
    * デフォルトで用意されいているゾーン
        * `drop`(設定変更不可) - あらゆる(外部からの返信)パケットを破棄, 実質通信できない
        * `block`(設定変更不可) - あらゆる受信パケットを拒否, 内部->外部の通信は可能
        * `public`
        * `external`
        * `dmz`
        * `work`
        * `home`
        * `internal`
        * `trusted`(設定変更不可) - あらゆるパケットが受信許可される
* 設定ファイル群
    * `/usr/lib/firewalld/zones` - ゾーンの定義、xmlファイルを配置
    * `/usr/lib/firewalld/services` - サービスの定義、xmlファイルを配置
    * `/usr/lib/firewalld/icmptypes` - icmpタイプの定義、xmlファイルを配置
* `firewall-cmd`コマンド
    * ゾーンの設定
        * `firewall-cmd --get-default-zone` - デフォルトゾーンの確認
        * `firewall-cmd --set-default-zone=HOGE` - デフォルトゾーンの変更
        * `firewall-cmd --list-all-zones` - ゾーンごとの設定一覧
        * `firewall-cmd --get-active-zones` - 現在activeなゾーンの確認
        * `firewall-cmd --reload` - 定義ファイル追加・変更の反映、起動時の設定を再読み込み
        * `firewall-cmd --complete-reload` - 起動時の設定を再読み込み, コネクショントラッキングのステート情報も初期化
        * サービス
            * `firewall-cmd --get-services` - 定義済のサービスとICMPタイプの一覧
            * `firewall-cmd --list-services --zone=public` - 許可されているサービスを表示
            * `firewall-cmd --query-service=http --zone=public` - 指定サービスが許可されているか確認
            * `firewall-cmd --add-service=http --zone=public` - 許可するサービスを追加
            * `firewall-cmd --remove-service=http --zone=public` - 許可するサービスを削除
        * ポート
            * `firewall-cmd --list-ports --zone=public` - 現在のポート開放設定を確認
            * `firewall-cmd --query-port=PORTID/PROTOCOL --zone=public` - ポート開放設定の存在を確認
            * `firewall-cmd --add-port=PORTID/PROTOCOL --zone=public` - ポート開放設定を追加
            * `firewall-cmd --remove-port=PORTID/PROTOCOL --zone=public` - ポート開放設定を削除
        * ICMPタイプ
            * `firewall-cmd --get-icmptypes` - 定義済のサービスとICMPタイプの一覧
            * `firewall-cmd --list-icmp-blocks --zone=public` - 禁止されているICMPタイプを表示
            * `firewall-cmd --query-icmp-block=echo-request --zone=public` - 指定ICMPタイプが禁止されているか確認
            * `firewall-cmd --add-icmp-block=echo-request --zone=public` - 禁止するICMPタイプを追加
            * `firewall-cmd --remove-icmp-block=echo-request --zone=public` - 禁止するICMPタイプを削除
        * IPマスカレード
            * `firewall-cmd --query-masquerade --zone=public` - 現在の設定を確認
            * `firewall-cmd --add-masquerade --zone=public` - IPマスカレードを有効化
            * `firewall-cmd --remove-masquerade --zone=public` - IPマスカレードを無効化
        * ポートフォワード(DNA(P)T)
            * `firewall-cmd --list-forward-ports --zone=public` - 現在の設定を確認
            * `firewall-cmd --query-forward-port=RULE --zone=public` - 変換ルールの存在を確認
            * `firewall-cmd --add-forward-port=RULE --zone=public` - 変換ルールを追加
            * `firewall-cmd --remove-forward-port=RULE --zone=public` - 変換ルールを削除
    * NICポートに対するゾーンの適用
        * `firewall-cmd --list-interface --zone=trusted` - trustedゾーンが適用されているNICポートを表示
        * `firewall-cmd --query-interface=eth1 --zone=trusted` - eth1の適用ゾーンがtrustedであるか確認
        * `firewall-cmd --add-interface=eth1 --zone=trusted` - eth1にtrustedゾーンを適用
        * `firewall-cmd --change-interface=eth1 --zone=public` - eth1の適用ゾーンをpublicに変更
        * `firewall-cmd --remove-interface=eth1` - eth1の適用ゾーンを除去
        * これらコマンドでは`--permanent`を付けると __起動時の設定変更__ になる(デフォルトは __実行中の設定変更__ )
        * `--zone`省略時は __デフォルトゾーン__ (デフォは`public`) が対象
    * 送信元IPのサブネットに対するゾーンの適用
        * `firewall-cmd --list-sources --zone=trusted` - trustedゾーンが適用されているサブネットを表示
        * `firewall-cmd --query-source=192.168.122.0/24 --zone=trusted` - サブネットの適用ゾーンがtrustedであるか確認
        * `firewall-cmd --add-source=192.168.122.0/24 --zone=trusted` - サブネットにtrustedゾーンを適用
        * `firewall-cmd --change-source=192.168.122.0/24 --zone=public` - サブネットの適用ゾーンをpublicに変更
        * `firewall-cmd --remove-source=192.168.122.0/24` - サブネットの適用ゾーンを除去
        * `--zone`省略時は __デフォルトゾーン__ (デフォは`public`) が対象
    * 例 : 社内LANからのアクセスを許可したい

      ```
      # firewall-cmd --add-source=10.0.0.0/8 --zone=work
      success
      ```


### NetworkManager (nmtui/nmcliコマンド)
* __ネットワークの設定は設定ファイルを直接編集せず、NetworkManager(CUIなら`nmcli`コマンド)を介して行う
* [2.3. Using the NetworkManager Command Line Tool, nmcli](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/sec-Using_the_NetworkManager_Command_Line_Tool_nmcli.html)




### Chrony

* __ntp__ の代替


### yum





## Bookmark

* [systemdを本番運用してわかったこと - mixi Engineers' Blog](http://alpha.mixi.co.jp/entry/2013/12063/)
* [Systemd入門(1) - Unitの概念を理解する - めもめも](http://d.hatena.ne.jp/enakai00/20130914/1379146157)
