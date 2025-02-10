# VPN構築（手動）

- os
  - ubuntu22.04-x86_64
- root pass
  - 0_Ayano

## 詳細

1. SSH接続

   ```
   ssh root@160.251.167.223 -p 22
   ```

2. 作業用ユーザーの追加とroot権限の付与

   ```
   adduser tkoya
   passwd tkoya
   # set pass : kd0_Ayano
   
   gpasswd -a tkoya sudo
   exit
   ```

3. アカウントを指定してSSH接続

   ```
   ssh tkoya@160.251.167.223 -p 22
   ```

4. wheelグループ以外のユーザーがsudoコマンドを使えないようする

   ```
   sudo vim /etc/pam.d/su
   # auth required pam_wheel.so deny group=nosu #を外す
   ```

5. セキュリティ設定

   ```
   sudo vim /etc/ssh/sshd_config
   # PasswordAuthentication no
   # PermitRootLogin no
   
   sudo systemctl restart sshd.service
   ```

6. ubuntuの更新

   ```
   sudo apt update
   sudo apt upgrade
   ```

7. 使用するパッケージをインストール

   ```
   sudo apt install -y gcc make dnsmasq
   sudo apt install net-tools
   ```

8. NIC名を確認　＝＞eth0

   ```
   ifconfig
   ```

9. VPSのIPアドレスを調べる　＝＞160.251.167.223

   ```
   curl globalip.me
   ```

10. softetherのインストール

    ```
    wget https://jp.softether-download.com/files/softether/v4.42-9798-rtm-2023.06.30-tree/Linux/SoftEther_VPN_Server/64bit_-_Intel_x64_or_AMD64/softether-vpnserver-v4.42-9798-rtm-2023.06.30-linux-x64-64bit.tar.gz
    tar xvf softether-vpnserver*
    cd vpnserver
    make
    chmod 600 *
    chmod 700 vpncmd
    chmod 700 vpnserver
    cd
    sudo \cp -r -f ./vpnserver/ /usr/local/
    sudo \rm -rf ./vpnserver
    ls -lra /usr/local/vpnserver
    rm -rf softether-vpnserver*

11. softetherの設定

    1. softetherの起動

       ```
       sudo /usr/local/vpnserver/vpnserver start
       sudo /usr/local/vpnserver/vpncmd /server localhost

    2. パスワードの設定

       ```
       ServerPasswordSet kd0_Ayano

    3. vpn hubの作成

       ```
       HubCreate vpnhub1 /PASSWORD:kd0_Ayano
       HubDelete DEFAULT
       BridgeCreate vpnhub1 /DEVICE:soft /TAP:yes

    4. vpn hub内にアカウントを作成

       ```
       Hub vpnhub1
       GroupCreate Admin /REALNAME:none /NOTE:none
       UserCreate tkoya /GROUP:Admin /NOTE:none /REALNAME:none
       UserPasswordSet tkoya /PASSWORD:kd0_Ayano
       exit

12. softetherを自動起動

    ```
    sudo vim /lib/systemd/system/vpnserver.service
    
    [Unit]
    Description=SoftEther VPN Server
    After=network.target
    
    [Service]
    Type=forking
    ExecStart=/usr/local/vpnserver/vpnserver start
    ExecStop=/usr/local/vpnserver/vpnserver stop
    ExecStartPost=/bin/sleep 05
    ExecStartPost=/bin/bash /root/softether-iptables.sh
    ExecStartPost=/bin/sleep 03
    ExecStartPost=/bin/systemctl start dnsmasq.service
    ExecReload=/bin/sleep 05
    ExecReload=/bin/bash /root/softether-iptables.sh
    ExecReload=/bin/sleep 03
    ExecReload=/bin/systemctl restart dnsmasq.service
    ExecStopPost=/bin/systemctl stop dnsmasq.service
    Restart=always
    
    [Install]
    WantedBy=multi-user.target

13. softetherのiptablesの設定

    ```
    sudo vim /root/softether-iptables.sh
    
    #!/bin/bash
    TAP_ADDR=192.168.30.1
    TAP_INTERFACE=tap_soft
    VPN_SUBNET=192.168.30.0/24
    NET_INTERFACE=eth0
    VPNEXTERNALIP=160.251.167.223
    
    iptables -F && iptables -X
    /sbin/ifconfig $TAP_INTERFACE $TAP_ADDR
    
    iptables -t nat -A POSTROUTING -s $VPN_SUBNET -j SNAT --to-source $VPNEXTERNALIP
    
    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
    
    iptables -A INPUT -s $VPN_SUBNET -m state --state NEW -j ACCEPT
    iptables -A OUTPUT -s $VPN_SUBNET -m state --state NEW -j ACCEPT
    iptables -A FORWARD -s $VPN_SUBNET -m state --state NEW -j ACCEPT

14. /root/softether-iptables.shの権限を変更

    ```
    sudo chmod +x /root/softether-iptables.sh

