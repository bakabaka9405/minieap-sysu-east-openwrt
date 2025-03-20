# MiniEAP in SYSU East for OpenWrt

为中山大学东校园有线网魔改的 MiniEAP，运行在 OpenWrt 上。仅在 AX6S 和极路由 4 上进行过测试，不保证稳定性和时效性。

对原版的修改仅为移除`eap_state_machine.c`中状态机的状态停留次数限制，抓包分析可知中山大学东校园锐捷认证的心跳包形式为服务器每隔 30s 向客户端发送一次用户名请求，而不是常见的客户端定期向服务器主动发送心跳包。这种不常见的形式会让 MiniEAP 程序状态机认为自身正处于认证前循环而直接退出，故作此修改。认证算法上和原版无区别。

要交叉编译并在 OpenWrt 上部署，可以按以下步骤进行：

1. 交叉编译在 Linux 上进行
2. 下载你的路由器对应架构的 OpenWrt SDK
3. 配置好`PATH`和`STAGING_DIR`
4. 交叉编译 libpcap 得到`libpcap.a`
5. 修改该 repo 中的`config.mk`文件，分别在`CUSTOM_CFLAGS`和`CUSTOM_LIBS`配置好 libpcap 的 include 目录和`libpcap.a`路径
6. 交叉编译 MiniEAP 得到可执行文件
7. 通过 ssh 或其他方法将可执行文件上传到 OpenWrt，并赋予执行权限

如果出现编译失败或运行时出现乱码等现象，请参考原版编译指引。

启动指令参考：

```bash
./minieap -k 1 -u username -p password -n wan -b 3 --save --module rjv3 --fake-dns2 114.114.114.114 --fake-serial fakeserial --if-impl libpcap -t 3000000
```

请自行修改`username`，`password`，`fakeserial`字段，前两个字段作用显然，`fakeserial`理论上随便填？但是似乎不能不填，原因费解。如果路由器的 wan 口名称不是`wan`，也需要修改。

`-t 3000000`的作用是将本机发送心跳包的间隔调成尽可能大，因为本机无需主动发送心跳包。也可以直接修改源代码关闭心跳包逻辑，但是太懒了（

没有做成 ipk 包，同样是因为懒。

`-k 1` 的作用是杀死其他正在运行的 MiniEAP 程序并且继续执行本程序。

`-b 3`的作用是让 MiniEAP 在后台运行，如果是为了调试可以删去。后台运行时要查看 MiniEAP 的输出，可以使用以下指令：

```bash
cat /var/log/minieap.log
```

初次运行时可能会卡在寻找服务器的阶段，可能需要多执行几次。成功认证的特征为输出“认证成功”字样，之后每隔半分钟输出一次“正在回应用户名请求"。

偶尔断连是正常现象，你鸭破网就这b样。~~不过这学期好多了~~

以下为原版 README：

MiniEAP
=======




