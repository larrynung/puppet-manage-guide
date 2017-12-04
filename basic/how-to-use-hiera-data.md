# 怎麼使用 Hiera data

## 什麼是 Hiera

Hiera 是 Puppet 內建的數據查詢系統，如果要管理好 Puppet，Hiera 絕對是不可或缺的項目，Hiera 支援 YAML 和 JSON 的格式編輯。

## 為什麼使用 Hiera

Hiera 擁有多層架構的特性，你可以為尚未定義的 data 設定一個 default hiera data，後續再給另一個 hiera data。

舉例：

hiera.yaml 是 hiera 預設讀取的檔案。

```yaml
# hiera.yaml
---
version: 5

defaults:
  datadir: data
  data_hash: yaml_data

hierarchy:
  - name: "Per-node data"
    path: "nodes/%{::trusted.certname}.yaml"
    
  - name: "Common data"
    path: "common.yaml"
```

定義 command.yaml 的 ntp_server

```yaml
# data/command.yaml
profile::base::ntp_server: 'time.stdtime.gov.tw'
```

但 web.puppet.com 必須修改 ntp_server 的值。

```yaml
# data/nodes/web.puppet.com.yaml
profile::base::ntp_server: 'time.google.com'
```

按照上面的範例，common.yaml 和 web.puppet.com.yaml 都擁有 ntp_server，但是依照 hiera 的讀取順序，在 web.puppet.com.yaml 就取到了，所以 common.yaml 的 ntp_server 就被丟掉了。


這是因為在 Hiera 是按照 facts 讀到的 **第一個** 數據，讀取的順序：

  - data/nodes/web.puppet.com.yaml
  - data/location/portland/ops.yaml
  - data/groups/ops.yaml
  - data/os/RedHat.yaml
  - data/common.yaml

除了 facts 的讀取順序以外，在 Hiera 5 之後還支援了 three layers 的架構：

  - Global ($confdir/hiera.yaml)
  - Environment ($ENVIRONMENT/hiera.yaml)
  - Module ($MODULE/hiera.yaml) 

透過這麼多層的 data 設計，就能讓你的 hiera 符合各種環境。


## 在 Hiera 中處理變數

當你常用 Hiera 在比較複雜的環境，你可能會遇到在不同的變數需要相同的值，或是比較奇怪的符號，Hiera 就支援以下這幾項讓你用來處理這些情境。

### 用 lookup 來取得其他的 hiera data

lookup 適合讓你用來取得別的 data 後塞入另一個 data 的內容中，例如：

```yaml
---
php_version: '7.0'
php_package: "php%{lookup('php_version')}"
```
> php_package 的值等於 'php7.0'。

### 用 alias 直接對應相同的 hiera data

或是直接把變數 alias 到另一個 data

```yaml
---
original:
  - 'one'
  - 'two'
aliased: "%{alias('original')}"
```

> aliased 的值等於 original

### literal 用來處理有 ‘%’ 的 hiera data

由於 hiera 的變數是 '%'，但如果你想儲存有 '%' 的值的話預設會被當成變數處理，要跳脫 '%' 就會用上 literal

```yaml
---
server_name: "%{literal('%')}{SERVER_NAME}"
```

> server_name 的值等於 ${SERVER_NAME}。

### 用來解釋 facts 的 scope

個人覺得 scope 沒什麼用途，主要是用來解釋 facts 的變數，實際上對 data 沒有特別改變

```yaml
smtpserver: "mail.%{facts.domain}"
smtpserver: "mail.%{scope('facts.domain')}"
```
> 以上兩者的值相同。





















