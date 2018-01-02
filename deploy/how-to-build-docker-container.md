# 如何使用 Puppet 佈署 Docker

`Puppet` + `Docker` 這兩個工具的結合可以利用 [image_build][image_build] 這個 module 來讓 Puppet 支援 Docker。

image_build 是由 [Puppetlabs][puppetlabs] 官方所維護的一個專案，其作用是讓 puppet 支援 `docker`、`aci` 這兩個 command。

## `aci`

aci (App Container Image) 是用來 build [App Container][app-container] 的工具，但小弟還沒有應用到，所以就不假會了。 

## `docker`

`docker` 這個 command 支援用來跑 build image，另外還可以產生 dockerfile，這個 dockerfile 可以直接拿去給 docker 使用。

### 安裝

在有安裝 puppet 的機器上執行 module install

```
$ puppet module install puppetlabs/image_build
```

執行 `puppet docker` 就會看到 `docker`功能啟用了，即使在 macOS 上也能動

```
$ puppet help docker
USAGE: puppet docker <action> [--from STRING]
...
```

### 範例

這個範例可以在 [puppet-docker-apache2-php7](https://github.com/shazi7804/puppet-docker-apache2-php7) 這邊找到。

整個目錄結構

```
├── Puppetfile
├── manifests
│   └── init.pp
└── metadata.yaml
```

- [**Puppetfile**][puppetfile] 用來定義 include 的 module。

```
forge 'https://forgeapi.puppetlabs.com'

mod 'puppetlabs/apache'
mod 'puppetlabs/stdlib'
mod 'puppetlabs/concat'
mod 'puppetlabs/apt'
```

模組的來源從 [puppetforge][puppetforge]，除此之外還可以從 Github 取得。

- **manifests/init.pp** 寫這個 image 的 resource 定義。

```
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
$dependency_apt = ['locale-gen', 'software-properties-common', 'python-software-properties']
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

- **metadata.yaml** 最後是這個 container 的 image data。

```
cmd: "/usr/sbin/apache2,-DFOREGROUND"
expose: 80
image_name: shazi7804/apache
```

準備好這些檔案後就可以開始 build image

```
$ puppet docker build
...
Successfully built f1ef9868c711
Successfully tagged shazi7804/apache:latest
```

## 產生 Dockerfile

我覺得這是相較於其他 build docker 的工具比較有趣的地方，image_build 提供產生 Dockerfile，對 Docker 還不熟的人很有幫助。

你只要簡單的執行一行指令就可以輸出 Dockerfile 內容

```
$ puppet docker dockerfile
```

或是直接輸出檔案

```
$ puppet docker dockerfile > Dockerfile
```

## 總結

用 Puppet build Docker 其實算是讓 puppet 的支援度更完整，如果你很熟 Docker 的話 build image 的工作直接寫 Dockerfile 可能比較輕鬆簡單。

而且 Puppet 在 build docker 其實是在 Docker 內跑 `puppet apply` 整體的 build time 比原生的 Docker build 速度較慢。

目前 image_build 的功能還沒有算很完整，最好用的 `files`、`templates` 並沒有支援。

個人評價目前只能算是堪用的一項功能。

[image_build]: https://github.com/puppetlabs/puppetlabs-image_build
[puppetlabs]: https://github.com/puppetlabs
[app-container]: https://github.com/appc/spec
[puppetfile]: https://puppet.com/docs/pe/2017.3/code_management/puppetfile.html
[puppetforge]: https://forge.puppet.com