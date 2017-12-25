# 用 templates 來處理檔案內容

templates 在 Puppet 佔有舉足輕重的地位，從簡單到複雜的設定檔都可以透過 templates 來寫成你想像的 file。

## What's Templates ? 

`templates` 是基於組合語言， 擅長用來管理複雜的設定，你只要提供簡單的輸入就可以輸出一個複雜的設定檔，templates 用於 file resource 的 content。


## Templates 支援的語言

目前 `templates` 支援兩種語言寫法

- [EPP][EPP] (Embedded Puppet) 基於 Puppet 自身的語言，目前只在 Puppet 4 以上的版本支援。
- [ERB][ERB] (Embedded Ruby) 基於 Ruby 語言的寫法，適用所有版本。

兩者看起來 Puppet 想用 EPP 來取代 ERB，寫起來 EPP 和 ERB 的差異不大，但是如果你還不太熟 Puppet，或是比較熟 Ruby 建議可以從 ERB 開始寫起，畢竟網路資料目前還是以 ERB 為主。

## 寫一個 Templates

這篇會以 ERB 做示範怎麼寫 templates，首先先拿 file 來 include template。

```puppet
class ntp::config (
  String $driftfile = '/var/lib/ntp/drift',
  Boolean $tinker = true,
  Array $restrict = [ '127.0.0.1', '::1' ],
){
  file { '/etc/ntp.conf':
    ensure => file,
    content => templates("${module_name}"/ntp.conf.erb),
  }
}
```

把要 input 到 ntp.conf.erb 的參數先定義好 type。

### ERB 用變數寫設定

```erb
driftfile <%= @driftfile %>
```

- Output

```
driftfile /var/lib/ntp/drift
```

### ERB 的 if 用法

```erb
# ntp.conf: Managed by puppet.
<% if @tinker == true -%>
tinker panic 0
<% end -%>
```

- Output

```
# ntp.conf: Managed by puppet.
tinker panic 0
```

### ERB 用 if 判斷 type

```erb
<% if @restrict != [] -%>
<% @restrict.flatten.each do |restrict| -%>
restrict <%= restrict %>
<% end -%>
<% end -%>
```

- Output

```
restrict 127.0.0.1
restrict ::1
```

先用簡單的範例來寫 ERB，因為有了判斷式的幫助所以設定檔就能按照不同的 input 而變化。


[EPP]: https://puppet.com/docs/puppet/5.3/lang_template_epp.html
[ERB]: https://puppet.com/docs/puppet/5.3/lang_template_erb.html
