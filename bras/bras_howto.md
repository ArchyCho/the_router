��������! ������� �� �����������. ��� ��������� ���������
������ � ���������� ������.

# 1. ��������

� ���� howto ����� ����������� ��������� ��������� bras ������� � ������� ��������� TheRouter
������������ ������� Gentoo Linux � ���� �������������� �������� (Dhcpd, FreeRadius, Mysql, Quagga).

BRAS ������ ����� ������������ � ��������� �������� ���� � ����� ��������� ��������� ������:

 * ���������� ���������������� ethernet qinq ������;
 * ��������� � �������������� ������������� � ������� ��������� Radius;
 * ��������������� ������� ��������������� ������������� �� c���������� ��� ����;
 * ����������� �������� �������������;
 * ������������� ����������������� ������� � ������� ������������ ���������� �������������;
 * ������������� ����� ������� ������������� � pool ����� ������� � ������� NAT;
	
# 2. ��������� ������������ ������� Gentoo � ��������� TheRouter

������ <a href="https://github.com/alexk99/the_router/blob/master/install.md">����������</a>
���������� DPDK � TheRouter �� ������ � ������������ �������� Gentoo Linux.

# 3. ����� ����, � ������� ����� ���������� BRAS.

���� ������� �� linux ����� H4, �� ������� ���������� TheRouter � ����������� ���
��� ������ ��������� (Dhcpd, FreeRadius, Mysql, Quagga), ������������ uplink ��������������,
������������� � ��������, � ����������� Subscriber 1-4. ���������� 1 � 2 ���������� �����
��������� �����, 3-4 ����� L2 broadcast domain.

## 3.1. L2 ����� ����

<img src="http://therouter.net/images/bras/bras howto l2.png">

## 3.2. L3 ����� ����

<img src="http://therouter.net/images/bras/bras_howto_l3.png">

