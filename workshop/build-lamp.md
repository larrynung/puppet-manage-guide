# 實作 LAMP


LAMP 是最常用到的 Web Server 架構之一，用 Puppet 來實作這種常用的架構是再實用不過。

## 學習項目

- resource
  - package
  - file
- data type
  - Array
- puppet file

## 實作目標

- apache2 + libapache2-mod-fcgid
- mpm_worker
- php7.0
- mysql-server

## Dependencies

在這個 workshop 會需要用到以下 module

- puppetlabs-apache
- puppetlabs-mysql
  
## LAB Start

### Apache

先定義 apache 的安全資訊，避免洩漏出版本訊息，並且先停用 mpm_module，因為待會要用 worker mode。

```puppet
class { 'apache':
  server_tokens    => 'Prod',
  server_signature => 'Off',
  mpm_module       => false,
}
```

再來就是開啟要用的 module，可以用 Array 的方式讓你寫重複的 Puppet code。

```puppet
$apache_modules = ['rewrite', 'actions', 'ssl', 'worker']

$apache_modules.each |String $module| {
  class { "apache::mod::${module}": }
}
```

再來是 VirtualHost 的部份，讓 Apache 可以跑 CGI

```
class { 'apache::vhosts':
  vhosts => {
    'demo' => {
      'server_name'   => 'example.com',
      'docroot'       => '/var/www/demo',
      'docroot_owner' => 'www-data',
      'docroot_group' => 'www-data',
      'port'          => '80',
      'override'      => 'All',
      'options'       => ['-Indexes', '+ExecCGI'],
    },
  },
}
```

### Fcgid

特別處理 fcgid，重複的參數可以考慮用變數來處理。

```puppet
class { 'apache::mod::fcgid':
  options => {
    'AddHandler'          => 'fcgid-script .php',
    'FcgidWrapper'        => '/usr/local/bin/php-wrapper .php',
    'FcgidConnectTimeout' => 20,
  }
}

file { '/usr/local/bin/php-wrapper':
  ensure => file,
  owner  => root,
  group  => root,
  mode   => '0755',
  source => "puppet:///modules/${module_name}/fcgid_php/php-wrapper",
}
```

php-wrapper 的參數調整參考官方

>PHP applications are usually configured using the FcgidWrapper directive and a corresponding wrapper script. The wrapper script can be an appropriate place to define any environment variables required by the application, such as PHP_FCGI_MAX_REQUESTS or anything else. (Environment variables can also be set with FcgidInitialEnv, but they then apply to all applications.)

php-wrapper 因為是用 files (puppet://) 的形式儲存，所以會放在 files/fcgid_php/php-wrapper，這邊讓 Puppet 抓這隻檔案。

```sh
#!/bin/sh
# Set desired PHP_FCGI_* environment variables.
# Example:
# PHP FastCGI processes exit after 500 requests by default.
PHP_FCGI_MAX_REQUESTS=10000
export PHP_FCGI_MAX_REQUESTS

# Replace with the path to your FastCGI-enabled PHP executable
exec /usr/bin/php-cgi
```

### PHP

用 package 安裝一狗票的 php package

```puppet
$php_package_list = ['libapache2-mod-fcgid',
                     'php7.0',
                     'php7.0-cgi',
                     'php7.0-cli',
                     'php7.0-common',
                     'php7.0-mysql',
                     'php7.0-mcrypt',
                     'php7.0-gd',
                     'php7.0-xmlrpc',
                     'php7.0-curl']
package { $php_package_list:
  ensure  => present,
}
```

### MySQL

這邊就會用到機敏資訊 password 的部份，就可以使用 [hiera-yaml](advanced/how-to-encrypt-hiera-data.md) 來儲存密碼。

```puppet
class { '::mysql::server':
  root_password           => 'strongpassword',
  remove_default_accounts => true,
}
```

驗證的部份就放個 phpinfo 來試試

```
file { '/var/www/demo/index.php':
    ensure  => file,
    owner   => 'www-data',
    group   => 'www-data',
    mode    => '0400',
    content => '<?php phpinfo(); ?>',
  }
```

最後確認能跑出 phpinfo 大致上就可以開始 deploy 你的 code 上線囉 !!













