# Openwrt之LUCI学习
## 1.LUCI简介
LUCI是一个小巧的语言，诞生于2008年3月份，目的是为OpenWrt固件从Opwnwrt的第一个分支 [White Russian](https://openwrt.org/about/history) 到随后的分支 [Kamikaze](https://openwrt.org/about/history) 实现快速配置接口。轻量级 LUA语言的官方版本只包括一个精简的核心和最基本的库。这使得LUA体积小、启动速度快，从而适合嵌入在别的程序里。UCI（Unified Configuration Interface，统一配置接口）是OpenWrt中为实现所有系统配置的一个统一接口。LuCI,即是这两个项目的合体，可以实现路由的网页配置界面。

### 1.1 Lua简介
Lua 由巴西里约热内卢天主教大学（Pontifical Catholic University of Rio de Janeiro）里的一个研究小组于 1993 年开发的，该小组成员有：Roberto Ierusalimschy、Waldemar Celes 和 Luiz Henrique de Figueiredo。Lua是一种轻量小巧的脚本语言，用标准C语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。Lua有以下特性：
- 轻量级: 它用标准C语言编写并以源代码形式开放，编译后仅仅一百余K，可以很方便的嵌入别的程序里。
- 可扩展: Lua提供了非常易于使用的扩展接口和机制：由宿主语言(通常是C或C++)提供这些功能，Lua可以使用它们，就像是本来就内置的功能一样。
- 支持面向过程(procedure-oriented)编程和函数式编程(functional programming)；
- 自动内存管理；只提供了一种通用类型的表（table），用它可以实现数组，哈希表，集合，对象；
- 语言内置模式匹配；闭包(closure)；函数也可以看做一个值；提供多线程（协同进程，并非操作系统所支持的线程）支持；
- 通过闭包和table可以很方便地支持面向对象编程所需要的一些关键机制，比如数据抽象，虚函数，继承和重载等。


## 2. Luci的安装
### 2.1 从openwrt源安装
1. 转到OpenWrt根目录。
2. 输入 `./scripts/feeds update`
3. 输入 `./scripts/feeds install -a -p luci`
4. 输入 `make menuconfig`
5. 在”LuCI”菜单下你将找到所有的组件。
### 2.2 从OpenWrt 安装包版本库：
1. 添加一行文字到你的`/etc/opkg.conf`中，即将LuCI添加到版本库中:
```
src luci http://downloads.openwrt/kamikaze/8.09.2/YOUR_ARCHITECTURE/packages
```
2. 输入 `opkg update`
3.
    - LuCI 简版，输入: `opkg install luci-light` ;
    - LuCI 普通版: `opkg install luci`;
    - 自定义模块的安装: `opkg install luci-app-*`.
4. 为了实现HTTPS支持，需要安装`luci-ssl meta`安装包
5. 由于opkg-installed服务是默认关闭的，你需要手动开启使它能够开机启动：

```shell
root@OpenWrt:~# /etc/init.d/uhttpd enable
root@OpenWrt:~# /etc/init.d/uhttpd start
```


## 3. LUCI的基本架构
LUCI可以归属为web开发行列。因为它的内容都是直接在浏览器里体现出来的。它的基本架构跟很多WEB开发语言的框架一样，都是MVC架构。以下给出一些简单的解释：

- **M(Model):** 模型层，路径是/usr/lib/lua/luci/model/。这个是数据处理的具体代码的层。该层中有一个cbi文件夹，那里面是预定义的一些的逻辑文件。

- **V（View）:** 视图层，路径是/usr/lib/lua/luci/view/。这个很容易理解，就是存放视图文件的地方。LUCI里的视图文件是.htm文件。

- **C（Controller）:** 控制层，路径是/usr/lib/lua/luci/controller/admin/。这个层最主要的功能是设定路由；它还有一个功能跟模型层一致——处理数据。
根目录: /usr/lib/lua/luci
资源文件存放目录: /www/luci-static/

### 3.1 MVC框架
MVC全名是Model View Controller，是模型(model)－视图(view)－控制器(controller)的缩写，一种软件设计典范，用一种业务逻辑、数据、界面显示分离的方法组织代码，将业务逻辑聚集到一个部件里面，在改进和个性化定制界面及用户交互的同时，不需要重新编写业务逻辑。MVC被独特的发展起来用于映射传统的输入、处理和输出功能在一个逻辑的图形化用户界面的结构中。MVC 分层有助于管理复杂的应用程序，因为您可以在一个时间内专门关注一个方面。例如，您可以在不依赖业务逻辑的情况下专注于视图设计。同时也让应用程序的测试更加容易。
- Model（模型）是应用程序中用于处理应用程序数据逻辑的部分。通常模型对象负责在数据库中存取数据。
- View（视图）是应用程序中处理数据显示的部分。通常视图是依据模型数据创建的。
- Controller（控制器）是应用程序中处理用户交互的部分。通常控制器负责从视图读取数据，控制用户输入，并向模型发送数据。

![avatar](https://baike.baidu.com/pic/MVC框架/9241230/0/ac6eddc451da81cb26660e7e5066d01608243184?fr=lemma&ct=single#aid=0&pic=b03533fa828ba61edbddc04d4034970a304e59a4)

## 4. 开始建立新页面
以配置network文件为例，实现一个testnet模块需要完成下面的步骤。
步骤：

1. 建立一个配置文件
1. 定义控制层
1. 定义模型层
### 4.1 建立一个配置文件
之前介绍的时候有说到，LUCI是用于存取配置文件的信息，所以我们的若要新建一个模块，需要一个配置文件。配置文件的路径一般是在/etc/config/。由于现在我们是需要配置network文件，而这个是系统必备文件，它已经存在，所以我们不需要新建，但我们要了解一下里面的结构是怎么样的：


```
config interface 'loopback'
        option ifname 'lo'
        option proto 'static'
        option ipaddr '127.0.0.1'
        option netmask '255.0.0.0'

config interface 'lan'
        option type 'bridge'
        option ifname 'eth1'
        option proto 'static'
        option ipaddr '192.168.1.1'
        option netmask '255.255.255.0'
        option ip6assign '60'
```

这个是配置里有内容，而其他配置文件内容格式也是差不多：


```
-- 编辑配置文件
root@openWrt:~# vim /etc/config/<config name>
-- 配置文件格式
config <section type> <section name>
    option <option name> <option value>
    option <option name> <option value>
    option <option name> <option value>
```

那么我们建立一个新的配置给testnet模块使用

```
config interface testnets
    option ifname 'testnets'
    option type 'test'
    option ipaddr '192.168.1.1'
    option net '192.168.251.1'
```

### 4.2 定义控制层
控制层中，我们主要定义访问的路由。每一个模块的控制都是独立的，那样才能方便控制。路由的定义是在控制器的`index()`函数里。控制器里还可以定义其他的函数用于处理处理，相信这一点有过WEB开发经验的童鞋们都知道。
接下来我们建立一个能以/admin/testnet访问的路由
话不多说，直接上代码：


```
-- /controller/admin/testnet.lua
module("luci.controller.admin.testnet")
function index()
    entry({"admin", "testnet", "index"}, cbi("admin_test/net"), translate("Test Net"), 10)
end

function test()
    return 'This is a test function.'
end
```

`module("luci.controller.admin.testnet")`: 这一句是命名空间的声明，luci指的是`/usr/lib/lua/luci`这个的文件夹，也就是LUCI的根目录
`entry({"admin", "testnet", "index"}, cbi("admin_test/net")`, `translate("Test Net"), 10)`: 这一句必须写在控制器文件的`index()`函数里。它的格式定义是这样子的：

`entry(path, target[[, title][, order]])
path`: 即路由规则，格式是`{"admin","testnet","index"[[,"..."][,"..."]]}`基本上可以无限延伸，但一般不建议这么干，到五六层已经很深了，再潜就不好了。
`target`: 即页面指向，格式是`cbi("admin_test/net")`。这里的`cbi`的函数指的是调用`/luci/model/cbi/`里的`admin_test/`里的`net.lua`文件。它还有其他的使用方法：

cbi("...")：调用/luci/model/cbi/文件夹中的指定lua文件，这里指向的文件相当于指向一个处理函数。使用的是LUCI自带逻辑的处理方法，本文就是使用这种方法来生成的页面。
template("...")：调用/luci/view/文件夹里指定的.htm视图文件。这个方法是直接调用视图。在视图里我们也可以嵌入代码读取配置。
alias("..."): 这个是重定向函数， 一般用于顶级菜单上，使其重定向到指定的子菜单。
call("..."): 这个函数的作用是将该路由指向控制下的某个函数，一般用于处理数据，作用与指向模型层的cbi("...")函数类似，只是一个指向到其他文件，一个仍是存在于控制层文件内。
post("..."): 这个函数在openwrt里的其他模块有使用过，本人研究了一下，其作用于call方法类似，但在使用的时候似乎没有成功，若是有成功使用过的猿友，欢迎交流探讨。
*　title: 即标题展示，这个是设置在菜单里显示的内容项。格式有_("Test Net")或translate("Test Net")，前一个使用我也没有成功（有点摸不清楚），后一个是调用translate('...')函数，用于与后期的语言包进行适配。该项可以用nil代替，表示为不在菜单栏显示。

order: 这个是排序，为子菜单进行排序，序号以1开始，最大不限。该项也可以忽略，表示为不在菜单栏显示。
*　若想让某项隐藏，可以写成以下格式：

entry({"admin", "testnet", "index"}, cbi("admin_test/net"), nil)
3> 定义模型层
OK，说了那么多，终于到了我们最关键的模型层。在模型层，LUCI拥有一套自动生成机制，接下来让我们一一介绍。先上代码再解释——

-- /model/cbi/admin_test/net.lua
m = Map("network", translate("Test Net"))

s = m:section(NamedSection, "testnets", "interface", translate("Net Configuration")
s.addremove = true
s.anonymouse = true

ifname = s:option(Value, "ifname", translate("Ifname: "))
ifname.datatype = 'string'

itype = s:option(Value, "type", translate("Type: "))
itype.datatype = 'string'

ipaddr = s:option(Value, "ipaddr", translate("Ipaddr: "))
ipaddr.datatype = 'ipaddr'

net = s:option(Value, "net", translate("Net:"))
net.datatype = 'ipaddr'

return m
m = Map("network", translate("Test Net"))：这一句是连接配置文件，是以LUCI机制的必不可少的语句。第一个参数是配置文件的名称，第二个参数页面的大标题。
s = m:section(NamedSection, "testnets", "interface", translate("Net Configuration")：这一句是以section`的name名连接到指定的section。

第一个参数是指定存取section的方法，本文用的是以section的name名进行查找的方式NamedSecton。其他还有几种方式：

TypedName: 根据section的type来进行存取
SimpleSection：（这个没用过，就不多说了）
Table：以表格的形式体现section
Tab：以标签的形式体现section
第二个参数是section的名字
第三个参数是section的类型
第四个参数是模块标题的名称
其他类型的代码示例分别如下：

-- TypedSection
s = m:section(TypedSection, "interface", translate("Net Configuration")

-- SimpleSection
s = m:section(SimpleSection, "interface", translate("Net Configuration")

-- Table
s = m:section(TypedSection, "interface", translate("Net Configuration")--也可以用NamedSection
s.Table(Table, "Table Title")

-- Tab
s = m:section(TypedSection, "interface", translate("Net Configuration")--也可以用NamedSection
s.Table(Tab, "Tab Title")
s.addremove = false：打开/关闭添加删除按钮，这个为true时，会在页面上添加一个添加和一个删除按钮，这样就能够在页面快速进行增减项了。不过有一点要注意的是，那样的页面比较丑……
s.anonymouse = true：这一句是不显示section的名字在页面上。还可以加上s.template = 'admin_test/net'这样的语句
net = s:option(Value, "net", translate("Net:"))：这一句是连接具体的option。

第一个参数是显示类型。常用的显示类型有：

Value（普通文本框）、ListValue（下拉列表）、Flag（复选框）、MultiValue（文本域）、DummyValue（纯文本）、TextValue（多行input）、Button（按钮）、StaticList（静态列表）、DynamicList（动态列表）
第二个参数是option的名称
第三个参数是显示项的说明
net.datatype = 'ipaddr'：这一句是指定option的类型。这里的是ip地址类型，其他还有很多类型：

neg、list、bool（布尔类型）、uinteger、integer（整型）、ufloat、float（浮点型）、ipaddr（IP地址）、ip4addr（IP4型IP地址）、ip4prefix（IP4前缀）、ip6addr、ip6prefix、port、portrange、macaddr、hostname、host、network、wpakey、wepkey、string、directory、file、device、uciname、range、min、max、rangelength、minlength、maxlength、phonedigit
这时访问/admin/testnet即可以看到页面了。

clipboard.png