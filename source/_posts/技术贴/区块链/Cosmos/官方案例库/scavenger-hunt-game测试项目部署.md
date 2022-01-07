---
title: scavenger_hunt_game测试项目部署
tags:
  - cosmos
categories:
  - technical
  - cosmos
toc: true
declare: true
date: 2021-03-01 15:07:45
---

# **scavenger hunt** game

cosmos官方给出的拾荒者狩猎游戏运行**部署细节/重点记录,以及文档翻译**

官方地址:[The Game | Cosmos SDK Tutorials](https://tutorials.cosmos.network/scavenge/tutorial/02-the-game.html)

文档翻译部分来源于:https://blog.csdn.net/lk2684753/article/details/113849468

<!-- more -->

## 1.安装starport

starport是cosmos官方的项目部署脚手架工具

Mac下的安装:

==注意⚠️:要安装0.13.1版本==,因为官网的教程安装的是0.13.1

![YmWghE](http://xwjpics.gumptlu.work/qinniu_uPic/YmWghE.png)

不会安装旧版本可见:https://blog.csdn.net/weixin_43988498/article/details/114359578?spm=1001.2014.3001.5501

不能直接用如下命令,Homebrew会默认安装最新版

```shell
brew install tendermint/tap/starport
```

其他安装:[starport/2 Install.md at develop · tendermint/starport (github.com)](https://github.com/tendermint/starport/blob/develop/docs/1 Introduction/2 Install.md)

## 2.创建/拷贝项目

初始化

```shell
$ starport app --help
Generates an empty application

Usage:
  starport app [github.com/org/repo] [flags]

Flags:
      --address-prefix string   Address prefix (default "cosmos")
  -h, --help                    help for app
      --sdk-version string      Target Cosmos-SDK Version -launchpad -stargate (default "stargate")

```

```shell
$ starport app github.com/github-username/scavenge --sdk-version="launchpad"

⭐️ Successfully created a Cosmos app 'scavenge'.
👉 Get started with the following commands:

 % cd scavenge
 % starport serve

NOTE: add --verbose flag for verbose (detailed) output.
```

出现的问题:

* 问题一:

  starport requires protoc installed.

  Please, follow instructions on https://grpc.io/docs/protoc-installation

  原因:环境未安装protoc

  解决:`brew install protobuf`

* 问题二

  ```shell
  scavenge/query.proto:4:1: warning: Import google/api/annotations.proto is unused.
  
  scavenge/query.proto:5:1: warning: Import cosmos/base/query/v1beta1/pagination.proto is unused.
  
  protoc-gen-gocosmos: program not found or is not executable
  
  Please specify a program using absolute path or make sure the program is available in your PATH system variable
  
  --gocosmos_out: protoc-gen-gocosmos: Plugin failed with status code 1.
  
  : exit status 1
  ```

  原因:没有把`GOPATH/bin`加入到环境变量中,导致protoc-gen-gocosmos等无法编译可执行程序到bin目录下

  解决:添加环境变量,再次运行

  ```
  export GOPATH="/Users/XXXX/projects/go_projects"
  export PATH=$PATH:$GOROOT/bin:$GOPATH/src:$GOPATH/bin
  ```

## 3.启动项目

`starport serve`

* 问题一

  ```shell
  npm ERR! code E404
  npm ERR! 404 Not Found - GET https://registry.npm.taobao.org/@types/node/-/node-13.13.36.tgz - [not_found] document not found
  npm ERR! 404 
  npm ERR! 404  '@types/node@https://registry.npm.taobao.org/@types/node/-/node-13.13.36.tgz' is not in the npm registry.
  npm ERR! 404 You should bug the author to publish it (or use the name yourself!)
  npm ERR! 404 
  npm ERR! 404 Note that you can also install from a
  npm ERR! 404 tarball, folder, http url, or git url.
  
  npm ERR! A complete log of this run can be found in:
  npm ERR!     /Users/xwj/.npm/_logs/2021-03-01T07_24_26_696Z-debug.log
  ```

  原因:部分依赖包淘宝镜像中不存在

  解决:修改为原始的源`npm config set registry https://registry.npmjs.org/`再次启动

  淘宝源`npm config set registry https://registry.npmjs.org/`

## 4.成功启动

![ebEzzD](http://xwjpics.gumptlu.work/qinniu_uPic/ebEzzD.png)

![5wjuhQ](http://xwjpics.gumptlu.work/qinniu_uPic/5wjuhQ.png)

如果启动界面不是如上所示,那么请检查starport的版本号要 >v0.14.0

最新版下载命令`curl https://get.starport.network/starport! | bash`

## 5.添加脚手架类型

命令`starport type`,作用:**给每一个type生成CRUD的操作**

在项目文件夹下打开一个新终端，然后运行以下starport type命令来生成我们的scavenge类型

`starport type scavenge description solutionHash reward solution scavenger`

我们还要创建第二种类型，Commit以防止前面提到的提交的解决方案在前端运行

`starport type commit solutionHash solutionScavengerHash`

到目前为止,Starport脚手架帮助我们搭建了必须的文件和函数

下面将会根据游戏的需求更改这些函数与文件.

* 问题一

  `open go.mod: no such file or directory`

  执行的路径不对,需要在scavenge文件夹下执行,不然会提示找不到go.mod文件夹

## 6.Message模块

开始第一个模块的编写,CRUD

Create

Messages type已经创建在`./x/scavenge/types/`文件夹下的`MsgCommitSolution`,但是我们要在中删除掉一些我们不需要的字段

```go
package types

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)

var _ sdk.Msg = &MsgCreateScavenge{}

type MsgCreateScavenge struct {
	Creator sdk.AccAddress `json:"creator" yaml:"creator"`
	Description string `json:"description" yaml:"description"`
	SolutionHash string `json:"solution_hash" yaml:"solution_hash"`
	Reward sdk.Coins `json:"reward" yaml:"reward"`
}


func NewMsgCreateScavenge(creator sdk.AccAddress, description string, solutionHash string, reward sdk.Coins) *MsgCreateScavenge {
	return &MsgCreateScavenge{
		Creator:      creator,
		Description:  description,
		SolutionHash: solutionHash,
		Reward:       reward,
	}
}

func (msg *MsgCreateScavenge) Route() string {
	return RouterKey
}

func (msg *MsgCreateScavenge) Type() string {
	return "CreateScavenge"
}

func (msg *MsgCreateScavenge) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Creator}
}

func (msg *MsgCreateScavenge) GetSignBytes() []byte {
	bz := ModuleCdc.MustMarshalJSON(msg)
	return sdk.MustSortJSON(bz)
}

//基本验证
func (msg *MsgCreateScavenge) ValidateBasic() error {
	if msg.Creator.Empty() {
		return sdkerrors.Wrap(sdkerrors.ErrInvalidAddress, "creator can't be empty")
	}
	if msg.SolutionHash == "" {
		return sdkerrors.Wrap(sdkerrors.ErrInvalidRequest, "solutionHash can't be empty")
	}
	return nil
}
```

注意,所有的Message都需要继承`sdk.Msg`接口

==**MsgCreateScavenge结构**==

- `Creator` - Who created it. This uses the `sdk.AccAddress` type which represents an account in the app controlled by public key cryptograhy.

  **Message的创建者,`sdk.AccAddress`代表由公钥密码体系创建的应用程序账户**

- `Description` - The question to be solved or description of the challenge.

  **要解决的问题以及挑战的描述**

- `SolutionHash` - The scrambled solution.

  **混乱的解决方案**

- `Reward` - This is the bounty that is awarded to whoever submits the answer first.

  **奖励给第一个提交答案的人的奖赏**

该`Msg`界面还需要设置其他方法，例如，验证的内容`struct`以及确认消息是由创建者签名并提交的。

既然可以创建清除方法，那么唯一的其他基本操作就是能够解决它。如前所述，这应分为两个单独的操作：`MsgCommitSolution`和`MsgRevealSolution`

==**MsgCommitSolution结构**==

 **重命名`./x/scavenge/types/MsgCreateCommit.go`为`./x/scavenge/types/MsgCommitSolution.go`**

修改后为如下内容:

```go
package types

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)

var _ sdk.Msg = &MsgCommitSolution{}

type MsgCommitSolution struct {
	Scavenger             sdk.AccAddress `json:"scavenger" yaml:"scavenger"`                         // address of the scavenger
	SolutionHash          string         `json:"solutionhash" yaml:"solutionhash"`                   // solutionhash of the scavenge
	SolutionScavengerHash string         `json:"solutionScavengerHash" yaml:"solutionScavengerHash"` // solution hash of the scavenge
}

// NewMsgCommitSolution creates a new MsgCommitSolution instance
func NewMsgCommitSolution(scavenger sdk.AccAddress, solutionHash string, solutionScavengerHash string) MsgCommitSolution {
	return MsgCommitSolution{
		Scavenger:             scavenger,
		SolutionHash:          solutionHash,
		SolutionScavengerHash: solutionScavengerHash,
	}
}

func (msg MsgCommitSolution) Route() string {
	return RouterKey
}

func (msg MsgCommitSolution) Type() string {
	return "CreateCommit"
}

func (msg MsgCommitSolution) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{sdk.AccAddress(msg.Scavenger)}
}

func (msg MsgCommitSolution) GetSignBytes() []byte {
	bz := ModuleCdc.MustMarshalJSON(msg)
	return sdk.MustSortJSON(bz)
}

func (msg MsgCommitSolution) ValidateBasic() error {
	if msg.Scavenger.Empty() {
		return sdkerrors.Wrap(sdkerrors.ErrInvalidAddress, "scavenger can't be empty")
	}
	return nil
}
```

消息struct包含揭示解决方案时的所有必要信息：

* Scavenger -谁在透露解决方案。

* SolutionHash -混乱的解决方案（哈希）。

* SolutionScavengerHash -这是解决方案和解决方案的人的哈希组合。

该消息也实现了sdk.Msg接口。

==**MsgRevealSolution**==

此消息类型应该存在`./x/scavenge/types/MsgRevealSolution.go`,将该文件修改为:

```go
package types

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"

	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)

// MsgRevealSolution
// ------------------------------------------------------------------------------
var _ sdk.Msg = &MsgRevealSolution{}

// MsgRevealSolution - struct for unjailing jailed validator
type MsgRevealSolution struct {
	Scavenger    sdk.AccAddress `json:"scavenger" yaml:"scavenger"`       // address of the scavenger scavenger
	SolutionHash string         `json:"solutionHash" yaml:"solutionHash"` // SolutionHash of the scavenge
	Solution     string         `json:"solution" yaml:"solution"`         // solution of the scavenge
}

// NewMsgRevealSolution creates a new MsgRevealSolution instance
func NewMsgRevealSolution(scavenger sdk.AccAddress, solution string) MsgRevealSolution {

	var solutionHash = sha256.Sum256([]byte(solution))
	var solutionHashString = hex.EncodeToString(solutionHash[:])

	return MsgRevealSolution{
		Scavenger:    scavenger,
		SolutionHash: solutionHashString,
		Solution:     solution,
	}
}

// RevealSolutionConst is RevealSolution Constant
const RevealSolutionConst = "RevealSolution"

// nolint
func (msg MsgRevealSolution) Route() string { return RouterKey }
func (msg MsgRevealSolution) Type() string  { return RevealSolutionConst }
func (msg MsgRevealSolution) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{sdk.AccAddress(msg.Scavenger)}
}

// GetSignBytes gets the bytes for the message signer to sign on
func (msg MsgRevealSolution) GetSignBytes() []byte {
	bz := ModuleCdc.MustMarshalJSON(msg)
	return sdk.MustSortJSON(bz)
}

// ValidateBasic validity check for the AnteHandler
func (msg MsgRevealSolution) ValidateBasic() error {
	if msg.Scavenger.Empty() {
		return sdkerrors.Wrap(sdkerrors.ErrInvalidAddress, "scavenger can't be empty")
	}
	if msg.SolutionHash == "" {
		return sdkerrors.Wrap(sdkerrors.ErrInvalidRequest, "solutionScavengerHash can't be empty")
	}
	if msg.Solution == "" {
		return sdkerrors.Wrap(sdkerrors.ErrInvalidRequest, "solutionHash can't be empty")
	}

	var solutionHash = sha256.Sum256([]byte(msg.Solution))
	var solutionHashString = hex.EncodeToString(solutionHash[:])

	if msg.SolutionHash != solutionHashString {
		return sdkerrors.Wrap(sdkerrors.ErrInvalidRequest, fmt.Sprintf("Hash of solution (%s) doesn't equal solutionHash (%s)", msg.SolutionHash, solutionHashString))
	}
	return nil
}
```

消息`struct`包含揭示解决方案时的所有必要信息：

- `Scavenger` -谁在透露解决方案。
- `SolutionHash` -混乱的解决方案。
- `Solution` -解决方案的纯文本版本。

该消息也实现了`sdk.Msg`接口。

特别是看`ValidateBasic`功能。它验证是否进行了所有必要的输入以显示解决方案，并从提交的解决方案中创建了sha256哈希。

==MsgSetScavenge、MsgDeleteScavenge、MsgSetCommit、MsgDeleteCommit==

按文档一致即可

==Codec==

定义消息后，我们需要向编码器描述如何将其存储为字节。为此，我们编辑位于的文件`./x/scavenge/types/codec.go`。通过如下描述我们的类型，它们将与我们的编码库一起使用

```go
package types

import (
	"github.com/cosmos/cosmos-sdk/codec"
)

// RegisterCodec registers concrete types on codec
func RegisterCodec(cdc *codec.Codec) {
	// this line is used by starport scaffolding # 1
	cdc.RegisterConcrete(MsgCommitSolution{}, "scavenge/CreateCommit", nil)
	cdc.RegisterConcrete(MsgSetCommit{}, "scavenge/SetCommit", nil)
	cdc.RegisterConcrete(MsgDeleteCommit{}, "scavenge/DeleteCommit", nil)
	cdc.RegisterConcrete(MsgCreateScavenge{}, "scavenge/CreateScavenge", nil)
	cdc.RegisterConcrete(MsgSetScavenge{}, "scavenge/SetScavenge", nil)
	cdc.RegisterConcrete(MsgDeleteScavenge{}, "scavenge/DeleteScavenge", nil)
	cdc.RegisterConcrete(MsgRevealSolution{}, "scavenge/MsgRevealSolution", nil)
}

// ModuleCdc defines the module codec
var ModuleCdc *codec.Codec

func init() {
	ModuleCdc = codec.New()
	RegisterCodec(ModuleCdc)
	codec.RegisterCrypto(ModuleCdc)
	ModuleCdc.Seal()
}
```

修改完这些文件后再次启动`starport serve`会出现错误,不用担心,后续全部修改完毕之后就ok了

我们已经拥有Message模块了,但是**我们需要一些地方去存储他们发送的信息.所有相关的静态数据都与Keeper模块相关**

## 7.Keep模块

使用该`starport`命令后，您应该`Keeper`在处有一个样板`./x/scavenge/keeper/keeper.go`。它包含了像基本功能引用一个基本的函数`Set`，`Get`和`Delete`。

管理器Keeper将所有数据存储在模块中。**有时一个模块会导入另一个模块的管理器Keeper。这将允许在模块之间共享和修改状态**。由于我们在处理模块中的coin作为赏金奖励，因此我们需要访问`bank`模块的管理员（我们称之为CoinKeeper）。看看我们完成的`Keeper`文件，你可以看到那里的`bank`管理员被引用，以及如何`Set`，`Get`以及`Delete`

**Keeper、scavenge、commit详细代码见文档**

您可能会注意到`types.Commit`和`types.Scavenge`贯穿了整个参考Keeper。这些是定义的新结构，`./x/scavenge/types/type<Type>.go`(`typeCommit 、typeScavenge`)其中包含有关不同Scavenge挑战和针对这些挑战的不同已提交解决方案的所有必要信息。它们看起来类似于Msg我们之前看到的类型，因为它们包含相似的信息。我们将对模版文件进行一些修改。

在`TypeScavenge.go`文件中，我们需要删除该`ID`字段，因为我们将使用`SolutionHash`键作为键。我们还需要更新`Reward`到`sdk.Coins`，以及`Scavenger`到`sdk.AccAddress`，所以我们可以一次性解决。

修改完成后的结果:

```go
package types

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
)

type Scavenge struct {
	Creator sdk.AccAddress `json:"creator" yaml:"creator"`
    Description string `json:"description" yaml:"description"`
    SolutionHash string `json:"solutionHash" yaml:"solutionHash"`
    Reward sdk.Coins `json:"reward" yaml:"reward"`
    Solution string `json:"solution" yaml:"solution"`
    Scavenger sdk.AccAddress `json:"scavenger" yaml:"scavenger"`
}
```

对于`TypeCommit.go`文件我们需要删除ID字段,并且重命名Creator字段为Scavenger

```go
package types

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
)

type Commit struct {
	Scavenger sdk.AccAddress `json:"scavenger" yaml:"scavenger"`
    SolutionHash string `json:"solutionHash" yaml:"solutionHash"`
    SolutionScavengerHash string `json:"solutionScavengerHash" yaml:"solutionScavengerHash"`
}
```

您可以想象，未解决的字段`Scavenge`将包含`Solution`和`Scavenger`字段的空值。您可能还注意到每种类型都有该`String`方法。这有助于将结构呈现为字符串

### **Prefixes**

您可能会注意到的使用`types.ScavengePrefix`，`types.ScavengeCountPrefix`以及`types.CommitPrefix`或`types.CommitCountPrefix`。这些定义在一个名为的文件中`./x/scavenge/types/key.go`，可帮助我们保持Keeper组织良好。该Keeper实际上只是一个键值存储。这意味着，与Object`javascript`中的相似，所有值都在键下引用。要访问值，您需要知道存储它的键。这有点像唯一标识符（UID）。

在存储a时，==`Scavenge`我们使用的密钥`SolutionHash`作为唯一ID==，对于a时，==`Commit`我们使用的密钥`SolutionScavengeHash`==。但是，由于我们将这两种数据类型存储在同一位置，因此我们可能**希望区分用作键的哈希类型。我们可以通过在散列上添加前缀来做到这一点**，从而使我们能够识别出哪一个。因为`Scavenge`我们看到了前缀`scavenge-value`和`scavenge-count`，所以`Commit`我们看到了前缀`commit-value`和`commit-count`。
所以在`key.go`文件中可以看到如下内容:

```go
package types

const (
	// ModuleName is the name of the module
	ModuleName = "scavenge"

	// StoreKey to be used when creating the KVStore
	StoreKey = ModuleName

	// RouterKey to be used for routing msgs
	RouterKey = ModuleName

	// QuerierRoute to be used for querier msgs
	QuerierRoute = ModuleName
)

const (
	ScavengePrefix = "scavenge-value-"
	ScavengeCountPrefix = "scavenge-count-"
)

const (
	CommitPrefix = "commit-value-"
	CommitCountPrefix = "commit-count-"
)
```

### **Iterators**

有时，您可能想直接通过其键访问一个 `Commit`或一个 `Scavenge`。这就是为什么我们有方法`GetCommit`和的原因`GetScavenge`。

但是，有时您会想要`Scavenge`一次或一次获取所有内容`Commit`。为此，我们使用称为的迭代器`KVStorePrefixIterator`。此实用程序来自`cosmos sdk`并在密钥存储上进行迭代。如果提供前缀，它将仅对包含该前缀的键进行迭代。由于我们为`Scavenge`和`Commit`定义了前缀，因此我们可以在此处使用它们以仅返回所需的数据类型。


目前你已经知道了`Commit`和`Scavenge`的存储位置,我们需要将Messages连接到此存储.这个过程叫做`handling`消息,并且它是实现在`Handler`中.

## 8.Handler模块

为了使消息到达`Keeper`，它必须经过`Handler`。在这里可以应用逻辑来允许或拒绝一个 `Message`成功。这也是逻辑准确显示状态更改应如何在`Keeper`中进行的地方。**==如果您熟悉Model View Controller（MVC）架构，Keeper有点像Model，Handler有点像Controller==**。如果您熟悉`React / Redux` 或`Vue / Vuex`架构，Keeper有点像`Reducer / Store`，而`Handler`有点像`Actions`。

我们的处理程序Handler将进入`./x/scavenge/handler.go`并遵循样板中列出的建议。我们将创建一个名为单独的文件处理功能，`handler<Action.go`为我们的每一个三种`Message`类型`MsgCreateScavenge`，`MsgCommitSolution`和`MsgRevealSolution`。

运行`starport type`命令应该已经添加了`handlerMsgCreateScavenge.go`和`handlerMsgCreateCommit.go`文件。本质上，您可以重命名`handlerMsgCreateCommit`为`handlerMsgCommitSolution`。制作一份副本并将其用作的模板`handlerMsgRevealSolution`。

文件修改见官方文档

### moduleAcct

你可能注意到handlerMsgCreateScavenge和handlerMsgRevealSolution处理函数中使用了moduleAcct。**该帐户不受公钥对控制，而是对该实际模块拥有的帐户的引用**。它被用来持有与scavenge连接的赏金，直到该scavenge被解决，在这一点上，赏金支付给解决了scavenge的帐户。

### Events

每个处理程序的末尾是一个EventManager，它将在事务内**创建日志**，以显示有关在处理此消息期间发生的情况的信息。这对于希望确切了解状态转换结果发生的客户端软件很有用。这些事件使用一系列预定义的类型，可以在`./x/scavenge/types/events.go`以下类型中找到它们

现在我们创建了必要的管道去更新状态,我们需要考虑用什么方法去查询它们. 通常，这是通过REST端点或CLI完成的.这两个客户端都与查询状态的应用程序部分交互，称为`Querier`

## 9.Querier

为了查询应用程序的数据，我们需要使用来使其可访问`Querier`。该应用程序的一部分`Keeper`与访问状态并返回状态一起工作。`Querier`定义在`./x/scavenge/keeper/querier.go`。我们的`starport`工具为我们提供了一些外观方面的建议，类似于`Handler`我们想要处理不同查询路线的建议。

您可以`Querier`针对许多不同类型的查询在内建立许多不同的路由，但我们将只进行三个：

* `listCommit` 将列出所有提交

* `getCommit` 将得到一个提交 solutionScavengerHash

* `listScavenge` 将列出所有Scavenge

* `getScavenge` 将会得到一次Scavenge 的 solutionHash

合并到switch语句中，并充实每个函数，该文件应如下所示

```go
package keeper

import (
  // this line is used by starport scaffolding # 1
	"github.com/github-username/scavenge/x/scavenge/types"
		
	
		
	abci "github.com/tendermint/tendermint/abci/types"

	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)

// NewQuerier creates a new querier for scavenge clients.
func NewQuerier(k Keeper) sdk.Querier {
	return func(ctx sdk.Context, path []string, req abci.RequestQuery) ([]byte, error) {
		switch path[0] {
    // this line is used by starport scaffolding # 2
		case types.QueryListCommit:
			return listCommit(ctx, k)
		case types.QueryGetCommit:
			return getCommit(ctx, path[1:], k)
		case types.QueryListScavenge:
			return listScavenge(ctx, k)
		case types.QueryGetScavenge:
			return getScavenge(ctx, path[1:], k)
		default:
			return nil, sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, "unknown scavenge query endpoint")
		}
	}
}
```



### Types

您可能会注意到，我们在初始`switch`语句中使用了四种不同的导入类型。这些在我们的`./x/scavenge/types/querier.go`文件中定义为简单字符串。该文件应如下

```go
package types

const (
	QueryListScavenge = "list-scavenge"
	QueryGetScavenge  = "get-scavenge"
)

const (
	QueryListCommit = "list-commit"
	QueryGetCommit  = "get-commit"
)
```

我们的查询非常简单，因为我们已经`Keeper`为访问状态配备了所有必需的功能。您也可以在这里看到正在使用的迭代器。

现在，我们已经创建了模块的所有基本操作，我们希望使它们可访问。我们可以使用CLI客户端和REST客户端来做到这一点。在本教程中，我们将创建一个CLI客户端

## 10.CLI

命令行界面（CLI）将在应用程序在某处机器上运行后帮助我们与它进行交互。**每个模块在CLI内都有自己的名称空间，这使它能够创建和签名要由该模块处理的消息。**它还具有查询该模块状态的功能。与该应用程序的其余部分结合使用时，CLI将允许您执行诸如为新帐户生成密钥或检查您已经与该应用程序进行交互的状态之类的操作

我们的模块CLI被分成两个文件名为`tx.go`以及`query.go`分别位于`./x/scavenge/client/cli/`。一个文件用于进行**包含消息的事务**，这些消息最终将更新我们的状态。另一个是进行**查询**，这将使我们能够从状态中读取信息

### **tx.go**

该tx.go文件包含GetTxCmdCosmos SDK中的标准方法。稍后在module.go文件中引用该文件，该文件准确描述了模块具有的属性。这使得在实际应用程序级别更容易合并出于不同原因的不同模块。毕竟，我们现在将重点放在模块上，但是稍后我们将创建一个利用该模块以及Cosmos SDK中已经可用的其他模块的应用程序。

在内部，GetTxCmd我们创建一个新的模块特定命令并调用它scavenge。在此命令中，我们为定义的每种消息类型添加一个子命令：

GetCmdCreateScavenge
GetCmdCommitSolution
GetCmdRevealSolution
每个函数都从Cobra CLI工具中获取参数以创建一个新的msg，对其进行签名并将其提交给要处理的应用程序。这些函数应该放在`tx.go`和`tx<Type>.go`文件中

### query.go

该query.go文件包含类似的Cobra命令，这些命令保留了一个新的名称空间来引用我们的scavenge模块。但是，`query.go`和`query<Type>.go`文件不是创建和提交消息，而是创建查询并以人类可读的形式返回结果。它处理的查询与我们querier.go先前在文件中定义的查询相同：

* GetCmdListCommit

* GetCmdGetCommit

* GetCmdListScavenge

* GetCmdGetScavenge

### REST

按照文档修改

## 11.运行游戏

`scavengecli tx scavenge create-scavenge "What's brown and sticky?" "A stick" 69token --from user1`

问题:What's brown and sticky

答案:A stick

并且设置69的奖励金

成功上链返回:

![Ggcayy](http://xwjpics.gumptlu.work/qinniu_uPic/Ggcayy.png)

查询你的交易

`scavengecli q tx <txhash>` (txhash是你的hash,注意不要带<>)

返回结果会显示高度

![34mTUt](http://xwjpics.gumptlu.work/qinniu_uPic/34mTUt.png)

另一个用户回答:

`scavengecli tx scavenge commit-solution "A stick" --from user2 -y	`

![s2KMUL](http://xwjpics.gumptlu.work/qinniu_uPic/s2KMUL.png)

查询:

![boIm78](http://xwjpics.gumptlu.work/qinniu_uPic/boIm78.png)

solutionScavengerHash是solution和自己账户的组合

![dycrHP](http://xwjpics.gumptlu.work/qinniu_uPic/dycrHP.png)

游戏的思路:

scavenge可以看做问题,而solution就是答案, 问题的提出者给出问题与答案, 对应的取Hash,如果其他人给出的答案与问题相hash的结果与正确答案一致,那么就说明回答正确.

user2回答正确后,查询其余额变化

![BoEqk5](http://xwjpics.gumptlu.work/qinniu_uPic/BoEqk5.png)

增加了69

如果您想看一下已完成的scavenge工作，可以先查询**所有**scavenge工作

`scavengecli q scavenge list-scavenge`

![M6mvbq](http://xwjpics.gumptlu.work/qinniu_uPic/M6mvbq.png)

单独查询某个scavenge:

`scavengecli q scavenge get-scavenge 2f9457a6e8fb202f9e10389a143a383106268c460743dd59d723c0f82d9ba906`

![BVKXln](http://xwjpics.gumptlu.work/qinniu_uPic/BVKXln.png)

