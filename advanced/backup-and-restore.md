# 備份與還原

建置完 Puppet 之後最重要的議題就是 maintian，其中一項最基本也最重要的就是災難復原，雖然現在都會架構在
 High Availability 的情況下，但還是要有一個備份機制以防萬一。

- CA 憑證

    - /etc/puppetlabs/puppet/ssl

> 這是 Puppet 最重要的項目，因為是 CA Trust 的方式，你的 CA 憑證如果沒有憑證所有的 node 都必須重新 trust。

- Configuration
    - /etc/puppetlabs
    - /opt/puppetlabs

> 這兩個位置是 puppet 主要安裝的目錄，懶人備份法可以直接 backup 這兩個目錄就好了。

- PuppetDB

> PuppetDB 是儲存所有 Node report 記錄，如果你的環境是需要稽核或反查某些事件的話，Puppetdb 就很重要。

- Hiera-eyaml Key

> 和憑證列為重要項目之一，所有的 hiera-eyaml 加密字串都必須依靠這把 hiera-eyaml 產生的 Key 來做加解密，如果不見了，你就要重新加密這些字串 (前提是你知道這些字串原本的數值)。

上述是使用 Puppet 的重點備份項目，如果你需要重建只要把上述的檔案放回原本相關位置，重啟 Puppet Server 就可以完成還原。
