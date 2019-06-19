# pyats

- General Information
- Website: https://developer.cisco.com/pyats/
- Documentation: https://developer.cisco.com/docs/pyats/
- Support: pyats-support-ext@cisco.com

# 自己紹介
勝男です。
ネットワークの自動化やってます
何かあればhttps://twitter.com/katu7414
まで

# pyats+genie
Cisco によって開発さている python ベースのテストフレームワークみたいです。
cisco live 2019で@tahigashからお話を聞いてこれは使えるかもしれないと思って
今回導入からcisco live US 2019でやっていたデモを実行するところまでを記事にしてみました。
Ansibleとは異なりネットワーク機器に特化しているためネットワークエンジニアには嬉しい機能もたくさんあります
またOSSでありもちろんコミットすることも可能です。
新しいネットワークの自動化もしくはコードへの一歩になるのでは？と思います。
今回はMacOSで構築しました。
## pyatsとは
Python Automated Test Systems の略で、python3 ベースのテストフレームワークのことです。
ciscoが開発しています。
## genieとは
pyATS のフレームワークの上で動く機能ライブラリのことです。
Genie のライブラリはOSSとなっており、誰でもコミットすることが可能です。
## デモ
### 準備
詳細URL→https://github.com/CiscoTestAutomation/CL-DEVWKS-2808
現在python virtual environmentが必須になっているので入れていきます

```
mkdir ~/pyats_genie
cd ~/pyats_genie
python3 -m venv .
 
source bin/activate
pip install --upgrade pip setuptools
pip install pyats genie
```


```
git clone https://github.com/CiscoTestAutomation/CL-DEVWKS-2808.git workshop
cd workshop
```
すると以下のようなファイルが展開される

```
(pyats_genie) shimakatsuyanoMacBook-puro:workshop shimakatsuya$ ls
LICENSE		README.md	env		files		mac_setup.sh	recordings	workshop.md
```

これで準備完了
次にfiles配下に既に作成済みのyamlファイルがあるので確認する。

simple-testbed.yaml

```
devices:
    csr1000v-1:
        type: router
        os: iosxe
        connections:
            console:
                protocol: telnet
                ip: 172.25.192.90
                port: 17001

```


続いてこのyamlファイルが構文的に正しいか確認します。

```
(pyats_genie) shimakatsuyanoMacBook-puro:workshop shimakatsuya$ pyats validate testbed files/simple-testbed.yaml
Loading testbed file: files/simple-testbed.yaml
--------------------------------------------------------------------------------

Testbed Name:
    simple-testbed

Testbed Devices:
.
`-- csr1000v-1 [router/iosxe]

Warning Messages
----------------
 - Device 'csr1000v-1' missing 'platform' definition
 - Device 'csr1000v-1' has no interface definitions

YAML Lint Messages
------------------

```


platformとinterfaceの定義がないと怒られますが今回使う分には問題ないのでそのまま進めましょう
続いてPythonにこのtestbedファイルを読み込ませていきます。

その前に環境を読み込ませます。
今回は実機や仮想環境はないのでUniconを使用してそこにまるで仮想環境があるようにします。
この環境についてはrecordings配下にあるので確認するといいと思います。

実行コマンド

```
export unicon_replay=`pwd`/recordings/replay
export unicon_speed=10
```
これで環境の準備は設定完了です。
それではPythonのシェルを起動して以下のコマンドを投入していきましょう

```
from genie.testbed import load

# load the testbed file
testbed = load('files/simple-testbed.yaml')

# let's see our testbed devices
testbed.devices
# TopologyDict({'csr1000v-1': <Device csr1000v-1 at 0x10cd0a3c8>})

# get the device we are interested in
csr = testbed.devices['csr1000v-1']

