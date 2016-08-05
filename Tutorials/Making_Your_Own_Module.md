## 什么是 Tessel 模块?

>所谓模块就是指有明确功能的外围小设备。也就是说其设计目的应该是单一、目的明确的，或者是功能非常相近也可以。但不能是一个简单的各种板载功能的累加。这个设计思路有助于降低系统复杂性和成本，平衡功耗，并最大化软硬件的复用性。

>*[–Tessel 硬件设计文档](https://tessel.io/docs/modules#module-design-philosophy)*

Tessel 平台的目标之一就是 "让硬件互联变得跟一个 npm install" 一样简单。
如果你需要一个加速器、低功耗蓝牙（BLE）或者需要给硬件增加 SIM 卡，或者其它[已经有官方支持或者 USB 模组支持的功能][modules_page]，你可以直接买一块现成模块，插上，npm instal，然后开始爽！

目前已经有一个社区贡献并驱动的[第三方 npm 方式模块库][third_party_modules]: 就是说这些不同的模组的驱动、使用说明等等都已经打包好，放到正式 npm 库里面了。

但如果现有官方出的或者社区第三方做的模块都不能满足你的需求该怎么办呢？简单！自己做一个！

本文档就是来说明如何给 Tessel 2 的板子上添加一个你需要的自制模块的。

### 鼓励鼓励！

DIY 模组看起来是一个最好能甩给那些知道欧姆定律之类的家伙的任务。如何焊接，还有为什么拿舌头舔一下 9V 会让你感觉不太好... 等等这些问题都让人感觉望而却步。但事实上，只需要简单的了解一些电气基本知识就可以完全胜任这个任务，如果你恰好有会编程，而且还挺聪明很擅长搞一些酷酷的东西出来的话，继续读，一起搞就对了！

## 模块设计基础

在你开始 DIY 硬件模组冒险之旅之前，我们还是先把以下几个基本的概念说清楚比较好。

每个 Tessel 的外设模组都应该包含以下 5 部分：

  1. 电源
  2. 通信能力
  3. 软件
  4. 说明文档
  5. 可分享

了解了这几点之后，我们就可以开始正式进入你的模块 DIY 之旅了！让我们先从电源部分入手。

### 电源
在电子元件的世界里，无论是各种感应器、显示屏、按键还是舵机，首要考虑的都是供电。所有这些都需要供电， Tessel 的外设模组也不能例外。在这个简单的例子中，你可以把电源看成从 Tessel 引出的，一正一负两根线。

Tessel 本身可以由几种不同的供电方式，但无论你选的是那种，最终供到板子上的都是 3.3V 的电。有点类似于 Tessel 的 "电压语言"，它跟所有东西都是只讲 3.3V 的话。

这块原型底板最好的地方之一就是，3.3V 电压和接地被放在了板子的两端，就像下图这样，你可以轻易的给你自己设计的模块选择供电。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/diy-module-guide/diymod_power.jpg" /></h1>

<p style="text-align:center;"><em>原型底板上，电源被分在了两头</em></p>

### 特别考量
如果你自制的模组上，所有的元器件都是工作在 3.3V，那么你的模组设计就会非常简单。你要做的就是简单的把电源两根线接上，然后把相应的引脚接上就可以了(比如像底下介绍的[自制显示模组](#screen_example)）。但更多的时候，你面临的是很多非 3.3V 电压的元器件，比如像[舵机模组][servo_module]这个示例。

常见的舵机都是工作在 5V 电压。这是它们的 “电压语言”，所以这点跟 Tessel 的 ”电压语言“ 接不上，硬接的话会导致舵机无法正常工作。舵机耗电一般都比较多，有可能会占用 Tessel 大部分的电源供应。为解决这个问题，在舵机模块边上我们增加了一个 5V 的电源接口用来外接供电。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/diy-module-guide/servo_jack_identified.jpg" /></h1>

<p style="text-align:center;"><em>舵机模组上的外接供电口</em></p>

从[舵机模块文件][servo_schematic]上看, 多节模块是通过 I2C 接口完成通讯的，提供的电压是 3.3V，但舵机模组用的是通过 J2 接口接入的 5V 电压。

本文档并不像在电源部分纠缠不清，但必须再次强调一下，如果你的模组需要外接电源，那么你必须用到[电压切换][level_shifting]。为了简化模块设计，我们强烈建议你使用 3.3V 电压。

### 关于电源的几个警告
以下是几个跟电子元器件打交道时，涉及到电源需要注意的点。

  * 插拔 Tessel 或者外设模块之前 ***一定要*** 先断电。
  * 不要随便提高供电电压，除非你明确的知道自己在做什么，比如假设你直接给 Tessel 接入 5V 电压极有可能把它烧毁。
  * 绝对不能直接把电源正极接地。这会造成[短路](http://en.wikipedia.org/wiki/Short_circuit)，从而验证破坏你的板子和心情。
  * 接入 Tessel 之前一定先想清楚并小心操作。

## 通讯

一旦决定好了如何给自有模组供电，接下来就需要搞定模组和 Tessel 之间的通讯问题了。

在互联网的世界里，有标准的 HTTP、HTTPS、FTP 等等这些现成的协议供不同的系统之间完成通讯。其实在硬件的世界里也同样存在着类似的协议， Tessel 支持以下 4 种比较常见的协议用来跟模组通讯。

  * [GPIO][comm_gpio]
  * [SPI][comm_spi]
  * [I2C][comm_i2c]
  * [UART][comm_uart]

因为 Tessel 把大部分[协议设计的累活儿都干了][hardware_api]，所以对你来说在设计自己模块的时候不太需要过于关注这部分，但如果你喜欢更深入的钻研的话，这里有一份[关于这4个协议在 Tessel 实现的简单说明文档](https://tessel.io/docs/communicationProtocols).

### 那么，到底哪种协议是我需要的呢？
既然有 4 种协议可选，那到底哪个才是最适合我们模组的呢？好消息是这个问题基本取决于你要做的是那类的模组。比如说，对[红外探头][pir_project] 模组来说，当它检测到有物体移动时会拉高一个 pin 叫的电压，这可以让你很容易的通过 GPIO 方式获取，类似的例子对各种传感器基本都适用，比如在[气候传感模组][climate_module]上的 Si7020 温湿度传感器就是通过 [I2C 协议][comm_i2]通讯的。一般来讲，像传感器这样的硬件基本都只支持一种协议，所以问题简单了，用就行了。

除此之外，还有些硬件是同时支持 SPI / I2C 接口的，这种随便选哪个都行。我们的建议是能选 SPI 的就先用 SPI，留着 I2C 可以给其他可能的扩展。

## 软件

上面电源和通讯的问题搞定之后，接下来就可以开始用 JavaScript 来跟模块通讯了。这就是 Tessel 作为开源硬件的真正魅力所在。我们在[官方模组][modules_page]中用到了所有的这 4 种协议，并在 [github 中开源][tessel_github]了出来，任何人都可以拷贝、编辑、使用。

为自己的模组设计好一个完整的 API 可以帮助别人更好的把模组集成到他们的工程中。第一原则是尽可能的让接口简单明了，第二原则是在第一原则的基础之上尽可能多的暴露更多接口。你可以在[为 Tessel 提供定制模块并编写 Tessel 风格的文档][third_party_doc]一文中找到更多的有用信息。

## 文档和分享

当你完成上面的几步之后，接下来就可以让更多的人了解你的成果，让更多的人使用你的模组了。我们推荐你做以下几步来让社区共享你的劳动成果。保持第三方模组和官方模组的产品、文档风格一致是一件对所有人都很重要的事。

### 创建一个 Git 库
把你的代码放到一个 git 库中，让更多的人可以访问到，并加入到他们的工程中去，为此我们准备了一个实例库，你可以从 fork 这个示例库开始创建你自己的 git repo.

[第三方模组软件 git 库示例][repo_template]

### 写文档
可能你创建了世界上最牛掰的 Tessel 模组，但如果没有文档说明怎么用，没人了解怎么办？当你搞定了所有的硬件、软件和 API 之后，写一份明了的文档告诉别人怎么使用就变成了一个非常重要的任务。这事儿可以从开始创建一个包含如何安装使用、简单示例和 API 说明的 [README][readme_template] 开始。

### 在 NPM 中发布你的代码
上面的步骤都完成之后，接下来就可以完成 Tessel 理念里面的 "让硬件连接像 npm install 一样简单"！ 如果你之前没有向 NPM 提交过代码，可以从下面几行命令开始：

    npm set init.author.name "Your Name"
    npm set init.author.email "you@example.com"
    npm set init.author.url "http://yourblog.com"

    npm adduser

以上几行命令设定了你在 NPM 的作者信息。接下来就是创建你的 package.json 文件了，这个文件应该放在你的代码库目录下，但我们推荐你先用 npm init 命令，在工程目录下生成它。

```
npm init
```

编辑 package.json 文件里面的相关信息，比如你的名字、软件版本号、软件说明等等。在关键词一栏里，我们推荐加上 "tessel"，这样其他的 Tessel 用户就可以很方便的通过 tessel 关键词检索到你的模组信息。 package.json 的结构很清晰，遵循了[npm package.json 标准][package_json_standard]， 并且加入了 **硬件** 部分。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/diy-module-guide/package_hardware_section.png" /></h1>

<p style="text-align:center;"><em>package.json 里的 **硬件说明部分** </em></p>

这是一个 Tessel 特定项，你需要手动添加一下，里面列出来了一个文件和文件夹的清单，在你提交到 NPM 的时候这些文件和文件夹不会被提交。

创建好 package.json 之后，就可以正式发布了。在项目的根目录下执行下面的命令

    npm publish ./

### 创建一个工程项目主页
[Tessel 工程项目页][tessel_projects]是一个向 Tessel 社区分享项目的好方式。只需要简单的提供几点信息、一张图片或者直接用你的 readme.md 文件作为内容也行。

### 提交你的模组
我们会一直很欢迎第三方提交模组到[第三方模组列表 ][third_party_modules]，所以，如果你想让你的模组出现在 Tessel 的模组列表页 [tessel.io/modules][modules_page]，那就赶紧去填个表吧，我们非常高兴可以早日看到它。

[第三方模组提交表单][module_submission]

[为 Tessel 提供定制模块并编写 Tessel 风格的文档][third_party_doc] 里面详细的描述了以上过程。


## 第一个 DIY 模组示例
到目前为止，我们已经知道了如何定制一个模组，让我们按照这个思路来定制一个简单的示例。在 Tessel 板子上有一个简单的按钮，但或许你会想用模组的方式添加更多，Kelsey 已经做了一篇非常好的[通过 GPIO  添加按键][orig_button_project_page]的文档，接下来让我们按照她的说明做一个出来。

### 电源部分
你可能觉得一个按键不需要电源，部分正确，按键本身确实不需要电源，我们用按钮来控制电源从而在 GPIO 上面产生不同的高低电平。

在 Tessel 上，所有没有连接的 GPIO 都是高电平，因为在 Tessel 内部，主芯片把这样的所有引脚都输出了 3.3V 正电压。

另一个电源线是接地，我们把它街道按键的另一个引脚上。接那个脚没有要求，因为按键部分左右，只是负责电流的开断。

### 通讯部分
如上面提到的，通讯部分是由你的模组决定的。在按键这个例子中，我们选用 GPIO，因为我们希望读取按键的状态。

Tessel 的两个 port 上都有不少这样的[数字 I/O pin 可以完成这个任务 ](https://tessel.io/docs/hardwareAPI#pin-mapping)，你可以随便选两个你喜欢的。

在这个例子中，我们选 port A 里面的 pin 1，我们会把这个 pin 连到按键的一段，在按键没有被按下时，该 pin 脚的状态是高电平，或者是 true，当我们按下这个按键，就是给该 pin 脚接地了，然后状态就变成了低电平，正如我们设计所想。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/diy-module-guide/switch_schematic.png" /></h1>

<p style="text-align:center"><em>按键连线图</em></p>

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/diy-module-guide/button_module_angle.jpg" width=300 /></h1>

<p style="text-align:center"><em>焊接好的模组板</em></p>

不要被焊接板吓到，这种简单的工序唯一有挑战的就是如何使用热喷胶枪，但这也是小菜一碟。

[这篇 Sparkfun 来的帖子](https://learn.sparkfun.com/tutorials/how-to-solder---through-hole-soldering) 是一个学习焊接的很好切入点。

让我们继续，把模块插上，**注意！！绝不要在 Tessel 上电的时候插拔模块！**

### 软件部分
以上步骤都完成之后就可以开始 coding 了。

事实上我们决定直接用 Kelsey 的代码，稍作改动，因为她遵循了编码风格并且把代码放到了 NPM 上，我们其实不需要写什么，更好的是她甚至还在文档中提供了一份[快速指引说明](https://github.com/Frijol/tessel-button#quick-start-i-just-want-a-button)，让我们直接用就好了。


  1. 安装 tessel-gpio-button 包，这样我们就可以复用 Kelsey 的成果了。

    ```npm install tessel-gpio-button```

  2. 创建一个名字类似 **myButton.js** 这样的文件，并把她在快速说明文档中提到的部分拷贝进去，差不多像这样：
```js
    // examples/button.js
    // Count button presses

    var tessel = require('tessel');
    var buttonLib = require('../');
    var myButton = buttonLib.use(tessel.port['GPIO'].pin['G3']);

    var i = 0;

    myButton.on('ready', function () {
      myButton.on('press', function () {
        i++;
        console.log('Press', i);
      });

      myButton.on('release', function () {
        i++;
        console.log('Release', i);
      });
    });
```

这段代码可以执行的很好，但还是有两个地方我们可以继续优化，看出来了吗？

首先，我们不要用 "require('../')" 这种方式引入包，更好的方式是直接使用包名: require('tessel-gpio-button')。

第二，代码中她是用的 Tessel 1 的 G3 这个 pin，我们模块上焊接的是 Port A 里面的 pin 1，所以代码需要稍微改一下，需要改的也就是 _myButton_ 的定义部分，改成这样：


```js
var myButton = buttonLib.use(tessel.port['A'].pin[1]);
```

保存，运行试试。

```
tessel run myButton.js
```

每次你按按键的时候都应该有 log 输出。

恭喜你！你已经完成了你的第一个 Tessel 外设模组的设计！

### 文档和分享
上面第一个模组严格来讲其实不能算是数，因为 Kelsey 已经创建好了 NPM 包我们可以拿来重用，尤其是我们不需要写文档，也不需要做软件分享。这没什么不对的，写的代码越少越好，下面是一个很好的例子说明如何写文档、如何在社区中分享代码。

我们可以通过创建[一个项目页面][button_project_page] 的方式说明我们该如何把 Kelsey 的按键模组推进到下一步，做成插件模组。我们做出来了模组，接下来我们应该写好相关文档让其他需要的人来使用它。


[自定义按键模组实例页面][button_project_page]

<a name="screen_example"></a>
## 自定义屏幕模组
上面这个简单的小项目我们已经有点儿经验了，是时候进入到下一步。目前大家最迫切的是想要一个屏幕的模组。显示屏之所有有点复杂是因为有好多种方式可选，包括 7 段数码管、LCD 显示、OLED 显示、电阻触摸屏，电容触摸屏等等，显示模组是一个很好的拿来自定义的模组。

在嵌入式项目中，比较有名的是 Nokia 5110 的屏，得益于该屏幕的简单交互特性，和低成本，让我们看看如何按照上面的流程来创建一个显示模组。

我们用[Sparkfun 的 Nokia 5110 拆机屏][nokia_sparkfun]来做示例，当然你也可以用 [Adfruit 版的屏幕][nokia_adafruit] 或者[从 Ebay 上买一个][nokia_ebay]也可以。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/diy-module-guide/nokia5110.png" /></h1>

<p style="text-align:center"><em>Nokia 5110 LCD 显示屏</em></p>

### 电源部分
5110 可以支持从 2.7V 到 3.3V 的电压，也就是说只要在这个区间的所有电压都可以驱动该屏幕。因为 Tessel 的 port 已经有 3.3V 输出，所以我们不需要再为了挂接它而做额外的工作。我们需要做的就是把屏幕的 VCC或者正极电源接到相应的正极，GND 接地即可。

由于屏幕尺寸的问题，我们会用到两个 port 来接这个模块，虽然只用一个 port 就可以把所有东西接好。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/diy-module-guide/doublewide_module.jpg" /></h1>

<p style="text-align:center"><em>双面模组</em></p>

### 通讯部分
像按键模组一样，通讯部分也无需费心选择， Nokia 5110 用了一个简单修改过的 [SPI][comm_spi] 协议来完成跟 Tessel 之间的通讯，我们直接用就好。

相比普通的 SPI 协议，5110 多一个 **D/C** pin 脚，用来区分屏幕我们发送过去的数据是普通的 SPI 命令还是实际的显示数据。

这个 D/C pin 脚是用简单的高低点评来控制的，用 GPIO 完全可以胜任。

下表里面说明了所有的通讯细节，和我们该如何对接到 Tessel 的 port 上去。

| Nokia 5110 Pin | 示例模块接法                               |
|----------------|------------------------------------------|
| SCE            | G1                                       |
| RST            | 通过一个 10K 电阻连到 3.3V 上去              |
| D/C            | G2                                       |
| MOSI           | MOSI                                     |
| SCLK           | SCK                                      |
| LED            | 通过一个 330 ohm 的电阻连到 G3              |

#### 设计部分
Nokia 5110 有 4 个连接可以直接用 GPIO 来接，D/C 和 SCE pin 需要用来获取显示数据。连完后就剩下一个 GPIO 和一个 RST 和 LED 了，有这么几个可选的办法。

  1. 用一个 10K 的把 RST 和 3.3V 连接，可以保护你避免在代码中重置屏幕，并可以通过剩余的 GPIO 来控制屏幕背光。
  2. 用一个 330 ohm 的电阻连接 LED 和 3.3V 可以保持背光常亮，这样做可以留一个 GPIO 口，以便在 JavaScript 里面用来重置屏幕。
  3. 因为我们用了两个 port，所以你可以用临近的 port 上的 GPIO 来接 LED 和 RST。
  4. 把片选 pin （SCE）接地，空出来一个 GPIO 可以用来控制 LED 和 RST，让片选 pin 保持低电平。这样确保 **没有其他的 SPI 设备可以同时连到 Tessel 上来**

我们确定采用第一个方案，因为其实基本不需要在代码中 reset 屏幕，而且这个方案允许我们通过那条 GPIO 来控制背光，这是自定义模组的另一个好处，你可以自有决定这个模块如何适应你的项目。

我们按照 [Graphic LCD Hookup Guide][screen_hookup_guide] 的指引把所有部件连接起来。然后，在你做焊接之前，最好是能通过一片[面包板](https://learn.sparkfun.com/tutorials/how-to-use-a-breadboard) 测试一下，确保模组是按照你希望的方式工作的。

焊接完成之后模组应该长成这个样子。

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/diy-module-guide/screen_soldered.jpg" /></h1>

<p style="text-align:center"><em>Nokia 5110 soldered to a large proto-module board</em></p>

### 软件部分
连上屏幕线之后就可以开始写代码了。我们遵从 [Git Repo Template][repo_template] 的规则，创建一个名字为 **tessel-nokia5110**  的文件夹，进到这个而文件夹，然后运行 `t2 init` 来创建 **index.js** 文件，在该文件中我们会按照 [the example index.js template][index_template] 的指引来完成我们的 API 调用。

Nokia 的这个模组非常的流行，网上有大量的示例代码和相关库我们可以拿来交互，所以没必要重新发明轮子，我们要做的是，用 JavaScript 来控制这块屏幕。

我们拿 [simple Arduino library][screen_arduino_code] 来控制这个屏幕，并把它改造成 [支持 JavaScript 的版本][screen_github]。我们的 API 非常简单，只暴露了一个时间和几个方法。

#### 事件处理
Nokia5110.**on**('ready', callback(err, screen)) - 当屏幕对象被初始化完成之后调用。

#### 方法处理
Nokia5110.**gotoXY**(x,y,[callback(err)]) - 把游标设置到 (x, y) 位置。

Nokia5110.**character**(char, [callback(err)]) - 写一个单字符到屏幕上。

Nokia5110.**string**(data, [callback(err)]) - 写一个字符串

Nokia5110.**bitmap**(bitmapData, [callback(err)]) - 从 _bitmapData_ 取一个单色图来显示

Nokia5110.**clear**([callback(err)]) - 清屏

Nokia5110.**setBacklight**(state) - 如果 _state_ 为 true 则打开屏幕背光，否则关上。

### 文档
现在模组已经可以正常互联并工作了，是时候完善一下相关文档了。

关于文档我们已经强调过几次其重要性了，而且只需要花费几分钟就可以搞定。不管你自己认为多么微不足道的地方，别人都可能因为不清楚而无法使用，所以赶快把文档完善起来吧！

在当前这个例子中，我们会拿 [README.md 文件][readme_template] 为例，[为我们的 API 创建一个结构良好的文档][screen_github].

接下来，我们会创建一个 **examples** 文件夹来说明如何使用这个模组

### 共享
是时候跟世界共享一下你的成果了：

  * 创建一个[Git 仓库并在线上传相应代码][screen_github]
  * [提交模组到 NPM][screen_npm]
  * [为自己的模组创建一个项目工程页][screen_project_page]
  * [Submitting][module_submission] it to the [third-party module list][third_party_modules]

<h1 style="text-align:center;"><img src="https://s3.amazonaws.com/technicalmachine-assets/tutorials/diy-module-guide/screen_connected.jpg" /></h1>

<p style="text-align:center"><em>完成的显示模组</em></p>

### 资源列表
以下资源可以帮你更快的开始定制自己的模组。

#### 电源
  * [Tessel 的电源][power_options]
  * [电压切换][level_shifting]

#### 通讯
  * [Tessel 模组通讯协议][comm_protocols]

#### 软件
  * [为 Tessel 提供定制模块并编写 Tessel 风格的文档][third_party_doc]
  * [Tessel 硬件 API][hardware_api]
  * [All first-party module code on Github][tessel_github]

#### 文档
  * [Git 软件仓库模板][repo_template]
  * [README.md 模板][readme_template]

#### 共享
  * [发布到 NPM][npm_tutorial]
  * [package.json 标准定义][package_json_standard]
  * [Tessel 项目页面][tessel_projects]
  * [第三方模组提交表单][module_submission]

[modules_page]: https://tessel.io/modules
[third_party_modules]: https://tessel.io/modules#third-party
[tessel_github]: https://github.com/tessel
[repo_template]: https://github.com/tessel/style/tree/master/Templates
[index_template]: https://github.com/tessel/style/blob/master/Templates/index.js
[package_json_standard]: https://www.npmjs.org/doc/files/package.json.html
[npm_tutorial]: https://gist.github.com/coolaj86/1318304
[readme_template]: https://github.com/tessel/style/blob/master/module_RM_template.md
[hardware_api]: https://tessel.io/docs/hardwareAPI
[tessel_projects]: https://projects.tessel.io/projects
[module_submission]: https://docs.google.com/forms/d/1Zod-EjAIilRrCJX0Nt6k6TrFO-oREeBWMdBmNMw9Zxc/viewform
[level_shifting]: https://learn.sparkfun.com/tutorials/voltage-dividers
[power_options]: https://tessel.io/docs/power
[servo_module]: https://tessel.io/modules#module-servo
[servo_schematic]: http://design-files.tessel.io.s3.amazonaws.com/2014.06.06/Modules/Servo/TM-03-03.pdf
[climate_module]: https://tessel.io/modules#module-climate
[pir_project]: https://projects.tessel.io/projects/pir
[button_project_page]: https://projects.tessel.io/projects/button-proto-module/
[orig_button_project_page]: https://projects.tessel.io/projects/a-button-on-tessel
[comm_protocols]: https://tessel.io/docs/communicationProtocols
[comm_gpio]: https://tessel.io/docs/communicationProtocols#gpio
[comm_spi]: https://tessel.io/docs/communicationProtocols#spi
[comm_i2c]: https://tessel.io/docs/communicationProtocols#i2c
[comm_uart]: https://tessel.io/docs/communicationProtocols#uart
[third_party_doc]: https://github.com/tessel/docs/blob/master/tutorials/make-external-hardware-library.md
[nokia_sparkfun]: https://www.sparkfun.com/products/10168
[nokia_adafruit]: https://www.adafruit.com/products/338
[nokia_ebay]: http://www.ebay.com/sch/i.html?_from=R40&_trksid=p2050601.m570.l1313.TR6.TRC1.A0.H0.Xnokia+5110&_nkw=nokia+5110&_sacat=0)
[screen_arduino_code]: http://dlnmh9ip6v2uc.cloudfront.net/datasheets/LCD/Monochrome/Nokia_5110_Example.pde
[screen_github]: https://github.com/sidwarkd/tessel-nokia5110
[screen_npm]: https://www.npmjs.org/package/tessel-nokia5110
[screen_hookup_guide]: https://learn.sparkfun.com/tutorials/graphic-lcd-hookup-guide
[screen_project_page]: https://projects.tessel.io/projects/nokia-5110-graphic-lcd-proto-module
