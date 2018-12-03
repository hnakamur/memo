==============
UbuntuでIPoE
==============

* :作成日: 2018-12-01
* :更新日: 2018-12-04

はじめに
========

`IIJmioひかり <https://www.iijmio.jp/imh/>`_ とIIJmio ひかり電話と `IIJmioひかり IPoEオプション <https://www.iijmio.jp/imh/ipoe/>`_ を利用しているのですが、Ubuntu 18.04.1 LTSでIPoEを使うようにしたメモです。

構成図
======

ネットワーク配線図は以下の通りです。

.. blockdiag::

    blockdiag {
      orientation = portrait;

      home-gw -- beagle, primergy;
      beagle, primergy -- hub;
      hub -- wifi-router;

      wifi-router -- thinkpad, android, ipad [style = dashed, label = "wifi"];
    }

ホームゲートウェイは `RT-500KI <https://www.ntt-west.co.jp/kiki/download/flets/rt500ki/>`_ です。
WiFiルータは `AtermWG1800HP2 <http://www.aterm.jp/product/atermstation/product/warpstar/wg1800hp2/>`_ ですがWANポートが1つしかないため、primergyとbeagleの2台をつなぐためにハブを挟んでいます。
ハブはI-O DATAの `ETG-ESH5 <https://www.iodata.jp/product/lan/hub/etg-esh5/>`_ です。
WiFiルータはブリッジモードを使用しています。

論理ネットワーク図は以下の通りです。

.. nwdiag::

    nwdiag {
      inet [shape = cloud];
      inet -- home-gw;

      network dmz {
          address = "192.168.1.x/24"

          home-gw [address = "ipv6:prefix:xxxx, 192.168.1.1"];
          beagle [address = "ipv6:prefix:zzzz, vip:192.168.1.2, 192.168.1.3"];
          primergy [address = "ipv6:prefix:yyyy, 192.168.1.4"];
      }
      network internal {
          address = "192.168.2.x/24";

          beagle [address = "vip:192.168.2.1, 192.168.2.2"];
          primergy [address = "192.168.2.3"];
          thinkpad;
          android;
          ipad;
      }
    }

ホームゲートウェイの設定
========================

* DHCPサーバの無効化
    - [詳細設定]/[DHCPv4サーバ設定]で[DHCPv4サーバ機能]の[使用する]チェックボックスをオフ
* LAN側の静的ルーティング
    - [詳細設定]/[LAN側静的ルーティング設定]で以下のように設定します。

.. list-table::
   :stub-columns: 1

   * - 宛先IPアドレス/マスク長
     - 192.168.2.0/24
   * - ゲートウェイ
     - 192.168.1.2

サーバのネットワーク設定
========================

beagleの :code:`/etc/netplan/01-netcfg.yaml` は以下の通りです。

.. code:: text

   network:
     version: 2
     renderer: networkd
     ethernets:
       enp3s0:
	 dhcp4: no
	 addresses: [192.168.1.3/24]
	 gateway4: 192.168.1.1
	 nameservers:
	   addresses: [192.168.1.1]
	 dhcp6: no
       enp4s0:
	 dhcp4: no
	 addresses: [192.168.2.2/24]

primergyの :code:`/etc/netplan/01-netcfg.yaml` は以下の通りです。

.. code:: text

   network:
     version: 2
     renderer: networkd
     ethernets:
       enp0s25:
	 dhcp4: no
	 addresses: [192.168.1.4/24]
	 gateway4: 192.168.1.1
	 nameservers:
	   addresses: [192.168.1.1]
	 dhcp6: no
       enp2s0:
	 dhcp4: no
	 addresses: [192.168.2.3/24]
	 dhcp6: no

:code:`sudo netplan apply` コマンドで反映しました。

IPv4フォワーディングの有効化
=============================

IPv4のルータとして機能させるため、以下のコマンドを実行してIPv4フォワーディングを有効化します。

.. code:: console

   sudo sysctl net.ipv4.ip_forward=1

OS再起動時にもこの設定がされるように :code:`/etc/sysctl.d/99-sysctl.conf` 内の以下の行をアンコメントします。