��� �������� ����������� ���������� the_router, ����������� � ���������������� �����, ���
��������� v3 � v20. v3 ��������� the_router � ��� ������������ uplink'��. v20 ����������
������� ���� the_router (radius client TheRouter'� � �.�.) � ������� ������ linux, �������
�������� �� ��� �� ����� h4, �� ���������� ���� ������� �����. 

# 4. ������ TheRouter, �������� L2 ��������� � �������������.

## 4.1. ������ TheRouter

the_router --proc-type=primary -c 0xF --lcores='0@0,1@1,2@2,3@3' \
 --syslog='daemon' -n2 -w 0000:60:00.0 -w 0000:60:00.1  --vdev 'eth_bond0,mode=4,slave=0000:60:00.0,slave=0000:60:00.1,xmit_policy=l23' \
 -- -c /etc/router_bras_dhcp_relay_lag.conf

## 4.2. ���������������� ���� TheRouter

���� �������� ����� ����������������� ����� TheRouter.
��������� ������ �������� �� ����� ����� �������� � ����������� �������� ����� howto.

/etc/router_bras_dhcp_relay_lag.conf

	startup {
	  sysctl set mbuf 8192
	  sysctl set log_level 7
	
	  # LAG (slave ports 0,1)
	  port 0 mtu 1500 tpid 0x8100 state enabled flags dynamic_vif bond_slaves 1,2
	
	  rx_queue port 0 queue 0 lcore 1
	  rx_queue port 0 queue 1 lcore 2
	  rx_queue port 0 queue 2 lcore 3
	
	  sysctl set global_packet_counters 1
	  sysctl set arp_cache_timeout 300
	  sysctl set arp_cache_size 65536
	  sysctl set dynamic_vif_ttl 600
	
	  sysctl set dhcp_relay_enabled 1
	}
	
	runtime {
	  # blackhole multicast addresses
	  ip route add 224.0.0.0/4 unreachable
	
	  radius_client add src ip 192.168.20.1
	
	
	  ip route add 10.10.0.0/24 unreachable
	
	  # link with local linux host
	  vif add name v20 port 0 type dot1q cvid 20
	  ip addr add 192.168.20.1/24 dev v20
	
	  # home network link (vlan3)
	  vif add name v3 port 0 type dot1q cvid 3 flags npf_on, kni
	  ip addr add 192.168.1.112/24 dev v3
	  #ip route add 0.0.0.0/0 via 192.168.1.3 src 192.168.1.112
	
	  # cisco L3 connected
	  vif add name v21 port 0 type dot1q cvid 21 flags kni,l3_subs
	  ip addr add 192.168.21.1/24 dev v21
	
	  # L2 connected (zyxel 5Ghz WiFi)
	  vif add name v5 port 0 type dot1q cvid 5 flags kni,l2_subs
	  ip addr add 192.168.5.1/24 dev v5
	
	
	  ## static arp records
	  # radius server
	  arp add 192.168.20.2 90:e2:ba:4b:b3:17 dev v20 static
	
	  dhcp_relay 192.168.20.2
	
	  radius_client add server 192.168.20.2
	  radius_client set secret "secret"
	
	
	  # PBR
	  ip route table add rt_bl
	
	  u32set create ips1 size 4096 bucket_size 16
	  u32set create l2s1 size 4096 bucket_size 16
	  subsc u32set init ips1 l2s1
	
	  ip pbr rule add prio 10 u32set ips1 type "ip" table rt_bl
	  ip pbr rule add prio 20 u32set l2s1 type "l2" table rt_bl
	
	
	  # NPF
	  # npf load "/etc/npf.conf.accept_all_2"
	
	  npf load "/etc/npf.conf.bras_dhcp_relay"
	}

## 4.3. �������� ���������

### 4.3.1. ��������, ��� ������� ��� ��������� � ���������������� ����� ����������

	h4 src # rcli sh vif
	name    port    vid     mac                     type    flags   idx     ingress_car     egress_car
	v20     0       0.20    00:1B:21:3C:69:44       dot1q           10      -       -
	v5      0       0.5     00:1B:21:3C:69:44       dot1q   kni,l2 subs     13      -       -
	v3      0       0.3     00:1B:21:3C:69:44       dot1q   kni,NPF 11      -       -

### 4.3.2. ARP

	h4 src # rcli sh arp
	port    vid     ip      mac     type    state
	0       0.20    192.168.20.2    90:E2:BA:4B:B3:17       static  ok
	0       0.5     192.168.5.124   A8:5B:78:09:0C:E1       dynamic ok
	0       0.3     192.168.1.3     D4:CA:6D:7C:D0:DC       dynamic ok

### 4.3.3. ICMP

	h4 src # rcli ping -c3 192.168.1.3
	Ping 192.168.1.3 56(84) bytes of data.
	reply 56 bytes icmp_seq=1 time=0.283 ms
	reply 56 bytes icmp_seq=2 time=0.279 ms
	reply 56 bytes icmp_seq=3 time=0.278 ms
	---
	Ping 192.168.1.3 statistics:
	sent: 3, recv: 3 (100%), lost: 0 (0%)
	round-trip min/avg/max = 0.278/0.280/0.283

	h4 src # rcli ping -c3 192.168.20.2
	Ping 192.168.20.2 56(84) bytes of data.
	reply 56 bytes icmp_seq=1 time=0.501 ms
	reply 56 bytes icmp_seq=2 time=0.557 ms
	reply 56 bytes icmp_seq=3 time=0.279 ms
	---
	Ping 192.168.20.2 statistics:
	sent: 3, recv: 3 (100%), lost: 0 (0%)
	round-trip min/avg/max = 0.279/0.445/0.557

## 4.4. ������ KNI ����������� � ���������� � Quagga

��������� �������� �������� ���������� c Quagga �������
�� �������� 
<a href="https://github.com/alexk99/the_router/blob/master/quagga_bgp.md#dynamic-routing-integration-with-quagga-routing-suite">
Dynamic routing. Integration with Quagga routing suite
</a>

### 4.4.1. KNI ����������

�NI ���������� ��������� ��� ������� vif ���������� TheRouter,
����� ������� �� ��������������� � ������� ����������������� � ������� ������-���� ���������
������������ �������������. � ����� ������� TheRouter ������������� bgp peer � uplink ��������,
����� �������� default �������.

����� ������� TheRouter ���������� ������� KNI ���������� � ������ �� MAC ������.
��� ����� KNI ���������� ������ ���� ����� MAC ������, ��������������� ��� VIF ���������� TheRouter.

� ����� ������� ��������� ���� KNI �������� c ������ rkni_v3 ��� v3 ���������� TheRouter.
MAC ����� VIF ���������� v3 ����� ������� � ������ ������� 'rcli sh vif', ����������� ����.

	#!/bin/bash
	ip link set up rkni_v3
	ip link set dev rkni_v3 address 00:1B:21:3C:69:44

### 4.4.2. ������ quagga

������� ���������� ��������� � ��������� zebra.

���������������� ���� /etc/quagga/zebra.conf

	hostname h4
	
	! Set both of these passwords
	password xxx
	enable password xxx
	
	# rt1
	table 250
	
	! Turn off welcome messages
	no banner motd
	log file /var/log/quagga/zebra.log

������������ ������� ������ � ���� ����� - ��� ��������� table 250, ������� �������
zebr� �������������� ��� ���������� �� ������� ������������� �������� �� � ��������
������� ������������� linux, � � ������� c id 250. Id ������� ������ ���� �������
�������� � ����� /etc/iproute2/rt_tables

	250     rt1

��� ����������, ����� ���������� �������� �� ������ ������ linux, �.�. ��� �������������
������������� ��� TheRouter � ����� �������� ��� ����� "zebra FIB push interface" ���������.

������ zebra

	/etc/init.d/zebra start

�������� zebra, ����� ��������� bgpd.

���������������� ���� /etc/quagga/bgpd.conf

	h4# cat /etc/quagga/bgpd.conf
	!
	! Zebra configuration saved from vty
	!   2017/05/09 15:50:41
	!
	hostname h4
	password xxx
	log syslog
	!
	router bgp 64512
	 bgp router-id 192.168.1.112
	 network 10.111.0.0/29
	 neighbor 192.168.1.3 remote-as 64513
	 exit
	!
	line vty
	!

����� ������ ������������ ��� c uplink ��������������� 192.168.1.3
� ����� ���� 10.111.0.0/29, ������� ����� ������������ ��� NAT ��������������
ip ������� ����������� � ������� ������ ���������������� � ��������.

����������: �.�. � �������� ���� ��� �������������� ����� �������,
������� � �������� ������� ������������ ����� ���� 10.111.0.0/29, � �����������
�������������� NAT �������������� �� Uplink ������� � ����������� ����� ip �����, 
����������� �� ���. 

������ bgpd

	/etc/init.d/bgpd start
	
��������, ��� default (0.0.0.0/0) ������� ������� ������� � ������������� �� linux ������� � � the_router.

Linux ������� rt1

	h4 src # ip route ls table rt1
	default via 192.168.1.3 dev rkni_v3  proto zebra  metric 20

�������� ������� ������������� TheRouter:

	h4 src # rcli sh ip route
	224.0.0.0/4 is unreachable
	192.168.21.0/24 C dev v21 src 192.168.21.1
	192.168.5.0/24 C dev v5 src 192.168.5.1
	192.168.20.0/24 C dev v20 src 192.168.20.1
	192.168.1.0/24 C dev v3 src 192.168.1.112
	10.10.0.0/24 is unreachable
	0.0.0.0/0 via 192.168.1.3 dev v3 src 192.168.1.112

# 5. ��������� Radius.

## 5.1. ��������� TheRouter ������� radius

������� ���� ��������� (� ������� ����������) ����� Radius �������,
ip ����� ��������� radius ��������, ����������� TheRouter, 
� ����� ��� ������� � ������� ������.

	radius_client add server 192.168.20.2
	radius_client add src ip 192.168.20.1
	radius_client set secret "secret"

Ip ����� "src ip" ������ ���� �������� �� ��������� TheRouter,
����� ������� ������� radius ������. � ����� ������ ��� ��������� v20,
����������� TheRouter c linux ������ ����� h4. 

	# link with local linux host
	vif add name v20 port 0 type dot1q cvid 20
	ip addr add 192.168.20.1/24 dev v20

Ip ����� 192.168.20.1 �������� �� linux ������� ����� 20.

	9: vlan20@eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
	    link/ether 90:e2:ba:4b:b3:17 brd ff:ff:ff:ff:ff:ff
	    inet 192.168.20.2/24 brd 192.168.20.255 scope global vlan20
	       valid_lft forever preferred_lft forever

## 5.2. ��������� Radius ������� �� ������� Linux

� �������� radius ������� ������������ ����� FreeRadius.
� �� ���� ��������� ��������� ��� ���������, �.�. � FreeRadius ������� ���� ���� ������������ � 
� �������� ���� ��������� �������� ��� �������������.

������� ���� ����� ��������� sql �������, �������������� ������������� radius ��������� TheRouter,
� c����� VSA TheRouter'� (dictionary).

/etc/raddb/sql/mysql/dialup.conf

	## router_bras_dhcp_relay.conf
	## pbr.
	authorize_reply_query = "SELECT 1, '%{SQL-User-Name}', 'therouter_ingress_cir', '200', '=' \
	UNION SELECT 2, '%{SQL-User-Name}', 'therouter_engress_cir', '200', '+=' \
	UNION SELECT 3, '%{SQL-User-Name}', 'therouter_ipv4_addr', GetIpoeUserService(%{request:therouter_port_id}, '%{request:therouter_outer_vid}', '%{request:therouter_inner_vid}'), '+=' \
	UNION SELECT 4, '%{SQL-User-Name}', 'therouter_ipv4_mask', '24', '+=' \
	UNION SELECT 5, '%{SQL-User-Name}', 'therouter_ip_unnumbered', '1', '+='"

���� sql ������ ����� ����������� FreeRadius'�� ��� ������������ ������ �� 
radius ������ TheRouter'� �� ����������� ����������, ������������� ����� ��������� ����.
Mysql �������� ��������� GetIpoeUserService ���������� ip ����� ���������� �� ������ ����� � id �����,
����� ������� ��������� ��������� � TheRouter. ���������� radius ������ ����� ������������ TheRouter'�� ���
��������� ������������� ������� � ������� ���������� �� �������� ip unnumbered.

�������� �������� ip unnumbered ��� ������������ "Vlan per subscriber" ����������� � ��
��������� � ������� radius ������� � ������� <a href="https://github.com/alexk99/the_router/blob/master/bras/subsriber_management.md#vlan-per-subscriber">Vlan per subscriber</a>

### 5.2.2. FreeRadius dictionary

�������� � /etc/raddb/dictionary ��������� �������

	VENDOR       TheRouter     12345
	BEGIN-VENDOR TheRouter
	    ATTRIBUTE therouter_ingress_cir 1 integer
	    ATTRIBUTE therouter_engress_cir 2 integer
	    ATTRIBUTE therouter_ipv4_addr 3 integer
	    ATTRIBUTE therouter_ipv4_mask 4 integer
	    ATTRIBUTE therouter_outer_vid 5 integer
	    ATTRIBUTE therouter_inner_vid 6 integer
	    ATTRIBUTE therouter_ip_unnumbered 7 integer
	    ATTRIBUTE therouter_port_id 8 integer
	    ATTRIBUTE therouter_ipv4_gw 9 integer
	    ATTRIBUTE therouter_pbr 10 integer
	END-VENDOR   TheRouter

# 6. ��������� � �������� ������/����������� ����������� (subscriber)

## 6.1. ������������ "Vlan per subscriber" ���������� �����������

��������� ���������� � "vlan per subscriber" ����������� ������� � 
<a href="https://github.com/alexk99/the_router/blob/master/bras/subsriber_management.md#vlan-per-subscriber"
����������� �������</a>, � ���� ���� � ����� �������� ������ �� ���������� ������� bras'�, ���������������� �����.

������������ (����������� �� ����� ������ TheRouter'�, � �� � ���������������� �����)
���������� ��������� �� �����. � ����� ������� ���� ����� ���� ���� (port 0), �������
����� TheRouter ���� � ������������ ��������� ����� ���������� � ����� ������ ���� dynamic_vif.

	port 0 mtu 1500 tpid 0x8100 state enabled flags dynamic_vif bond_slaves 1,2

�������� �� ����� �����, �� ������������� �� ������ �� �����������, TheRouter �������
����� ������������ ���������, ��������, ��� subscriber'� � ����� 0.50 (svid.cvid), ��� ������� ��� ��������
�����������. 

����������� - ��� �������� ������� � ��������� ������ c ������� Radius ���������.
������ ����������� ����� �������� ��� ������ � ����� subscriber'�, � ����� ��� ������, ����������� 
��� ��������� IP ���������� � ������������� (� ������ ip unnumbered).

### 6.1.1. ����� ���������� � ������������ �����������

	h4 ~ # rcli sh subsc
	vlan    subsc ip        mode    port    ingress_car     egress_car      rx_pkts tx_pkts
		...
	0.130                   1       0       200 mbit/s      200 mbit/s      0       0
		...

������ ������������ ����������� ��������� ������ �� ������� L2/L3 ������, ������� ��������� ����
�������� �������, �������� subsc ip, � ������ ������������ �����������.

### 6.1.2. �������� ������������� ��� ������������ �����������

�.�. radius ����� ����������� �������� ������������� ���������� �������� VSA ������� "therouter_ip_unnumbered",
�� TheRouter ������� ������� ��� IP ������ ����������, ������������� ����� ���� ���������.
��������� �������� ����� �������� ��������� � ������� <a href="https://github.com/alexk99/the_router/blob/master/bras/subsriber_management.md#�����-��-������-�����������">
����� �� ������ �����������</a>

��������, ��� ������� ������

	h4 ~ # rcli sh ip route
	...
	10.10.0.11/32 C dev dvif0.130 src 10.10.0.1
	...

����� 10.10.0.11 - ��� ip �����, ��������� � radius ������. dvif0.130 - ��� �������� ������������� ����������,
��� 0.130 - ��� svid.cvid ����� ����.

## 6.2. L2/L3 connected ����������

### 6.2.1. L2/L3 ����������
��������� ���������� � L2 �����������/������� ������� � �������
<a href="https://github.com/alexk99/the_router/blob/master/bras/subsriber_management.md#l2-connected-subscribers"
L2 connected subscribers</a>, � L3 ����������� � ������� 
<a href="https://github.com/alexk99/the_router/blob/master/bras/subsriber_management.md#l3-connected-subscribers"
L3 connected subscribers</a>

� ���� ���� ����� ����������� ���������� ������� ���������������� ����� BRAS'�.

� ������� �� vlan per subscriber ������ � ������������� ������������, ������� ����������
�������� �������� ����������� �������������� �� ������ ���������� ����������� IP, ��� L2/L3
����������� �� ��������� ����� ����������, � ��������� ��� ����������� ������, ������� �����������
���� ��� ����� ������������� vif ���������� TheRouter'�. ������� �� �������� ���������� �� vif ����������,
� �������� ������ ���� l2_subs ��� l3_subs.

� ����� ������� L2 ������ ��������� �� ���������� v5, 

	vif add name v5 port 0 type dot1q cvid 5 flags kni,l2_subs

� L3 ������ ��������� �� ���������� v21.

	vif add name v21 port 0 type dot1q cvid 21 flags kni,l3_subs	

������� L2 ������ �� L3 ����������� ������ � ���, ��� L2 ������ ������
������ ip ������, ��� � MAC ����� ��������, ��������������� ������. ���
�������� (��� �� �����������) ��������� (�������� ���������) ����������� MAC ������
�������, ���������� ����� ������, MAC ������, ������������ � ������, � � ������ ���������,
��������� ����������� ��������.

����������� �������� L2/L3 ������ ���������� ���������� ����������� �������� ������������
�����������, �� ����������� ����, ��� ������ - �� �������� ��������� �����������, ������� ��������� IP
��� ��� ��������� �� �����.

� ���������, ������ ����� ������������ ����������� �������, ����������� ����� ��� � PBR ������������� ��
�������.

### 6.2.2. ����� ���������� � L2/L3 �������

	h4 ~ # rcli sh subsc
	vlan    subsc ip        mode    port    ingress_car     egress_car      rx_pkts tx_pkts
	        192.168.5.132   1       0       200 mbit/s      200 mbit/s      0       0
	        192.168.5.133   1       0       200 mbit/s      200 mbit/s      0       0

# 7. ��������� DHCP � DHCP Relay ��� ������ �������� �����������

� TheRouter ����������� ���������������� DHCP Relay.
������������ ��������� �� ������ ������ ��������� - ��� ip ����� DHCP �������,
�������� TheRouter ����� �������������� ��� ���������� dhcp �������.

	dhcp_relay 192.168.20.2

� ����� ������� ������ ip ����� ����� 20 linux ����� ����� h4, ��� �������
dhcpd ������.

## 7.1. ������ /etc/dhcp/dhcpd.conf
	
��� ������ dhcpd ����������������� �����, � ������� ������� ���������
���� subscriber'��. Dhcpd ������ ������� ������� �� ���������� v3 (c��� 192.168.20.0)
� �������������� ���� �������� subscriber'�� �� �� mac �������.

	
	#
	# Global parameters
	#
	default-lease-time 864000; # 10 days
	max-lease-time 864000;
	option domain-name "home";
	option domain-name-servers 192.168.1.3;
	ddns-update-style none;
	ddns-updates off;
	authoritative;
	
	shared-network dd {
	
		subnet 10.10.0.0 netmask 255.255.255.0 {
		    option subnet-mask 255.255.255.0;
		    option routers 10.10.0.1;
		
		    pool {
		        range 10.10.0.50 10.10.0.200;
		        default-lease-time 72000; # 20 hours
		        max-lease-time 72000;
		        allow unknown clients;
		    }
		}
		
		subnet 192.168.20.0 netmask 255.255.255.0 {
		}
	
	}
	
	# 0 - hostname, 1 - host ip, 2 - host mac
	
	host c1 {
	  hardware ethernet F8:32:E4:72:61:1B;
	  fixed-address 10.10.0.12;
	}
	
	host c2 {
	  hardware ethernet 00:88:65:36:39:4c;
	  fixed-address 10.10.0.11;
	}
	
## 7.2. ������������� DHCP �������

����� ������ dhcpd ������� ������� �������� ������ ���������� ������ ���� �������� �������
�� �������, ������� �������������� ������� � dhcpd. � ����� ������ dhcpd ������� ���������
� ������ 10.10.0.1 - ��� IP ����� ���� ������������ �����������, ����������� �� �������� ip
unnumbered �, �������������, ������� ���������� �����. TheRouter ������������� dhcp �������,
���������� �� ����������� ����� ������������ ����������, ���������� � ��� ����� 10.10.0.1,
����� dhcpd ������ ���� ���� ������� ������� ���������� ������.

��� ������������ ������ �� ��� ����� dhcpd

	Aug 23 21:07:24 h4 dhcpd[2888]: DHCPREQUEST for 10.10.0.11 (192.168.20.2) from 00:88:65:36:39:4c via 10.10.0.1
	Aug 23 21:07:24 h4 dhcpd[2888]: DHCPACK on 10.10.0.11 to 00:88:65:36:39:4c via 10.10.0.1

� ����� ������ ������� �� ������ ������� ������������ ������: "via 10.10.0.1".

������� �� linux ����� h4 ������ ���� ������ ������� �� ������ 10.10.0.1.
���� ������� ������ ����� � TheRouter, � ����� � ��� ��������� ����� ���� 20,
������� ������� �������� ��� ���:

	ip route add 10.10.0.1 via 192.168.20.1

# 8. ��������� ��������������� ��������������� ����������� �� ���������� ����

����������� � ������� ��������� PBR (Policy based routing) � ������� � �������
<a href="https://github.com/alexk99/the_router/blob/master/bras/subsriber_management.md#����������-��������-���������������-�����������">���������� �������� ��������������� �����������</a>

# 9. ��������� NAT

��������� NAT ����������� � ��������� �����, � ����������� ��������� ���������� 

	npf load "/etc/npf.conf.bras_dhcp_relay"

���� /etc/npf.conf.bras_dhcp_relay

	map v3 netmap 10.111.0.0/29
	
	#alg "icmp"
	
	group default {
	  pass stateful final on v3 all
	}
	
��������� "map v3 netmap 10.111.0.0/29" ����������� NAT �������������� IP ������� ��������� ���� ������� (��������,���������)
���������� v3 � ������ �� ������� 10.111.0.0/29. �������� NAT ���������� ��������� �������� ����� ��������� � ����� �������
�� �������� ������� 10.111.0.0/29 ��������� �������:
	����� ����� ������������ ��� ����� 10.111.0.0 or (32 - 29 == 3 � c������ 2 == 8) ������� 8 ��� ������ ��������� ������.

�.�. ���� � ��� �� ���������� ����� (��������) ����� ������ ����������������� � ���� � ��� �� ������� �����.
