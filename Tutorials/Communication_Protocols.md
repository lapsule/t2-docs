# Tessel 2 外接模块通信说明

Tessel 2 上提供了两个 10-pin 接口来支持模块定制。每个接口上有两个 pin 脚用于供电（一个3.3V，一个接地），另外8个 pin 脚是通用 GPIO（General Purpose Input and Output），可以用来跟主模组进行通信。

所有以上 8 个 pin 脚的通信都可以通过改变电压来控制：有些可以通过 0V～3V 的可变电压来以模拟 pin 脚的方式，有些通过简单的有／无电压（0V／3V，数字 pin 脚）来控制，还有一些根据特定协议的不同，通过关／闭相关的数字 pin 脚实现。这里用到的通信协议（比如 SPI、I2C 和 UART）都是通过定义一系列的 pin 脚用法来实现的，这些用来编解码复杂信息的协议，本质上跟[摩尔斯代码](http://en.wikipedia.org/wiki/Morse_code)是一样的，只不过这些协议是用来进行电子通讯的。


在电子设备中，有 4 种常用的协议， Tessel 都可以支持，并且可以[以 JavaScript 的方式交互](https://tessel.io/docs/hardwareAPI)。

  * [GPIO](#gpio)
  * [SPI](#spi)
  * [I2C](#i2c)
  * [UART](#uart)

本文会简单说明一下这几种协议，并分别介绍一些每种协议的优缺点。

如有兴趣，你可以在这里查到在不同的协议中，[不同的 pin 脚是怎样使用](https://github.com/tessel/t2-docs/blob/master/hardware-api.md#pin-mapping)。

## 快速参考
在设计模块时，通常你会根据你所需功能来选择相应的协议。另外还需考量芯片或者模组是否提供了你需要的 pin 脚、还有所选的协议传输速度等等。下面这个表可以用做一个协议选型的快速参考。

| 协议 	|          # 所需 pin 脚              |          最大传输速率            	|
|:--------:	|:-------------------------------------:|:-------------------------------:|
| GPIO     	|          **1**         	             	| 1kHz                            |
| SPI      	| **3+** (MOSI, MISO, SCK + 1 GPIO pin) | 25MBit/s                        |
| I2C      	|          **2** (SCL and SDA)        	| 100kHz or 400kHz  可选	|
| UART     	|           **2** (TX and RX)           | 8Mbit/s                        	|

## 关于电平位移的说明
以下所有关于通信协议的讨论和图表说明都是假设你的操作电压是 3.3V下，像 Tessel 这样。

如果你的模块是工作在 5V、1.8V 或者其它非 3.3V 的电压下，一定注意了！直接用非 3.3V 的电流直连会损坏你的 Tessel 板子或者你的模块！直接用非 3.3V 的电流直连会损坏你的 Tessel 板子或者你的模块！！直接用非 3.3V 的电流直连会损坏你的 Tessel 板子或者你的模块！！！好了，重要的事情说三遍。

还有个办法，也可以利用 `电平位移` 技术来处理不同的电压。 Sparkfun 有一篇很好的讲解[如何处理电平位移和分压器的文章][level_shifting]，对想了解这部分的人来说是一个很好的起点。

当然，避免此类麻烦最直接的办法是不要用非 3.3V 的设备 ：）

<a name="gpio"></a>
## GPIO
**优点:** 简单，只需占用一个 pin 脚。

**缺点:** 无法发送大量的复杂数据。

目前来讲，通过 GPIO 来做通讯是最简单的方式。 严格讲，GPIO 其实算不上一个 `协议`。它只是你写代码的时候手动控制相关 pin 脚开、关，和读取其电平是高还是低的一种最基本的方式。

Tessel 的 GPIO pin 脚默认都被设置成了输入模式。

[看看那些 pin 脚可以通过这种方式控制](https://github.com/tessel/t2-docs/blob/master/hardware-api.md#pin-mapping)

#### 输入
处于数字输入模式的 GPIO pin 脚可以通过软件的方式读取其当前的 pin 脚状态：高电平（ 1 或者 true）、低电平（ 0 或者 false）。

下面是一个读取 port A 上 pin 2 这个 pin 脚状态的代码片段：

```js
var tessel = require('tessel'); // Import tessel
var pin = tessel.port.A.pin[2]; // Select pin 2 on port A
pin.read(function(error, value) {
  // Print the pin value to the console
  console.log(value);
});
```

GPIO 非常适合用在像[开关, 按键][button_post]这样的简单外设上，像[动作检测][pir_project]这种也可以。但还是要提醒大家注意， Tessel 的 pin 脚只能处理 3.3V 或者以下的外设。

##### 敏感的读者可能很想知道如何监测数字 pin 脚的状态了:

很明确的是，如果该数字输入 pin 脚处在高电平状态，那它肯定是 3.3V，如果是低电平状态，肯定是接地状态。但是，如果该 pin 脚的电压处在这两个值之间，比如是 2V 该怎么监测呢？

大部分同学的第一印象可能是所有超过 1.65V （0 到 3.3V的中间值）的状态都属于高电平，所有低于这个值的都是低电平，听起来很合理吧，但其实不是这么玩儿的。

一个值到底是高电平还是低电平，其实是由处理器说了算。以 Tessel 2 开发板为例，这个事其实是 Atmel SAM D21 决定的，[SAM D21 说明文档](http://www.atmel.com/Images/Atmel-42181-SAM-D21_Datasheet.pdf)写的很清楚，在这颗处理器看来，至少超过 Tessel 板载电压(VDD, 3.3V) 55% 的电压才算是高电平（也就是说 1.815V 或者之上），还规定了板载电压的 30% 或者更低的（VIL）电平信号才算是低电平（0.99V 或者之下）。


![SAM D21 电气特征](http://i.imgur.com/QsfwGSn.png)

<p style="text-align:center"><em>A screen capture from the electrical characteristics section of [Atmel's SAM D21 datasheet](http://www.atmel.com/Images/Atmel-42181-SAM-D21_Datasheet.pdf)</em></p>

那么介于 0.99V 和 1.815V 之间的电平怎么办？事实上，读取这个范围的输入 pin 脚会返回一个未知的非确定值。这就是为什么我们一直强调说要确保连到 Tessel 输入 pin 脚的外设模块的供电要介于 VIH 和 3.3V 或者接地电压到 VIL 之间的原因。

[关于 GPIO 的更多例子和资料见这里][gpio_code_example]

#### 输出 pin
当一个 pin 脚被用作点子输出 pin 时，它可以被设置成高电平 (on/1/true) 或者低电平 (off/0/false) 两种状态。对 Tessel 板子来说，高电平意味着输出 3.3V，低电平代表 0V。

数字输出 pin 对那些可以接受简单的 on/off 信号的硬件来说很有用。接下来的代码片段演示了如何设置 port A 上面的 pin 2 作为电子输出 pin：

```js
var tessel = require('tessel'); // Import tessel
var pin = tessel.port.A.pin[2]; // Select pin 2 on port A
pin.output(1);  // Turn pin high (on)
pin.read(function(error, value) {
  // Print the pin value to the console
  console.log(value);
  pin.output(0);  // Turn pin low (off)
});
```

[简单 LEDs][led_example] 演示了几个使用 GPIO 作为数字输出 pin 的几个例子。用到了一个 [relay][relay_module] 模块来控制开、关状态。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/communication-protocols/320px-5mm_Red_LED.jpg" /></h1>

<p style="text-align:center"><em>数字输出 pin 是控制 LED 的完美方法。本图片基于 [Creative Commons Attribution-Share Alike 2.0 Generic](http://creativecommons.org/licenses/by-sa/2.0/deed.en) 协议.</em></p>

[More GPIO example code and information][gpio_code_example]

<a name="spi"></a>
## SPI
**优点:** 传输速录快，支持一条总线上连接多设备，支持双路通信。

**缺点:** 至少需要 3 根 pin 脚。

SPI 是 [Serial Peripheral Interface][spi_wikipedia] 的简称。SPI 协议通过两条通讯线，允许在 Tessel 和外设模块之间，一次交换 1 byte 的数据。 这尤其合适用在像读写感应器指令这样的场景。

SPI 协议是一种 Master/Slave 协议，也就是说，任何时候总有一个主设备来决定如何跟一台或者多台 slave 设备通信。把 master 设备想象成一位交警，它指导所有的从设备何时开始通信、合适停止通信。

在创建 Tessel 外设模块的时候， Tessel 总是作为 master 设备存在，你创建的其它外设模块以 slave 的方式接入 Tessel。

SPI 协议至少需要 3 根 pin 脚进行通信，一般来讲会用 4 根（除了电源连接之外）。下图说明了 SPI 的连接（箭头说明了数据流向）。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/communication-protocols/Simple_SPI.png" /></h1>

<p style="text-align:center"><em>两条红线组成了跟 slave 设备通信的共享总线。绿色线是 slave 设备返回数据给 master 的共享总线。蓝色的用作给每台外设模块独立的切片选择信号。 </em></p>


#### SCK
时钟信号被用做 Tessel 和外设模组的通信信号同步。两个链接的设备需要有一个彼此都理解的关于传输速录的约定。也就是通常所说的波特率（baud）或者比特率。 时钟信号就是用来给通讯的两个设备提供用于同步交换数据速率的一个参考。

如果没有这个同步时钟信号，相连的设备就没办法理解彼此传过来的数据。

一个时钟周期只能传输一个 byte 的数据（详情见上面的图表）。

#### MOSI
MOSI 是 **M**aster **O**ut **S**lave **I**n 的简写，在 Tessel 里面用来把数据传输给外设模块，Tessel 会把编码后的数据从这个 pin 脚发出去。

#### MISO
MISO 是 **M**aster **I**n **S**lave **O**ut 正好相反，是用来让外设模块发数据给 Tessel 的。

#### SS or CS
这个 pin 脚，也就是通常所说的 **从选择 (SS)** 或者叫 **芯片选择 (CS)** pin，是用来让主芯片通知从设备它什么时候可以开始发送数据的。我们一般管他叫 CS，你可以在 datasheets 或者其它文档中看到。

给 Tessel 设计新的 SPI 外设模块的时候，CS 连接会占用 Tessel port 的一个 pin 脚。


下面的图说明了 SPI 协议中不同的 pin 脚是如何通过频繁开关来传送有用数据的，在这个例子中，主芯片发送了 `S` 的 ASCII 编码，而从设备则回复了一个 `F`。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/communication-protocols/spi_diagram.jpg" /></h1>

<p style="text-align:center"><em>SPI 数据传输时序图。基于 Sparkfun 的 [原图](https://dlnmh9ip6v2uc.cloudfront.net/assets/c/7/8/7/d/52ddb2dcce395fed638b4567.png)修改， [CC BY-NC-SA 3.0](http://creativecommons.org/licenses/by-nc-sa/3.0/)</em></p>

需要强调的一点是，主设备负责初始化所有的通信。初始化完成之后，第一件事就是把 CS/SS pin 脚拉低，以便通知从设备马上准备开始进行数据传输。

在整个数据传输过程中，主设备都会让 CS/SS pin 保持低电平状态。

在保持 CS/SS pin 脚低电平的状态下，主设备会通过频繁的关闭动作在时钟 pin （SCK）上发送数据，同时控制 MOSI pin 脚发送它想发的数据给从设备。

上图中的绿色数字说明了 byte 中的每个 bit 是如何传输的。

看起来很复杂，但其实 Tessel 帮你把这些操作都通通包装了起来，你需要做的仅仅是像下面这个代码段类似的写 javascript，下面的代码演示了如何在 port A 上面使用 SPI 协议。

```js
var portA = tessel.port['A'];
var spi = new portA.SPI({
  clockSpeed: 4000000 // 4MHz
});

spi.transfer(new Buffer([0xde, 0xad, 0xbe, 0xef]), function (err, rx) {
  console.log('buffer returned by SPI slave:', rx);
});
```

[更多 SPI 的例子和资料见这里][spi_code_example]

<a name="i2c"></a>
## I2C
**优点:** 只需要 2 个 pin 脚，一条宗宪可以接多个设备，双向传输数据。

**缺点:** 有可能产生设备地址冲突，没有 SPI 协议那么快。

I2C 是 [Inter-Integrated Circuit][i2c_wikipedia] 的简写，发音是  "I 方 C", "I two C" 或者 "I-I-C"。I2C 协议允许一个设备通过一条数据 pin 和一条时钟 pin 来跟其他一个或多个设备传输数据。

I2C 是一个主从协议，也就是说任何时候都有一个熟设备来控制跟另外一个或多个设备的数据传输。

I2C 仅需 2 个 pin 进行数据传输：

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/communication-protocols/Simple_I2C.png" /></h1>


#### SCL
时钟信号 pin 用来确保 Tessel 和他的外围模块数据传输是同步的。连接的设备之间需要一种确保了解彼此数据传输速度的机制。有时候也被称作 baud 或者 bitrate。该 pin 脚提供的时钟信号为连接的设备如何交换数据提供了一个参考。如果没有该信号作为参考，连接的设备之间将无法正常的在数据 pin 脚上传输数据。

#### SDA
SDA 是主设备和从设备之间用来传输数据的 pin 脚。无需单独为主从设备设立两路单独的数据往来 pin，设备之间可以通过一根 pin 脚实现数据的传输。主设备负责掌控传输顺序，以确保同一时间内只有一个设备在发送数据。

由于多个设备可以共用一条 SDA pin 脚，主控设备需要知道如何区分这些设备，并控制任意时间点痘只有一台设备在发送数据。I2C 协议引入了 **设备地址** 的概念来在该 pin 脚上协调数据的传输。

任何一个通过 I2C 协议接入的设备都有一个内部地址，这个地址对所有接入 Tessel 的外设来说都是唯一的。该地址通常都是由该设备的生产商来决定的。有时候也可以通过设备生产上提供的特殊工具来自己修改该地址。在 Tessel 里面，作为一个主控设备，必须知道所有联入的从设备地址，并可以利用该地址通知相应的从设备，告诉他们什么时候可以开始通信。


<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/communication-protocols/Multi_I2C.png" /></h1>

<p style="text-align:center"><em>Tessel 和 从设备时如何通过 I2C 传输数据的</em></p>

下图说明了 SDA 和 SCL 两个 pin 脚在 I2C 协议中是如何协调发送数据的。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/communication-protocols/i2c_modified_timing.png" /></h1>

要开始传输数据，主设备先通过拉低 SDA pin 脚电平再拉低 SCL 电平的方式来创建一个起始条件。

接着，主设备会把所有它想与之通信的从设备地址广播出去。注意，时钟信号是由很多的 bit 信号组成的，这些 0/1 信号告诉从设备何时、由那台从设备跟主设备通讯。

设置好地址后，主设备通过发送读／写指令来处理是发送数据给从设备，还是从从设备读取数据。

完成地址广播之后，主设备要么马上开始发送数据给从设备，要么发送一个注册过的设备地址，用以接受从该设备发送的数据。

整个过程完成之后，主控设备会发送一个停止信号，也就是先拉高 SCL 的电平，然后再拉高 SDA 的电平。

有一点点的复杂，但这些细节其实 Tessel 已经包装好了，你只需要想下面这样写 JavaScript 就行了:

```js
var tessel = require('tessel'); // import tessel
var portA = tessel.port['A']; // use Port A
var slaveAddress = 0xDE; // This is the address of the attached module/sensor
var i2c = new portA.I2C(slaveAddress)

i2c.send(new Buffer([0xde, 0xad, 0xbe, 0xef]), function (err) {
  console.log("I'm done sending the data");
  // Can also use err for error handling
})
```

[更多 I2C 的例子和资料][i2c_code_example]

<a name="uart"></a>
## UART
**优点:** 广泛的支持，可以双向通信。

**缺点:** 不能共享数据 pin，传输速度比 SPI 和 I2C 都慢。


UART 是 [Universal Asynchronous Receiver/Transmitter][uart_wikipedia] 的缩写，也就是串口。实现也非常容易理解，只需要两个 pin 脚，一个是数据发送 pin ： **TX** ，一个数据接收 pin **RX** ， Tessel 通过 TX 发送数据，通过 RX 接收数据。

### TX
Tessel 通过 TX 发送数据。

### RX
外设模块通过 RX 发送数据给 Tessel。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/communication-protocols/uart_example.jpg" /></h1>

<p style="text-align:center;"><em>基于 UART 的数据传输。</em></p>

UART 数据传输是从某一个 pin 脚（TX 或者RX）被拉低成低电平开始的。然后 5 到 8 个数据 bit 发送过去。上图说明了总共 8 个 bit 的数据是如何传输的。

紧跟着数据，一个可选的[奇偶位](http://en.wikipedia.org/wiki/Parity_bit)被发送出去，接着是 1 到 2 个停止 bit，然后发送端将该 pin 脚电平拉高

为确保该协议可以工作，发送／接收双方需要在以下几点达成一致：

  1. 每个包里面包含多少 bit 的数据（5 到 8）？
  2. 数据传输速率是多少？也就是所谓的波特率。
  3. 是否有奇偶校验位，有的话是高还是低。
  4. 数据传输完成之后几 bit 的停止数据需要发送。

如果你想通过 UART 接入某一个特定模块，那么以上的问题应该都在该模块的 datasheet 里面有说明，根据这些信息，你就可以正确的配置并使用该模组，用 JavaScript 的方式如下：

```js
var port = tessel.port['A'];
var uart = new port.UART({
  dataBits: 8,
  baudrate: 115200,
  parity: "none",
  stopBits: 1
});

uart.write('Hello UART');
```

[更多 UART 的实例和资料][uart_code_example]




[led_example]: http://start.tessel.io/blinky
[relay_module]: https://tessel.io/modules#module-relay
[sd_module]: https://tessel.io/modules#module-sdcard
[climate_module]: https://tessel.io/modules#module-climate
[ambient_module]: https://tessel.io/modules#module-ambient
[accel_module]: https://tessel.io/modules#module-accelerometer
[ble_module]: https://tessel.io/modules#module-ble
[gps_module]: https://tessel.io/modules#module-gps
[spi_wikipedia]: http://en.wikipedia.org/wiki/Serial_Peripheral_Interface_Bus
[i2c_wikipedia]: http://en.wikipedia.org/wiki/I%C2%B2C
[uart_wikipedia]: http://en.wikipedia.org/wiki/Universal_asynchronous_receiver/transmitter
[level_shifting]: https://learn.sparkfun.com/tutorials/voltage-dividers
[spi_code_example]: https://tessel.io/docs/hardwareAPI#spi
[gpio_code_example]: https://tessel.io/docs/hardwareAPI#digital-pins
[i2c_code_example]: https://tessel.io/docs/hardwareAPI#i2c
[uart_code_example]: https://tessel.io/docs/hardwareAPI#uart
[pir_project]: https://projects.tessel.io/projects/pir
[button_post]: https://projects.tessel.io/projects/a-button-on-tessel
[ambient_index]: https://github.com/tessel/ambient-attx4
[audio_index]: https://github.com/tessel/audio-vs1053b
[camera_index]: https://github.com/tessel/camera-vc0706
[sd_index]: https://github.com/tessel/sdcard
[nrf_index]: https://github.com/tessel/rf-nrf24
[accel_index]: https://github.com/tessel/accel-mma84
[climate_index]: https://github.com/tessel/climate-si7005
[rfid_index]: https://github.com/tessel/rfid-pn532
[servo_index]: https://github.com/tessel/servo-pca9685
[ble_index]: https://github.com/tessel/ble-ble113a
[gps_index]: https://github.com/tessel/gps-a2235h
[gprs_index]: https://github.com/tessel/gprs-sim900
[ir_index]: https://github.com/tessel/ir-attx4