.. code:: text

   # Uncomment the next line to enable packet forwarding for IPv4
   net.ipv4.ip_forward=1


IPoEの設定
==========

上記のように配線した状態で :code:`ip a` でIPアドレスを確認するとIPv6のグローバルアドレスがついていました。

:code:`ip -6 tunnel add` コマンドでmodeにipip6（IPv4 over IPv6トンネル）を指定することでトンネルを作成します。以下のようなシェルスクリプト :code:`/usr/local/sbin/ipoe.sh` を作成しました。

.. code:: sh

   #!/bin/sh -x
   set -eu

   #. /etc/default/ipoe

   tunnel_name="${TUNNEL_NAME}"
   wan_interface="${WAN_INTERFACE}"
   gateway_addr="${GATEWAY_ADDR}"

   case "$1" in
   start)
     while :; do
       local_addr=$(/sbin/ip a | /usr/bin/awk '/mngtmpaddr/ {print substr($2, 1, index($2, "/") - 1); exit}')
       if [ -n "$local_addr" ]; then
	 break
       fi

       sleep 1
     done
     /sbin/ip -6 tunnel add "${tunnel_name}" mode ipip6 remote "${gateway_addr}" local "${local_addr}" dev "${wan_interface}"
     /sbin/ip link set "${tunnel_name}" up
     /sbin/ip route delete default || :
     /sbin/ip route add default dev "${tunnel_name}"
     ;;
   stop)
     /sbin/ip link set "${tunnel_name}" down
     /sbin/ip -6 tunnel del "${tunnel_name}"
     /sbin/ip route add default dev "${wan_interface}"
     ;;
   *)
     echo "Usage: ipoe.sh (start|stop)"
     exit 1
     ;;
   esac

:code:`local_addr` を設定している箇所は以下のような行からIPv6アドレスを取り出しています。もしmngtmpaddrを含む行が複数ある場合は最初の行を採用します。

.. code:: text

   inet6 xxxx:xxx:xxx:xxxx:xxxx:xxx:xxxx:xxxx/64 scope global dynamic mngtmpaddr noprefixroute


サービス定義ファイル :code:`/etc/systemd/system/ipoe.service` は以下のようにしました。
`IPv6 トンネルブローカー設定 - ArchWiki <https://wiki.archlinux.jp/index.php/IPv6_%E3%83%88%E3%83%B3%E3%83%8D%E3%83%AB%E3%83%96%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%BC%E8%A8%AD%E5%AE%9A>`_ でsystemd-networkdを使ったトンネルの設定方法も見つけたのですが、こちらは試していません。

.. code:: text

   [Unit]
   Description=IPoE tunnel
   After=network.target

   [Service]
   Type=oneshot
   RemainAfterExit=yes
   EnvironmentFile=/etc/default/ipoe
   ExecStart=/usr/local/sbin/ipoe.sh start
   ExecStop=/usr/local/sbin/ipoe.sh stop

   [Install]
   WantedBy=multi-user.target

設定ファイル :code:`/etc/default/ipoe` は以下の通りです。

.. code:: text

   # tunnel device name
   TUNNEL_NAME=ip6tnl1

   # WAN network interface
   WAN_INTERFACE=enp0s25

   # Gateway address
   # You can get this address by running the following command.
   # dig -t aaaa gw.transix.jp
   GATEWAY_ADDR=2404:8e01::feed:100

:code:`WAN_INTERFACE` は実際の環境に応じて変更してください。
:code:`GATEWAY_ADDR` は :code:`dig -t aaaa gw.transix.jp` で調べられます。
NTT西日本エリアだと :code:`2404:8e01::feed:100` と :code:`2404:8e01::feed:101` のラウンドロビンになっていました。
`YAMAHA RTX1200の設定マニュアル <https://faq.interlink.or.jp/faq2/View/wcDisplayContent.aspx?id=220>`_ によるとNTT東日本エリアでは :code:`2404:8e00::feed:100` と :code:`2404:8e00::feed:101` で、エリアは `NTT東日本、NTT西日本のサービス提供地域について <http://www.ntt.co.jp/product/category/area.html>`_ で確認できます。

以下のコマンドで反映しました。