# connect and run commands
csr.connect()
csr.execute('show interfaces')
```

実際の投入ログ

```
(pyats_genie) shimakatsuyanoMacBook-puro:workshop shimakatsuya$ python3
Python 3.7.2 (default, Dec 27 2018, 07:35:52)
[Clang 10.0.0 (clang-1000.11.45.5)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> from genie.testbed import load
>>> testbed = load('files/simple-testbed.yaml')
>>> testbed.devices
TopologyDict({'csr1000v-1': <Device csr1000v-1 at 0x11399bdd8>})
>>> csr = testbed.devices['csr1000v-1']
>>> csr.connect()
[2019-06-18 12:37:32,091] +++ csr1000v-1 logfile /tmp/csr1000v-1-cli-20190618T123732091.log +++
[2019-06-18 12:37:32,092] +++ Unicon plugin iosxe +++
"Escape character is '^]'.\r\n\r\ncsr1000v-1#"
>>> csr.execute('show interfaces')
'GigabitEthernet1 is up, line protocol is up \r\n  Hardware is CSR vNIC, address is 5e00.0000.0000 (bia 5e00.0000.0000)\r\n  Internet address is 10.255.8.19/16\r\n  MTU 1500 bytes, BW 1000000 Kbit/sec, DLY 10 usec, \r\n     reliability 255/255, txload 1/255, rxload 1/255\r\n (ログが長すぎたので一部カットしてます)
>>> exit()
```

接続できてそしてshow inteefacesが取れてそうですね。
ただ見にくいので整形した形でみたいですね。それはこの後やるgenie shellで見ることが出来るのでそれまではこの形です。

---
では続きいきます。
次はもう少し拡大したyamlファイルを確認します。これも/files配下にあります。
2台作成したyamlです。
two-device-testbed.yaml

```
testbed:
    name: CLUS-19-DevWks-2808

devices:
    nx-osv-1:
        type: switch
        os: nos
        alias: 'uut'
        tacacs:
            login_prompt: "login:"
            password_prompt: "Password:"
            username: "admin"
        passwords:
            tacacs: Cisc0123
            enable: admin
            line: admin
        connections:
            console:
                protocol: telnet
                ip: "172.25.192.90"
                port: 17028

    csr1000v-1:
        type: router
        os: iosxe
        alias: helper
        tacacs:
            login_prompt: 'login:'
            password_prompt: 'Password:'
            username: cisco
        passwords:
            tacacs: cisco
            enable: cisco
            line: cisco
        connections:
            console:
              protocol: telnet
              ip: 172.25.192.90
              port: 17002
```
続いて実行するPython

```
from genie.testbed import load

# load the testbed file
testbed = load('two-device-testbed.yaml')

# because we assigned aliases to each device, we can refer by alias instead
nx = testbed.devices['uut']
csr = testbed.devices['helper']

# connect and run commands
for device in [nx, csr]:
    device.connect()
    device.execute('show version')

```
実行結果(一部のみ抜粋)

```
[2019-06-18 12:39:03,065] +++ nx-osv-1 logfile /tmp/nx-osv-1-default-20190618T123903064.log +++
[2019-06-18 12:39:03,066] +++ Unicon plugin generic +++
"Escape character is '^]'.\r\n\r\r\n\rnx-osv-1# "
'Cisco Nexus Operating System (NX-OS) Software\r\nTAC support: http://www.cisco.com/tac\r\nDocuments: http://www.cisco.com/en/US/products/ps9372/tsd_products_support_series_home.html\r\nCopyright (c) 2002-2016, Cisco Systems, Inc. All rights reserved.\r\nThe copyrights to certain works contained herein are owned by\r\nother third parties and are used and distributed under license.\r\nSome parts of this software are covered under the GNU Public\r\nLicense. A copy of the license is available at\r\nhttp://www.gnu.org/licenses/gpl.html.\r\n\r\nNX-OSv is a demo version of the Nexus Operating System\r\n\r\nSoftware\r\n  loader:    version N/A\r\n  kickstart: version 7.3(0)D1(1)\r\n  system:    version 7.3

[2019-06-18 12:39:03,349] +++ csr1000v-1 logfile /tmp/csr1000v-1-cli-20190618T123903349.log +++
[2019-06-18 12:39:03,350] +++ Unicon plugin iosxe +++
"Escape character is '^]'.\r\n\r\ncsr1000v-1#"
'Cisco IOS XE Software, Version 16.09.01\r\nCisco IOS Software [Fuji], Virtual XE Software (X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 16.9.1, RELEASE SOFTWARE (fc2)\r\nTechnical Support: http://www.cisco.com/techsupport\r\nCopyright (c) 1986-2018
```

それぞれ取れてそうですね。続けてこんなyamlを書くことが出来るという紹介があったのでのせておきます。

---

Ansibelとまた異なる部分になりますが複数の接続方法を試すことができます。

今回のyaml

```
devices:
    csr1000v-1:
        type: router
        os: iosxe
        connections:
            console:
                protocol: telnet
                ip: 172.25.192.90
                port: 17001
            mgmt:
                protocol: ssh
                ip: 10.1.3.50
```

---


次はAnsibleやterraformを書くときに気になるパスワードの下りです。
もちろん環境変数を利用してファイルに直接パスワードを書かなくても良い方法もあります。
それが次のような書き方です。

```
devices:
    csr1000v-1:
        type: router
        os: iosxe
        alias: helper
        tacacs:
            username: "%ENV{DEVICE_USERNAME}"
        passwords:
            tacacs: "%ENV{DEVICE_TACACS_PWD}"
            enable: "%ENV{DEVICE_ENABLE_PWD}"
            line: "%ENV{DEVICE_LINE_PWD}"
        connections:
            console:
              protocol: telnet
              ip: 172.25.192.90
              port: 17002
```

---

続いてgenie shellというシェルを利用して様々な出力を確認します。
まずfiles配下にあるworkshop-testbed.yamlを確認します。

```
testbed:
    name: CL-DEVWKS-2808-WORKSHOP-MOCK-TB

devices:
    nx-osv-1:
        type: 'router'
        os: 'nxos'
        alias: 'uut'
        connections:
            console:
                command: mock_device_cli --os nxos --mock_data_dir recordings/yamls/nxos --state execute
                protocol: mock

    csr1000v-1:
        type: 'router'
        os: "iosxe"
        alias: 'helper'
        connections:
            console:
                command: mock_device_cli --os iosxe --mock_data_dir recordings/yamls/csr --state execute
                protocol: mock
```

ここからわかることですがおそらくrecordings/yamlsにあるyamlを参考にして仮想ルーター的な物を作っているのかな〜という感じがします。
それでは実際にそういう動作を示すのか？
実行してみましょう

```
>>> from genie.testbed import load
>>> testbed = load('files/workshop-testbed.yaml')
>>> uut = testbed.devices['uut']
>>> uut.connect()
[2019-06-18 13:35:28,349] +++ nx-osv-1 logfile /tmp/nx-osv-1-cli-20190618T133528348.log +++
[2019-06-18 13:35:28,350] +++ Unicon plugin nxos +++
"Escape character is '^]'.\r\n\r\r\n\rnx-osv-1# "
>>> exit()
(pyats_genie) shimakatsuyanoMacBook-puro:workshop shimakatsuya$ genie shell --testbed-file files/workshop-testbed.yaml
Welcome to Genie Interactive Shell
==================================
Python 3.7.2 (default, Dec 27 2018, 07:35:52)
[Clang 10.0.0 (clang-1000.11.45.5)]

>>> from genie.conf import Genie
>>> testbed = Genie.init('files/workshop-testbed.yaml')
>>> with open('None', 'rb') as f:
>>>     pickle_data = dill.load(f)
-------------------------------------------------------------------------------
>>>
```

genie shellが起動しましたね
ではこの状態でネットワーク機器の情報を見ていきましょう。

```
>>> uut = testbed.devices['uut']
>>> uut.connect()
[2019-06-18 13:47:21,804] +++ nx-osv-1 logfile /tmp/nx-osv-1-cli-20190618T134721801.log +++
[2019-06-18 13:47:21,805] +++ Unicon plugin nxos +++
nx-osv-1#
+++ nx-osv-1: connecting command ' +++
Escape character is '^]'.

nx-osv-1#
"Escape character is '^]'.\r\n\r\r\n\rnx-osv-1# "
>>>
```

はいこれで接続できました。ではshow versionを見ていきましょう。

```
>>> uut.parse('show version')
nx-osv-1#
+++ nx-osv-1: execute command 'show version' +++
show version
Cisco Nexus Operating System (NX-OS) Software
TAC support: http://www.cisco.com/tac
Documents: http://www.cisco.com/en/US/products/ps9372/tsd_products_support_series_home.html
Copyright (c) 2002-2016, Cisco Systems, Inc. All rights reserved.
The copyrights to certain works contained herein are owned by
other third parties and are used and distributed under license.
Some parts of this software are covered under the GNU Public
License. A copy of the license is available at
http://www.gnu.org/licenses/gpl.html.

NX-OSv is a demo version of the Nexus Operating System

Software
  loader:    version N/A
  kickstart: version 7.3(0)D1(1)
  system:    version 7.3(0)D1(1)
  kickstart image file is: bootflash:///titanium-d1-kickstart.7.3.0.D1.1.bin
  kickstart compile time:  1/11/2016 16:00:00 [02/11/2016 10:30:12]
  system image file is:    bootflash:///titanium-d1.7.3.0.D1.1.bin
  system compile time:     1/11/2016 16:00:00 [02/11/2016 13:08:11]

```

綺麗だ整形されている。これは見やすいですね。ちゃんとparseしてくれているので良さそうですね。
続いてshow interfaceをparseしてみましょう。(内容が多いため一部のみのせます)

```
loopback0 is up
admin state is up
  Hardware: Loopback
  Internet Address is 10.2.2.2/32
  MTU 1500 bytes, BW 8000000 Kbit, DLY 5000 usec
  reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation LOOPBACK, medium is broadcast
  Port mode is routed
  Auto-mdix is turned off
    18270 packets input 913055 bytes
    0 multicast frames 0 compressed
    0 input errors 0 frame 0 overrun 0 fifo
    0 packets output 0 bytes 0 underruns
    0 output errors 0 collisions 0 fifo
    0 out_carrier_errors

loopback1 is up
admin state is up
  Hardware: Loopback
  Internet Address is 10.22.22.22/32
  MTU 1500 bytes, BW 8000000 Kbit, DLY 5000 usec
  reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation LOOPBACK, medium is broadcast
  Port mode is routed
  Auto-mdix is turned off
    0 packets input 0 bytes
    0 multicast frames 0 compressed
    0 input errors 0 frame 0 overrun 0 fifo
    0 packets output 0 bytes 0 underruns
    0 output errors 0 collisions 0 fifo
    0 out_carrier_errors
```

これもちゃんと整形されていて良さそうですね。
どんなものがparseされるのか一覧ページがあるので
見てみてください→https://pubhub.devnetcloud.com/media/pyats-packages/docs/genie/genie_libs/#/parsers
またOSSなのでもちろんソースコードも読めます。→https://github.com/CiscoTestAutomation/genieparser/blob/master/src/genie/libs/parser/nxos/show_interface.py#L145

---
まだ続きます。
続いては機器によって微妙にコマンドは微妙に違いますがそういったものは気にすることはなく
情報を取得出来るという例をBGPを例にやっていきます。
そのモデルリストは次のページで確認することもできます。
https://pubhub.devnetcloud.com/media/pyats-packages/docs/genie/genie_libs/#/models
実行するコマンド

```
# get our device
uut = testbed.devices['uut']

# connect to it
uut.connect()

# let's learn a whole model instead
bgp = uut.learn('bgp')

# now let's look at what we learnt
import pprint
pprint.pprint(bgp.info)
```

実行結果

```
>>> bgp = uut.learn('bgp')
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp process vrf all' +++
show bgp process vrf all

BGP Process Information
BGP Process ID                 : 8613
BGP Protocol Started, reason:  : configuration
BGP Protocol Tag               : 65000
BGP Protocol State             : Running
BGP MMODE                      : Not Initialized
BGP Memory State               : OK
BGP asformat                   : asplain

BGP attributes information
Number of attribute entries    : 2
HWM of attribute entries       : 2
Bytes used by entries          : 200
Entries pending delete         : 0
HWM of entries pending delete  : 0
BGP paths per attribute HWM    : 1
BGP AS path entries            : 0
Bytes used by AS path entries  : 0

Information regarding configured VRFs:

BGP Information for VRF default
VRF Id                         : 1
VRF state                      : UP
Router-ID                      : 10.2.2.2
Configured Router-ID           : 10.2.2.2
Confed-ID                      : 0
Cluster-ID                     : 0.0.0.0
No. of configured peers        : 1
No. of pending config peers    : 0
No. of established peers       : 1
VRF RD                         : Not configured

    Information for address family IPv4 Unicast in VRF default
    Table Id                   : 1
    Table state                : UP
    Peers      Active-peers    Routes     Paths      Networks   Aggregates
    1          1               2          2          1          0
    Redistribution
        None

    Wait for IGP convergence is not configured


    Nexthop trigger-delay
        critical 3000 ms
        non-critical 10000 ms

    Information for address family IPv6 Unicast in VRF default
    Table Id                   : 80000001
    Table state                : UP
    Peers      Active-peers    Routes     Paths      Networks   Aggregates
    0          0               0          0          0          0

    Redistribution
        None

    Wait for IGP convergence is not configured


    Nexthop trigger-delay
        critical 3000 ms
        non-critical 10000 ms
nx-osv-1#
+++ nx-osv-1: execute command 'show running-config | inc peer-session' +++
show running-config | inc peer-session
Could not learn <class 'genie.libs.parser.nxos.show_bgp.ShowBgpPeerSession'>
Parser Output is empty
nx-osv-1#
+++ nx-osv-1: execute command 'show running-config | inc peer-policy' +++
show running-config | inc peer-policy
Could not learn <class 'genie.libs.parser.nxos.show_bgp.ShowBgpPeerPolicy'>
Parser Output is empty
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp vrf all all dampening parameters' +++
show bgp vrf all all dampening parameters
Could not learn <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllAllDampeningParameters'>
Parser Output is empty
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp vrf all all nexthop-database' +++
show bgp vrf all all nexthop-database

Next Hop table for VRF default, address family IPv4 Unicast:
Next-hop trigger-delay(miliseconds)
  Critical: 3000 Non-critical: 10000
IPv4 Next-hop table

Nexthop: 0.0.0.0, Flags: 0x2, Refcount: 1, IGP cost: 0
IGP Route type: 0, IGP preference: 0
Nexthop is not-attached local unreachable not-labeled
Nexthop last resolved: never, using 0.0.0.0/0
Metric next advertise: Never
RNH epoch: 0

Nexthop: 10.1.1.1, Flags: 0x1, Refcount: 1, IGP cost: 41
IGP Route type: 0, IGP preference: 110
Attached nexthop: 10.0.2.1, Interface: Ethernet2/2
Attached nexthop: 10.0.1.1, Interface: Ethernet2/1
Nexthop is not-attached not-local reachable not-labeled
Nexthop last resolved: 00:01:31, using 10.1.1.1/32
Metric next advertise: Never
RNH epoch: 3
IPv6 Next-hop table

Next Hop table for VRF default, address family IPv6 Unicast:
Next-hop trigger-delay(miliseconds)
  Critical: 3000 Non-critical: 10000
IPv4 Next-hop table
IPv6 Next-hop table
nx-osv-1#
+++ nx-osv-1: execute command 'show routing vrf all' +++
show routing vrf all
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.0.1.0/24, ubest/mbest: 1/0, attached
    *via 10.0.1.2, Eth2/1, [0/0], 00:01:41, direct
10.0.1.2/32, ubest/mbest: 1/0, attached
    *via 10.0.1.2, Eth2/1, [0/0], 00:01:41, local
10.0.2.0/24, ubest/mbest: 1/0, attached
    *via 10.0.2.2, Eth2/2, [0/0], 6d01h, direct
10.0.2.2/32, ubest/mbest: 1/0, attached
    *via 10.0.2.2, Eth2/2, [0/0], 6d01h, local
10.1.1.1/32, ubest/mbest: 2/0
    *via 10.0.1.1, Eth2/1, [110/41], 00:01:32, ospf-1, intra
    *via 10.0.2.1, Eth2/2, [110/41], 00:01:32, ospf-1, intra
10.2.2.2/32, ubest/mbest: 2/0, attached
    *via 10.2.2.2, Lo0, [0/0], 6d01h, local
    *via 10.2.2.2, Lo0, [0/0], 6d01h, direct
10.11.11.11/32, ubest/mbest: 1/0
    *via 10.1.1.1, [200/0], 00:01:32, bgp-65000, internal, tag 65000,
10.22.22.22/32, ubest/mbest: 2/0, attached
    *via 10.22.22.22, Lo1, [0/0], 6d01h, local
    *via 10.22.22.22, Lo1, [0/0], 6d01h, direct
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp vrf all all' +++
show bgp vrf all all
BGP routing table information for VRF default, address family IPv4 Unicast
BGP table version is 7, local router ID is 10.2.2.2
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i10.11.11.11/32     10.1.1.1                 0        100          0 i
*>l10.22.22.22/32     0.0.0.0                           100      32768 i
nx-osv-1#
+++ nx-osv-1: execute command 'show vrf' +++
show vrf
VRF-Name                           VRF-ID State   Reason
default                                 1 Up      --
management                              2 Up      --
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp vrf default all neighbors' +++
show bgp vrf default all neighbors
BGP neighbor is 10.1.1.1,  remote AS 65000, ibgp link, Peer index 1
  BGP version 4, remote router ID 10.1.1.1
  BGP state = Established, up for 6d01h
  Using loopback0 as update source for this peer
  Last read 00:00:18, hold time = 180, keepalive interval is 60 seconds
  Last written 00:00:27, keepalive timer expiry due 00:00:32
  Received 9588 messages, 0 notifications, 0 bytes in queue
  Sent 8711 messages, 0 notifications, 0 bytes in queue
  Connections established 1, dropped 0
  Last reset by us never, due to No error
  Last reset by peer never, due to No error

  Neighbor capabilities:
  Dynamic capability: advertised (mp, refresh, gr)
  Dynamic capability (old): advertised
  Route refresh capability (new): advertised received
  Route refresh capability (old): advertised received
  4-Byte AS capability: advertised received
  Address family IPv4 Unicast: advertised received
  Graceful Restart capability: advertised

  Graceful Restart Parameters:
  Address families advertised to peer:
    IPv4 Unicast
  Address families received from peer:
  Forwarding state preserved by peer for:
  Restart time advertised to peer: 120 seconds
  Stale time for routes advertised by peer: 300 seconds
  Extended Next Hop Encoding Capability: advertised

  Message statistics:
                              Sent               Rcvd
  Opens:                         1                  1
  Notifications:                 0                  0
  Updates:                       2                  2
  Keepalives:                 8708               9585
  Route Refresh:                 0                  0
  Capability:                    0                  0
  Total:                      8711               9588
  Total bytes:              165505             182175
  Bytes in queue:                0                  0

  For address family: IPv4 Unicast
  BGP table version 7, neighbor version 7
  1 accepted paths consume 80 bytes of memory
  1 sent paths
  Third-party Nexthop will not be computed.

  Local host: 10.2.2.2, Local port: 179
  Foreign host: 10.1.1.1, Foreign port: 24407
  fd = 60
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp vrf management all neighbors' +++
show bgp vrf management all neighbors
Unknown vrf management
Could not learn <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllNeighbors'>
Parser Output is empty
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp vrf all all summary' +++
show bgp vrf all all summary
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 10.2.2.2, local AS number 65000
BGP table version is 7, IPv4 Unicast config peers 1, capable peers 1
2 network entries and 2 paths using 288 bytes of memory
BGP attribute entries [2/288], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.1.1.1        4 65000    9588    8711        7    0    0    6d01h 1

BGP summary information for VRF default, address family IPv6 Unicast
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp vrf default all neighbors 10.1.1.1 advertised-routes' +++
show bgp vrf default all neighbors 10.1.1.1 advertised-routes

Peer 10.1.1.1 routes for address family IPv4 Unicast:
BGP table version is 7, local router ID is 10.2.2.2
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup

   Network            Next Hop            Metric     LocPrf     Weight Path
*>l10.22.22.22/32     0.0.0.0                           100      32768 i


Peer 10.1.1.1 routes for address family IPv4 Multicast:

Peer 10.1.1.1 routes for address family IPv6 Unicast:
BGP table version is 2, local router ID is 10.2.2.2
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup

   Network            Next Hop            Metric     LocPrf     Weight Path

Peer 10.1.1.1 routes for address family IPv6 Multicast:

Peer 10.1.1.1 routes for address family VPNv4 Unicast:

Peer 10.1.1.1 routes for address family VPNv6 Unicast:

Peer 10.1.1.1 routes for address family IPv4 MDT:

Peer 10.1.1.1 routes for address family IPv6 Label Unicast:

Peer 10.1.1.1 routes for address family L2VPN VPLS:

Peer 10.1.1.1 routes for address family IPv4 MVPN:

Peer 10.1.1.1 routes for address family IPv6 MVPN:

Peer 10.1.1.1 routes for address family IPv4 Label Unicast:

Peer 10.1.1.1 routes for address family L2VPN EVPN:
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp vrf default all neighbors 10.1.1.1 routes' +++
show bgp vrf default all neighbors 10.1.1.1 routes

Peer 10.1.1.1 routes for address family IPv4 Unicast:
BGP table version is 7, local router ID is 10.2.2.2
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup

   Network            Next Hop            Metric     LocPrf     Weight Path
*>i10.11.11.11/32     10.1.1.1                 0        100          0 i


Peer 10.1.1.1 routes for address family IPv4 Multicast:

Peer 10.1.1.1 routes for address family IPv6 Unicast:
BGP table version is 2, local router ID is 10.2.2.2
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-injected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup

   Network            Next Hop            Metric     LocPrf     Weight Path

Peer 10.1.1.1 routes for address family IPv6 Multicast:

Peer 10.1.1.1 routes for address family VPNv4 Unicast:

Peer 10.1.1.1 routes for address family VPNv6 Unicast:

Peer 10.1.1.1 routes for address family IPv4 MDT:

Peer 10.1.1.1 routes for address family IPv6 Label Unicast:

Peer 10.1.1.1 routes for address family L2VPN VPLS:

Peer 10.1.1.1 routes for address family IPv4 MVPN:

Peer 10.1.1.1 routes for address family IPv6 MVPN:

Peer 10.1.1.1 routes for address family IPv4 Label Unicast:

Peer 10.1.1.1 routes for address family L2VPN EVPN:
nx-osv-1#
+++ nx-osv-1: execute command 'show bgp vrf default all neighbors 10.1.1.1 received-routes' +++
show bgp vrf default all neighbors 10.1.1.1 received-routes

Inbound soft reconfiguration for IPv4 Unicast not enabled on 10.1.1.1

Inbound soft reconfiguration for IPv4 Multicast not enabled on 10.1.1.1

Inbound soft reconfiguration for IPv6 Unicast not enabled on 10.1.1.1

Inbound soft reconfiguration for IPv6 Multicast not enabled on 10.1.1.1

Inbound soft reconfiguration for VPNv4 Unicast not enabled on 10.1.1.1

Inbound soft reconfiguration for VPNv6 Unicast not enabled on 10.1.1.1

Inbound soft reconfiguration for IPv4 MDT not enabled on 10.1.1.1

Inbound soft reconfiguration for IPv6 Label Unicast not enabled on 10.1.1.1

Inbound soft reconfiguration for L2VPN VPLS not enabled on 10.1.1.1

Inbound soft reconfiguration for IPv4 MVPN not enabled on 10.1.1.1

Inbound soft reconfiguration for IPv6 MVPN not enabled on 10.1.1.1

Inbound soft reconfiguration for IPv4 Label Unicast not enabled on 10.1.1.1

Inbound soft reconfiguration for L2VPN EVPN not enabled on 10.1.1.1
Could not learn <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllNeighborsReceivedRoutes'>
Parser Output is empty
+====================================================================================================================================================+
| Commands for learning feature 'Bgp'                                                                                                                |
+====================================================================================================================================================+
| - Parsed commands                                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------|
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpProcessVrfAll'>                                                                              |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllAllNextHopDatabase'>                                                                   |
|   cmd: <class 'genie.libs.parser.nxos.show_routing.ShowRoutingVrfAll'>                                                                             |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllAll'>                                                                                  |
|   cmd: <class 'genie.libs.parser.nxos.show_vrf.ShowVrf'>                                                                                           |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllNeighbors'>, arguments: {'vrf':'default'}                                              |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllAllSummary'>                                                                           |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllNeighborsAdvertisedRoutes'>, arguments: {'neighbor':'10.1.1.1','vrf':'default'}        |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllNeighborsRoutes'>, arguments: {'neighbor':'10.1.1.1','vrf':'default'}                  |
|====================================================================================================================================================|
| - Commands with empty output                                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------|
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpPeerSession'>                                                                                |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpPeerPolicy'>                                                                                 |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllAllDampeningParameters'>                                                               |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllNeighbors'>, arguments: {'vrf':'management'}                                           |
|   cmd: <class 'genie.libs.parser.nxos.show_bgp.ShowBgpVrfAllNeighborsReceivedRoutes'>, arguments: {'neighbor':'10.1.1.1','vrf':'default'}          |
|====================================================================================================================================================|

```

続いて辞書型で書き出します。


```

>>> import pprint
>>> pprint.pprint(bgp.info)
{'instance': {'default': {'bgp_id': 65000,
                          'protocol_state': 'running',
                          'vrf': {'default': {'address_family': {'ipv4 unicast': {'distance_internal_as': 200,
                                                                                  'nexthop_trigger_delay_critical': 3000,
                                                                                  'nexthop_trigger_delay_non_critical': 10000,
                                                                                  'nexthop_trigger_enable': True},
                                                                 'ipv6 unicast': {'nexthop_trigger_delay_critical': 3000,
                                                                                  'nexthop_trigger_delay_non_critical': 10000,
                                                                                  'nexthop_trigger_enable': True}},
                                              'cluster_id': '0.0.0.0',
                                              'confederation_identifier': 0,
                                              'neighbor': {'10.1.1.1': {'address_family': {'ipv4 unicast': {'bgp_table_version': 7,
                                                                                                            'session_state': 'established'}},
                                                                        'bgp_negotiated_capabilities': {'dynamic_capability': 'advertised '
                                                                                                                              '(mp, '
                                                                                                                              'refresh, '
                                                                                                                              'gr)',
                                                                                                        'dynamic_capability_old': 'advertised',
                                                                                                        'graceful_restart': 'advertised',
                                                                                                        'route_refresh': 'advertised '
                                                                                                                         'received',
                                                                                                        'route_refresh_old': 'advertised '
                                                                                                                             'received'},
                                                                        'bgp_negotiated_keepalive_timers': {'hold_time': 180,
                                                                                                            'keepalive_interval': 60},
                                                                        'bgp_neighbor_counters': {'messages': {'received': {'bytes_in_queue': 0,
                                                                                                                            'capability': 0,
                                                                                                                            'keepalives': 9585,
                                                                                                                            'notifications': 0,
                                                                                                                            'opens': 1,
                                                                                                                            'route_refresh': 0,
                                                                                                                            'total': 9588,
                                                                                                                            'total_bytes': 182175,
                                                                                                                            'updates': 2},
                                                                                                               'sent': {'bytes_in_queue': 0,
                                                                                                                        'capability': 0,
                                                                                                                        'keepalives': 8708,
                                                                                                                        'notifications': 0,
                                                                                                                        'opens': 1,
                                                                                                                        'route_refresh': 0,
                                                                                                                        'total': 8711,
                                                                                                                        'total_bytes': 165505,
                                                                                                                        'updates': 2}}},
                                                                        'bgp_session_transport': {'connection': {'last_reset': 'never',
                                                                                                                 'reset_reason': 'no '
                                                                                                                                 'error',
                                                                                                                 'state': 'established'},
                                                                                                  'transport': {'foreign_host': '10.1.1.1',
                                                                                                                'foreign_port': '24407',
                                                                                                                'local_host': '10.2.2.2',
                                                                                                                'local_port': '179'}},
                                                                        'bgp_version': 4,
                                                                        'holdtime': 180,
                                                                        'keepalive_interval': 60,
                                                                        'local_as_as_no': 'None',
                                                                        'remote_as': 65000,
                                                                        'session_state': 'established',
                                                                        'shutdown': False,
                                                                        'up_time': '6d01h',
                                                                        'update_source': 'loopback0'}},
                                              'router_id': '10.2.2.2'}}}}}
>>>

```

実行結果からわかるようにそれぞれのOSのコマンドの差分を気にしないでbgp.infoなどで取り出せるので楽ですね
特にios,ios-xe,ios-xrの差分を気にしないでいいのは楽ですね。

---
ようやく最後です。
最後にここまでやった情報を一括で抽出する物を作成します

test.py

```

import os
HERE = os.path.dirname(__file__)

from tabulate import tabulate
from genie.testbed import load

if __name__ == '__main__':
    testbed = load(os.path.join(HERE, 'files/workshop-testbed.yaml'))

    uut = testbed.devices['uut']
    helper = testbed.devices['helper']

    uut.connect()
    info = uut.learn('platform')
    bgp = uut.learn('bgp')

    # print useful information
    print('\n' + '-'*80)
    print('Hostname: %s' % uut.name)
    print('Software Version: %s %s\n' % (info.os, info.version))

    nbr_info = []
    for bgp_instance in bgp.info['instance']:
        for vrf in bgp.info['instance'][bgp_instance]['vrf']:
            for nbr in bgp.info['instance'][bgp_instance]['vrf'][vrf]['neighbor']:
                state = bgp.info['instance'][bgp_instance]['vrf'][vrf]['neighbor'][nbr]['session_state']
                nbr_info.append((bgp_instance, vrf, nbr, state))

    print(tabulate(nbr_info, headers = ['BGP Instance', 'VRF', 'Neighbor', 'State']))

    active_nbr = len([i for i in nbr_info if i[-1].lower() == 'established'])
    print('\nTotal # of Active Neighbors: %s' % active_nbr)
    print('-'*80 + '\n')

```
実行してみましょう

```
(pyats_genie) shimakatsuyanoMacBook-puro:workshop shimakatsuya$ python test.py
[2019-06-18 14:53:29,022] +++ nx-osv-1 logfile /tmp/nx-osv-1-cli-20190618T145329021.log +++
[2019-06-18 14:53:29,022] +++ Unicon plugin nxos +++

--------------------------------------------------------------------------------
Hostname: nx-osv-1
Software Version: NX-OS 7.3(0)D1(1)

BGP Instance    VRF      Neighbor    State
--------------  -------  ----------  -----------
default         default  10.1.1.1    established

Total # of Active Neighbors: 1
--------------------------------------------------------------------------------

```

綺麗に取れましたね。
## 感想
parseしてくれるライブラリーも多いし良さそうな雰囲気は感じる。
だけどネットワークを自動化するにも色々あるがこれは状態確認するツールとしてかなり優秀な感じがあります。
また状態を変更するAnsibleとの相性の良さも感じます。
ただ学習コストはAnsibleより高い感じがします。なのでこれを導入してどうやって継続させていくか？はちゃんと考えないといけないと思います。
後ドキュメントも少ないので増やさないと厳しいかなと感じます。

自分がネットワークの自動化をするならAnsibleで変更してこれで状態確認ですかね？
みなさんどうですか？

自分にこのツールを紹介してくれた@tahigashさんありがとうございます。
