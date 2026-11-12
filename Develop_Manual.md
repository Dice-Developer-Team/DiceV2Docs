# Dice!嵌入模块开发手册

## （原Shiki Plugin Manual）

## 简介

本手册是对Dice!2.4.2(build570)新增的自定义指令功能、Dice!2.5.1(build577)新增的自定义任务、Dice!2.6.0支持调用Lua的关键词回复、Dice!2.6.4(build612)支持的事件处理、Dice!2.7.0(build640)支持的JavaScript与Python脚本等所作的说明，**当前对应最新版本Dice!2.7.1(670)**。通过在mod目录安装扩展模块（lua/js/py脚本与toml配置均可），或使用指令录入，或在plugin目录放入lua脚本，Dice!将监听特定消息或事件并做出响应，也可基于时刻或循环处理定时任务，从而实现骰主对骰娘的深度化定制。扩展模块旨在以下方面实现对Dice!内建功能的补充：

1. 过于客制化而无法使用现有数据结构表现的功能，如：基于D20结果的数值分段回复；
2. 同人性质或版本各异的规则，如：JOJO团、圣杯团、骑士团、方舟团等；
3. 因受众过偏不适合写入骰娘源代码的规则；
4. 客制化的非骰娘功能，如：好感度系统；

从零编写脚本需要对Dice所支持脚本语言（lua/javascript/python）的其中一种有一定的了解，**建议使用VS Code等工具编辑脚本**。如果实在脚本苦手，手册附录准备了现成的样例脚本，你也**可以在Shiki官方群或[论坛](https://forum.kokona.tech/)内找到Shiki免费发布共享的脚本**。Shiki的桌游指令集已收录：DND检定、ShadowRun、双重十字、永夜后日谈（命中检定）、祸不单行、山屋惊魂。**如果你只要安装现成模块，只需记住在webui界面【模块管理】子标签【远程资源】下选择安装；或手动将mod文件放入mod文件夹后启动或.system load。**如果你实在有想实现的指令，没办法修改现成脚本实现，也确实找不到人白嫖的话，可以**将本手册交给Agent阅读**。愿所有人能从Dice!骰娘处获得更好的用户体验，拥有自己独一无二的骰娘。

Shiki的开发官方群:【一群】928626681【二群】1029435374

<div align="right">——安研色Shiki</div>

## 版权声明

**Dice!使用AGPLv3许可**，如果你将Dice!与Lua脚本结合用于分发或通过网络提供服务，那么Dice!与所搭载脚本将作为整体使用AGPLv3许可，任何接收者或用户基于AGPLv3许可，均完全享有获取并公开完整代码的自由。

同样地，本手册使用AGPLv3许可，如果您通过付费方式获取到本手册或本手册附带的Lua脚本，那么您显然付出了不必要的代价。

**特别地，若您将Dice!源代码从C++重写为其他语言，这种翻译行为属于著作权法意义上的修改，因此您翻译的代码一旦分发或用于网络交互，也强制使用AGPLv3协议且必须保留Dice!开发者的署名。AGPLv3文本格式参考：**

```lua
--[[ 
Copyright (C) 2018-2022 w4123溯洄
Copyright (C) 2019-2026 String.Empty
This program is free software: you can redistribute it and/or modify it under the terms of the GNU Affero General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License along with this program. If not, see <http://www.gnu.org/licenses/>. 
]]
```

## 开始之前：三条路径与目录总表

Dice!允许用三种方式嵌入自己的功能，**它们的语言支持和生效方式不同**，先选对载体，再谈怎么写：

| 载体 | 位置 | 支持语言 | 适用场景 | 生效方式 |
| --- | --- | --- | --- | --- |
| mod模块 | `存档目录/mod/mod名.json` + 同名文件夹 | lua / js / py / toml / yaml / xml | 要发布、热插拔、带规则书或角色卡模板 | `.mod on/off/reload`，初次加载自动开启 |
| 直接设置 | `.reply set`指令 / WebUI | lua / js / py语句 | 临时试做、单条回复 | 立即生效 |
| plugin脚本（最屎山最草台） | `存档目录/plugin/*.lua` | **仅Lua**，另支持`*.toml`写reply | 只想给自家骰娘加几条指令，不打算发布 | `.system load`，改内容保存即生效 |

**先分清这两个目录**（下文统一用这两个称呼）：

- **程序目录**：`DiceDriver.*.exe`所在目录（一键包中通常即根目录）。
- **存档目录**：`程序目录/Dice<骰娘账号>/`，每个骰娘一份，**本手册说的"存档目录"都指这里**。

```
程序目录/
|-- DiceDriver.*.exe
|-- data/image、data/record   ← mod的image/audio会复制到这里，可按相对路径填CQ码
|-- Diceki/lua、Diceki/py、Diceki/js  ← 公共库：lua的require路径、python的模块路径、js的loadJS路径
|-- python312/                ← Python运行时（含标准库目录或python312.zip）
|-- Dice<骰娘账号>/           ← 存档目录
    |-- mod/                  ← 模块：mod名.json + mod名/
    |-- plugin/               ← 脚本：*.lua（不会自动创建）
    |-- selfdata/             ← 脚本自建数据（getSelfData读取）
    |-- conf/、user/          ← 配置与用户数据，脚本一般只读
```

指令脚本最短的跑通方式：在`存档目录/plugin/hello.lua`写入下面内容，`.system load`后在群里发`hello`。

```lua
--/plugin/hello.lua
msg_order = {
    ["hello"] = function(msg)        --值可以是函数，也可以是同文件内的函数名
        return "hello,world"
    end,
}
```

若这一句没有回应，先确认三件事：文件在`plugin/`下且后缀为`.lua`；`.system load`的回执里能看到读取到的脚本数与指令数；控制台/通知窗口没有Lua报错。

## 脚本语言嵌入说明

Dice!在更新过程中支持嵌入Lua(Dice!2.4.2)、Python(Dice!2.7.0)、JavaScript(Dice!2.7.0)脚本，包括执行语句或执行文件。后文中的(build+)表示特性的最低支持版本。

三种语言共用一条兼容性判定规则：

> **值是字符串，且能在`script/`目录下找到同名文件（不含后缀）→ 当作文件名；否则当作语句执行。**

同一份逻辑因此有四种写法：`.reply set`里直接写语句、`.reply set`里写文件名、mod的`reply/*.toml`里写文件名、mod的`reply/*.lua`里直接写函数。下文按语言分别举例。用`.reply set`设置时，各字段之间以**换行或Tab**分隔，值里可以有逗号、等号和引号。

### Lua嵌入方式

#### ①在脚本文件的msg_order表中登记

前缀指令最直接的形式，只能定义前缀匹配关键词的指令。`msg_order`的**值有两种写法**：函数，或本文件内函数名（字符串）。

```lua
--/plugin/hello_world.lua
function hello(msg)
    return "hello,world"
end

msg_order = {
    ["hello"] = "hello",	--值为字符串时，取本文件内的同名函数
}
```

也可以直接写匿名函数，文件仍会先整体执行一遍，仅用于准备全局环境：

```lua
--/plugin/hello_world.lua
msg_order = {
    ["hello"] = function(msg)
        return "hello,world"
    end,
}
```

#### ②指定语句

常用于直接以`.reply`指令或webui设置回复：

>.reply set Title = hello_world
>
>Prefix=hello
>
>Lua=return "hello,world"


```toml
#/mod/hello_lua/reply/hello_world.toml
#可以在toml文件内写入脚本语句但不推荐
[reply.hello]
type = "Order"
keyword.prefix = "hello"
echo.lua = """return 'hello,world'"""
```

```lua
--/mod/hello_lua/reply/hello_world.lua
--可以这么写但完全不推荐，请采用④
msg_reply.hello = {
    type = "Order",
    echo = {
        lua = [[return "hello,world"]]
    },
}
```

#### ③指定脚本文件

既可用于直接以`.reply`指令或webui设置回复，又可用于mod内设置：

>  .reply set Title = hello_world
>
> Prefix=hello
>
> Lua="hello_world"

```toml
#/mod/hello_lua/reply/hello_world.toml
[reply.hello]
type = "Order"
keyword.prefix = "hello"
echo.lua = "hello_world"
```

```lua
--/mod/hello_lua/script/hello_world.lua
return "hello,world"
```

#### ④直接定义函数

```lua
--/mod/hello_lua/reply/hello_world.lua
msg_reply.hello = {
    type = "Order",
    echo = {
        lua = function(msg)
            return "hello,world"
        end
    }
}
```

### Lua嵌入特性

- 每次lua脚本执行都在独立的虚拟机(lua_State)中，全局变量相互独立，**不跨次保留**；
- 每次新建虚拟机时将预置函数写入全局；
- **执行文件形式时，文件会先被完整执行一遍**，再取`msg_order`中登记的函数调用。因此写在文件顶层的`os.time()`、`getUserConf()`之类每次指令都会重新求值；若值是匿名函数，文件只用于准备全局环境，函数体另行执行；
- Dice!传入的数值一律是浮点型（Lua number），拼进字符串时会带小数部分（如`"233.0"`），脚本内自己写的整数常量不受影响。**取账号ID或群号时请统一按字符串处理**，需要比较时先用`string.format("%.0f",n)`归一；
- 运行返回值可文本化时，将作为消息回复；最多返回2个值，第1个为公屏回复，第2个为私聊回复（暗骰/暗抽/暗检定会同时发给ob用户）；
- 向`msg`写入的自定义字段，可以在返回文本中用`{字段名}`引用，这是把脚本数据交给回执模板的常用手法（例：`msg.li = ...`后`return "结果：\n{li}"`）。

### JavaScript嵌入方式

*(build640+)*JavaScript同样分"语句"与"文件"两种。**判定规则与Lua、Python一致**：值能在`script/`目录下找到同名文件（不含后缀）就当文件名，否则当语句执行。`.reply set`里`JS=hello_world`与`JS=msg.echo("hi")`的区别，仅在于前者存在同名文件。

#### ①指定语句

常用于直接以`.reply`指令或webui设置回复：

>.reply set Title = hello_world
>
>Prefix=hello
>
>JS="hello,world"


```toml
#/mod/hello_js/reply/hello_world.toml
#可以在toml文件内写入脚本语句但不推荐
[reply.hello]
type = "Order"
keyword.prefix = "hello"
echo.js = """msg.echo("hello,world")"""
```

#### ②指定脚本文件

既可用于直接以`.reply`指令或webui设置回复，又可用于mod内设置：

>  .reply set Title = hello_world
>
>  Prefix=hello
>
>  JS=hello_world

```toml
#/mod/hello_js/reply/hello_world.toml
[reply.hello]
type = "Order"
keyword.prefix = "hello"
echo.js = "hello_world"
```

```javascript
//mod/hello_js/script/hello_world.js
msg.echo("hello,world");
```

### JavaScript嵌入特性

- Dice!采用（修改后支持MSVC的）QuickJS实现JavaScript的嵌入，支持到ES2020的大部分语法，**没有`require`**，模块请用ESM的`import`或内置的`loadJS`；
- 所有js脚本执行共用一个运行时(JSRuntime)，每次的**执行上下文**(JSContext)相互独立，全局变量不跨次保留；
- 每次新建执行上下文时将预置函数写入全局（共14个，见附录）；
- 回复有两种写法：`msg.echo("文本")`立即发送；或让语句/文件的**求值结果为字符串**，Dice!会将其作为回复；
- 事件与定时任务的`action`中，事件对象用`this`指代（`this.uid`、`this.gid`、`this.fromMsg`……），**不是`event.uid`**；
- `loadJS(文件名)`先查mod的`script/`目录，再查`程序目录/Diceki/js/`，以ES模块方式执行并返回其导出内容。

### Python嵌入方式

*(build639+)*Python同样分"语句"与"文件"，其中**文件形式的内联判定与其他语言一致**（能命中`script/`下同名文件即为文件）。但Python有一处必须记住的差异：**内联语句只会被当作单个表达式求值**，因此不能写`import`、赋值或多行语句，需要多句逻辑时请写进`script/xxx.py`。

#### ①指定语句

常用于直接以`.reply`指令或webui设置回复：

>.reply set Title = hello_world
>
>Prefix=hello
>
>Py=msg.echo("hello,world")


```toml
#/mod/hello_py/reply/hello_world.toml
#可以在toml文件内写入脚本语句但不推荐
[reply.hello]
type = "Order"
keyword.prefix = "hello"
echo.py = """msg.echo("hello,world")"""
```

#### ②指定脚本文件

既可用于直接以`.reply`指令或webui设置回复，又可用于mod内设置：

>  .reply set Title = hello_world
>
>  Prefix=hello
>
>  Py=hello_world

```toml
#/mod/hello_py/reply/hello_world.toml
[reply.hello]
type = "Order"
keyword.prefix = "hello"
echo.py = "hello_world"
```

```python
#/mod/hello_py/script/hello_world.py
msg.echo("hello,world")
```

### Python嵌入特性

- Dice!采用CPython实现Python的嵌入，运行时为**3.12**。程序会按`程序目录/python312`、`py312`、`python`、`bin`、`程序目录`的顺序寻找含标准库的目录，Windows下可将`python3.dll`、`python312.dll`与`python312.zip`放入`程序目录`后，设置Master配置项`EnablePython`=1并重启；未找到运行时时日志会提示"未找到可用的 Python 运行时目录"；
- 所有py脚本共用一个解释器，`__main__`的全局命名空间长期存在，**全局变量会跨次保留甚至互相污染**，需慎用；回复脚本的上下文对象放在一次性局部命名空间里（`msg`/`self`），事件脚本的上下文对象本身就是局部命名空间；
- *(build670+)*全局初始化时已执行一次`from dicemaid import *`，因此**脚本无需导入**即可直接调用`log`、`getPlayerCard`、`Actor`等，回复脚本与事件脚本都一样；显式再写`import`也无副作用；
- *(build670+)*脚本里的上下文对象叫`self`：`self.echo(文本)`、`self.uid`、`self.blocked = True` 都可用；回复脚本另有`msg`这个别名（历史写法，与`self`指向同一对象），事件脚本没有`msg`；
- **本版构建的已知限制**：CPython是静态链接进`w4123.Dice.dll`的，而`.pyd`扩展会另外加载一份`python312.dll`，两套运行时混用会直接打崩宿主。因此py脚本里**不能import需要`.pyd`的模块**（`unicodedata`、`_ssl`、`_ctypes`、`_decimal`、`_hashlib`等），也**不要在源码里写非ASCII标识符**（词法分析会隐式加载`unicodedata`，同样崩）。纯Python模块（`json`、`random`、`re`）与内建模块（`math`、`sys`、`os`）不受影响；
- **Python不会用返回值回复**：无论语句还是文件，都必须调用`msg.echo("文本")`才会发消息（这一点与Lua/lua的`return`不同）；出错时会回复统一的脚本运行错误提示；
- 向用户数据中写入元组(Tuple)时视同列表(List)处理，写入集合(set)时映射为Set；
- 事件脚本的顶层赋值写进的是上下文对象本身，**只能保存可转换为`AttrVar`的值**（字符串、数字、布尔、表、集合、ID）；`def`/`class`/`import` 绑定的函数、类、模块对象无法转换，名字随即失效，详见[附录的事件样例](#python写法)；
- 事件与定时任务的`action`同理：文件走`script/`目录，内联值只能是单个表达式。

## 模块Mod

**模块是Dice!外置可热插拔的扩展文件集**，是远程装卸、开关的单元。Mod采用类Paradox风格结构，由`存档目录/mod/`下的json文件及其同名文件夹构成：`模块名.json`写有标题、作者、版本、说明与帮助文档；同名目录`模块名/`下按类型存放各类文件。多个Mod存在同名项目（script文件、event、reply、speech、rulebook条目）时，按加载排序**后来居上**。

> 从仓库或压缩包安装时（`.mod get`、WebUI远程资源），Dice!会把包内的`descriptor.json`复制为`模块名.json`。自己手写时**文件名取模块名即可，且必须与目录名一致**，否则只读json不读目录。

*阅读说明：下文中形如`(build609+)`的注释用于说明该特性在何时被添加或最后编辑，开发者应当在制作mod时，将`dice_build`设定为满足mod内所有特性要求的最小build。*

```
存档目录/
|-- mod/
    |-- mod_name.json          ← 模块主文件，文件名=模块名
    |-- mod_name/
        |-- event/             ← 事件（lua / toml）
        |-- reply/             ← 关键词回复与自定义指令（lua / toml）
        |-- script/            ← 可被调用的脚本（lua / js / py）
        |-- speech/            ← 自定义回执文本（yaml）
        |-- rulebook/          ← 规则集（yaml）
        |-- model/             ← 角色卡模板（xml）
        |-- image/             ← 图片，加载时复制到 程序目录/data/image
        |-- audio/             ← 语音，加载时复制到 程序目录/data/record
```

### Mod主文件

```json
{
    "title":"角色卡栏位扩展",
    "ver":"1.0.0",
    "author":"安研色Shiki",
    "dice_build":654,
    "brief":"角色卡栏位扩展（状态栏、物品栏、法术栏、专长栏、笔记栏等）",
    "require":[],
    "comment":"",
    "helpdoc":{
        "扩展栏位":"在角色卡上追加状态栏、物品栏、法术栏、专长栏与笔记栏",
        "栏位指令":".st show 物品栏 //查看指定栏位"
    }
}
```

<table><thead><tr><th>字段</th><th>类型</th><th>说明</th></tr></thead><tbody>
<tr><td>title</td><td>string</td><td>模块名称，会显示在.mod list与帮助中</td></tr>
<tr><td>ver</td><td>string</td><td>模块版本号</td></tr>
<tr><td>author</td><td>string</td><td>作者署名</td></tr>
<tr><td>dice_build</td><td>number</td><td>可运行的最低Dice!build，低于此项的Dice!会放弃读取该模块</td></tr>
<tr><td>brief</td><td>string</td><td>模块简介</td></tr>
<tr><td>require</td><td>array</td><td>前置模块名列表。缺失则放弃加载；若前置模块排在本模块之后，只会在日志中告警</td></tr>
<tr><td>comment</td><td>string</td><td>不会写入任何输出，仅作文件内注释</td></tr>
<tr><td>helpdoc</td><td>object</td><td>帮助词条表，键为词条名、值为帮助文本，可被.helpdoc与.help取到</td></tr>
<tr><td>speech</td><td>object</td><td>可选，等效于speech/*.yaml中的回执文本</td></tr>
</tbody></table>

mod主文件读取成功后，Dice!将尝试进一步读取同名子目录*(build599+)*。**各子目录均为可选**，缺哪个就跳过哪个。

### Mod子目录

#### event

以lua*(build609+)*或toml*(build636+)*形式向表`event`写入事件。load时读入，修改后需要`.system load`应用。

一个事件由三部分组成：**ID**（表键，唯一，同名覆盖）、**trigger**（触发方式，缺省时只能由规则集卡带或脚本按ID调用）、**action**（被执行的内容）。

<table><thead><tr><th>trigger</th><th>取值</th><th>说明</th></tr></thead><tbody>
<tr><td>hook</td><td>事件名</td><td>代理Dice!内置时点，取值见[附录：事件event样例](#附录：事件event样例)。其中StartUp/DayEnd/DayNew在独立线程执行</td></tr>
<tr><td>clock</td><td>表或表数组</td><td>每日指定时点触发（过时不补），只读取hour与minute（不读秒、不读星期，受TimeZoneLag修正），可传多个时点</td></tr>
<tr><td>cycle</td><td>秒数或表</td><td>计时循环，表内可用second/minute/hour/day相加；Dice!初始化后会立刻先执行一次</td></tr>
</tbody></table>


`action`的值是"`script/`下的文件名"（不含后缀），或直接写语句；lua还可直接写函数：

- `{ lua = "daily.good_morning" }` 调用`script/daily/good_morning.lua`
- `{ js = "good_morning" }` 调用`script/good_morning.js`
- `{ py = "good_morning" }` 调用`script/good_morning.py`
- `{ lua = function() ... end }` 内联函数（**不接收参数**，事件对象用全局`event`）
- `{ lua = 'return "..."' }` / `{ js = "this.echo(...)" }` / `{ py = "..." }` 内联语句

lua侧在调用前会先将事件上下文注册为`event`，因此`event.uid`、`event.gid`、`event.fromMsg`等字段可直接读写；js侧事件对象以`this`提供；**py侧把上下文直接当作脚本的局部命名空间**，字段以裸名出现（`uid`、`gid`、`fromMsg`），回写同名变量即写回上下文；需要显式引用该对象时用`self`（`self.uid`、`self.echo()`）。

##### lua格式

```lua
event.listen_friend_request = { --该事件的ID，唯一对应，同名覆盖
    title = "好友审核", --事件标题
    trigger = { --触发方式，无则只能直接通过ID调用
        hook = "FriendRequest"	--代理内置的好友申请事件
    },
    --调用script/listen_friend_request.lua
    action = { lua = "listen_friend_request" } 
}
event.good_morning = {    
    title = "早安",
    trigger = {
        clock = { --在每日指定时点触发，可设置多个时点
            {hour=7,minute=30}
        }
    },
    --调用script/daily/good_morning.lua
    action = { lua = "daily.good_morning" } 
}
event.heartbeat = {    
    title = "心跳",
    trigger = {
        cycle = { --计时循环,所有cycle事件在Dice!初始化后立刻执行第一次
            minute=5,second=30 --循环执行的间隔
        }
    },
    action = { lua = "heartbeat" } 
}
```

##### toml格式

```toml
[event.listen_friend_request]   
title = "好友审核"
trigger.hook = "FriendRequest"
action.lua = "listen_friend_request"

[event.good_morning]   
title = "早安"
trigger.clock = [{hour=7,minute=30}]
action.lua = "daily.good_morning"

[event.heartbeat]   
title = "心跳"
trigger.cycle = { minute=5,second=30 }
action.lua = "heartbeat"
```

#### reply

以lua*(build603+)*或toml*(build636+)*形式向表`msg_reply`（toml为`reply`）写入关键词回复。load时读入，修改后需要`.system load`应用。

每一条回复由五个字段决定：

**keyword**，触发形式，可同时使用多种（`match`/`prefix`/`search`/`regex`）：

<table><thead><tr><th>键</th><th>匹配方式</th><th>说明</th></tr></thead><tbody>
<tr><td>match</td><td>完全匹配</td><td>消息全文与关键词完全一致</td></tr>
<tr><td>prefix</td><td>前缀匹配</td><td>消息以关键词开头，剩余部分存入msg.suffix</td></tr>
<tr><td>search</td><td>模糊匹配</td><td>消息中包含关键词即触发，禁止自我触发（骰娘自己的消息不匹配）</td></tr>
<tr><td>regex</td><td>正则匹配</td><td>ECMAScript正则，整条消息需完全匹配；消息长度上限400字符</td></tr>
</tbody></table>

值可以是单个字符串或字符串数组；用`.reply set`设置多个关键词时以 **|** 分隔。

**type**，触发性质，决定受哪一个开关控制：

<table><thead><tr><th>值</th><th>说明</th></tr></thead><tbody>
<tr><td>Order</td><td>自定义指令，先于内建指令匹配，受"停用指令"/.bot off控制</td></tr>
<tr><td>Reply</td><td>自定义回复，在内建指令之后匹配，受"禁用回复"控制</td></tr>
<tr><td>Both</td><td>既参与指令匹配也参与回复匹配；开关按指令处理（受停用指令控制）</td></tr>
<tr><td>Game</td><td>局内/规则集指令，仅在game局中且对应规则集下匹配*(build649+)*</td></tr>
<tr><td>Nor</td><td>不参与任何自动匹配，只能由脚本或指令显式调用</td></tr>
</tbody></table>

**echo**，回复内容，共五种形态：

<table><thead><tr><th>写法</th><th>行为</th></tr></thead><tbody>
<tr><td>字符串</td><td>纯文本回复，会做花括号转义</td></tr>
<tr><td>数组</td><td>按牌堆处理，随机抽一条</td></tr>
<tr><td>{ lua = 文件名或函数 }</td><td>调用lua脚本，函数返回值即回复</td></tr>
<tr><td>{ js = 文件名或语句 }</td><td>调用js，msg.echo()或求值结果即回复</td></tr>
<tr><td>{ py = 文件名或表达式 }</td><td>调用py，必须msg.echo()才会发出</td></tr>
</tbody></table>
**limit**，触发限制，见下。**rule**，归属规则集，见[rulebook](#rulebook)。

##### limit限制项

table形式的键值表示"条件类型 = 条件内容"。冷却与限额的时间/次数单位分别是**秒**与**次**。

<table><thead><tr><th>键</th><th>取值</th><th>说明</th></tr></thead><tbody>
<tr><td>cd</td><td>数字 或 {user=60}</td><td>冷却。直接给数字为当前聊天窗口冷却；user为每用户、global为全局</td></tr>
<tr><td>today</td><td>数字 或 {user=1}</td><td>当日次数上限，作用域同上</td></tr>
<tr><td>lock</td><td>true / 字符串 / {user=true}</td><td>同步锁：同一条回复同时只处理一次，可具名以便跨回复互斥</td></tr>
<tr><td>prob</td><td>1~99</td><td>以该百分比概率触发</td></tr>
<tr><td>user_id</td><td>数字/数组 或 {nor=...}</td><td>仅这些用户触发；nor为反向（这些用户不触发）</td></tr>
<tr><td>grp_id</td><td>数字/数组 或 {nor=...}</td><td>仅这些群触发；nor同上</td></tr>
</tbody></table>

冷却与限额都可以附带自定义提示文本：`cd = { user = 60, echo = "还不到时候哦" }`。同一张表里可并列多个作用域，如`cd = { user = 60, global = 5 }`表示用户冷却60秒、全局冷却5秒；若用`.reply set`的字符串形式设置，同类条件之间以`&`连接，不同类之间以`;`分隔。

##### lua格式

```lua
msg_reply.good_morning = {	--该条msg_reply的id，唯一对应，同名覆盖
    keyword = {
        match = "早",
        prefix = {"早安","早上好"},
    },
    limit = {
        today = { user = 1 },	--每用户当日触发次数
    },
    echo = {
        lua = "reply_good_morning"	--调用文件名对应lua
    }
}
msg_reply.good_night = {
    keyword = {
        prefix = "晚安",
    },
    limit = {
        cd = { user = 60 },
        today = { user = 2 },
    },
    echo = "{reply_good_night}"	--直接回复文本
}
```

##### toml格式

```toml
[reply.good_morning]
keyword = { match = "早", prefix = ["早安","早上好"] }
limit.today.user = 1
echo.lua = "reply_good_morning"

[reply.good_night]
keyword.prefix = "晚安"
limit = { cd = { user = 60 }, today = { user = 2 }}
echo = "{reply_good_night}"
```

##### 规则集专属指令

带上`rule`字段后，该回复不再进入全局匹配，只在该规则集生效；配合`type = "Game"`，即为“只有当前桌使用该规则时才会响应的指令”。下面是COC7扩展模块的实例写法（同名Order则将COC的sn设置为无game/未设定rule时的兜底指令，与其他规则优先级视mod排序而定）：

```toml
#/mod/COC7扩展/reply/sn.toml
[reply.coc7_sn]
type = "Game"
rule = "COC7"
keyword.prefix = ".sn"
limit.grp_id.nor = 0
echo.lua = "sn"

[reply._sn]
type = "Order"
keyword.prefix = ".sn"
limit.grp_id.nor = 0
echo.lua = "sn"
```

#### script

`script/`目录存放**可被调用的脚本**，支持lua、js、py三种后缀，文件名（不含后缀）即调用名，可放在子目录里：

- `reply`/`event`中`echo`/`action`的值命中同名文件时按文件执行，否则按语句执行；
- 跨脚本调用：lua用`loadLua("子目录/文件名")`，js用`loadJS("文件名")`；py用`import`，但其搜索路径只含`存档目录/plugin`与`程序目录/Diceki/py`，要引用本mod的其他py脚本需先`sys.path.append`所在目录；
- lua另有约定：子目录下的`init.lua`以其**目录名**注册（`script/daily/init.lua`→`loadLua("daily")`）；
- script中的文件**不会被预加载**，而是调用时实时读取，因此热更新后不需要`.system load`；
- *推荐script文件名及文件内不出现中文字符。*

```lua
---reply_good_morning.lua
clock_now = now = os.date("*t")
if(now.hour>12)then
    msg.hour = now.hour
    return "{reply_good_morning_late}"
elseif(now.hour<5)then
    return "{reply_good_morning_early}"
else
    return "{reply_good_morning}"
end
```

```javascript
//reply_good_morning.js
const now = new Date();
msg.hour = now.getHours();
if (msg.hour > 12) {
    msg.echo("{reply_good_morning_late}");
} else if (msg.hour < 5) {
    msg.echo("{reply_good_morning_early}");
} else {
    msg.echo("{reply_good_morning}");
}
```

```python
#reply_good_morning.py
import time
msg.hour = time.localtime().tm_hour
if msg.hour > 12:
    msg.echo("{reply_good_morning_late}")
elif msg.hour < 5:
    msg.echo("{reply_good_morning_early}")
else:
    msg.echo("{reply_good_morning}")
```

三份脚本同名不同后缀，回复里用哪个后缀就调哪个：

```toml
[reply.good_morning]
keyword.prefix = "早安"
echo.lua = "reply_good_morning"	#或 echo.js / echo.py
```

注意三种语言的**返回方式不同**：lua用`return`；js可以用`msg.echo()`，也可以让最后一条表达式返回字符串；py**只能用`msg.echo()`**，返回值不会被当成回复。

#### speech

*(build603+)*台词speech是自定义回执文本的上位，可直接由花括号转义。每项条目可存多条文本，等效于单抽放回的牌堆或{sample}。load时读入，修改后需要`.system load`应用。

```yaml
reply_good_morning:
 - "早上好啊{nick}"
 - "那{nick}也早安哦"
reply_good_morning_late: "{hour}点的早安？{nick}起得可真早呢"
reply_good_morning_early: "早安，{nick}的睡眠还够吗？"
reply_good_night: 
 - "晚安呐"
 - "也祝{nick}晚安"
strRollDice: "{pc}掷骰：{res}"
strRollRegularSuccess: 
 - 成功了哦
 - "成功 看来{self}还是护佑着{nick}的"
```

#### rulebook

*(build647+)*规则集，一个yaml文件描述一本规则书。**这项能力供查询与局内指令使用**：`.rule set 规则名`后，`msg.game.rule`即为该规则，桌面上的`type="Game"`指令随之生效。

```yaml
rule: COC7                 #规则集名，与.rule set、reply的rule字段一致
manual:                    #词条表，可被.rule s/.rules检索
  跳跃: "大于等于50则成功"
  大失败: "96-100"
tape:                      #卡带，按名登记的action，由listen_cassette按ID调用
  game_start:
    lua: "COC7.start"
```

命名可用`母-子`表示继承：名为`COC7-1920`的规则集以`COC7`为母本，未定义的词条与卡带向母本回溯。

#### model

*(build654+)*角色卡模板，`model/`下放xml模板文件。根元素`<model name="COC7" alias="COC/coc7">`的名字即模板名（`alias`以`/`分隔，*(build657+)*起可建对应别名）；其下`<basic>`定义基础字段、`<property>`定义派生属性（`<any name alias text>`，`text`可为`dicexp`/`javascript`）、`<presets>`定义预设、`<script>`内可写`<trigger name time>`挂触发器。

模板与脚本强相关（触发器里能拿到`this.char`、`this.game`，并可用`setGroupAttr`写回），完整语法目前以官方模块为准，可直接参考`DiceExtern/DiceModIndex/COC7-TRPG/model/COC7.xml`。

#### image

*(build633+)*图片文件夹，会在mod加载时复制到`程序目录/data/image`。发送时按**相对路径**填CQ码即可，如`[CQ:image,file=myMod/roll.png]`（对应模块内`image/myMod/roll.png`）。

#### audio

*(build633+)*语音文件夹，会在mod加载时复制到`程序目录/data/record`，同样按相对路径填CQ码发送，如`[CQ:record,file=myMod/hello.mp3]`。


## Lua快速教程

*如果你已经了解过脚本语言lua，请跳过该部分。由于function使用小括号()输入参数，下文使用中括号[]表示参数可省略*

### Lua基本介绍

Lua是一种用标准C语言编写并开源的嵌入式脚本语言。

Lua语句不使用分号或其它标点作结尾，不使用大括号或缩进表示作用域，使用"end"作为函数、条件判断、while循环的结束。单行注释以"--"开头，与或非运算符使用"`and`""`or`""`not`"。Lua字符串与数组的首位索引均为1，而不是一般编程语言的0。

### Lua标识符

- 逻辑运算: 与`and`; 或`or`; 非`not`

- 运算符号: 赋值`=`; 等号`==`; 不等号`~=`; 小于号`<` ; 小于等于号`<=`; 四则运算`+-*/`; 取余`%`; 乘幂`^`; 取整除法`//`; 连接`..` ; 

- 取长度#: `#table` 或 `#string` 

- 函数参数():`function(args)`

- 表索引键值（index）[]: `table[key]`

- 字符串（不可混用）:`"字符串"`/`'字符串'`/`[[不转义换行等符号的字符串]]`

- 单行注释 `--`

  ```lua
  -- lua单行注释
  ```

- 多行注释 `-[[]]`

  ```lua
  --[[
  lua多行注释
  ]]
  ```

### Lua数据类型

Lua中的变量一共有8种基本数据类型，具体类型在赋值时自动判断而不需要声明。编写Dice!脚本需要了解其中6种：空(nil)、逻辑型(boolean)、双精度浮点数(number)、字符串(string)、函数(function) 和 表(table)。使用type()函数可以返回一个表示变量类型的文本值，如type(msg)=="table"。

#### 空值nil

nil类型只有nil一种值，表示无效值，比如一个没有赋值过的变量，访问表中不存在的key，缺失的函数返回值。函数不返回相当于返回nil，对变量赋值nil等于删除该变量。*例：试图用string.match匹配字符串时，如果匹配失败，则只返回nil。nil不能视作空字符串参与运算，所以需要预先考虑变量为nil的情况。*

#### 逻辑值（布尔）boolean

boolean类型有两种值：真true，假false。所有变量都可以自动转换为逻辑值，其他非空变量均视为真，空变量(nil)视为假。

#### 数值number

lua中所有的数都可视为双精度浮点数(double)。

string与number进行算术运算时（如`"6"*5`），若string可转换为数字，将进行自动类型转换，否则报错；string与number进行取相等运算时自动为false（如`"123"~=123`）；string与number用不等号比较时必定报错（如`"6">5`）

#### 文本值（字符串）string

Lua中的string可以用三种方式表示:`"双引号"、'单引号'、[[双层中括号]]`，其中双层中括号内引号、换行符等特殊字符不需要转义。string不能使用加减符号，可以通过".."连接。在string前加"#"表示该字符串的长度。

- **str1..str2** 

  连接字符串str1与str2，若连接到数字，将数字自动保留6位小数打印。

- **string.match(str, pattern[, init])**

  从str的init位置起，寻找（Lua自定义正则）匹配pattern的子串，成功返回匹配到的string，失败返回nil。init默认为1。

  例:`string.match(msg.fromMsg,"%d*",#order_name+1)`从指令名的后一位开始，从消息中匹配数字。

- **string.len(str)**

  返回str的长度，等效于`#str。`

- **string.format(...)**

  将特定类型变量按格式转化为string。

  例: `string.format("%.0f",die)`将掷骰结果按保留0位小数的浮点数打印。

- **string.sub(str, init [, end])**

  从str中截取从init位置到end的子串，end默认-1（截取到最后）

  例:`string.sub(msg.fromMsg,#order_name+1)`截取消息从指令名后一位开始的文本

- **string.find (str, substr, [init, [end]])** 

  在str从init到end的位置间从左至右寻找匹配substr的子串。如匹配成功，返回str中子串的首位，否则返回nil。

- **string.upper(str)** 将str中所有字母全部大写。

- **string.lower(str)** 将str中所有字母全部小写。

- **string.rep(str, n)** 返回将str重复n遍后连接的string。

  例:`string.rep("木大",8)`返回"木大木大木大木大木大木大木大木大"。

#### 表table

table是存储变量的容器(关联数组)，可以通过不同的索引访问对应的值，格式为`table[key]`。

当table的键值为纯ANSI字符时（非纯数字、无中文等多字节字符），可用'.'代替[""]格式，如`msg.fromQQ`等价于`msg["fromQQ"]`，但不能用`msg_order.抱Shiki`来代替`msg_order["抱Shiki"]`

- **table.concat(tab[, sep])** 

  将表tab中所有元素作为string连接，使用sep分隔。

  例:`table.concat(die_pool,'+')`将所有骰目通过加号连接。

- **table.insert(tab, [pos, ]value)** 

  在table的pos位置插入value，默认插入在末尾。

  例:`table.insert(die_pool,die)`将本次骰目加入结果池中

- **table.sort (tab [, comp])**

  将tab按comp函数进行排序，默认进行升序排序。

  例: `table.sort(die_pool,function(a,b) return (a>b) end)`当需要为掷骰结果取最大的若干个时，就需要预先为骰目降序排序。

```lua
--初始化空表
msg_order = {}
--向表中插入键值对
msg_order[".duel"] = mdice_duel
```

#### 函数function

函数在定义时需要声明function 函数名(参数列表)，使用return返回，使用end作为函数体的结尾。函数不需要规定返回值的类型和数量，甚至可以在不同的位置返回不同的值。返回多个值时，值之间以','分隔。*不要在return的下一行接一般语句。*

```
function max_min(a,b)
	if(a>b)then
		return a,b
	else
		return b,a
	end
end
```

### Lua流程控制

#### if()then...elseif()then..else...end

if(语句为真)then [执行语句] end。可使用多个elseif()then来进行判定语句为假时的后续判定。

**注意：elseif与else if是两种格式**，后者表示else作用域内额外嵌套一层if结构，因此比前者额外多一个end。

```lua
if(favor < 20)then
	return "{nick}的好感度还不够哟"
elseif(favor < 60)then
	return "只给{nick}一下哦~就一下"
else
	return "那就随{nick}的便啦"
end
```
#### for循环(数值)

```lua
for i=1, cnt_dice do	--表示i初始值为1，每次循环+1，大于cnt_dice时停止循环，相当于执行cnt_dice次
    local dice_point = ranint(1,6)
    table.insert(dice_pool,dice_point)  
    sum = sum + dice_point
end  
```
#### while()do 循环

```lua
while(cnt_dice > 0) do	--条件为真则继续循环，相当于执行cnt_dice次
    local dice_point = ranint(1,6)
    table.insert(dice_pool,dice_point)  
    sum = sum + dice_point
    cnt_dice = cnt_dice - 1
end  
```
#### repeat...until 循环

```lua
repeat	--第一次无条件执行，相当于执行cnt_dice次但至少为1次
    local dice_point = ranint(1,6)
    table.insert(dice_pool,dice_point)  
    sum = sum + dice_point
    cnt_dice = cnt_dice - 1
until( cnt_dice < 1 )	--条件为真则跳出循环
```

好了，现在你已经了解了lua的全部基本语法，可以自己动手自定义几乎任何跑团规则的掷骰指令，或是自制牌堆/好感互动指令了~

### 编写帮助

- lua没有三目运算符，但可以通过短路运算符`or`和`and`实现类似效果；

  `0.1+0.2==0.3 and 'true' or 'false'`


## 前缀指令脚本

### 匹配顺序

Dice!收到消息后依次尝试，任一步命中即返回，因此自定义指令可以覆盖内建指令：

1. 基础指令：`.authorize/.dismiss/.warning/.master/.bot/.helpdoc/.help`；
2. 若当前聊天在`game`局中：规则集卡带 → 规则集/局内指令（`type="Game"`，见[rulebook](#rulebook)）；
3. 自定义指令：`type="Order"`或`Both`的回复，消息前缀匹配后，未匹配的部分存为`msg.suffix`；
4. 内建指令；
5. 自定义回复：`type="Reply"`或`Both`的回复。

> **`plugin/`目录只支持Lua**（另可放写`[reply.x]`的toml）。js/py写的指令必须做成mod，在`reply`的`echo.js`/`echo.py`里指定。

### 指令函数

指令函数的参数msg是一张记录消息上下文的表（实现为Lua的userdata，以Dice定义的Context为元表），可直接读写字段，也可写入自定义字段供回执转义使用。

#### msg常用字段

<table><thead><tr><th>字段</th><th>说明</th></tr></thead><tbody>
<tr><td>fromMsg</td><td>消息全文</td></tr>
<tr><td>suffix</td><td>前缀指令匹配后剩下的部分（仅前缀匹配时有定义）</td></tr>
<tr><td>uid / fromQQ / fromUser</td><td>发送者账号，三者等价</td></tr>
<tr><td>gid / fromGroup</td><td>来源群号，私聊时为空</td></tr>
<tr><td>chid</td><td>频道号，仅频道消息有值</td></tr>
<tr><td>msgid</td><td>消息id</td></tr>
<tr><td>time</td><td>消息时间戳（秒）</td></tr>
<tr><td>nick</td><td>称呼（按群名片/全局称呼/昵称依次取）</td></tr>
<tr><td>pc</td><td>角色卡名，无卡时退化为称呼</td></tr>
<tr><td>at / @</td><td>指向发送者的at码</td></tr>
<tr><td>user / grp(group)</td><td>用户对象/群对象，可继续取属性</td></tr>
<tr><td>char</td><td>发送者在本群的角色卡，即Actor</td></tr>
<tr><td>grpAuth</td><td>发送者在本群的权限（1群员/2管理/3群主）</td></tr>
<tr><td>game</td><td>当前聊天的桌，即GameTable，无桌时为空</td></tr>
<tr><td>reply_title</td><td>命中的回复/指令名</td></tr>
<tr><td>lang</td><td>脚本出错时被置为出错语言</td></tr>
</tbody></table>

此外，`msg["自定义字段"] = 值`写入的内容会保留到本次回复完成，可在返回文本里用`{自定义字段}`引用。

#### 返回值

指令函数的返回值可以有0到2个，如有，第一个返回值表示直接回复的语句，第二个返回值表示私聊回复的语句（用于暗骰/暗抽/暗检定，会同时发送给ob用户）。由于Dice!在发送消息时会将换页符`\f`视为消息分段发送，实际上可以通过一个返回值返回多段回复。

### 指令注册

lua脚本读取目录为`存档目录/plugin/`（plugin文件夹不会自动创建）。在启动或`.system load`时，Dice!会遍历该目录：

- `*.lua`：读取其中的`msg_order`表，键为前缀触发词，**值可以是函数，也可以是本文件内的函数名（字符串）**；
- `*.toml`：读取其中的`[reply.x]`节，等效于mod的`reply/*.toml`。

```lua
--/plugin/mymod.lua
local function roll_d20(msg)
    return "{pc}掷骰：" .. ranint(1, 20)
end

msg_order = {
    [".d20"] = "roll_d20",      --字符串：取本文件内同名函数
    [".d20x"] = roll_d20,       --函数：直接引用
}
```

**要点：注册与执行是两次不同的加载。**注册时把整个文件执行一遍取出`msg_order`；每条指令执行时又会**新建一个虚拟机、重跑整个文件**，再调用目标函数。所以：

- 全局变量不跨次保留，不要拿全局变量做缓存；
- 写在文件顶层的语句（读文件、读配置、`os.time()`）**每条指令都会重新执行一次**，重活请放进函数体内或用`loadLua`缓存；
- 只有**增删文件、修改触发词或函数名**需要`.system load`，只改函数体内容保存即生效。

### 定时任务

在plugin的任一lua文件中写`task_call`表，键为任务名、值为函数（不接收参数）：

```lua
--/plugin/mymod.lua
task_call.daily_report = function()
    sendMsg("早安，今日也要好好跑团哦", 0, getDiceQQ())  --无msg可用，需显式指定目标
end
```

用`.admin clock`把它挂到时间点：

```
.admin clock daily_report 7:30      #每天7:30执行
.admin clock -daily_report 7:30     #移除该时间点
.admin clock                        #查看当前定时列表
```

内置任务名`off`/`on`/`save`/`clear`分别对应休息、开工、备份、清理。定时任务在独立线程执行，没有`msg`，需要发消息请用`sendMsg`。

### 脚本调试

lua的代码错误要到运行报错时才暴露，但虚拟机的报错不会终结宿主程序，Dice!会将报错信息发送到通知窗口。脚本可能在两个时点报错：**加载脚本文件**时和**运行指令函数**时。由于指令注册时会将整个文件作为脚本运行，基本的语法错误（多了个end、少了括号、return之后没有end）会在注册时立即暴露；更多错误（几乎全部是参数类型不对）要到函数执行时才出现，出现后需要第一时间热修复。

js与py没有"加载期"检查：语法错误要等第一次执行时才暴露，报错写入控制台日志（通知窗口级别），并向聊天窗口回复统一的脚本运行错误提示。排查顺序建议：先看控制台最后一条报错的文件名与行号 → 用`.reply set`把可疑的一句单独设为回复试跑 → 确认无误再放回脚本。

### 扩展说明

*如果看不懂这部分内容，可以略过*

指令注册时会将整个lua文件运行一遍。如果担心函数体过于复杂影响读取，建议将主体放置到plugin的子文件夹，使用loadLua调用。指令注册时不会递归遍历子文件夹，因此可以降低不必要的开销。例如，在Lua实现了Maid角色卡模板功能，保存在`PC/Maid.lua`，这样其他脚本可使用`loadLua("PC/Maid")`重复调用。js用`loadJS("文件名")`；py用`import`，`plugin/`与`Diceki/py`已在`sys.path`中。

### 样例模板

```lua
--声明表msg_order,初始化时会遍历plugin文件，读取msg_order表中的指令名
--msg_order中的键值对表示 前缀指令->函数名
msg_order = {}
--声明触发词，允许多个触发词对应一个函数
order_word = "触发词"
order_word_1 = "触发词1"
order_word_2 = "触发词2"
--声明指令函数主体，函数名可自定义
function custom_order(msg)
    return "回复语句"
end
msg_order[order_word] = "custom_order"	--value为字符串格式且与指令函数名一致
msg_order[order_word_1] = "custom_order"
msg_order[order_word_2] = "custom_order"
--注意：本手册所提供脚本样例并非固定，仅基于模板化考虑选定该格式。
```


## 开发帮助

### 类型处理

lua使用`type()`以`string`类型返回变量类型，js使用`typeof()`以字符串形式返回变量类型，python使用`type()`以`type`类型返回变量类型。

<table><thead><tr><th>变量</th><th>type(lua)</th><th>typeof(js)</th><th>type(py)</th></tr></thead><tbody>
<tr><td>未定义</td><td>nil</td><td>undefined</td><td>报错：is not defined</td></tr>
    <tr><td>1==0</td><td>boolean</td><td>boolean</td><td>bool</td></tr>
    <tr><td>0</td><td>number</td><td>number</td><td>int</td></tr>
    <tr><td>3.14</td><td>number</td><td>number</td><td>float</td></tr>
    <tr><td>'hello'</td><td>string</td><td>string</td><td>str</td></tr>
</tbody></table>

### 常用语句

#### 类型转换

lua字符串连接时会自动类型转换为字符串，但Dice!传入的数值是浮点型，拼接时会带小数部分（如`"233.0"`），需要整齐的数字请手动使用`string.format`


```lua
--lua
local i = 233
local pi = 3.14
string.format("%d",i)	--整数转字符串，如果参数#2不为整数会报错
string.format("%.0f",pi)	--指定小数位数转字符串
print("pi="..math.pi)
```
JavaScript字符串连接时会自动类型转换

```javascript
//js
log("pi="+Math.PI)
```
python字符串连接时不会自动类型转换

```python3
#python
pi = 3.14
print("pi=" + str(pi))

float() #转换为浮点型
int() #转换为整形
bin() #转换为2进制
hex() #转换为16进制
oct() #转换为8进制
bool() #返回bool值
list() #返回列表
tuple() #返回元组
dict() #返回字典对象
set() #返回集合对象
complex() #虚数转换
bytes() #返回字节数组
bytearray() #返回可变的字节数组
str() #返回字符类型
```

#### 字符串操作


```lua
--lua
--去除首尾空格
str:match("^[%s]*(.-)[%s]*$")
```

```javascript
//js
//去除首尾空格
str.trim();
```

```python
#python
#去除首尾空格
str.strip()
```

#### 时间日期操作

```lua
--lua
--获取现在离1970年1月1日0时所过秒数
local t_now = os.time()

--格式化输出日期(YYYY-MM-DD HH:mm:ss，以系统所在时区)
local fmt_now = os.date("%Y-%m-%d %H:%M:%S")
local fmt_dt = os.date("%Y-%m-%d %H:%M:%S", t_now)
```

```js
//js
var d_now = new Date();
//获取现在离1970年1月1日0时所过毫秒数
var t_now = new Date().getTime();

//格式化输出日期(YYYY-MM-DD HH:mm:ss，以系统所在时区)
function pad(num) {
    return num.toString().padStart(2, '0');
}
function formatDateTime(date = new Date()) {
    return date.getFullYear()+'-'+pad(date.getMonth() + 1)+'-'+pad(date.getDate())+' '
    +pad(date.getHours())+':'+pad(date.getMinutes())+':'+pad(date.getSeconds());
}
var fmt_now = formatDateTime()
var fmt_dt = formatDateTime(dt_now)
```

````python
#python
import time
#获取现在离1970年1月1日0时所过秒数
t_now = time.time()

#格式化输出日期(YYYY-MM-DD HH:mm:ss，以系统所在时区)
fmt_now = time.strftime('%Y-%m-%d %H:%M:%S')
fmt_dt = time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(t_now))
````

### 文件的字符编码问题

**除lua外，所有文件（js/py/toml/yaml/json/xml）一律使用UTF-8（不带BOM）。**

- **Lua**：Dice!会逐个文件检测编码，UTF-8（推荐）与GBK都能读，并在该虚拟机生命周期内按检测结果收发字符串。这是唯一允许GBK的脚本语言；
- **JavaScript**：QuickJS按字节读入源码并当作UTF-8解析，GBK的中文字面量会变成乱码（ASCII部分仍可运行，问题更隐蔽）；
- **Python**：解释器以UTF-8模式启动，源码按UTF-8解析；
- **数据文件**：toml/yaml/json/xml均要求UTF-8。跨语言读写同一份数据时，UTF-8是唯一稳妥的选择；
- *(build666+)* Dice!内部字符串已统一为UTF-8，仅在驱动器接口处转码，因此历史脚本里为GBK写的补偿转换可以逐步去掉；
- lua调用`http`时，网页内容按原样返回，若目标网页为GBK需自行转码；
- 若`require`加载**文件名含非ASCII字符**的lua模块时报"module not found"，改用`loadLua("子目录/文件名")`：它按脚本名查找并处理路径转码。

## 附录：Dice!预置Lua元表及全局函数

**调用前注意Dice版本是否匹配！**Dice!2.7.1预置**21条全局函数**、**6个预置库/元表**（`Set`、`Context`、`GameTable`、`SelfData`、`Actor`、`http`），其中`http`有4条函数、`Context`有4个方法。

*类型为number的参数，一般也可传入可数字化的字符串，如'msg.fromGroup'。*

*`Context`可用的字段见[msg常用字段](#msg常用字段)。*

### Context元表

*(build602+)* **上下文Context**，承载event/reply中的上下文信息，如uid、gid等。

#### get(self, field, defaultVal)

取上下文中的变量，如#3省略，则`context:get(field)`可等效于`context[field]`。

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>上下文</td><td>Context</td><td>形如context:get()可省略#1</td></tr>
<tr><td>变量字段</td><td>string</td><td>字段会先进行一次转义</td></tr>
<tr><td>候补值</td><td>任意</td><td>如未定义该变量则返回该值</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead><tbody>
<tr><td>任意</td><td>待取变量</td></tr>
</tbody></table>

#### format(self, rawString)

*(build619+)* 在该上下文中转义文本。

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>上下文</td><td>Context</td><td>形如msg:format()可省略#1</td></tr>
<tr><td>待转义文本</td><td>string</td><td></td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead><tbody>
<tr><td>string</td><td>转义后文本</td></tr>
</tbody></table>
#### echo(self, replyMsg, isRaw)

*(build602+)* 回复消息，有来源聊天窗口的事件也可以用`event:echo(replyMsg)`。

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>上下文</td><td>Context</td><td>形如msg:echo()可省略#1</td></tr>
<tr><td>待回复消息</td><td>string</td><td></td></tr>
<tr><td>是否禁用转义</td><td>boolean</td><td>可省略，默认转义</td></tr>
</tbody></table>

#### inc(self, field, val)

自增变量，仅对数字有效。

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>上下文</td><td>Context</td><td>形如context:set()可省略#1</td></tr>
<tr><td>变量字段</td><td>string</td><td>字段会先进行一次转义</td></tr>
<tr><td>自增量</td><td>number</td><td>可省略，表示+1</td></tr>
</tbody></table>
### Actor元表

*(build644+)* **角色卡Actor**。由`getPlayerCard`取得，也可直接取`msg.char`。属性可直接读写（`pc.理智`、`pc["理智"]`），以`__`开头的是元属性（`pc.__Name`卡名、`pc.__DefaultDice`默认骰、`pc.__DefaultDiceExp`默认掷骰表达式）。

#### get(key)

取角色卡属性，等价于`pc[key]`。

#### set(key, val)

写角色卡属性，等价于`pc[key] = val`；写入`nil`表示删除。

#### locked(key)

*(build648+)*该属性是否被锁定，返回boolean。锁定属性常见于规则要求的"不可随意修改"项。

#### lock(key) / unlock(key)

*(build648+)*加锁/解锁，返回boolean表示是否改变了状态。

```lua
local pc = getPlayerCard(msg.uid, msg.gid)
if not pc:locked("理智") then
    pc:set("理智", (pc.理智 or 0) - 1)
    msg.hp = pc.理智
else
    return "理智已被锁定，无法修改"
end
```

#### rollDice(exp)

调用角色卡的默认骰*(__DefaultDice)*及默认掷骰表达式*(__DefaultDiceExp)*进行掷骰，返回table记录掷骰结果。

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>掷骰表达式</td><td>string</td><td></td></tr>
</tbody></table>

<table><thead><tr><th>返回值字段</th><th>字段类型</th><th>说明</th></tr></thead><tbody>
<tr><td>expr</td><td>string</td><td>规范化后的表达式</td></tr>
<tr><td>sum</td><td>number</td><td>掷骰结果（表达式合法时）</td></tr>
<tr><td>expansion</td><td>string</td><td>掷骰展开式（表达式合法时）</td></tr>
<tr><td>error</td><td>number</td><td>错误类型（表达式非法时）</td></tr>
</tbody></table>

### Set元表

*(build646+)* **集合Set**，存放不重复的键（字符串或数字），用于名单去重、权限判定、玩家列表等场景。`game`桌内的`pls`/`obs`/`gms`就是Set。

```lua
local list = Set.new()
list:add(msg.uid)          --返回true表示新加入
if list[10001] then        --同 list:in(10001)
    return "已在名单内"
end
list:remove(10001)
for _, uid in ipairs(list:totable()) do ... end
#list                       --元素个数
tostring(list)              --"{10001,10002}"
```

<table><thead><tr><th>成员</th><th>说明</th></tr></thead><tbody>
<tr><td>Set.new()</td><td>新建空集合</td></tr>
<tr><td>add(v)</td><td>加入元素，返回boolean表示是否新增</td></tr>
<tr><td>in(v) / set[v]</td><td>是否存在，返回boolean或nil</td></tr>
<tr><td>remove(v)</td><td>移除元素，返回boolean</td></tr>
<tr><td>totable()</td><td>转为数组table，便于ipairs遍历</td></tr>
<tr><td>#set</td><td>元素个数</td></tr>
</tbody></table>

### SelfData元表

**自建数据**。`getSelfData("文件名")`取`存档目录/selfdata/`下的json/toml/yaml，写入即落盘，适合存模块级配置与累计数据。同名文件全局共用同一份内存数据。

```lua
local conf = getSelfData("favor.json")
local limit = conf.daily_limit or 3      --字段直接读
conf.total = (conf.total or 0) + 1       --字段直接写，立即保存
conf:set("daily_limit", limit)           --等价写法
local all = conf:get()                   --取整张表
```

<table><thead><tr><th>成员</th><th>说明</th></tr></thead><tbody>
<tr><td>get([key][, default])</td><td>省略key时返回整表；key不存在时返回default</td></tr>
<tr><td>set(key, val) 或 set(表)</td><td>写单个键，或整份覆盖</td></tr>
<tr><td>data.key / data[key]</td><td>直接读写字段</td></tr>
</tbody></table>

### GameTable元表

*(build649+)* **游戏桌GameTable**。`msg.game`取得，仅当该聊天已用`game`指令开局时存在，否则为`nil`。

```lua
local game = msg.game
if not game then return "{help:game}" end
game.rule = "COC7"                     --写桌面字段（等价 game:set("rule","COC7")）
for _, uid in ipairs(game.pls:totable()) do
    local pc = getPlayerCard(uid, msg.gid)
end
game:message("桌上各位，请投掷先攻")   --向桌面所在聊天发一条消息
```

<table><thead><tr><th>成员</th><th>说明</th></tr></thead><tbody>
<tr><td>set(key, val) / set(表)</td><td>写桌面字段</td></tr>
<tr><td>message(文本)</td><td>向桌面所在聊天发送消息</td></tr>
<tr><td>name</td><td>桌号</td></tr>
<tr><td>pls / obs / gms</td><td>玩家/旁观/管理，均为Set</td></tr>
<tr><td>rule</td><td>当前规则集名，对应.rule set</td></tr>
<tr><td>log_name / log_file</td><td>日志名与日志文件</td></tr>
<tr><td>其余字段</td><td>game.任意名读取.game set写入的自定义项</td></tr>
</tbody></table>

### log(info[,notice_level])

*(build598+)*生成指定通知类型的日志，可同时发送到复数类别通知窗口

```lua
log("这是一条通知",1,2)
```

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>日志内容</td><td>string</td><td>待输出日志内容</td></tr>
<tr><td>通知窗口级别</td><td>number</td><td>选填，若空则只输出到框架日志界面</td></tr>
</tbody></table>


### loadLua(scriptName)

运行Lua文件，返回目标脚本的返回值
参数使用相对路径且无后缀，根目录为plugin文件夹*(build575+)*或mod内script文件夹*(build575+)*
与Lua自带的require函数不同，目标文件定义的变量会保持生命周期

```lua
loadLua("PC/COC7")
```
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>lua文件名</td><td>string</td><td>待调用mod/script/文件或plugin/文件</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>同文件内返回值类型</td><td>执行指定文件后的返回值</td></tr>
</tbody></table>

### ranint(low,high)

取随机数
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>随机区间下限</td><td>number</td><td>整数</td></tr>
<tr><td>随机区间上限</td><td>number</td><td>整数</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>number</td><td>生成随机数</td></tr>
</tbody></table>
### getDiceQQ()

取DiceMaid自身账号
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>string</td><td>取骰子自身账号</td></tr>
</tbody></table>

### getDiceDir()

取Dice存档目录，用于自行读写文件
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>string</td><td>取Dice存档目录</td></tr>
</tbody></table>

### getSelfData(fileName)

取`存档目录/selfdata/`下的数据文件，返回**SelfData**（见[SelfData元表](#selfdata元表)）。文件不存在时自动新建。

```lua
local conf = getSelfData("favor.json")
```

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>文件名</td><td>string</td><td>相对存档目录/selfdata/，可带子目录；后缀决定存储格式：.json/.toml/.yaml(.yml)，无后缀按二进制处理，基本不用</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>SelfData</td><td>该数据文件的读写句柄</td></tr>
</tbody></table>

### eventMsg(msg, gid, uid)

虚构一条消息进行处理，不计入指令频度。可使用参数列表`eventMsg(msg, gid, uid)`或*(build608+)*参数包形式`eventMsg(pkg)`.

```lua
eventMsg(".rc Rider Kick:70 踢西鹿", msg.gid, msg.uid)
eventMsg({
        fromMsg = ".rc Rider Kick:70 踢西鹿",
        gid = msg.gid,
        uid = msg.uid,
})
```
<table><thead><tr><th>输入参数/pkg子项</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>消息文本/fromMsg</td><td>string</td><td></td></tr>
<tr><td>来源群/gid</td><td>number</td><td>可以为空</td></tr>
<tr><td>发送者/uid</td><td>number</td><td></td></tr>
</tbody></table>

### sendMsg(msg, gid, uid)

可使用参数列表`sendMsg(msg, gid, uid)`或*(build619+)*参数包形式`sendMsg(pkg)`发送.

```lua
sendMsg("早安哟", msg.fromGroup, msg.fromQQ)
```
<table><thead><tr><th>输入参数/pkg子项</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>fwdMsg</td><td>string</td><td>待发送消息</td></tr>
<tr><td>gid</td><td>number</td><td>私聊时为空</td></tr>
<tr><td>uid</td><td>number</td><td>群聊时可以为空</td></tr>
<tr><td>chid</td><td>number</td><td>频道id，仅参数包可用</td></tr>
</tbody></table>

### getUserToday(userID, keyConf, defaultVal)

取用户今日数据项。所有当日数据会在系统时间24时清空。

```lua
getUserToday(msg.uid, "jrrp")
```
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>number</td><td></td></tr>
<tr><td>配置项</td><td>string</td><td>待取配置项</td></tr>
<tr><td>候补值</td><td>任意</td><td>配置项不存在时返回该值，为空则返回0</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead><tbody>
<tr><td>任意</td><td>待取值</td></tr>
</tbody></table>
<table><thead><tr><th>内嵌字段</th><th>说明</th></tr></thead><tbody>
<tr><td>jrrp</td><td>今日人品（.jrrp结果，不存在时会从服务器统一获取并存入本地）</td></tr>
<tr><td>frq</td><td>当日指令量（uid取0时，为全局总指令量）</td></tr>
</tbody></table>

### setUserToday(userID, keyConf, val)

存用户今日数据项

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>number</td><td></td></tr>
<tr><td>配置项</td><td>string</td><td>待存配置项</td></tr>
<tr><td>配置值</td><td>任意</td><td>待存入数据</td></tr>
</tbody></table>

### getUserConf(userID, keyConf, defaultVal)

取用户配置，配置项带\*标记表示会另行计算而非调用存储数据。*(build613+)*参数1可以为空，此时遍历所有**记录了该属性**的用户并返回以账号=属性值为键值对的table。

```lua
getUserConf(msg.fromQQ, "favor", 0)
getUserConf(nil, "favor") --返回所有用户的favor列表
```
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>number</td><td></td></tr>
<tr><td>配置项</td><td>string</td><td>待取配置项</td></tr>
<tr><td>候补值</td><td>任意</td><td>配置项不存在时返回该值</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>任意</td><td>待取值</td></tr>
</tbody></table>
<table><thead><tr><th>特殊配置项</th><th>说明</th></tr></thead><tbody>
<tr><td>trust</td><td>用户信任（仅4以下可编辑）</td></tr>
<tr><td>firstCreate</td><td>用户记录创建（初次使用）时间 [时间戳，秒]</td></tr>
<tr><td>lastUpdate</td><td>用户记录最后更新时间 [时间戳，秒]</td></tr>
<tr><td>name</td><td>*用户账号昵称（只读）</td></tr>
<tr><td>nick</td><td>*全局称呼（备取账号昵称）</td></tr>
<tr><td>nick#群号</td><td>*特定群内的称呼（备取群名片->全局称呼->账号昵称）</td></tr>
<tr><td>nn</td><td>*全局nn</td></tr>
<tr><td>nn#群号</td><td>*特定群内的nn</td></tr>
</tbody></table>

### setUserConf(userID, keyConf, val)

存用户配置项
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>number</td><td></td></tr>
<tr><td>配置项</td><td>string</td><td>待存配置项</td></tr>
<tr><td>配置值</td><td>任意</td><td>待存入数据</td></tr>
</tbody></table>

### getGroupConf(groupID, keyConf, defaultVal)

取群配置，配置项带\*标记表示会另行计算而非调用存储数据。*(build613+)*群号可以为空，此时遍历所有**记录了该属性**的群并返回以群号=属性值为键值对的table。

```lua
getGroupConf(msg.gid, "rc房规", 0)
```
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>群号</td><td>number</td><td></td></tr>
<tr><td>配置项</td><td>string</td><td>待取配置项</td></tr>
<tr><td>候补值</td><td>任意</td><td>配置项不存在时返回该值</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>任意</td><td>待取值</td></tr>
</tbody></table>
<table><thead><tr><th>特殊配置项</th><th>说明</th></tr></thead><tbody>
<tr><td>name</td><td>*群名称（只读）</td></tr>
<tr><td>size</td><td>*群人数（只读）</td></tr>
<tr><td>maxsize</td><td>*群规模（只读）</td></tr>
<tr><td>firstCreate</td><td>用户记录创建（初次使用）时间 [时间戳，秒]</td></tr>
<tr><td>lastUpdate</td><td>用户记录最后更新时间 [时间戳，秒]</td></tr>
<tr><td>members</td><td>(build615+)*群用户列表</td></tr>
<tr><td>admins</td><td>(build615+)*群管理列表</td></tr>
<tr><td>card#群员账号</td><td>(build615+)*群名片</td></tr>
<tr><td>auth#群员账号</td><td>(build615+)*群权限（只读） 1-群员;2-管理;3-群主</td></tr>
<tr><td>lst#群员账号</td><td>(build615+)*最后发言时间（只读） [时间戳，秒]</td></tr>
</tbody></table>


### setGroupConf(groupID, keyConf, val)

存群配置项
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>群号</td><td>number</td><td></td></tr>
<tr><td>配置项</td><td>string</td><td>待存配置项</td></tr>
<tr><td>配置值</td><td>任意</td><td>待存入数据</td></tr>
</tbody></table>

### getPlayerCard(userID, groupIDOrName)

取角色卡，返回**Actor**（不是table）。第二参数可传群号（取该群绑定的卡），也可传卡名（string）；都省略时取默认卡。

```lua
local pc = getPlayerCard(msg.uid, msg.gid)      --本群绑定的卡
local alt = getPlayerCard(msg.uid, "调查员")     --按卡名取
```

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>number</td><td></td></tr>
<tr><td>群号或卡名</td><td>number / string</td><td>群号取该群绑定的卡；字符串按卡名取；为空取默认卡</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>Actor</td><td>角色卡，可用pc.属性名、pc:rollDice()等，见[Actor元表](#actor元表)</td></tr>
</tbody></table>

### getPlayerCardAttr(userID, groupID, keyAttr, defaultVal)

取角色卡属性

```lua
getPlayerCardAttr(msg.fromQQ, msg.fromGroup, "理智", val_default)
```
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>number</td><td></td></tr>
<tr><td>群号</td><td>number</td><td></td></tr>
<tr><td>属性名</td><td>string</td><td>待取属性</td></tr>
<tr><td>候补值</td><td>任意</td><td>属性不存在时返回该值</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>任意</td><td>待取属性</td></tr>
</tbody></table>

### setPlayerCardAttr(userID, groupID, keyConf, val)

存角色卡属性
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>number</td><td></td></tr>
<tr><td>群号</td><td>number</td><td></td></tr>
<tr><td>属性名</td><td>string</td><td>待存属性</td></tr>
<tr><td>属性值</td><td>任意</td><td>待存数据</td></tr>
</tbody></table>

### mkDirs(pathDir)

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>文件夹路径</td><td>string</td><td>递归创建该文件夹</td></tr>
</tbody></table>

### sleepTime(ms)
<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>等待毫秒数</td><td>number</td><td></td></tr>
</tbody></table>
### http

*(build590+)*

#### http.get

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>待访问url</td><td>string</td><td>若含须转义字符须用urlEncode转义</td></tr>
</tbody></table>
<table><thead><tr><th>返回值</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>连接是否成功</td><td>boolean</td><td></td></tr>
<tr><td>网页返回内容</td><td>string</td><td>访问失败则返回错误原因</td></tr>
</tbody></table>
#### http.post

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>待访问url</td><td>string</td><td>若含须转义字符须用urlEncode转义</td></tr>
<tr><td>post数据</td><td>string</td><td>(build634+)如为table则自动序列化为json格式</td></tr>
<tr><td>header</td><td>string</td><td>(build606+)可省略，默认Content-Type: application/json，(build629+)如为table将自动拼接</td></tr>
</tbody></table>
<table><thead><tr><th>返回值</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>连接是否成功</td><td>boolean</td><td></td></tr>
<tr><td>网页返回内容</td><td>string</td><td>访问失败则返回错误原因</td></tr>
</tbody></table>
#### http.urlEncode

将url中须转义的字符进行转义。

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>待编码url</td><td>string</td><td></td></tr>
</tbody></table>
<table><thead><tr><th>返回值</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>编码后url</td><td>string</td><td></td></tr>
</tbody></table>

#### http.urlDecode

还原url中转义的字符。

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>待解码url</td><td>string</td><td></td></tr>
</tbody></table>
<table><thead><tr><th>返回值</th><th>变量类型</th><th>说明</th></tr></thead><tbody
<tr><td>解码后url</td><td>string</td><td></td></tr>
</tbody></table>
## 附录：Dice预置JavaScript原型及全局函数

*(build640+)*本附录对应Dice!2.7.1，共**14个全局函数**，另有`Context`、`Set`、`SelfData`、`GameTable`、`Actor`五个原型/类型。各函数的参数顺序与语义与Lua侧基本一致，差异集中在：

- 没有`loadLua`/`ranint`/`mkDirs`/`sleepTime`/`getPlayerCardAttr`/`setPlayerCardAttr`。取随机数用`Math.random()`，文件操作请import`os`；
- 读取用户/群配置的函数名为`getUserAttr`/`setUserAttr`/`getGroupAttr`/`setGroupAttr`；
- 加载其他脚本用`loadJS(文件名)`（先查mod的`script/`，再查`程序目录/Diceki/js/`，按ES模块执行）；
- 上下文对象即全局`msg`（事件中为`this`），字段与Lua一致，见[msg常用字段](#msg常用字段)。

### Actor原型

*(build644+)* **角色卡Actor**。由`getPlayerCard(uid[, gid|name])`取得。属性可直接读写（`pc.生命`、`pc["生命"]`），`__`开头为元属性（`pc.__Name`）。

<table><thead><tr><th>成员</th><th>说明</th></tr></thead><tbody>
<tr><td>set(key, val)</td><td>写属性（也可直接`pc[key] = val`）</td></tr>
<tr><td>属性读取</td><td>直接`pc[key]`或`pc.生命`，无`get`方法</td></tr>
<tr><td>rollDice(exp)</td><td>按角色卡默认骰掷骰，见下</td></tr>
<tr><td>locked(key)</td><td>属性是否被锁定，返回boolean</td></tr>
<tr><td>lock(key) / unlock(key)</td><td>加锁/解锁，返回boolean</td></tr>
</tbody></table>

#### rollDice(exp)

调用角色卡的默认骰*(__DefaultDice)*及默认掷骰表达式*(__DefaultDiceExp)*进行掷骰，返回object记录掷骰结果。

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>掷骰表达式</td><td>String</td><td></td></tr>
</tbody></table>

<table><thead><tr><th>返回值字段</th><th>字段类型</th><th>说明</th></tr></thead><tbody>
<tr><td>expr</td><td>String</td><td>规范化后的表达式</td></tr>
<tr><td>sum</td><td>Number</td><td>掷骰结果（表达式合法时）</td></tr>
<tr><td>expansion</td><td>String</td><td>掷骰展开式（表达式合法时）</td></tr>
<tr><td>error</td><td>Number</td><td>错误类型（表达式非法时）</td></tr>
</tbody></table>

### Set

*(build646+)*集合，方法`add(v)`、`in(v)`、`toArray()`，用`new Set()`或`Set()`新建：

```javascript
const list = new Set();
list.add(msg.uid);
if (list.in(10001)) { msg.echo("已在名单内"); }
for (const uid of list.toArray()) { /* ... */ }
```

### SelfData

`getSelfData("文件名")`取`存档目录/selfdata/`下的数据文件，字段直接读写，写入即落盘：

```javascript
const conf = getSelfData("favor.json");
const limit = conf.daily_limit || 3;
conf.total = (conf.total || 0) + 1;
conf.append("daily_limit", limit);   //等价于 conf.daily_limit = limit
```

### GameTable

*(build649+)*`msg.game`取得，仅局面存在时可用。可读写字段（`game.rule = "COC7"`），并可用`message`向桌面发送消息：

```javascript
const game = msg.game;
if (!game) { msg.echo("{help:game}"); } else { game.message("请各位投掷先攻"); }
```

<table><thead><tr><th>成员</th><th>说明</th></tr></thead><tbody>
<tr><td>message(文本)</td><td>向桌面所在聊天发送消息</td></tr>
<tr><td>name</td><td>桌号</td></tr>
<tr><td>pls / obs / gms</td><td>玩家/旁观/管理，均为集合</td></tr>
<tr><td>rule</td><td>当前规则集名</td></tr>
</tbody></table>

### log(info[,notice_level])

发送日志

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>日志内容</td><td>String</td><td>待输出日志内容</td></tr>
<tr><td>通知窗口级别</td><td>Number</td><td>选填，若空则只输出到框架日志界面</td></tr>
</tbody></table>

### getDiceID()

取DiceMaid自身账号

<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>Number</td><td>取骰子自身账号</td></tr>
</tbody></table>
### getDiceDir()

取Dice存档目录，用于自行读写文件

<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>String</td><td>取Dice存档目录</td></tr>
</tbody></table>

### getSelfData(fileName)

取`存档目录/selfdata/`下的数据文件，返回SelfData对象，用法见上文[SelfData](#selfdata)。

```javascript
const conf = getSelfData("favor.json");
```

### getPlayerCard(userID[, groupID][, name])

取角色卡，返回**Actor**（见上文[Actor原型](#actor原型)）。第二参数传群号取该群绑定的卡，传卡名按名取，省略取默认卡。

```javascript
const pc = getPlayerCard(msg.uid, msg.gid);
msg.echo(pc.rollDice("1d100").expansion);
```

### loadJS(fileName)

*(build640+)*加载并执行另一份js，返回其导出内容。先按脚本名查mod的`script/`，再查`程序目录/Diceki/js/`，以ES模块方式执行。

```javascript
const util = loadJS("common/util");
```

### eventMsg(msg, gid, uid)

虚构一条消息进行处理，不计入指令频度。可使用参数列表`eventMsg(msg, gid, uid)`或*(build608+)*参数包形式`eventMsg(pkg)`.

```javascript
eventMsg(".rc Rider Kick:70 踢", msg.gid, msg.uid);
eventMsg({
        fromMsg: ".rc Rider Kick:70 踢",
        gid: msg.gid,
        uid: msg.uid,
});
```

<table><thead><tr><th>输入参数/pkg子项</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>消息文本/fromMsg</td><td>String</td><td></td></tr>
<tr><td>来源群/gid</td><td>Number</td><td>可以为空</td></tr>
<tr><td>发送者/uid</td><td>Number</td><td></td></tr>
</tbody></table>

### sendMsg(msg, gid, uid)

可使用参数列表`sendMsg(msg, gid, uid)`或*(build619+)*参数包形式`sendMsg(pkg)`发送.

```javascript
sendMsg("早安哟", msg.fromGroup, msg.fromQQ);
sendMsg({ fwdMsg: "早安哟", gid: msg.gid, uid: msg.uid });
```

<table><thead><tr><th>输入参数/pkg子项</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>fwdMsg</td><td>String</td><td>待发送消息</td></tr>
<tr><td>gid</td><td>Number</td><td>私聊时为空</td></tr>
<tr><td>uid</td><td>Number</td><td>群聊时可以为空</td></tr>
<tr><td>chid</td><td>Number</td><td>频道id，仅参数包可用</td></tr>
</tbody></table>

### getUserToday(userID, keyConf, defaultVal)

取用户今日数据项。特别地，配置项为"jrrp"时，所取值同`.jrrp`结果。所有当日数据会在系统时间24时清空。

```javascript
getUserToday(msg.uid, "jrrp")
```

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>Number</td><td></td></tr>
<tr><td>配置项</td><td>String</td><td>待取配置项</td></tr>
<tr><td>候补值</td><td>任意</td><td>配置项不存在时返回该值，为空则返回0</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead><tbody>
<tr><td>任意</td><td>待取值</td></tr>
</tbody></table>

### setUserToday(userID, keyConf, val)

存用户今日数据项

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>Number</td><td></td></tr>
<tr><td>配置项</td><td>String</td><td>待存配置项</td></tr>
<tr><td>配置值</td><td>任意</td><td>待存入数据</td></tr>
</tbody></table>

### getUserAttr(userID, keyConf, defaultVal)

取用户配置，配置项带\*标记表示会另行计算而非调用存储数据。*(build613+)*参数1可以为空，此时遍历所有**记录了该属性**的用户并返回以账号=属性值为键值对的table。

```javascript
getUserAttr(msg.uid, "favor", 0);
getUserAttr(undefined, "favor");    //省略用户号（undefined）返回所有用户的favor表
```

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>Number</td><td></td></tr>
<tr><td>配置项</td><td>String</td><td>待取配置项</td></tr>
<tr><td>候补值</td><td>任意</td><td>配置项不存在时返回该值</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>任意</td><td>待取值</td></tr>
</tbody></table>
<table><thead><tr><th>特殊配置项</th><th>说明</th></tr></thead><tbody>
<tr><td>trust</td><td>用户信任（仅4以下可编辑）</td></tr>
<tr><td>firstCreate</td><td>用户记录创建（初次使用）时间 [时间戳，秒]</td></tr>
<tr><td>lastUpdate</td><td>用户记录最后更新时间 [时间戳，秒]</td></tr>
<tr><td>name</td><td>*用户账号昵称（只读）</td></tr>
<tr><td>nick</td><td>*全局称呼（备取账号昵称）</td></tr>
<tr><td>nick#群号</td><td>*特定群内的称呼（备取群名片->全局称呼->账号昵称）</td></tr>
<tr><td>nn</td><td>*全局nn</td></tr>
<tr><td>nn#群号</td><td>*特定群内的nn</td></tr>
</tbody></table>


### setUserAttr(userID, keyConf, val)

存用户配置项

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>用户账号</td><td>Number</td><td></td></tr>
<tr><td>配置项</td><td>String</td><td>待存配置项</td></tr>
<tr><td>配置值</td><td>任意</td><td>待存入数据</td></tr>
</tbody></table>

### getGroupAttr(groupID, keyConf, defaultVal)

取群配置，配置项带\*标记表示会另行计算而非调用存储数据。*(build613+)*群号可以为空，此时遍历所有**记录了该属性**的群并返回以群号=属性值为键值对的table。

```javascript
getGroupAttr(msg.gid, "rc房规", 0)
```

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>群号</td><td>Number</td><td></td></tr>
<tr><td>配置项</td><td>String</td><td>待取配置项</td></tr>
<tr><td>候补值</td><td>任意</td><td>配置项不存在时返回该值</td></tr>
</tbody></table>
<table><thead><tr><th>返回值类型</th><th>说明</th></tr></thead>
<tbody><tr><td>任意</td><td>待取值</td></tr>
</tbody></table>
<table><thead><tr><th>特殊配置项</th><th>说明</th></tr></thead><tbody>
<tr><td>name</td><td>*群名称（只读）</td></tr>
<tr><td>size</td><td>*群人数（只读）</td></tr>
<tr><td>maxsize</td><td>*群规模（只读）</td></tr>
<tr><td>firstCreate</td><td>用户记录创建（初次使用）时间 [时间戳，秒]</td></tr>
<tr><td>lastUpdate</td><td>用户记录最后更新时间 [时间戳，秒]</td></tr>
<tr><td>members</td><td>群用户列表</td></tr>
<tr><td>admins</td><td>群管理列表</td></tr>
<tr><td>card#群员账号</td><td>*群名片</td></tr>
<tr><td>auth#群员账号</td><td>*群权限（只读） 1-群员;2-管理;3-群主</td></tr>
<tr><td>lst#群员账号</td><td>*最后发言时间（只读） [时间戳，秒]</td></tr>
</tbody></table>

### setGroupAttr(groupID, keyConf, val)

存群配置项

<table><thead><tr><th>输入参数</th><th>变量类型</th><th>说明</th></tr></thead><tbody>
<tr><td>群号</td><td>Number</td><td></td></tr>
<tr><td>配置项</td><td>String</td><td>待存配置项</td></tr>
<tr><td>配置值</td><td>任意</td><td>待存入数据</td></tr>
</tbody></table>

## 附录：Dice预置Python类及模块函数

*(build639+)*本附录对应Dice!2.7.1，`dicemaid`模块共**13条函数**，另有`Context`、`Actor`、`GameTable`、`SelfData`四个类型。

```python
# 无需导入：全局初始化时已执行 from dicemaid import *
msg.echo("hello,world")
```

与Lua侧的能力差异（**照抄Lua代码前务必核对**）：

- **没有**`Set`类型与`http`系列函数；
- **没有**`ranint`、`mkDirs`、`sleepTime`、`loadLua`、`getPlayerCardAttr`、`setPlayerCardAttr`。取随机数用`random.randint(low, high)`，建目录用`os.makedirs(path, exist_ok=True)`，延时用`time.sleep(秒)`，角色卡属性走`Actor`；
- 取用户/群配置的函数名是`getUserAttr`/`setUserAttr`/`getGroupAttr`/`setGroupAttr`（不是Lua的`getUserConf`）；
- **`sendMsg`的参数顺序与Lua/JS相反**：`sendMsg(msg, uid, gid[, chid])`。

### 函数总表

| 函数 | 参数 | 说明 |
| --- | --- | --- |
| `log` | `info[, level]` | 输出日志，`level`为通知窗口级别位 |
| `getDiceID` | 无 | 骰娘自身账号（int） |
| `getDiceDir` | 无 | 存档目录路径（str） |
| `getSelfData` | `文件名` | 取SelfData对象，见下 |
| `getGroupAttr` | `gid, key[, sub]` | 取群配置；`gid`传`None`时返回全部记录了该属性的群（dict）；`key`为`members`时可用`sub`取`card`/`auth`/`lst` |
| `setGroupAttr` | `gid, key, val` | 存群配置 |
| `getUserAttr` | `uid, key[, default]` | 取用户配置；`uid`传`None`时返回全部记录了该属性的用户（dict） |
| `setUserAttr` | `uid, key, val` | 存用户配置 |
| `getUserToday` | `uid, key[, default]` | 取用户今日数据（`jrrp`为今日人品） |
| `setUserToday` | `uid, key, val` | 存用户今日数据 |
| `getPlayerCard` | `uid[, gid][, name]` | 取角色卡，返回**Actor** |
| `sendMsg` | `msg, uid, gid[, chid]` | 向指定目标发送消息，`msg`可传文本或含`fwdMsg/uid/gid/chid`的dict |
| `eventMsg` | `msg, uid, gid` | 虚构一条消息走一遍完整处理，不计入指令频度 |

注意：**账号传`0`会抛`ValueError`**，要遍历全部请传`None`。

`getGroupAttr`支持的只读项：`name`群名、`size`群人数、`maxsize`群规模、`members`成员列表、`admins`管理列表、`card#群号`群名片、`auth#群号`权限(1群员/2管理/3群主)、`lst#群号`最后发言时间。
`getUserAttr`支持的只读项：`name`昵称、`nick`称呼、`nick#群号`群内称呼、`nn`/`nn#群号`、`trust`信任、`firstCreate`/`lastUpdate`。

### Context类

消息、事件、回复的上下文对象，字段用法与Lua一致：`self.uid`、`self.gid`、`self.fromMsg`、`self.suffix`……，也支持`self["uid"]`与直接赋值`self.gift = 1`（随后可在回执里用`{gift}`引用）。回复脚本里这个对象另有个名字叫`msg`，两者是同一个。

- 字段读取有两条路径，行为不同：**属性式**（`msg.不存在`）返回`None`；**下标式**（`msg["不存在"]`）抛`KeyError`，与dict一致；
- 只有3个方法，**没有`get`**。回复脚本里上下文对象叫`msg`（另有别名`self`），事件脚本里它就是局部命名空间，用裸名`self`引用：

| 方法 | 说明 |
| --- | --- |
| `echo(text[, isRaw])` | 立即回复；`isRaw`为真时不转义 |
| `format(text)` | 在该上下文中转义文本并返回str |
| `inc(field[, cnt])` | 变量自增，`cnt`默认1 |

### Actor类

由`getPlayerCard`取得，代表一张角色卡。属性可直接读写（`pc.理智`、`pc["理智"]`），以`__`开头的是元属性（`pc.__Name`卡名、`pc.__DefaultDice`默认骰）。

| 方法 | 说明 |
| --- | --- |
| `get(key)` | 取属性 |
| `set(key, val)` | 存属性 |
| `rollDice(exp)` | 按角色卡默认骰掷骰，返回含`expr`/`sum`/`expansion`/`error`的dict |
| `locked(key)` | 该属性是否被锁定（bool） |
| `lock(key)` | 加锁，返回bool |
| `unlock(key)` | 解锁，返回bool |

```python
pc = getPlayerCard(msg.uid, msg.gid)
if pc.locked("理智"):
    msg.echo("理智已被锁定，无法修改")
else:
    pc.set("理智", pc.get("理智", 0) - 1)
    rd = pc.rollDice("1d100")
    msg.echo("理智扣除1点，检定：" + rd.get("expansion", ""))
```

### GameTable类

`msg.game`取得当前聊天窗口的桌（无桌时为`None`，`game`指令的`new`后才有）。属性可直接读写（`msg.game.rule = "COC7"`）。

| 成员 | 说明 |
| --- | --- |
| `game.get(key)` / `game[key]` | 取桌面字段 |
| `name` | 桌号 |
| `pls` / `obs` / `gms` | 玩家/旁观/管理，返回Python的`set`，可直接遍历 |
| `rule` | 当前规则集名 |
| `log_name` / `log_file` | 日志名与日志文件 |

### SelfData类

`getSelfData("文件名")`取`存档目录/selfdata/`下的数据文件（json/toml/yaml），可属性式或下标式读写，适合存模块级配置与累计数据。方法只有`get(key)`与`set(key, val)`/`set(整份数据)`。

```python
conf = getSelfData("favor.json")
limit = conf.daily_limit or 3          #属性式读取（也可 conf["daily_limit"]）
conf.total = (conf.total or 0) + 1     #属性式写入，立即落盘
conf.set("daily_limit", limit)         #等价的按键写入
```

## 附录：自定义指令常用的Lua正则匹配

解析参数时，可使用`msg.suffix`来略过指令前缀匹配的部分，从之后的字符开始匹配。

```lua
--前缀匹配且指令参数为消息余下部分时，去除前后两端的空格
local rest = msg.suffix:match("^[%s]*(.-)[%s]*$")

--指令参数为单项整数时，直接用%d+表示匹配一个或多个数字，未输入数字时匹配失败返回nil
local cnt = string.match(msg.suffix,"%d+")

--同上，%d*表示匹配0或任意个数字，未输入数字时匹配成功返回空字符串""
--该匹配模式需要确保数字之前的其他字符已被排除
local cnt = string.match(msg.suffix,"%d*")

--参数使用空格分隔且不限数目，遍历匹配
local item,rest = "",string.match(msg.fromMsg,"^[%s]*(.-)[%s]*$",#order_select_name+1)
if(rest == "")then
    return "请输入参数"
end
local items = {}
repeat
    item,rest = string.match(rest,"^([^%s]*)[%s]*(.-)$")
    table.insert(items, item)
until(rest=="")
```

## 附录：常用在线语法校验/运行工具

- json: [在线JSON校验格式化工具](https://www.bejson.com/)
- toml: [在线TOML转YAML工具](https://tooltt.com/toml2yaml/)
- yaml: [YAML、YML在线编辑器(格式化校验)](https://www.bejson.com/validators/yaml_editor/)
- lua: [在线运行Lua](https://www.bejson.com/runcode/lua/)
  *所用Lua版本(5.3.5)略旧于Dice!内置(5.4)，可能产生差异。*
- javascript: [在线运行JavaScript](https://www.bejson.com/runcode/javascript/)
  *注意对js标准的支持存在差异。*
- python: [在线运行Python(3.8.1)](https://www.bejson.com/runcode/python3/)
  *所用Python版本(3.8.1)明显旧于Dice!内置的3.12，`match`语法、f-string细节等可能有差异。*
- lua/javascript/python: [Judge0 IDE - Free and open-source online code editor](https://ide.judge0.com/)

## 附录：lua常见报错说明

```lua
module '%s' not found:
	no field package.preload['%s']
	no file 
-- 没有把被引用的lua或dll文件放在指定位置（多见于require与loadLua）
-- 解决方式：把所需文件放入Dice存档目录/plugin/或Diceki/lua/，dll文件或require对象必须置于后者
attempt to call a nil value (global '%s')
-- 将空变量%s用作函数（global表示被调用的是全局变量，local表示本地变量，method表示索引方法）
attempt to index a nil value (global '%s')
-- 对空变量%s使用索引（只有table等结构可以索引，形如msg.fromMsg）
attempt to concatenate a nil value (local '%s')
-- 使用..连接字符串时连接了一个空变量%s
bad argument #1 to '%s' (string expected, got nil)
-- 函数%s的第1个参数类型错误，要求类型为string，但实际传入的参数为nil。特别地，got nil表示输入参数为nil或缺少参数
value expected
-- 要求参数，但没有传入
attempt to perform arithmetic on a nil value (global '%s')
-- 将一个空变量%s当做数字表示
bad argument #5 to 'format'(number has no integer representation)
-- 函数format的第5个参数类型错误，要求是整数，但传入是小数，或者是其余类型不能化为整数
'}' expected (to close '{' at line 169) near '%s'
-- 脚本第169行的左花括号缺少配对的右花括号。此错误也可以由表格内缺少逗号分隔、括号外的中文等原因造成
'end' expected (to close 'function' at line 240) near <eof>
-- 脚本第240行的function缺少收尾的end，<eof>表示文件结束（找到文件末也没找到）
'then' expected near 'end'
-- if then end逻辑结构缺少then
unexpected symbol near '%s'
-- 符号%s边有无法识读的符号，比如中文字符
attempt to get length of a nil value (local 'tab')
-- 对空变量tab作取长度运算（#）
attempt to add a 'string' with a 'string'
-- 对(不能化为数字的)字符串用加法'+'（字符串只能用连接'..'）
attempt to compare number with string
-- 对数字和(不能化为数字的)字符串用比较运算符
error loading module '%s' from file
-- 使用require "%s"时加载出错
no visible label '%s' for <goto> at line 
-- 在循环结构中跳转不存在的节点
invalid option '%s'
-- 传入的参数不是string或不在给定的字符串列表中
malformed number near '86400..'
-- 数字与字符连接符粘连导致误判为小数点，使得格式解析错误，需要插入空格
```

## 附录：事件event样例

### 内置时点

<table><thead><tr><th>时点</th><th>说明</th></tr></thead><tbody>
    <tr><td>StartUp</td><td>启动完成后</td></tr>
    <tr><td>FriendRequest</td><td>好友申请后，处理前</td></tr>
    <tr><td>FriendAdd</td><td>自动或被同意添加好友后，回复前</td></tr>
    <tr><td>GroupRequest</td><td>受群邀请后，处理前</td></tr>
    <tr><td>GroupKicked</td><td>被踢出群后，拉黑前</td></tr>
    <tr><td>GroupBanned</td><td>被禁言后，拉黑前</td></tr>
    <tr><td>GroupAdd</td><td>新入群后，欢迎前</td></tr>
    <tr><td>GroupAuthorize</td><td>群申请许可后，处理前</td></tr>
    <tr><td>LogEnd</td><td>日志完成后，上传前</td></tr>
    <tr><td>WhisperIgnored</td><td>私聊不识别为指令且未触发回复</td></tr>
    <tr><td>BlackInfoWarned</td><td>*(build664+)*收到其他骰娘/服务方的黑名单预警时，event.warning为预警条目</td></tr>
    <tr><td>DayEnd</td><td>（时差修正后）0点后刷新每日数据前</td></tr>
    <tr><td>DayNew</td><td>刷新每日数据后</td></tr>
</tbody></table>
- *逻辑上被踢出群不含解散，被禁言不含全群禁言*
- *GroupAuthorize、LogEnd、WhisperIgnored时点在消息处理时，对应Event为Message，其余事件Event为其自身*
- *StartUp、DayEnd、DayNew在独立线程中执行，可以安全地做耗时操作*

<table><thead><tr><th>event字段</th><th>说明</th></tr></thead><tbody>
    <tr><td>Event</td><td>基础事件名，多为Message</td></tr>
    <tr><td>uid</td><td>当事用户（同fromQQ/fromUser）</td></tr>
    <tr><td>gid</td><td>当事群</td></tr>
    <tr><td>fromMsg</td><td>消息文本/好友申请文本</td></tr>
    <tr><td>aimGroup</td><td>GroupAuthorize中待审核群</td></tr>
    <tr><td>AttachInfo</td><td>GroupAuthorize中申请文本</td></tr>
    <tr><td>log_file / log_path</td><td>LogEnd中日志文件名及路径</td></tr>
    <tr><td>warning</td><td>BlackInfoWarned中的预警条目</td></tr>
    <tr><td>hook</td><td>仅少数上下文自带（如BlackInfoWarned）；判断时点请用Event</td></tr>
    <tr><td>blocked</td><td>[回传]是否拦截该事件后续处理</td></tr>
    <tr><td>approval</td><td>[回传]是否同意申请</td></tr>
    <tr><td>ignored</td><td>[回传]是否忽略算入指令计数</td></tr>
</tbody></table>

回写只对当前事件生效：**不写`blocked`就等于放行**，Dice!会继续执行原生流程。

### 事件声明

一份事件声明可以只挂一种语言，也可以`action.lua`/`action.js`/`action.py`**同时挂三种**（按lua→js→py顺序执行）：

```toml
#/mod/xxx/event/hook.toml
[event.friend_request]
title = "好友申请"
trigger.hook = "FriendRequest"
action.lua = "listen_friend_request"	#或 action.js / action.py，值即script/下的文件名
```

lua也可以写成表的形式：

```lua
--/mod/xxx/event/hook.lua
event.friend_request = {
    title = "好友申请",
    trigger = { hook = "FriendRequest" },
    action = { lua = "listen_friend_request" }
}
```

### 上下文的取法

三种语言取事件字段的方式不同，**这是迁移时最需要注意的区别**：

<table><thead><tr><th>语言</th><th>上下文对象</th><th>读字段</th><th>回写</th><th>回复</th></tr></thead><tbody>
    <tr><td>Lua</td><td>全局event</td><td>event.uid</td><td>event.blocked = true</td><td>event:echo()（多数场景用log即可）</td></tr>
    <tr><td>JS</td><td>this</td><td>this.uid</td><td>this.blocked = true</td><td>this.echo() 或 sendMsg()</td></tr>
    <tr><td>Python</td><td>脚本自身的局部命名空间，用self引用它</td><td>uid 或 self.uid</td><td>blocked = True 或 self.blocked = True</td><td>self.echo(文本)；要指定目标时用 sendMsg(文本, uid, gid)</td></tr>
</tbody></table>

### Lua写法

```lua
--mod/xxx/script/listen_friend_request.lua
local answer = event.fromMsg
--如果验证方式为回答问题，则需要截去问题部分
local pos = string.find(answer, "答案：")
if pos then
    answer = string.sub(answer, pos + #"答案：")
end
local true_answer = "正确回案"
if answer == true_answer then
    log("收到" .. getUserConf(event.uid, "name") .. "(" .. event.uid .. ")的好友请求：\n"
        .. answer .. "\n回答通过√", 1)
    event.approval = true          --通过申请
else
    log("收到" .. getUserConf(event.uid, "name") .. "(" .. event.uid .. ")的好友请求：\n"
        .. answer .. "\n回答错误×", 1)
    event.approval = false
end
event.blocked = true               --终止连锁，不执行原生申请处理流程
```

```lua
--mod/xxx/script/listen_friend_add.lua
local nick = getUserConf(event.uid, "name")
getUserConf(event.uid, "firstMet", os.time())
log("新好友：" .. nick .. "(" .. event.uid .. ")", 1)
sendMsg("欢迎，" .. nick .. "使用~发送.help 协议 确认服务协议\n.help 设定 确认{self}设定\n.help 指令 查看指令列表", 0, event.uid)
```

```lua
--mod/xxx/script/listen_group_authorize.lua
local answer = event.AttachInfo or ""
local pos = string.find(answer, "答案：")
if pos then
    answer = string.sub(answer, pos + #"答案：")
end
if answer == "正确回案" then
    log("群" .. event.aimGroup .. "的许可申请通过√", 1)
    event.approval = true
else
    log("群" .. event.aimGroup .. "的许可申请被拒：\n" .. answer, 1)
    event.approval = false
end
event.blocked = true
```

```lua
--mod/xxx/script/listen_black_warned.lua
event.blocked = true                                   --先拦住，避免被自动拉黑
log("收到预警：" .. tostring(event.warning), 1)          --看清结构后再决定放行或核实
```

### JavaScript写法

事件对象是`this`（**不是`event`**）。`this`上可以直接读字段、写回字段，也带有`echo()`/`format()`/`get()`/`inc()`方法。

```javascript
//mod/xxx/script/listen_friend_request.js
let answer = this.fromMsg || "";
const pos = answer.indexOf("答案：");
if (pos >= 0) answer = answer.slice(pos + 3);
if (answer === "正确回案") {
    log("收到" + getUserAttr(this.uid, "name") + "(" + this.uid + ")的好友请求：\n"
        + answer + "\n回答通过√", 1);
    this.approval = true;          //通过申请
} else {
    log("收到" + getUserAttr(this.uid, "name") + "(" + this.uid + ")的好友请求：\n"
        + answer + "\n回答错误×", 1);
    this.approval = false;
}
this.blocked = true;               //终止连锁，不执行原生申请处理流程
```

```javascript
//mod/xxx/script/listen_friend_add.js
const nick = getUserAttr(this.uid, "name");
setUserAttr(this.uid, "firstMet", Math.floor(Date.now() / 1000));
log("新好友：" + nick + "(" + this.uid + ")", 1);
sendMsg("欢迎，" + nick + "使用~发送.help 协议 确认服务协议\n.help 设定 确认{self}设定\n.help 指令 查看指令列表", 0, this.uid);
```

```javascript
//mod/xxx/script/listen_group_authorize.js
let answer = this.AttachInfo || "";
const pos = answer.indexOf("答案：");
if (pos >= 0) answer = answer.slice(pos + 3);
if (answer === "正确回案") {
    log("群" + this.aimGroup + "的许可申请通过√", 1);
    this.approval = true;
} else {
    log("群" + this.aimGroup + "的许可申请被拒：\n" + answer, 1);
    this.approval = false;
}
this.blocked = true;
```

```javascript
//mod/xxx/script/listen_black_warned.js
this.blocked = true;                                  //先拦住，避免被自动拉黑
log("收到预警：" + JSON.stringify(this.warning), 1);   //看清结构后再决定放行或核实
```

### Python写法

**事件脚本里的上下文对象就是脚本自己的局部命名空间**：字段以裸名出现（`uid`、`fromMsg`），给同名变量赋值即写回事件；需要显式引用该对象时用`self`。回复用`self.echo(文本)`（等价于回复脚本里的`msg.echo`）；需要指定发送目标时才用`sendMsg(文本, uid, gid)`——**它的参数顺序与Lua/JS相反**。

> **事件脚本的顶层赋值写进的是上下文对象本身，而不是普通命名空间。** 上下文表只能保存可转换为`AttrVar`的值（字符串、数字、布尔、表、集合、ID）；函数、类、模块对象没有对应类型，`py_to_attr()` 转换失败会得到空值，写入时等同于删除该键。因此 `def`/`class`/`import` 执行完，名字就已经不在了，随后调用报 `name 'xxx' is not defined`。
>
> 需要在事件脚本里复用函数时，把模块放到 `存档目录/plugin/`（该目录在 `sys.path` 上），用 `exec("from myhelper import *", globals())` 注入到真正的全局命名空间 `__main__`（普通dict），之后即可正常调用。
>
> 回复脚本不受此限：它的局部命名空间就是普通dict，`def` 与 `import` 都正常。

```python
#mod/xxx/script/listen_friend_request.py
answer = fromMsg or ""
pos = answer.find("答案：")
if pos >= 0:
    answer = answer[pos + 3:]
if answer == "正确回案":
    log("收到" + getUserAttr(uid, "name") + "(" + str(uid) + ")的好友请求：\n"
        + answer + "\n回答通过√", 1)
    approval = True            #通过申请
else:
    log("收到" + getUserAttr(uid, "name") + "(" + str(uid) + ")的好友请求：\n"
        + answer + "\n回答错误×", 1)
    approval = False
blocked = True                 #终止连锁，不执行原生申请处理流程
```

```python
#mod/xxx/script/listen_friend_add.py
import time
nick = getUserAttr(uid, "name")
setUserAttr(uid, "firstMet", int(time.time()))
log("新好友：" + nick + "(" + str(uid) + ")", 1)
self.echo("欢迎，" + nick + "使用~发送.help 协议 确认服务协议\n.help 设定 确认{self}设定\n.help 指令 查看指令列表")
```

```python
#mod/xxx/script/listen_group_authorize.py
answer = AttachInfo or ""
pos = answer.find("答案：")
if pos >= 0:
    answer = answer[pos + 3:]
if answer == "正确回案":
    log("群" + str(aimGroup) + "的许可申请通过√", 1)
    approval = True
else:
    log("群" + str(aimGroup) + "的许可申请被拒：\n" + answer, 1)
    approval = False
blocked = True
```

```python
#mod/xxx/script/listen_black_warned.py
blocked = True                 #先拦住，避免被自动拉黑
log("收到预警：" + str(warning), 1)
```


## 附录：DND规则.rdc指令

.rdc(B/P) (+/-[加值]) ([检定理由]) ([成功阈值])
参数[优/劣势骰]:可选，B=2D20取大，P=2D20取小
参数[加值]:可选，加值最后修正到D20结果上
参数[检定理由]:可选，将显示在回执语句中
参数[成功阈值]:可选，将与掷骰结果比较，返回成功或失败
出于及时向用户提供指令帮助的考虑，空参将返回帮助

```lua
msg_order = {}

function intostring(num)
    if(num==nil)then
        return ""
    end
    return string.format("%.0f",num)
end

--DND
function roll_d20(msg)
    local rest=string.match(msg.fromMsg,"^[%s]*(.-)[%s]*$", 5)
    if(rest=="")then
        return [[
D20检定:.rdc
.rdc(B/P) (+/-[加值]) ([检定理由]) ([成功阈值])
参数[优/劣势骰]:可选，B=2D20取大，P=2D20取小
参数[加值]:可选，加值最后修正到D20结果上
参数[检定理由]:可选，将显示在回执语句中
参数[成功阈值]:可选，将与掷骰结果比较，返回成功或失败
空参返回本段说明文字]]
    end
    local bonus,sign,rest = string.match(rest,"^[%s]*([bBpP]?)[%s]*([+-]?)(.*)$")
    local addvalue = 0
    local modify = ""
    --加减符号判断有无加值
    if(sign ~= "")then
        modify,rest = string.match(rest,"^([%d]+)[%s]*(.*)$")
        if(modify==nil or modify=="")
        then
            return "请{pc}输入正确的加值"
        end
        addvalue = tonumber(modify)
        if(sign=='-')
        then
            addvalue = addvalue*-1;
        end
    end
    local skill,dc = string.match(rest,"^[%s]*(.-)[%s]*([%d]*)$")
    local dc = tonumber(dc)
    if(dc~=nil)then
        if(dc<1)then
            return "这你要怎么才能失败啊？"
        elseif(dc>99)then
            return "这你要怎么才能成功啊？"
        end
    end
    local die = ranint(1,20)
    local selected = die
    local res
    --考虑优势骰/劣势骰
    if(bonus=="")then
        res="D20"..sign..modify.."="..intostring(die)..sign..modify
    else
        local another=ranint(1,20)
        bonus=string.upper(bonus)
        if((another>die and bonus=="B")or(another<die and bonus=="P"))then
            selected=another
        end
        res=bonus.."("..intostring(die)..','..intostring(another)..")"..sign..modify.."->"..intostring(selected)..sign..modify
    end
    local strReply = "{pc}进行"..skill.."检定:"..res
    local final=selected+addvalue
    if(addvalue~=0)then
        strReply=strReply.."="..intostring(final)
    end
    if not dc then
        --对成功与否不做判定
    elseif(final>dc)
    then 
        strReply=strReply..'>'..intostring(dc).." {strRollRegularSuccess}"
    elseif(final<dc)
    then
        strReply=strReply..'<'..intostring(dc).." {strRollFailure}"
    else
        strReply=strReply..'='..intostring(dc).." {strRollRegularSuccess}"
    end
    return strReply
end
msg_order[".rdc"] = "roll_d20"
```

## 附录：ShadowRun规则.rsr指令

ShadowRun掷骰规则：掷X粒六面骰，点数为5或6记一次命中，过半数骰目出1记失误。自定义指令格式.rsr X。

```lua
msg_order = {}

function intostring(num)
    if(num==nil)then
        return ""
    end
    return string.format("%.0f",num)
end

--shadowrun
function roll_shadowrun(msg)
    local cnt = tonumber(string.match(msg.fromMsg,"%d+",5))
    if(not cnt)
    then
        return "shadowrun检定:.rsr[骰数] 出5/6视为命中\n出1过半视为失误"
    end
    local res="{pc}掷骰"..cnt.."D6: "
    cnt = tonumber(cnt)
    if(cnt<1)
    then
        return "{strZeroDiceErr}"
    elseif(cnt>100)
    then
        return "{strDiceTooBigErr}"
    else
        local cntSuc=0
        local cntFail=0
        local pool={}
        local glitch=""
        for times=1,cnt do
            local die=ranint(1,6)
            if(die>=5)then
                cntSuc = cntSuc + 1
            elseif(die==1)then
                cntFail = cntFail + 1
            end
            table.insert(pool,intostring(die))
        end
        if(cntFail > cnt/2)then
            glitch = " {strGlitch}"
        end
        local reply=res..table.concat(pool,'+').."->"..intostring(cntSuc)..glitch
        return reply
    end
end
msg_order[".rsr"] = "roll_shadowrun"
```

## 附录：仿MDice的.duel指令

```lua
msg_order = {} 

function intostring(num)
    if(num==nil)then
        return ""
    end
    return string.format("%.0f",num)
end

--惠惠给每个得分都写了专属回复，这里作为样例仅使用随机选取
reply_duel_win = {
    "不服？不服你就正面赢我啊",
    "你请我吃顿饭这次就算你赢，怎么样？",
    "哦吼？我觉得我甚至可以让你一个骰子",
    "（面露笑意地看着你）",
    "你觉得能赢我，嘿嘿，那是个幻觉，美丽的幻觉。",
    "再试也没用啦，会输一次就会输一百次哦",
}
reply_duel_lose = {
    "？！居然趁我一时大意。。。"
}
reply_duel_even = {
    "哼哼，平分秋色嘛"
}

function table_draw(tab)
    return tab[ranint(1,#tab)]
end

--惠系决斗
function mdice_duel(msg)
    local isDark=string.sub(msg.fromMsg,1,5)==".dark"
    local poolSelf, poolPlayer = {},{}
    local resSelf, resPlayer, resTotal, resEval = "","","",""
    local sumSelf, sumPlayer, sumTotal=0,0,0
    for times=1, 5 do
        local die = ranint(1,100)
        sumSelf = sumSelf + die
        table.insert(poolSelf,intostring(die))
    end
    resSelf="\n{self}\n"..table.concat(poolSelf,'、').."\n打点"..intostring(sumSelf)
    for times=1, 5 do
        local die = ranint(1,100)
        sumPlayer = sumPlayer + die
        table.insert(poolPlayer,intostring(die))
    end
    resPlayer="\n{pc}\n"..table.concat(poolPlayer,'、').."\n打点"..intostring(sumPlayer)
    sumTotal = sumPlayer-sumSelf;
    resTotal="\n————\n计"..intostring(sumTotal).."/10="..intostring(sumTotal/10).."分"
    if(sumSelf>sumPlayer)then
        if(isDark)then
            resEval = "发起黑暗决斗的你，应该做好觉悟了吧？"
            --好孩子不要用 eventMsg(".group ".. msg.fromGroup .." ban ".. msg.fromQQ .." "..intostring(sumSelf), 0, getDiceQQ())
        else
            resEval = table_draw(reply_duel_win)
        end
    elseif(sumSelf<sumPlayer)then
        resEval = table_draw(reply_duel_lose)
    else
        resEval = table_draw(reply_duel_even)
    end
    local reply = "看起来{pc}想向我发起决斗，呼呼，那我就接下了\n胜负的结果是——"..resSelf..resPlayer..resTotal.."\n"..resEval
    return reply --如有返回值表示回复文本，如有第二返回值表示暗骰文本
end
msg_order[".dark duel"] = "mdice_duel"
msg_order[".duel"] = "mdice_duel"
--指令注册格式 key->指令前缀 value->脚本内函数名
```

## 附录：仿SitaNya的.clock指令

```lua
msg_order = {} 

function intostring(num)
    if(num==nil)then
        return ""
    end
    return string.format("%.0f",num)
end

function print_chat(msg)
    if(msg.fromGroup == "0")then
        return "qq "..msg.fromQQ
    end
    return "group "..msg.fromGroup
end

function Sleep(n)
   --575+实现方式
   if n > 0 then sleepTime(n*1000) end
end

function sitanya_clock(msg)
    local seconds = string.match(msg.fromMsg,"%d+",7)
    if(not seconds)then
        return [[clock    计时器
出于技术验证仿塔骰写的指令，请勿滥用该指令。
用法：.clock [秒]
示例：.clock 60]]
    end
    if(tonumber(seconds)>600)then
        return "设置{self}定时失败:设置时长超过10分钟×"
    end
    eventMsg(".send "..print_chat(msg).." 您的定时器:<".. seconds ..">秒已开启计时",0,getDiceQQ())
    Sleep(seconds)
    return "{nick}的定时器:<".. seconds ..">秒已到期"
end
msg_order[".clock"] = "sitanya_clock"
```

