## 前言

据说不会上位机和游戏开发，都不好意思说自己会 C#

正好这俩我都不太会😂

这不来点一下上位机的技能树

这次的需求很简单，用 C# 模拟一个设备协议，实现不用去现场对接设备，也能先开发和调试上位机程序。

实际设备是用 RS-485 标准进行通信，模拟跑通之后，到现场只需要把RS-485 总线（A/B 差分线）插到 USB-RS485 转换器上就可以实现数据读取和指令下发了。

> PS: 最近把我的博客小重构了一下，欢迎访问体验一下: [http://blog.deali.cn/](https://github.com)

## 先放一些截图作为预告

本文要介绍只是最基础的前期工作

实际上这个项目要实现的是一个简单的物联网平台，不只是对接几台设备

系统的初版已经完成了，这里我先放几张截图

[![](https://img2024.cnblogs.com/blog/866942/202508/866942-20250826101840500-422112702.png)](https://github.com)

实时图表

[![](https://img2024.cnblogs.com/blog/866942/202508/866942-20250826101847642-405830855.png)](https://github.com)

设备控制

[![](https://img2024.cnblogs.com/blog/866942/202508/866942-20250826101854742-1820558714.png)](https://github.com)

就这几个吧，其他的还不是很完善

## 前提

OK 说回正题，模拟串口设备需要的前提是这些

* 首先已经拿到了详细的设备协议文档

  这个很关键，谁也没法摸黑去开发呀
* 操作系统: Windows/Linux

  很神奇吧，Linux居然也能开发上位机？事实上 Linux 模拟设备更方便

  不过为了方便开发调试，我这里还是以 Windows 系统为例

## 串口驱动

Windows 上模拟串口驱动: com0com

这个工具可以在系统里创建一对连通的 com 串口，比如 com3 <-> com4

在任何一端发信息，另一端都可以读取

我们就是用这个方式来模拟串口设备

PS: com0com 的图形界面需要安装 net framework 3.5 老古董才能用，我直接用命令行

> Linux的话可以使用 tty0tty
>
> [https://github.com/freemed/tty0tty](https://github.com):[wgetcloud全球加速器服务](https://wgetcloud6.org)

## 串口调试工具

串口调试工具开源的有很多

我这次试用了 llcom 和 Wu.CommTool

推荐 llcom，使用比较直观

项目地址: [https://github.com/chenxuuu/llcom](https://github.com)

可以直接在命令行安装

```
winget install llcom
```

界面长这样

[![](https://img2024.cnblogs.com/blog/866942/202508/866942-20250826101906223-1753292237.png)](https://github.com)

## com0com常用命令

前面说了 com0com 的图形界面需要安装 net framework 3.5

我肯定是不想安装这种老古董来污染我的电脑环境的

好在还有命令行可以用

这里列一些常用命令

### 查看当前有哪些虚拟串口

```
list
```

输出会显示每一对虚拟串口，例如：

```
CNCA0 PortName=COM5
CNCB0 PortName=COM6
```

这说明有一对虚拟串口：`COM5 <-> COM6`。

### 创建一对新的虚拟串口

```
install PortName=COM5 PortName=COM6
```

这会创建一对虚拟串口，分别命名为 `COM5` 和 `COM6`，它们互相连通。

👉 以后就可以让：

* 模拟器程序 监听 `COM5`
* 上位机/主程序 打开 `COM6`

这样它们互相通信，等同于 RS-485 设备在现场。

### 删除一对虚拟串口

```
remove 0
```

删除标识符为 `CNCA0` 和 `CNCB0` 的那一对（0 是编号，可以从 `list` 查到）。

### 修改已有端口的参数

比如要修改 `CNCA0` 的端口号：

```
change CNCA0 PortName=COM7
```

### 清理所有虚拟串口

```
uninstall
```

⚠️ 注意，这会把所有 com0com 的虚拟端口全删掉。

## 开发流程

1. 创建一对虚拟串口：

   ```
   install PortName=COM3 PortName=COM4
   ```
2. 编写 **模拟器程序**（C#），监听 `COM3`。
3. 上位机程序/串口调试助手连 `COM4`，输入指令，收到模拟器的返回

PS: 创建串口后在设备管理器可以看到

## 串口通信程序

用 C# 自带了 `System.IO.Ports` 工具，可以很方便实现串口通信，难怪那么多人用 C# 开发上位机

不过在 .NET Core 时代，这个库需要通过 nuget 安装

```
dotnet package add System.IO.Ports
```

这里我写了一个简单的串口模拟程序

```
using System.IO.Ports;
using System.Text;

Console.WriteLine("=== 协议模拟器 ===");

// 打开虚拟串口 (比如 COM5)
const string portName = "COM5";
var port = new SerialPort(portName, 9600, Parity.None, 8, StopBits.One);
port.Encoding = Encoding.ASCII;
port.Open();

Console.WriteLine($"模拟设备已启动，监听 {portName}...");

port.DataReceived += (s, e) => {
    try {
        var cmd = port.ReadExisting();
        Console.WriteLine($"收到: {cmd}");

        string response;

        // 协议模拟逻辑 (这里举例)
        if (cmd.Contains("temp", StringComparison.OrdinalIgnoreCase)) {
            // 模拟返回温度
            response = "01,temp=25.6\n";
        }
        else if (cmd.Contains("hum", StringComparison.OrdinalIgnoreCase)) {
            // 模拟返回湿度
            response = "01,hum=60%\n";
        }
        else {
            // 默认回应
            response = "01,ack\n";
        }
    }
    catch (TimeoutException) {
        // 超时继续监听
    }
    catch (Exception ex) {
        Console.WriteLine($"错误: {ex.Message}");
    }
};
```

## 实现效果

使用串口调试工具发送指令，C# 写的模拟程序这边收到后就返回响应了

[![](https://img2024.cnblogs.com/blog/866942/202508/866942-20250826101920183-1948291216.png)](https://github.com)

## 小结

IT寒冬什么的已经被说了好多次了

显而易见的，互联网的发展空间基本到头了，这俩年火热的AI也只是缩减了一批低端岗位而已，并不能把蛋糕做大

但换个角度看，正因为互联网不再是蓝海，才让我们重新注意到那些“传统”却始终不可或缺的领域。上位机开发就是这样一个方向。它不像移动互联网那样卷，但在工业控制、科研实验、自动化测试等场景里却有着稳定而长期的需求。无论是实验室里的一台设备，还是生产线上成百上千台 PLC，最终都需要一个可靠、可视化的上位机来管理和监控。

对入门者来说，C# 提供了友好的语法和强大的生态，足够快速地做出第一个能跑的 Demo —— 一个串口助手、一个数据采集可视化界面，甚至是一个小型的测试管理系统。随着学习深入，还可以接触到 Modbus、CAN 总线、OPC 等更复杂的协议，逐渐走向真正的工业应用。

未来的趋势不会停在“传统上位机”上。跨平台框架（.NET MAUI、Avalonia）、前后端融合（C# + Web 技术），甚至 AI 辅助的数据分析，都可能成为上位机开发的新方向。换句话说，这条路并不狭窄，它只是需要你把眼光从“卷互联网”转向“深耕行业”。

所以，如果你正处在迷茫期，不妨先从一个简单的上位机小项目开始做起。哪怕是一个串口监控工具，都可能成为你进入这个领域的第一块敲门砖。
