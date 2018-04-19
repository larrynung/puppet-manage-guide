## 什麼是 Vagrant ?

[`HashiCorp`][hashicorp] 出品的 ~~HashiCorp 產品必屬佳作~~ `Vagrant`，簡單介紹一下 `Vagrant` 是一款用於構建及配置虛擬開發環境的軟體，基於 `Ruby` 開發，主要使用 `Oracle` 的開源 `VirtualBox` 虛擬化系統。

`Vagrant` 支援 Chef、Puppet、Ansible .. 等快速的方式構建環境。

---

## Requires

在開始之前，要先安裝 `Vagrant`、`Virtualbox` 和 [`librarian-puppet`][librarian-puppet]。

---

### Virtualbox 安裝

`Virtualbox` 的官方 [Download](https://www.virtualbox.org/wiki/Downloads) 找到適合自己的環境安裝。

### Vagrent 安裝

`Vagrant` 的安裝也很簡單，以 `Ubuntu` 為例

```bash
$ wget https://releases.hashicorp.com/vagrant/2.0.1/vagrant_2.0.1_x86_64.deb
$ dpkg -i vagrant_2.0.1_x86_64.deb
$ vagrant version
Installed Version: 2.0.1
Latest Version: 2.0.1
```

### librarian-puppet 安裝

`librarian-puppet` 是幫助 `Puppet` 管理 `module` 的套件，只需要攥寫 `Puppetfile` 來管理 `module`，就像 php 的 [composer][composer]、nodejs 的 npm 一樣。

> 和 `librarian-puppet` 同性質的還有 [r10k][r10k]

由於 `librarian-puppet` 是用 `Ruby` 開發，所以使用 `gem` 來安裝 `librarian-puppet`

```bash
$ gem install librarian-puppet
```

---

## 用 Puppet 來 build Vagrant

其實用 `Puppet` 來 build Vagrant 和前面兩篇的 `Docker` 和 `Packer` 一樣，差別在於不同的 `Provisioning` 設定的方式不同。

先看檔案結構

```
├── Puppetfile
├── Vagrantfile
├── environments
│   └── development
│       └── manifests
│           └── init.pp
├── metadata.json
└── scripts
    ├── init.sh
    └── puppet_install.sh
```

- **environments/development/manifests/init.pp** 這邊寫 Puppet code

```puppet
$php_version = '7.0'

class { 'apache':
  server_tokens    => 'Prod',
  server_signature => 'Off',
  default_vhost    => false,
  mpm_module       => false,
}

apache::vhost { 'localhost':
  port           => 80,
  docroot        => '/var/www/html',
  docroot_owner  => 'www-data',
  docroot_group  => 'www-data',
  directoryindex => 'index.php',
  override       => 'All',
  options        => ['-Indexes', '+ExecCGI']
}

$default_modules = ['rewrite', 'actions', 'ssl', 'worker']
$default_modules.each |String $module| {
  class { "apache::mod::${module}": }
}

# Use fcgid run php
class { 'apache::mod::fcgid':
  options => {
    'AddHandler'          => 'fcgid-script .php',
    'FcgidWrapper'        => '/usr/local/bin/php-wrapper .php',
    'FcgidConnectTimeout' => 20,
  }
}

file { '/usr/local/bin/php-wrapper':
  ensure  => file,
  owner   => root,
  group   => root,
  mode    => '0755',
  content => "#!/bin/bash\nPHP_FCGI_MAX_REQUESTS=10000\nexport PHP_FCGI_MAX_REQUESTS\nexec /usr/bin/php-cgi";
}

# add 'ppa:ondrej/php' repository
$dependency_apt = ['locales', 'software-properties-common', 'python-software-properties']
package { $dependency_apt: ensure => present }

exec { 'install-ppa':
  provider    => 'shell',
  environment => ['LANG=en_US.UTF-8'],
  path        => '/bin:/usr/sbin:/usr/bin:/sbin',
  command     => "/usr/sbin/locale-gen en_US.UTF-8 && add-apt-repository -y ppa:ondrej/php && apt-get update",
  user        => 'root',
  unless      => 'apt-cache policy | grep ondrej/php',
  require     => Package[$dependency_apt]
}

$php_package_list = [ "php${php_version}",
                      "libapache2-mod-php${php_version}",
                      "php${php_version}-mysql",
                      "php${php_version}-cli",
                      "php${php_version}-cgi",
                      "php${php_version}-common",
                      "php${php_version}-mcrypt",
                      "php${php_version}-gd",
                      "php${php_version}-json",
                      "php${php_version}-bcmath",
                      "php${php_version}-mbstring",
                      "php${php_version}-xml",
                      "php${php_version}-xmlrpc",
                      "php${php_version}-zip",
                      "php${php_version}-soap",
                      "php${php_version}-sqlite3",
                      "php${php_version}-curl",
                      "php${php_version}-opcache",
                      "php${php_version}-readline",
                      'php-mongodb',
                      'php-memcached'
]

package { $php_package_list:
  ensure  => present,
  require => Exec['install-ppa'],
}

file { '/var/www/html/index.php':
  ensure  => present,
  content => '<?php phpinfo(); ?>',
}
```

- Puppetfile 也和前面兩天一樣，拿來管理 module。

```
forge 'https://forgeapi.puppetlabs.com'

mod 'puppetlabs/apache'
mod 'puppetlabs/stdlib'
mod 'puppetlabs/concat'
mod 'puppetlabs/apt'
```

- `Vagrantfile` 用來定義這個 `Vagrent box` 要做的主要設定檔

```
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/xenial64"

  config.vm.provision "shell", path: "scripts/init.sh"
  config.vm.provision "shell", path: "scripts/puppet_install.sh"

  # Enable the Puppet provisioner, with will look in manifests
  config.vm.provision "puppet" do |puppet|
    puppet.environment_path = "environments"
    puppet.environment = "development"
    puppet.manifests_path = "environments/development/manifests"
    puppet.manifest_file = "init.pp"
    puppet.module_path = "modules"
  end

  config.vm.network "forwarded_port", guest: 80, host: 8080, protocol: "tcp"

  config.vm.provision "shell", inline: "echo Enjoy your new vbox."
end
```

這個 `Vagrantfile` 拿 `Ubuntu 16.04` 當 source image，再執行 `Puppet` 前先跑 shell `init.sh` 和 `puppet_install.sh` 來初始化環境，否則 `Guest` 環境是沒有 `Puppet` 可以讓你執行的。

LAB 的最後會測試將 guest 80 port forward 到 host 的 8080 port 來測試是否正常。

- `scripts/init.sh` 這隻拿來改密碼，預設 `Ubuntu` 的密碼實在太不親民了。

```bash
#!/bin/bash
echo "ubuntu:ironman" | sudo chpasswd
```

> 使用者：ubuntu
> 密碼：ironman

- `scripts/puppet_install.sh` 這隻用來安裝 Puppet agent。

```bash
#!/bin/bash
wget https://apt.puppetlabs.com/puppet5-release-xenial.deb
sudo dpkg -i puppet5-release-xenial.deb
sudo apt update && sudo apt install puppet-agent -y
```

- `metadata.json` 這個是用來定義這個 Vagrant box 的 `metadata`，主要用來上傳 Vagrant box 顯示的資訊，沒有一定需要。 

```json
{
  "name": "shazi7804/apache2-php7",
  "description": "This box is build Apache2, PHP7 built on Ubuntu 16.04 LTS 64-bit.",
  "versions": [
    {
      "version": "0.0.1",
      "providers": [
        {
          "name": "virtualbox",
        }
      ]
    }
  ]
}
```

---

## Build

要 build 之前先用 `librarian-puppet` 產生 modules。

```bash
$ librarian-puppet install
```

檔案都準備好了之後就可以開始測試 `building`。

```bash
$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'ubuntu/xenial64'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'ubuntu/xenial64' is up to date...
...
...
==> default: Notice: Applied catalog in 168.25 seconds
==> default: Running provisioner: shell...
    default: Running: inline script
==> default: Enjoy your new vbox.
```

最後跑完最後一筆 shell 出現 `Enjoy your new vbox.` 就成功了，這時你的 Virtualbox 應該會出現一台虛擬機。

此時你可以檢查 http://localhost:8080 要能看到 `phpinfo`。


[hashicorp]: https://hashicorp.com
[librarian-puppet]: http://librarian-puppet.com
[composer]: https://getcomposer.org/
[r10k]: https://github.com/puppetlabs/r10k