15. dnsmasqの設定のバックアップを取る

    ```
    sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf-bak

16. dnsmasqの設定

    ```
    sudo /etc/dnsmasq.conf
    
    # In this case it is the Softether bridge
    interface=tap_soft
    
    # Don't ever listen to anything on eth0, you wouldn't want that.
    except-interface=eth0
    
    listen-address=192.168.30.1
    bind-interfaces
    
    # Let's give the connecting clients an internal IP
    dhcp-range=tap_soft,192.168.30.10,192.168.30.200,720h
    
    # Default route and dns
    dhcp-option=tap_soft,3,192.168.30.1
    
    # enable dhcp
    dhcp-authoritative
    
    # enable IPv6 Route Advertisements
    enable-ra
    
    #  have your simple hosts expanded to domain
    expand-hosts
    
    # Let dnsmasq use the dns servers in the order you chose.
    strict-order
    
    # Let's try not giving the same IP to all, right?
    dhcp-no-override
    
    # The following directives prevent dnsmasq from forwarding plain names (without any dots)
    # or addresses in the non-routed address space to the parent nameservers.
    domain-needed
    
    # Never forward addresses in the non-routed address spaces
    bogus-priv
    
    
    # blocks probe-machines attack
    stop-dns-rebind
    rebind-localhost-ok
    
    # Set the maximum number of concurrent DNS queries. The default value is 150. Adjust to your needs.
    dns-forward-max=300
    
    # stops dnsmasq from getting DNS server addresses from /etc/resolv.conf
    # but from below
    no-resolv
    no-poll
    
    # Prevent Windows 7 DHCPDISCOVER floods
    dhcp-option=252,"\n"
    
    # Use this DNS servers for incoming DNS requests
    server=1.1.1.1
    server=8.8.4.4
    
    # Use these IPv6 DNS Servers for lookups/ Google and OpenDNS
    server=2620:0:ccd::2
    server=2001:4860:4860::8888
    server=2001:4860:4860::8844
    
    # Set IPv4 DNS server for client machines # option:6
    dhcp-option=option:dns-server,192.168.30.1,176.103.130.130
    
    # Set IPv6 DNS server for clients
    dhcp-option=option6:dns-server,[2a00:5a60::ad2:0ff],[2a00:5a60::ad1:0ff]
    
    # How many DNS queries should we cache? By defaults this is 150
    # Can go up to 10k.
    cache-size=10000
    
    neg-ttl=80000
    local-ttl=3600
    
    # TTL
    dhcp-option=23,64
    
    # value as a four-byte integer - that's what microsoft wants. See
    dhcp-option=vendor:MSFT,2,1i
    
    dhcp-option=44,192.168.30.1 # set netbios-over-TCP/IP nameserver(s) aka WINS server(s)
    dhcp-option=45,192.168.30.1 # netbios datagram distribution server
    dhcp-option=46,8         # netbios node type
    dhcp-option=47
    
    read-ethers
    
    log-facility=/var/log/dnsmasq.log
    log-async=5
    
    log-dhcp
    quiet-dhcp6
    
    # Gateway
    dhcp-option=3,192.168.30.1

17. NATの設定

    ```
    sudo vim /etc/sysctl.conf
    
    net.core.somaxconn=4096
    net.ipv4.ip_forward=1
    net.ipv4.conf.all.send_redirects = 0
    net.ipv4.conf.all.accept_redirects = 1 
    net.ipv4.conf.all.rp_filter = 1
    net.ipv4.conf.default.send_redirects = 1
    net.ipv4.conf.default.proxy_arp = 0
    
    net.ipv6.conf.all.forwarding=1
    net.ipv6.conf.default.forwarding = 1
    net.ipv6.conf.tap_soft.accept_ra=2
    net.ipv6.conf.all.accept_ra = 1
    net.ipv6.conf.all.accept_source_route=1
    net.ipv6.conf.all.accept_redirects = 1
    net.ipv6.conf.all.proxy_ndp = 1

18. NATの設定を反映

    ```
    sudo sysctl -f

19. システムの再起動

    ```
    systemctl daemon-reload
    ```

20. ファイヤーウォールを設定

    ```
    sudo -s
    ufw enable
    ufw allow 22/tcp
    ufw allow 443/tcp
    ufw allow 80/tcp
    ufw allow 67/udp
    exit
    
21. サーバー自体を再起動

    ```
    reboot

22. ファイアウォールの適用

    ```
    sudo ufw enable

23. VPNとDNSマスクの状態を確認

    ```
    systemctl enable vpnserver
    systemctl enable dnsmasq
    systemctl start vpnserver
    systemctl status vpnserver
    
    systemctl start dnsmasq
    systemctl status dnsmasq
    systemctl restart dnsmasq


```
sudo vim /etc/ufw/sysctl.conf