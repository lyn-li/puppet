# 让Puppet运行在httpd之下

puppet使用SSL\(https\)协议来进行通讯，默认情况下，puppet server端使用基于Ruby的WEBRick HTTP服务器。由于WEBRick HTTP服务器在处理agent端的性能方面并不是很强劲，因此官方推介，在正式的生产环境中，把puppet-master运行在Apache的代理之下,使用Apache或者其他强劲的web服务器来处理客户的https请求。

```
yum install -y httpd httpd-devel mod_ssl \
ruby-devel rubygems gcc-c++ curl-devel zlib-devel make automake  openssl-devel
```

切换ruby的默认源

```
[root@puppet-master ~]# gem source ls
*** CURRENT SOURCES ***

https://rubygems.org/

[root@puppet-master ~]# gem source -d https://rubygems.org/
[root@puppet-master ~]# gem source -a https://gems.ruby-china.org
```

其实我们需要下载的gem包都可以在rubygem的官方库里面找到

```
https://rubygems.org/gems/passenger/versions/4.0.19
```

```
# gem install json
```

### 使用nginx转发不同环境的请求

```

```



