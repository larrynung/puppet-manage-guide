# 如何使用 Puppet + Packer 佈署 AWS AMI

Packer 是由 [Hashicorp][hashicorp] 所開發的一個 build image 工具。

相較於 Docker，Packer 比較像是一個工具，提供你在各種 platform build image，而 Docker 只是其中一個 platform。

## Requires

在開始之前，要先安裝 Packer 和 [librarian-puppet][librarian-puppet] 這兩個套件在要執行 build 的機器上。

### Packer 安裝

在 Packer [Download][packer-download] 頁面上尋找自己適合的 OS 安裝，我這邊是 MacOS。

```shell
$ wget https://releases.hashicorp.com/packer/1.1.3/packer_1.1.3_darwin_amd64.zip
$ unzip packer_1.1.3_darwin_amd64.zip
```

解開之後會得到一個 packer 的執行檔，把它放到 PATH 裡面方便執行

```shell
$ mv packer /usr/local/bin/
$ packer -h

usage: packer [--version] [--help] <command> [<args>]

Available commands are:
    build       build image(s) from template
    fix         fixes templates from old versions of packer
    inspect     see components of a template
    push        push a template and supporting files to a Packer build service
    validate    check that a template is valid
    version     Prints the Packer version
```

### librarian-puppet 安裝

librarian-puppet 是幫助 Puppet 管理 module 的套件，只需要攥寫 Puppetfile 來管理 module，就像 php 的 [composer][composer]、nodejs 的 npm 一樣。

> 和 librarian-puppet 同性質的還有 [r10k][r10k]

由於 librarian-puppet 是用 Ruby 開發，所以使用 gem 來安裝 librarian-puppet

```shell
$ gem install librarian-puppet
```

## 用 Puppet + Packer build AMI

這個範例

先來看檔案結構：

```
├── Makefile
├── Puppetfile
├── manifests
│   └── site.pp
├── packer.json
└── scripts
    └── puppet_install.sh
```

- manifests/init.pp 

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

- Puppetfile 是拿來管理 module。

```
forge 'https://forgeapi.puppetlabs.com'

mod 'puppetlabs/apache'
mod 'puppetlabs/stdlib'
mod 'puppetlabs/concat'
mod 'puppetlabs/apt'
```

- packer.json 這個是 packer 的主要設定檔

```json
{
    "variables": {
        "aws_profile": "{{env `AWS_PROFILE`}}"
    },
    "builders": [
        {
            "type": "amazon-ebs",
            "profile": "{{ user `aws_profile`}}",
            "name": "apache2-php70-ami",
            "ami_description": "apache2-php70 (ap-northeast-1)",
            "region": "ap-northeast-1",
            "source_ami": "ami-bec974d8",
            "instance_type": "t2.nano",
            "ssh_username": "ubuntu",
            "ami_name": "apache2-php70-ami-{{timestamp}}",
            "tags": {
                "Name": "apache2-php70-ami-{{timestamp}}"
            }
        }
    ],
    "provisioners": [
        {
            "type": "shell",
            "script": "scripts/puppet_install.sh",
            "pause_before": "10s"
        },
        {
            "type": "puppet-masterless",
            "manifest_file": "manifests/init.pp",
            "module_paths": "modules",
            "puppet_bin_dir": "/opt/puppetlabs/bin"
        }
    ]
}
```

[builders][packer-builder] 是拿來定義 AWS 的參數，我有另外把 AWS Profile 用變數設定 export。


[provisioners][packer-provisioners] 則是用什麼來 build image，這邊用到了 shell 來安裝 puppet-agent，而 puppet-masterless 來安裝 apache2 + php7。

- scripts/puppet_install.sh

用來安裝 puppet-agent 的 script。

```shell
#!/bin/bash
wget https://apt.puppetlabs.com/puppet5-release-xenial.deb
sudo dpkg -i puppet5-release-xenial.deb
sudo apt update && sudo apt-get install puppet-agent -y
```

- Makefile

這個檔案是用來方便 build 的工具，還有 AWS Profile Name

```
#
PROFILE?= demo_profile

#
.PHONY: build

#
build:
	@AWS_PROFILE=${PROFILE} packer build packer-apache2-php7.json
```

### 處理 AWS Profile

AWS Profile 大致帶過怎麼寫 ...

```
[demo_profile]
aws_access_key_id = foo
aws_secret_access_key = bar
```

### Build

要 build 之前先用 librarian-puppet 產生 modules。

```shell
$ librarian-puppet install
```

然後利用 Makefile 開始 build

```shell
$ make
apache2-php70-ami output will be in this color.

==> apache2-php70-ami: Prevalidating AMI Name...
    apache2-php70-ami: Found Image ID: ami-bec974d8
==> apache2-php70-ami: Creating temporary keypair: packer_5a4b3a79-a62b-d6ae-5724-f114405b9f62
...
...
==> apache2-php70-ami: Deleting temporary keypair...
Build 'apache2-php70-ami' finished.

==> Builds finished. The artifacts of successful builds are:
--> apache2-php70-ami: AMIs were created:

ap-northeast-1: ami-cbc654ad
```

最後出現 AMI ID 就代表 build 成功囉!!!


[hashicorp]: https://hashicorp.com
[packer-io]: https://www.packer.io
[packer-download]: https://www.packer.io/downloads.html
[librarian-puppet]: http://librarian-puppet.com
[composer]: https://getcomposer.org/
[r10k]: https://github.com/puppetlabs/r10k
[ironman-day21]: https://ithelp.ithome.com.tw/articles/10195062
[packer-builder]: https://www.packer.io/docs/builders/index.html
[packer-provisioners]: https://www.packer.io/docs/provisioners/index.html

