# 實作 Nginx and NodeJS

Nginx 和 NodeJS 也是在實務上常遇到的，面對這樣的常用的項目，我會讓他盡可能的彈性，透過不同的參數實現不同的設定，那麼 hiera 就是必會的功能。

## 學習項目

- hiera
- class include
- resource
  - file
- if/else
- data type

## 實作目標

- Nginx (Reverse Proxy)
- nvm

## Dependencies

在這個 workshop 會需要用到以下 module

- puppet-nginx (Vox Pupuli)
- corp104-nvm (104corp)
  
## LAB Start

這個 LAB 會實作到 hiera，所以我必須把所有可動的參數都拉出來去參考 hiera 的 data

## Nginx 

直接使用 Github 的 nginx 模組來安裝 nginx。

```puppet
class profile::nginx (
  Integer $worker_connections,
  Integer $worker_rlimit_nofile,
  String $server_tokens,
  Array $proxy_set_header,
  String $gzip,
  String $gzip_types,
  Integer $gzip_comp_level,
){
  class { 'nginx':
    server_tokens        => $server_tokens,
    worker_connections   => $worker_connections,
    worker_rlimit_nofile => $worker_rlimit_nofile,
    proxy_set_header     => $proxy_set_header,
    gzip                 => $gzip,
    gzip_types           => $gzip_types,
    gzip_comp_level      => $gzip_comp_level,
  }
}
```

在 nginx 這邊我僅定義簡單的 nginx global 設定，Proxy 的設定寫在 nodejs 這個 class。

## nodejs

在 nodejs 去 include nginx，考慮到 nvm 的安裝要到 internet 有可能要透過 Proxy，所以也加入 Proxy 的判斷。

```puppet
class profile::nginx::nodejs (
  String $node_version,
  Optional[String] $full_http_proxy,
  Array $upstream_members,
){
  include profile::nginx

  # generate nginx resource
  nginx::resource::upstream { "puppet-${::fqdn}":
    members => $upstream_members,
  }
  nginx::resource::server { $::fqdn:
    proxy => "http://puppet-${::fqdn}",
  }

  file {
    default:
      ensure  => absent,
      require => Package['nginx'],
      notify  => Service['nginx'],
    ;
    'default-conf':
      path => '/etc/nginx/conf.d/default.conf',
    ;
    'default-site-conf':
      path => '/etc/nginx/sites-enabled/default',
    ;
  }

  # include nvm module
  if $full_http_proxy {
    class { 'corp104_nvm':
      node_version => $node_version,
      http_proxy   => $full_http_proxy,
    }
  }
  else {
    class { 'corp104_nvm':
      node_version => $node_version,
    }
  }
}
```

最後用 [hiera](basic/how-to-use-hiera-data.md) 把參數補上。

```yaml
profile::nginx::worker_connections: 4096
profile::nginx::worker_rlimit_nofile: 204800
profile::nginx::server_tokens: 'off'
profile::nginx::proxy_set_header:
  - 'Host $host'
  - 'X-Real-IP $remote_addr'
  - 'X-Forwarded-For $proxy_add_x_forwarded_for'
  - 'X-Forwarded-Proto $http_x_forwarded_proto'
profile::nginx::stub_status: true
profile::nginx::gzip: 'on'
profile::nginx::gzip_types: 'text/plain text/css text/xml text/javascript application/json application/x-javascript application/javascript application/xml'
profile::nginx::gzip_comp_level: 6
profile::nginx::nodejs::node_version: '8.8.0'
profile::nginx::nodejs::upstream_members:
  - 'localhost:8080'
profile::nginx::nodejs::full_http_proxy: ~
```

從這個範例會安裝 Nginx 並且 Proxy 8080 port，nvm 則會安裝 nodejs 8.8.0 的版本。