.. code:: console

   sudo systemctl daemon-reload
   sudo systemctl start ipoe
   sudo systemctl enable ipoe

iptablesの設定
==============

ファイアウォールは `UFW <https://help.ubuntu.com/community/UFW>`_ は使わず、iptablesとiptables-persistentを使っています。

.. code:: console

   sudo apt install iptables-persistent

設定は以下の通りです。LXDとdockerを使っているのでそれらの設定も含まれています。

:code:`/etc/iptables/rules.v4`

.. code:: text

   # Generated by iptables-save v1.6.1 on Sat Dec  1 14:44:22 2018
   *filter
   :INPUT ACCEPT [238:15436]
   :FORWARD ACCEPT [0:0]
   :OUTPUT ACCEPT [114:36056]
   :DOCKER - [0:0]
   :DOCKER-ISOLATION-STAGE-1 - [0:0]
   :DOCKER-ISOLATION-STAGE-2 - [0:0]
   :DOCKER-USER - [0:0]
   :WAN_IN - [0:0]
   :WAN_LOCAL - [0:0]
   -A INPUT -i lxdbr0 -p tcp -m tcp --dport 53 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A INPUT -i lxdbr0 -p udp -m udp --dport 53 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A INPUT -i lxdbr0 -p udp -m udp --dport 67 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A INPUT -i enp0s25 -j WAN_LOCAL
   -A FORWARD -o lxdbr0 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A FORWARD -i lxdbr0 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A FORWARD -i enp0s25 -j WAN_IN
   -A FORWARD -j DOCKER-USER
   -A FORWARD -j DOCKER-ISOLATION-STAGE-1
   -A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
   -A FORWARD -o docker0 -j DOCKER
   -A FORWARD -i docker0 ! -o docker0 -j ACCEPT
   -A FORWARD -i docker0 -o docker0 -j ACCEPT
   -A OUTPUT -o lxdbr0 -p tcp -m tcp --sport 53 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A OUTPUT -o lxdbr0 -p udp -m udp --sport 53 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A OUTPUT -o lxdbr0 -p udp -m udp --sport 67 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A DOCKER-ISOLATION-STAGE-1 -i docker0 ! -o docker0 -j DOCKER-ISOLATION-STAGE-2
   -A DOCKER-ISOLATION-STAGE-1 -j RETURN
   -A DOCKER-ISOLATION-STAGE-2 -o docker0 -j DROP
   -A DOCKER-ISOLATION-STAGE-2 -j RETURN
   -A DOCKER-USER -j RETURN
   -A WAN_IN -m comment --comment WAN_IN-10 -m state --state RELATED,ESTABLISHED -j RETURN
   -A WAN_IN -m comment --comment WAN_IN-20 -m state --state INVALID -j DROP
   -A WAN_IN -s 192.168.1.0/24 -p icmp -m comment --comment WAN_IN-30 -j ACCEPT
   -A WAN_IN -m comment --comment "WAN_IN-10000 default-action drop" -j LOG --log-prefix "[WAN_IN-default-D]"
   -A WAN_IN -m comment --comment "WAN_IN-10000 default-action drop" -j DROP
   -A WAN_LOCAL -m comment --comment WAN_LOCAL-10 -m state --state RELATED,ESTABLISHED -j RETURN
   -A WAN_LOCAL -m comment --comment WAN_LOCAL-20 -m state --state INVALID -j DROP
   -A WAN_LOCAL -s 192.168.1.0/24 -p icmp -m comment --comment WAN_LOCAL-30 -j ACCEPT
   -A WAN_LOCAL -s 192.168.1.0/24 -p vrrp -m comment --comment WAN_LOCAL-40 -j ACCEPT
   -A WAN_LOCAL -m comment --comment "WAN_LOCAL-10000 default-action drop" -j LOG --log-prefix "[WAN_LOCAL-default-D]"
   -A WAN_LOCAL -m comment --comment "WAN_LOCAL-10000 default-action drop" -j DROP
   COMMIT
   # Completed on Sat Dec  1 14:44:22 2018
   # Generated by iptables-save v1.6.1 on Sat Dec  1 14:44:22 2018
   *mangle
   :PREROUTING ACCEPT [2054464:7594383842]
   :INPUT ACCEPT [1989651:7566369331]
   :FORWARD ACCEPT [64047:27954808]
   :OUTPUT ACCEPT [937916:146880228]
   :POSTROUTING ACCEPT [1005398:175609326]
   -A POSTROUTING -o lxdbr0 -p udp -m udp --dport 68 -m comment --comment "generated for LXD network lxdbr0" -j CHECKSUM --checksum-fill
   COMMIT
   # Completed on Sat Dec  1 14:44:22 2018
   # Generated by iptables-save v1.6.1 on Sat Dec  1 14:44:22 2018
   *nat
   :PREROUTING ACCEPT [9291:2179690]
   :INPUT ACCEPT [3252:483996]
   :OUTPUT ACCEPT [3623:517216]
   :POSTROUTING ACCEPT [3545:511992]
   -A POSTROUTING -s 10.70.121.0/24 ! -d 10.70.121.0/24 -m comment --comment "generated for LXD network lxdbr0" -j MASQUERADE
   COMMIT
   # Completed on Sat Dec  1 14:44:22 2018

