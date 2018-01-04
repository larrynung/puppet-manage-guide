# Role and Profile pattern 的管理方式

一開始用 `Puppet` 的時候就是把每個 `node` 的 `manifests` 像流水帳一樣寫完，但當 `node` 數量一多就會發現一直在寫重複的 `Puppet code`，當要大量修改所有 `node` 的某項參數或設定的時候，你就會開始吐血 ...

那麼先來看看針對不同情境下要怎麼處理。

## 少於 20 台 Node 的情況

在量少的狀況下會直接把 `resource` 寫在 `node`，像這樣

```puppet
node default {
  service { 'ntp':
    ensure => present
  }
  ...
}

node 'example.com' {
  service { 'apache2':
    ensure => present
  }
  ...
}
```

## 模組入場 - Resource 開始重複性高的情況下

重複的 `Resource` 寫多了就會覺得很腦殘，一直在做重複的事情，這時候模組化就派上用場了。

```puppet
node default {
  include ntp
}

node 'example.com' {
  include ntp
  include apache2
}
```

## Role and Profile - 20 台以上 Node 且複雜性高的時候

環境開始複雜而且不好管理的時候 `Role and Profile` 就可以進場了。

所以 Puppet 就提出了 [Role and Profile pattern][role-and-profile-pattern] 的管理方式，利於**可重用**、**可重構**、**可修改**的概念

![roles-and-profiles-overview](/assets/images/roles_and_profiles_overview.png)

Role and Profile 有幾個要點：

  - 使用大量 Normal module。
  - `Profile` 由很多 Normal module 組成。
  - `Role` 由很多 Profile module 組成。
  - 一個 node 只擁有一個 `Role`。

直接看範例。

## Profile example

```puppet
class profile::base {
  include ntp
}

class profile::user {
  include users
}
 
class profile::apache2 {
  class { 'apache':
    apache_version = '2.4',
  }
}
 
class profile::apache2::php {
  include profile::apache2
  include php
}
```

## Role example

```puppet
class role::webserver {
  include profile::base
  include profile::user
  include profile::apache2::php
}

class role::base {
  include profile::base
  include profile::user
}
```

## Node example

```puppet
node default {
  include role::base
}

node site {
  include role::webserver
}
```

當我有多個 `node`，就可以重複使用 `role` 直接 `include` 想要的設定，你只需要一行就能做到**可重用**。

當你要修改一個使用者的密碼，你只需要動 `profile::user`，而有 `include profile::user` 的 `Role` 都會跟著動來達到**可重構**。

然後再搭配 Hiera 來針對不同的情況給予參數來做到**可修改**。


[role-and-profile-pattern]: https://docs.puppet.com/pe/2017.2/r_n_p_intro.html