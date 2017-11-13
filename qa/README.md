# 疑難雜症 Q&A

除了一般正常的用法以外，偶爾還是會遇到一些奇怪的錯誤而卡關，有可能是你腦殘打結，或是手殘打錯，還是官方的蟲，通通都在這邊。

## 憑證篇

- **我的 Puppet Server 平時都運作很正常，但是偶爾會有零星的 Puppet agent 出現錯誤「SSL_connect returned=1 errno=0 state=unknown state: sslv3 alert certificate unknown」，這是什麼狀況？**

請檢查你的 Puppet Server 中是否有 sign not trust 的狀況。


## 效能篇

- **我的 Puppet Agent 變慢了我該怎麼辦？**

你的 Puppet Server 已經接近繁忙並且無法處理 Puppet agent 的 request。

Puppet Server 的 `max-active-instances` 決定可同時連接的 Puppet agent (JRuby)數量，你可以透過增加 Puppet Server 或是加大 CPU core 來調整 `max-active-instances` 的數量。

Puppet Server 在 5.1 之後提供 `retry requests` 的機制，即使 Puppet Server 當下繁忙也可以嘗試 retry 不用等待下次 sync (Puppet agent > 5.3.0)

## 已知 bug 篇

- **Puppet Agent 在同步時出現訊息「Unable to set ownership to puppet:puppet for log file: /var/log/puppetlabs/puppet/puppet.log」是怎麼回事？**

這是已知 Bug [PUP-7331][puppet-bug-pup-7331]，因為新版的 Agent 不再建立 puppet 使用者，但是在 puppet agent 在使用 `--logdest` 時仍然會將 log file 改變權限為 puppet:puppet 造成，目前已知在 5.x 版本修正。

[puppet-bug-pup-7331]: https://tickets.puppetlabs.com/browse/PUP-7331