:code:`/etc/iptables/rules.v6`

.. code:: text

   # Generated by ip6tables-save v1.6.1 on Mon Nov 12 18:31:46 2018
   *nat
   :PREROUTING ACCEPT [5513:616145]
   :INPUT ACCEPT [5243:585056]
   :OUTPUT ACCEPT [2714:318879]
   :POSTROUTING ACCEPT [2645:312423]
   -A POSTROUTING -s fd42:f7bd:987f:2f80::/64 ! -d fd42:f7bd:987f:2f80::/64 -m comment --comment "generated for LXD network lxdbr0" -j MASQUERADE
   COMMIT
   # Completed on Mon Nov 12 18:31:46 2018
   # Generated by ip6tables-save v1.6.1 on Mon Nov 12 18:31:46 2018
   *filter
   :INPUT ACCEPT [251:73519]
   :FORWARD ACCEPT [0:0]
   :OUTPUT ACCEPT [226:60652]
   :WANv6_IN - [0:0]
   :WANv6_LOCAL - [0:0]
   -A INPUT -i lxdbr0 -p tcp -m tcp --dport 53 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A INPUT -i lxdbr0 -p udp -m udp --dport 53 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A INPUT -i lxdbr0 -p udp -m udp --dport 547 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A INPUT -i enp0s25 -j WANv6_LOCAL
   -A FORWARD -o lxdbr0 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A FORWARD -i lxdbr0 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A FORWARD -i enp0s25 -j WANv6_IN
   -A OUTPUT -o lxdbr0 -p tcp -m tcp --sport 53 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A OUTPUT -o lxdbr0 -p udp -m udp --sport 53 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A OUTPUT -o lxdbr0 -p udp -m udp --sport 547 -m comment --comment "generated for LXD network lxdbr0" -j ACCEPT
   -A WANv6_IN -m comment --comment WANv6_IN-10 -m state --state RELATED,ESTABLISHED -j RETURN
   -A WANv6_IN -m comment --comment WANv6_IN-20 -m state --state INVALID -j DROP
   -A WANv6_IN -p ipv6-icmp -m comment --comment WANv6_IN-30 -j RETURN
   -A WANv6_IN -m comment --comment "WANv6_IN-10000 default-action drop" -j LOG --log-prefix "[WANv6_IN-default-D]"
   -A WANv6_IN -m comment --comment "WANv6_IN-10000 default-action drop" -j DROP
   -A WANv6_LOCAL -m comment --comment WANv6_LOCAL-10 -m state --state RELATED,ESTABLISHED -j RETURN
   -A WANv6_LOCAL -m comment --comment WANv6_LOCAL-20 -m state --state INVALID -j DROP
   -A WANv6_LOCAL -p ipv6-icmp -m comment --comment WANv6_LOCAL-30 -j RETURN
   -A WANv6_LOCAL -p udp -m comment --comment WANv6_LOCAL-40 -m udp --sport 547 --dport 546 -j RETURN
   -A WANv6_LOCAL -p ipip -m comment --comment WANv6_LOCAL-50 -j RETURN
   -A WANv6_LOCAL -p tcp -m comment --comment WANv6_LOCAL-60 -m tcp --dport 22 -j RETURN
   -A WANv6_LOCAL -m comment --comment "WANv6_LOCAL-10000 default-action drop" -j LOG --log-prefix "[WANv6_LOCAL-default-D]"
   -A WANv6_LOCAL -m comment --comment "WANv6_LOCAL-10000 default-action drop" -j DROP
   COMMIT
   # Completed on Mon Nov 12 18:31:46 2018

