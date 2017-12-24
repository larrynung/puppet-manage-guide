# Puppet 內建的檔案系統

這邊講的檔案系統是指拿來 deploy 到 `Node` 的檔案方式，各種方式都有優缺點，在以下會開始詳述該怎麼用這些資源。

講到檔案系統，其實就是在講 `file` 這個 Resource，提供兩種類型 `source` 和 `content`。

## file 的 source

顧名思義就是這個檔案的來源，那麼可以簡單分為三種 `puppet:`、`file` 和 `http:`。

### `puppet:` 從 Puppet Server 上抓 source

`puppet:` 是很常用用來 deploy file 的方式，`puppet:` 可以幫你把 Puppet Server 上的檔案丟到 `Node` 上面，是拿來做大量佈署很常見的一種方式。

- Example

把 `ntp.conf` 放在 `<MODULES DIRECTORY>/ntp/files/ntp.conf` 下來讓 file 取得。

```
file { '/etc/ntp.conf':
  source => [
    "puppet:///modules/ntp/ntp.conf"
  ]
}
```

或是可以用多個來源 `puppet:` 合併成一個檔案。

```
file { '/etc/nfs.conf':
  source => [
    "puppet:///modules/nfs/conf.${host}",
    "puppet:///modules/nfs/conf.${operatingsystem}",
    'puppet:///modules/nfs/conf'
  ]
}
```

優點：最簡單佈署檔案的方式，無腦。
缺點：設定檔必須放在 module 裡面，會利用多個檔案來合併成一個 file。

### `file:` 從 Node 上抓 source

這算是一個 hard code 的方式，`file:` 是從 Node 上既有的檔案上取得，這種情況通常比較少。

- Example

在 `Node` 的 `/tmp/ntp-example.conf` 有既有的檔案提供存取。

```
file { '/etc/ntp.conf':
  source => 'file:/tmp/ntp-example.conf',
}
```

優點：沒有優點。
缺點：要確保 Node 上已有檔案，否則會 deploy 失敗。

### `http:` 從 HTTP 上抓 source

這是一個較少人用，但是超好用的 source 取得方式，讓你從 http:// 協議來抓檔案。

- Example

```
file { '/etc/ntp.conf':
  source => 'http://website/config/ntp.conf',
}
```

優點：不適合放在 Puppet 上的檔案可以透過 Web Server 的形式來提供，像是 *.so 這類型的東西。
缺點：要另外維護 Web Server。

## file 的 content

`content` 是用來決定檔案內容，通常會有三種方式。

### 直接 input 的 content

直接塞檔案內容。

- Example

```
file { '/etc/motd':
  content => "The site $fqdn is managed by Puppet.",
}
```

優點：很直覺，適合簡短的內容。
缺點：不適合有比較多內容的檔案。

### file 從 Puppet Server 上抓 content

和 `source` 一樣直接指定路徑，只是這邊的路徑是以 Puppet Server 為主。

- Example

在 Puppet Server 的 /tmp/motd-example 有既有的檔案提供存取。

```
file { '/etc/motd':
  content => file("/tmp/motd-example"),
}
```

優點：適合不放在 Puppet Server repository 底下的檔案。
缺點：版控不好做，可能要從 CD 把這類型的檔案 deploy 到 Puppet Server。

### templates 從 Puppet Server 上的範本取得

Puppet 要用的好 templates 不能少，利用模版的方式讓所有設定可以參數化。

- Example

放在 Puppet Server 的 `<MODULES DIRECTORY>/motd/templates/motd.erb` 這個範本來參考。

```
file { '/etc/motd':
  content => templates("motd/motd.erb"),
}
```

優點：要寫一個彈性的設定檔的話非 templates 莫屬。
缺點：要多學習 erb 或 epp 模版語言。

`source` 和 `content` 只能擇一使用，又以 `content` 搭配 templates 能夠建立非常彈性的 config。


