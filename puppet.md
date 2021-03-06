# Puppet配置hiera数据源

## 1.简介和历史

Hiera是基于键值查询的数据配置工具，Hiera是一个可选工具，它的目标是：Hiera makes Puppet better by keeping site-specific data out of your manifests.  
它的出现使得代码逻辑和数据可以分离管理。  
在Puppet 2.x版本时代，Hiera作为一个独立的组件出现，若要使用则需要单独安装。在3.x版本之后，Hiera被整合到Puppet的代码中去。Hiera是Hierarchal单词的缩写，表明了其层次化数据格式的特点。

### 1.1实例说明

在使用hiera前，一个常见的manifests的init.pp文件是这么编写的：

```
class openstack(
  $enable_httpd = false,
){

   if $enable_httpd {
     package { 'httpd':}
   }
}
```

如果有多个Puppet环境都调用了这个基础函数，有些环境希望打开httpd，有些则不希望打开httpd，我们怎么办呢？如果我们建立多套代码库，那么不利于后期维护，所以Puppet社区就想了一个方法，把这些参数的输入，全部单独独立出来，代码逻辑不变，但是你在外部输入了不同的变量，就可以走不通的代码逻辑。就是说，我希望有一个文件，可以根据不同的环境设定我们的enable\_httpd的值。这就是Puppet的hiera的数据配置的基础功能。

所有的数据设置移到了hiera中:

```
vsftpd::enable: true
```

这里就定义了hiera的变量的值。

## 2. hiera.yaml配置文件

hiera.yaml是Hiera唯一的配置文件，它其中只有少数几个配置参数，但决定了Hiera不同的使用方式。

### 2.1 hiera初体验

在puppet.conf中通过设置hiera\_config参数来设置hiera.yaml文件的路径，默认值为：$confdir/hiera.yaml  
\(注意Puppet 4.x以上时，默认值变更为$codedir/hiera.yaml\)，查看本地的hiera的路径：

```
[root@puppet-master puppet]# puppet master --configprint hiera_config
/etc/puppet/hiera.yaml
```

接下来我们就自己建立一个hiera.yaml文件：

```
[root@puppet-master puppet]# mdkir -p /etc/puppet/hieradata
[root@puppet-master puppet]# cat hiera.yaml
---
:backends:
  - yaml
:hierarchy:
  - "global/base"
:yaml:
   :datadir: /etc/puppet/hieradata
```

上面的hierarchy部分定义了,我们将会在/etc/puppet/hieradata这个目录下面找哪个文件。我们接下来就需要去建立这个base文件，注意是，global／base.yaml这个文件，因为在backends中定义的文件后缀是yaml。

新建base.yaml文件：

```
[root@puppet-master puppet]# cat /etc/puppet/hieradata/global/base.yaml
enable_httpd: true
```

之后在module中设置：

```
class openstack(
  $enable_httpd = hiera('enable_httpd'),
){

notify { "$enable_httpd": }

}
```

之后在agent中运行测试：

```
[root@puppet-agent ~]# puppet agent   -t --server puppet-master.openstacklocal
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Caching catalog for puppet-agent.openstacklocal
Info: Applying configuration version '1504337788'
Notice: true
Notice: /Stage[main]/Openstack/Notify[true]/message: defined 'message' as 'true'
Notice: Finished catalog run in 0.03 seconds
```

我们发现已经可以通过

```
$text = hiera('enable_httpd')
```

获取到hiera里面的参数的值了。

### 2.2 设置不同的客户端路径

我们在上面设定一了一个固定的hiera的文件路径：

```
/etc/puppet/hieradata/global/base.yaml
```

但是在实际的生产环境中，却不是这样配置的。因为生产环境中常常有很多套环境，所以说我们的hiera的文件的路径需要支持变量，这里我们用的是域名的方式进行区分，不同的环境：

```
[root@puppet-master puppet]# cat hiera.yaml
---
:backends:
  - yaml
:hierarchy:
  - "%{::domain}/base"
:yaml:
# datadir is empty here, so hiera uses its defaults:
#  - /var/lib/hiera on *nix
#  - %CommonAppData%\PuppetLabs\hiera\var on Windows
#  When specifying a datadir, make sure the directory exists.
   :datadir: /etc/puppet/hieradata
```

注：hierarchy下可以写很多行，因为它是个列表项，如：

```
:hierarchy:
  - "%{::domain}/base1"
  - "%{::domain}/base2"
  - "%{::domain}/base3"
```

但是你会发现在运行的时候并不是所有的base1,base2,base3都会被加载，他只会去加载第一个匹配到的文件。



之后在puppet-agent中查看我们的域名：

```
[root@puppet-agent puppet]# hostname -d
openstacklocal
```

同样的，我们需要在puppet-master端建立对应的路径：

```
[root@puppet-master puppet]# cat /etc/puppet/hieradata/openstacklocal/base.yaml
test_hiera_domain: openstacklocal
```

修改我们master的模块代码进行测试：

```
class openstack(
  $test_hiera_domain = hiera('test_hiera_domain'),
){

  notify { "$test_hiera_domain": }

}
```

puppet-agent端运行测试：

```
[root@puppet-agent ~]# puppet agent   -t --server puppet-master.openstacklocal
Info: Retrieving pluginfacts
Info: Retrieving plugin
Info: Caching catalog for puppet-agent.openstacklocal
Info: Applying configuration version '1504342356'
Notice: openstacklocal
Notice: /Stage[main]/Openstack/Notify[openstacklocal]/message: defined 'message' as 'openstacklocal'
Notice: Finished catalog run in 0.03 seconds
```

我们能看到这里的运行结果就是对的。

参考资料：

[http://docs.puppetlabs.com/hiera/latest/](http://docs.puppetlabs.com/hiera/latest/)