ファイルの内容を変更したら以下のコマンドを実行して反映します。

.. code:: console

   sudo iptables-restore < /etc/iptables/rules.v4
   sudo ip6tables-restore < /etc/iptables/rules.v6

逆に設定をファイルに保存するには以下のコマンドを実行します。

.. code:: console

   sudo iptables-save | sudo tee /etc/iptables/rules.v4
   sudo ip6tables-save | sudo tee /etc/iptables/rules.v6

:code:`/etc/iptables/rules.v{4,6}` の内容はbitbucket.orgのプライベートレポジトリで管理しています。

keepalivedの設定
================

keepalivedを使ってVIP（仮想IPアドレス）を設定しています。
keepalivedは標準パッケージをインストールしました。

.. code:: console

   sudo apt install keepalived

:code:`/etc/keepalived/keepalived.conf` は以下の通りです。送信元のメールアドレスは途中を大文字にしてどこから送られたかを判別できるようにしています。

.. code:: text

   global_defs {
      notification_email {
	john.doe+home@example.com
      }
      notification_email_from jOhn.doe@example.com
      smtp_server localhost
      smtp_connect_timeout 30
      router_id 1
   }

   include vrrp.conf

beagleの :code:`/etc/keepalived/vrrp.conf` は以下の通りです。

.. code:: text

   vrrp_instance VRRP1 {
     state MASTER
     interface enp3s0
     virtual_router_id 1
     priority 200
     advert_int 1
     smtp_alert
     virtual_ipaddress {
       192.168.1.2/24
     }
   }

   vrrp_instance VRRP2 {
     state MASTER
     interface enp4s0
     virtual_router_id 2
     priority 200
     advert_int 1
     smtp_alert
     virtual_ipaddress {
       192.168.2.1/24
     }
   }

primergyの :code:`/etc/keepalived/vrrp.conf` は以下の通りです。

.. code:: text

   vrrp_instance VRRP1 {
     state BACKUP
     interface enp0s25
     virtual_router_id 1
     priority 100
     advert_int 1
     smtp_alert
     virtual_ipaddress {
       192.168.1.2/24
     }
   }

   vrrp_instance VRRP2 {
     state BACKUP
     interface enp2s0
     virtual_router_id 2
     priority 100
     advert_int 1
     smtp_alert
     virtual_ipaddress {
       192.168.2.1/24
     }
   }

stateとpriorityをbeagleとprimergyで上記のように設定することでbeagleのほうを優先するようにしています。

DHCPサーバの設定
================

DHCPサーバはisc-dhcp-serverパッケージを使っています。

.. code:: console

   sudo apt install isc-dhcp-server

beagleの :code:`/etc/dhcp/dhcpd.conf` は以下の通りです。

.. code:: text

   option domain-name-servers 192.168.1.1;

   default-lease-time 600;
   max-lease-time 7200;

   ddns-update-style none;

   subnet 192.168.2.0 netmask 255.255.255.0 {
     range 192.168.2.20 192.168.2.99;
     option routers 192.168.2.1;
   }

primergyの :code:`/etc/dhcp/dhcpd.conf` は以下の通りです。

.. code:: text

   option domain-name-servers 192.168.1.1;

   default-lease-time 600;
   max-lease-time 7200;

   ddns-update-style none;

   subnet 192.168.2.0 netmask 255.255.255.0 {
     range 192.168.2.100 192.168.2.189;
     option routers 192.168.2.1;
   }

beagleとprimergyでDHCPのrangeを重ならないように設定し、ゲートウェイにはVIPを指定しています。こうすることで片方のサーバが停止してもThinkPadなどDHCPクライアントのマシンはそのまま通信できると同僚に教わりました。ありがとうございます！
