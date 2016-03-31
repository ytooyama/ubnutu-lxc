#UbuntuでLXCを利用する
（Last Update: 2016/03/31）

本書は以下の環境にLXCをインストールした時の各操作のメモです。

- Ubuntu Trusty 14.04.4 Server
- LXC 1.0.8-0ubuntu0.3

##インストール

- LXCのインストール

```
$ sudo apt-get install lxc
```

- テンプレートのパス

```
/usr/share/lxc/templates/
```

- 展開される場所

```
/var/lib/lxc/
```

- キャッシュの場所

```
/var/cache/lxc
```

- デフォルトの設定

```
# cat /etc/lxc/default.conf
lxc.network.type = veth
lxc.network.link = lxcbr0
lxc.network.flags = up
lxc.network.hwaddr = 00:16:3e:xx:xx:xx
```

- コンテナの作成

```
$ sudo lxc-create [option]
```
--cleanをつけて実行すると、必要なパッケージの再取得を行ってからデプロイする。

- OSのデプロイ

```
$ sudo lxc-create -t download -n my-container
Downloading the image index
...
---
DIST	RELEASE	ARCH	VARIANT	BUILD
---
centos	6	amd64	default	20160331_02:16
centos	6	i386	default	20160331_02:16
centos	7	amd64	default	20160331_02:16
...
Distribution: ubuntu
Release: trusty
Architecture: amd64
Downloading the image index
Downloading the rootfs
...
```

lxc-createが終わると、次のようなメッセージが表示されます。

```
You just created an Ubuntu container (release=trusty, arch=amd64, variant=default)

To enable sshd, run: apt-get install openssh-server

For security reason, container images ship without user accounts
and without a root password.

Use lxc-attach or chroot directly into the rootfs to set a root password
or create user accounts.
```

lxc-attachしてユーザーを作成します。SSHサーバーが必要であればインストールしてサーバーを起動します。

```
$ sudo lxc-start -d -n my-container
$ sudo lxc-attach -n my-container
/# adduser user
/# passwd user
```

従来のやり方でOSをデプロイすることも可能です。

```
$ sudo lxc-create -t ubuntu -n my-container
...

##
# The default user is 'ubuntu' with password 'ubuntu'!
# Use the 'sudo' command to run tasks as root in the container.
##
```


###デプロイ時の注意点

同じテンプレートを使ってデプロイしている場合、リリースバージョンやアーキテクチャを切り替える場合、--cleanをつけて実行する必要がある。

##操作

- 一覧

```
$ sudo lxc-ls -f
```

- ゲストの状態

```
$ sudo lxc-info -n my-container
```

- 起動

-dオプションをつけないと、起動後コンソールに接続します。

```
$ sudo lxc-start -d -n my-container
```

- コンソールに接続

```
$ sudo lxc-console -n my-container
```

- コンソールから切断

```
Ctrl-a q
```

- 終了

```
$ sudo lxc-stop -n my-container
```

- 強制停止

```
$ sudo lxc-stop -n my-container --kill
```

- 削除

```
$ sudo lxc-destroy -n my-container
```

- スナップショット

```
$ sudo lxc-snapshot -n my-container
$ sudo lxc-snapshot -n my-container --list
snap0 (/var/lib/lxcsnaps/my-container) 2016:03:31 17:48:21
```

- スナップショットからのリストア

```
$ sudo lxc-snapshot -n my-container -r snap0 my-container-snap0
```


##物理ネットワークとの接続

次のようにLXCホストに設定すると、コンテナーを物理ネットワークと接続できます。

```
# The primary network interface
auto eth0
iface eth0 inet static
address 0.0.0.0

auto lxcbr0  #lxcbr0を上書きするぜよ
iface lxcbr0 inet static
address 192.168.1.5
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 8.8.4.4
bridge_ports eth0
bridge_stp off
bridge_maxwait 10
```

##DHCPが動いていないネットワークにコンテナーを接続する

コンテナー内のOSは、接続したネットワークにDHCPサーバーがあることを期待しますが、DHCPサーバーが無いネットワークに接続するには、次のように実行して接続します。

- コンテナーを起動。

```
$ sudo lxc-start -d -n my-container
```

- lxc-consoleでコンテナーに入って...

```
$ sudo lxc-console -n my-container
```

- コンテナー内のOSにStatic IP設定。

- 一旦コンテナーを停止

```
$ sudo lxc-stop -n my-container
```

- 再度コンテナーを起動

```
$ sudo lxc-start -d -n my-container
```

- すると...

```
$ sudo lxc-ls -f
NAME          STATE    IPV4         IPV6  AUTOSTART
---------------------------------------------------
my-container  RUNNING  192.168.1.6  -     NO　#IP取れた
```

- SSHアクセスする

```
~$ ssh ubuntu@192.168.1.6
ubuntu@192.168.1.6's password:
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 4.4.0-14-generic x86_64)

 * Documentation:  https://help.ubuntu.com/
Last login: Thu Mar 31 17:42:52 2016 from 192.168.1.5
ubuntu@my-container:~$
```