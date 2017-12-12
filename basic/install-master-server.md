# Day 5 - Master-Agent 架構 - Master 安裝

來到 Day-5 了阿 ～～～～～～ (無病呻吟)，感覺好辛苦。

今天要講 Puppet 中的 Master / Agent 這個架構了，然後直接就要帶大家來安裝一下 Puppet Server !!

要安裝之前首先先介紹一下：

## Puppet Master

Puppet Master 是由 Ruby 開發的 Application，Puppet Master 主要是用來跑整個 Puppet Code 編譯的核心，單單 Puppet Master 是無法運行的，還必須搭配 Server 當平台。

目前 Puppet Master 可以在兩種平台上運行：

1. Puppet Server (Java)
2. Rack Server (Ruby)

## Puppet Server 

Puppet Server 是由 Java 開發，運行在 JVM 的 Application 並且提供和 Puppet Master 相同功能的服務，採用 JRuby 可以直接運行 Puppet Code，所以在 Puppet Server 本身就已經內建 Puppet Master。

## Rack Server

Rack Server 是由 Ruby 開發的 Web Service，將 HTTPS 請求重新格式化後轉發給後端的 Puppet Master 再進行處理，簡單說 Rack Server 就是一個中繼 Service。

---

在 `Puppet 3` 之前都是採用 Rack + Puppet Master 的架構運行，可想而知中間的傳遞耗損了多少效能，所以在 `Puppet 4` 之後開始預設將 Server 這塊改用 Puppet Server 在整體效能上提昇了不少 (雖然跑在 Java 還是挺耗資源的)，兩者都還是能用，但是 Puppet 已經停止在 Rack 的支援，目前的開發重心都是以 Puppet Server 為主。

因為 Rack 是準備淘汰的東西，所以今天就會以 Puppet Server 為主軸介紹：

## Supported Platforms

目前 Puppet Server 支援在任何 POSIX 平台下安裝，以下平台可以直接用套件管理工具安裝 (i.e. yum, apt)

- RedHat
- Debian
- Fedora
- Ubuntu

除此之外最低必須使用 JDK 1.7 以上才跑的動。

## LAB 環境

在一開始先定義好 Puppet Master 和 Agent 運行的環境：

**Master**

- Operating system: Ubuntu 16.04
- IP Address: 192.168.10.10
- Domain : master.puppet.com

**Agent**

- Operating system: Ubuntu 16.04
- IP Address: 192.168.10.11
- Domain : agent.puppet.com

## 安裝 Puppet Server

1. Puppet 針對所有的主機皆須定義為 Domain。

    在 Puppet 官方有提到：
    
    > Name resolution: Every node must have a unique hostname. Forward and reverse DNS must both be configured correctly. (Instructions for configuring DNS are beyond the scope of this guide. If your site lacks DNS, you must write an /etc/hosts file on each node.)
        
    > Note: The default Puppet master hostname is puppet. Your agent nodes can be ready sooner if this hostname resolves to your Puppet master.
        
    如果你還沒有定義好 Puppet Master / Agent 的 Domain，可以先設定在 hosts。
        
    ```shell
    $ cat /etc/hosts
    192.168.10.10 master.puppet.com
    192.168.10.11 agent.puppet.com
    ```

1. Puppet Master 必須準確校時，Master 和 Agent 誤差超過 5 分鐘則 Puppet 交握失敗。

    ```shell
    $ sudo ntpdate time.stdtime.gov.tw
    $ sudo timedatectl set-timezone Asia/Taipei
    ```

1. 安裝 Puppet Master

    從官方 [repository][puppet-platform] 取得 Puppet package。

    ```shell
    $ wget https://apt.puppetlabs.com/puppet5-release-xenial.deb
    $ sudo dpkg -i puppet5-release-xenial.deb
    $ sudo apt-get update
    $ sudo apt-get install puppetserver
    ```
  
1. Puppet Master Memory 調整，預設為 2G。

    ```shell
    $ sudo vim /etc/default/puppetserver
    JAVA_ARGS="-Xms2g -Xmx2g"
    ```

1. 修改 Puppet 的主要設定檔 [puppet.conf][puppet-conf]。

    ```shell
    $ sudo vim /etc/puppetlabs/puppet/puppet.conf
    
    [main]
    certname = master.puppet.com
    ```
    
    certname 是用來生成這台 node 憑證使用，Puppet 之間的溝通是使用 SSL 交握。


1. 嘗試啟動 Puppet master

    ```shell
    $ sudo systemctl start puppetserver
    $ sudo systemctl enable puppetserver
    ```

1. 應該要 listen 8140 port，可以看到是由 Java 啟動，因為 Puppet Server 是用 Java 開發。

    ```shell
    $ ss -tunlp | grep 8140
    tcp  LISTEN  0  50  :::8140  :::*  users  (("java",pid=27866,fd=32))
    ```

1. 初始化 Puppet server 的 ca 憑證，如果沒有執行 Agent 會無法透過 CA 來產生 Agent 的 certificate。

    ```shell
    $ sudo puppet master --verbose --no-daemonize
    ```

1. 如果你有開啟 Firewall 記得 allow 8140。

    ```shell
    $ sudo ufw allow 8140
    ```

[puppet-platform]: https://docs.puppet.com/puppet/5.3/puppet_platform.html
[puppet-conf]: https://docs.puppet.com/puppet/5.3/configuration.html