这是一个实现了标准 EAP-MD5-Challenge 算法的 EAP 客户端，支持通过插件来修改标准数据包以通过特殊服务端的认证。目前带有一个实现锐捷 v3 (v4) 算法的插件。本插件的认证算法来自 [Hu Yunrui 的 MentoHUST 项目](https://github.com/hyrathb/mentohust)，在此表示感谢！

## 特性

#### 通用特性

* 模块化设计
可在 `config.mk` 中直观选择所需模块。如需添加模块，直接复制一份现有的 `minieap.mk` 并按需修改即可。

* 网络帧收发由插件模块完成
可根据平台差异使用不同的插件。目前提供 `libpcap` 和 Raw Socket 两种插件。前者兼容性好，但需链接 `libpcap`，体积较大；后者不需额外动态库，但只能在 Linux 上使用。可选择任意个模块参与编译，但运行时只能选取其中之一来使用。

* 数据包修改同样由插件完成
可以在不修改主要认证流程的情况下适配各种环境。可以启用多个插件，也可将一个插件启用多次。程序会让标准 EAP 算法生成的数据包按照命令行中 `--module` 参数的顺序让数据包流经这些插件。目前提供一个锐捷 v3 认证算法插件和一个打印流经的数据包大小的示例插件。

* 所有数据包生成逻辑均采用结构体对缓冲区进行读写，拒绝 magic number 从我做起！

#### 锐捷插件特性

* 认证算法来自 [Hu Yunrui 的 MentoHUST 项目](https://github.com/hyrathb/mentohust)
* 相比原本的 MentoHUST v3 (v4) 实现，能够支持更多的字段，更容易通过验证。
* 二次认证时，支持位于修改常规字段以外的 IP 地址、网关、主 DNS 等信息，更容易通过验证。
* 所有字段都通过收集来的信息直接构造而成，不采用修改数据包模板的方式，避免各场景下偏移量不同导致的认证失败或数据包无法解析问题。
* 所有字段生成逻辑均采用结构体对缓冲区进行读写，拒绝 magic number 从我做起 x2！
* 字段中所用到的常量都有宏定义来注明其含义，定长字段的长度也通过宏定义声明，拒绝 magic number 从我做起 x3！
* 支持通过命令行来附加新的字段，也可覆盖程序生成的字段。可以在不修改代码的情况下进行适配。
* 整体程序的内存占用比 MentoHUST 小约 78%（在 256 MB 内存的 ARMv7 平台上测试）。

## 编译

1. 编辑 `config.mk`，选择所需要的模块。
在以 `if_impl` 开头的模块中，Linux 环境建议只选择 `if_impl_sockraw` 模块，其他平台建议只选择 `if_impl_libpcap` 模块。
在以 `packet_plugin` 开头的模块中，请按需要选择。
注：若选择 `if_impl_libpcap`，将自动添加 `-lpcap` 选项。

2. 本程序需要使用 `getifaddrs`。
如果您的平台没有提供此函数，可自行寻找需要的实现，并在 `include/` 中添加 `ifaddrs.h`，在 `util/ifaddrs/` 目录中添加必要的 C 文件，最后在 `config.mk` 中选中 `ifaddrs` 模块即可。

3. 如果服务器消息乱码，可将 `config.mk` 中的 `ENABLE_ICONV` 置为 1.
若平台未提供iconv相关函数，需手动链接 `libiconv` 库。

4. 执行 `make` 即可在根目录下编译出可执行文件。

注1：如需要交叉编译，可参考 `config.mk` 中的示例。

注2：如需要链接外部库，请在 `COMMON_CFLAGS`、`COMMON_LDFLAGS`、`LIBS` 中加入合适的 `-I -L -l` 等选项。

## 运行

具体选项请参阅 `minieap -h` 的输出。这里列出必需的几个选项。

* `-u 用户名`
* `-p 密码`
* `-n 网卡名`

默认的网络帧收发模块是 `if_impl_sockraw`。如果要使用其他模块，如 `libpcap`，则必须指定 `--if-impl libpcap`。

默认不使用任何数据包修改器，将只会发送单纯的标准 EAP 数据包。 **如需使用锐捷认证，则必须指定 `--module rjv3`。** 可以指定多个 `--module` 参数，程序会按参数的顺序让数据包流经这些插件。

参数格式支持如下几种：

* `-u myname`
* `--username myname`

注意：暂不支持 `-umyname` 这种形式，这在插件的命令行解析中将带来错误。

示例：在 en0 上使用锐捷认证，以 `libpcap` 作为网络帧收发模块，并且在数据包流经锐捷认证插件前后都打印出数据包的大小：

```
minieap -u 201000000 -p 15000000000 -n en0 --module printer --module rjv3 --module printer --if-impl libpcap
```

## 注意事项

本项目刚成立不久，虽然有过测试，但无法保证高可靠性。欢迎大家提出意见，谢谢！

非常感谢 HustMoon 工作室以及 Hu Yunrui 同学对这个领域做出的贡献！
