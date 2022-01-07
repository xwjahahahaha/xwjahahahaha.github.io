---
title: Nameservice测试项目部署
tags:
  - cosmos
categories:
  - technical
  - cosmos
toc: true
declare: true
date: 2021-03-04 10:10:48
---

# 目录

[TOC]

# 介绍

# 1.Getting Started

使用**scratch**部署区块链

项目最后会构建一个Nameservice, 就是一个映射关系 string->other string(`map[string]string`)

<!-- more -->

类似于Namecoin、EN、DNS等,用户创建需要使用的域名

前置要求:

- [`golang` >1.15 (opens new window)](https://golang.org/doc/install)installed
- Github account and either [Github CLI (opens new window)](https://hub.github.com/)or [Github Desktop (64-bit required)(opens new window)](https://help.github.com/en/desktop/getting-started-with-github-desktop/installing-github-desktop)
- Desire to create your own blockchain!

* ==starport V0.13.1==注意版本, 可能默认下载的最新版. 各系统下载地址[install Starport (opens new window)](https://github.com/tendermint/starport/blob/develop/docs/1 Introduction/2 Install.md).

# 2.Application Goals

目标:用户购买域名,系统设置存储对应的value,域名都是最高出价售出

Here are the modules you will need for the nameservice application:

- `auth`: This module defines accounts and fees and gives access to these functionalities to the rest of your application.

  **定义账户和费用**

- `bank`: This module enables the application to create and manage tokens and token balances.

  **使应用程序管理与创建tokens**

- `staking` : This module enables the application to have validators that people can delegate to.

  **使得应用程序实现用户委托代理**

- `distribution` : This module give a functional way to passively distribute rewards between validators and delegators.

  **在验证者与代理者之间分配奖励**

- `slashing` : This module disincentivizes people with value staked in the network, ie. Validators.

  **以削减质押金额(例如Validators)来惩罚 **

- `supply` : This module holds the total supply of the chain.

  **控制区块链的总供应**

- `nameservice`: This module does not exist yet! It will handle the core logic for the `nameservice` application you are building. It is the main piece of software you have to work on to build your application.

  **nameservice的核心逻辑,由开发人员自己编写**

## State

token和用户的状态都已经在auth和bank模块中定义了,所以我们无需操心.

但是我们需要定义部分特别的与nameservice相关的状态

在SDK,所有的数据存储在叫做multistore的数据库中(k/v数据库)

我们使用一个存储域名和其代表的人的映射,这个结构体包含了域名的值、拥有者以及余额

## Messages

==消息包含在交易中==,它们触发状态交易

每个模块都定义了一系列的消息和操作它们的方法

nameservice应用中需要定义的两类基本消息:

- `MsgSetName`: This message allows name owners to set a value for a given name.
- `MsgBuyName`: This message allows accounts to buy a name and become its owner.
  * 购买当前的域名,你的价格要比前任支付的价格高才可以,如果该域名之前没有人购买,那么必须支付最小的金额即`MinPrice`

==一般性过程:==

1. 已经打包进区块的交易到达Tendermint节点

2. 节点通过ABCI与应用程序cosmos SDK交流,
3. cosmos SDK解码交易获取消息
4. SDK将消息路由到合适的模块去执行,执行遵循`Handler`(可认为是控制器)中的定义
5. `Handler`运行`Keeper`(可认为是module)进行数据的更新或删除等操作

# 3.Start your application

```shell
starport app github.com/user/nameservice --sdk-version="launchpad"
cd nameservice
```

如果你克隆了项目,那么`user`字段可以换成自己的项目地址

![iW71tT](http://xwjpics.gumptlu.work/qinniu_uPic/iW71tT.png)

==Tendermint通过来自网络的交易与应用程序交互的接口叫做 => ABCI==

结构如下:

```shell
+---------------------+
|                     |
|     Application     |
|                     |
+--------+---+--------+
         ^   |
         |   | ABCI
         |   v
+--------+---+--------+
|                     |
|                     |
|     Tendermint      |
|                     |
|                     |
+---------------------+

```

**这些接口cosmos SDK都提供了样板文件在`basseapp`中**,baseapp的作用可见cosmos开发基础.

在这个案例中可以使用`nameservice` & `NameServiceApp`这些types,这些type会嵌入到`baseapp`中

baseapp不能够识别用户自定义模块的路由和应用程序中自定义的用户接口, **您的应用程序的主要作用是定义这些路由, 另一个作用是定义初始状态**. 这些都要求在你的应用程序添加模块.

 `auth`, `bank`, `staking`, `distribution`, `slashing` and `nameservice`. The first five already exist, but not the last! 后面需要修改.

# 4.Types

创建`whois`类型

`starport type whois value price`

![qP4v1r](http://xwjpics.gumptlu.work/qinniu_uPic/qP4v1r.png)

目前我们只给whois类型添加了两个字段,我们还会对部分自动生成的字段进行修改, 删除`ID`字段, 代替`Creator`字段为`Owner`

脚手架工具为我们自动创建了以下文件:

在 `./x/nameservice/types`目录下:

 `MsgCreateWhois.go`, `MsgDeleteWhois.go`, `MsgSetWhois.go`, and `TypeWhois.go`.

## Whois

每一个域名都有三个字段

- Value - The value that a name resolves to. This is just an arbitrary string, but in the future you can modify this to require it fitting a specific format, such as an IP address, DNS Zone file, or blockchain address.

  **域名解析的值,string类型,但是后面可以自己设置为特定的数据类型**

- Owner - The address of the current owner of the name

  **域名的主人**

- Price - The price you will need to pay in order to buy the name

  **购买的价格**

To start your SDK module, define your `nameservice.Whois` struct in the `./x/nameservice/types/TypeWhois.go` file:

```go
package types

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
)

//定义最小域名转卖金额
var MinNamePrice = sdk.Coins{sdk.NewInt64Coin("nametoken", 1)}

type Whois struct {
	Creator sdk.AccAddress `json:"creator" yaml:"creator"`
	ID      string         `json:"id" yaml:"id"`
    Value string `json:"value" yaml:"value"`
    Price string `json:"price" yaml:"price"`
}


//返回一个新的Whois，因为新的域名还没有人购买，所以设置金额为最小
func NewWhois() Whois {
	return Whois{
		Price:   MinNamePrice,
	}
}
```

# 5.Key

在`key.go`文件中已经创建了模块需要的一些全局常量

```go
package types

const (
	// ModuleName is the name of the module
	// 模块名
	ModuleName = "nameservice"

	// StoreKey to be used when creating the KVStore
	// 模块存储空间的键/key
	StoreKey = ModuleName

	// RouterKey to be used for routing msgs
	// 路由消息的键名
	RouterKey = ModuleName

	// QuerierRoute to be used for querier msgs
	// 查询消息的查询路由名
	QuerierRoute = ModuleName
)

const (
	// whois结构体的前缀，即在模块空间中k/v的前缀key
	WhoisPrefix      = "whois-value-"
	// 存储总记数的前缀key，是WhoisPrefix的特例
	WhoisCountPrefix = "whois-count-"
)
```

# 6.Errors

定义了模块的自定义错误以及错误代码

```go
package types

import (
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)

var (
	ErrNameDoesNotExist = sdkerrors.Register(ModuleName, 1, "name does not exist")
)
```

你应该还定义错误的方法,当发生错误的时候.

# 7.Keeper

主要负责数据的增删改查

## Keeper Struct

Your `nameservice.Keeper` should already be defined in the `./x/nameservice/keeper/keeper.go` file. Let's have a short introduction of the `keeper.go` file.

```go
package keeper

import (
	// this line is used by starport scaffolding # 1
	"fmt"

	"github.com/tendermint/tendermint/libs/log"

	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/cosmos/cosmos-sdk/x/bank"
	"github.com/user/nameservice/x/nameservice/types"
)

// Keeper of the nameservice store
// Keeper结构体
type Keeper struct {
	CoinKeeper bank.Keeper			// bank模块, 底层管理token的模块
	storeKey   sdk.StoreKey			// cosmos-sdk/types的通用存储key类型
	cdc        *codec.Codec			// 底层编码模块
	// paramspace types.ParamSubspace
}

// NewKeeper creates a nameservice keeper
// 返回Keeper的对象
func NewKeeper(coinKeeper bank.Keeper, cdc *codec.Codec, key sdk.StoreKey) Keeper {
	keeper := Keeper{
		CoinKeeper: coinKeeper,
		storeKey:   key,
		cdc:        cdc,
		// paramspace: paramspace.WithKeyTable(types.ParamKeyTable()),
	}
	return keeper
}

// Logger returns a module-specific logger.
// 使用当前的模块名生成特定的log返回
func (k Keeper) Logger(ctx sdk.Context) log.Logger {
	return ctx.Logger().With("module", fmt.Sprintf("x/%s", types.ModuleName))
}
```

对于代码中import的两个不同的types:

- [`types` (as sdk) (opens new window)](https://godoc.org/github.com/cosmos/cosmos-sdk/types)- this contains commonly used types throughout the SDK.

  **包含整个sdk通常使用的类型**

- `types` - it contains `BankKeeper` you have defined in previous section.

  **包含之前介绍的`BankKeeper`**

Keeper结构体解释:

- `types.BankKeeper` - This is an interface you had defined on previous section to use `bank` module. Including it allows code in this module to call functions from the `bank` module. The SDK uses an [object capabilities (opens new window)](https://en.wikipedia.org/wiki/Object-capability_model)approach to accessing sections of the application state. This is to allow developers to employ a least authority approach, limiting the capabilities of a faulty or malicious module from affecting parts of state it doesn't need access to.

  **BankKeeper 这是在上一节介绍的bank,这里使用bank模块的接口。==它允许该模块中的代码调用bank模块中的函数==。SDK使用 object capabilities方法来访问应用程序状态的部分。这允许开发人员采用最小权限的方法，限制错误或恶意模块的功能，使其不影响它不需要访问的状态部分。**

- [`*codec.Codec` (opens new window)](https://godoc.org/github.com/cosmos/cosmos-sdk/codec#Codec)- This is a pointer to the codec that is used by Amino to encode and decode binary structs.

  **这是Amino用于编码和解码二进制结构的编解码器的指针,用于编码/解码数据**

- [`sdk.StoreKey` (opens new window)](https://godoc.org/github.com/cosmos/cosmos-sdk/types#StoreKey)- This is a store key which gates access to a `sdk.KVStore`which persists the state of your application: the Whois struct that the name points to (i.e. `map[name]Whois`).

  **==这是一个用于访问sdk 持久化应用程序状态的KVStore对应的key==。数据库存储也就是域名所指向的Whois结构体**

## Getters and Setters

In our `keeper` directory we find the `whois.go` file which has been created with our `starport type` command.

我们需要修改部分, 使用`Name`作为key去查找每一个`Whois`

```go
package keeper

import (
	"strconv"

	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"

	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/user/nameservice/x/nameservice/types"
)

// GetWhoisCount get the total number of whois
// 获取whois的所有数量
func (k Keeper) GetWhoisCount(ctx sdk.Context) int64 {
	// 获取keeper对象的key,也就是在key.go文件中定义的存储StoreKey常量
	// 提取上下文中对应key的store
	store := ctx.KVStore(k.storeKey)
	// 将key.go中的数量统计前缀转换为byte数组
	byteKey := []byte(types.WhoisCountPrefix)
	// 根据WhoisCountPrefix查询存储的数量值
	bz := store.Get(byteKey)

	// Count doesn't exist: no element
	// 不存在则返回默认的0
	if bz == nil {
		return 0
	}

	// Parse bytes
	// 解析字查询到的数量string => int
	count, err := strconv.ParseInt(string(bz), 10, 64)
	if err != nil {
		// Panic because the count should be always formattable to int64
		panic("cannot decode count")
	}

	return count
}

// SetWhoisCount set the total number of whois
// 存储whois的总数
func (k Keeper) SetWhoisCount(ctx sdk.Context, count int64) {
	// 获取数据库 => 格式化key => 存储
	store := ctx.KVStore(k.storeKey)
	byteKey := []byte(types.WhoisCountPrefix)
	bz := []byte(strconv.FormatInt(count, 10))
	// 设置
	store.Set(byteKey, bz)
}

// CreateWhois creates a whois. This function is included in starport type scaffolding.
// We won't use this function in our application, so it can be commented out.
// 脚手架自动创建的创建whois结构体的函数, 不想使用可以注释掉, 一般写在Tpye{your type}.go中,这里就是TypeWhois.go
// func (k Keeper) CreateWhois(ctx sdk.Context, whois types.Whois) {
// 	store := ctx.KVStore(k.storeKey)
// 	key := []byte(types.WhoisPrefix + whois.Value)
// 	value := k.cdc.MustMarshalBinaryLengthPrefixed(whois)
// 	store.Set(key, value)
// }

// GetWhois returns the whois information
// 获取whois结构体数据, key就是域名的name
func (k Keeper) GetWhois(ctx sdk.Context, key string) (types.Whois, error) {
	// 1. 获取namservice数据库
	store := ctx.KVStore(k.storeKey)
	var whois types.Whois
	// 2. 拼接key, whois前缀 + 获取参数key
	byteKey := []byte(types.WhoisPrefix + key)
	// 3. 使用keeper的codec类型cdc解码, 赋值给whois结构体
	err := k.cdc.UnmarshalBinaryLengthPrefixed(store.Get(byteKey), &whois)
	if err != nil {
		return whois, err
	}
	// 4. 返回
	return whois, nil
}

// SetWhois sets a whois. We modified this function to use the `name` value as the key instead of msg.ID
// 存储Whois, 我们修改了这个函数，使用' name '值作为键，而不是msg.ID
func (k Keeper) SetWhois(ctx sdk.Context, name string, whois types.Whois) {
	store := ctx.KVStore(k.storeKey)
	// 使用cdc编码参数whois结构体返回的bz是byte切片
	// MustMarshalBinaryLengthPrefixed 有Must代表不返回其错误直接Panic处理， 即使有err的话，没有的话返回可能的错误
	bz := k.cdc.MustMarshalBinaryLengthPrefixed(whois)
	// 使用' name '值作为键，而不是msg.ID
	key := []byte(types.WhoisPrefix + name)
	// 存储
	store.Set(key, bz)
}

// DeleteWhois deletes a whois
// 删除一个whois结构体
func (k Keeper) DeleteWhois(ctx sdk.Context, key string) {
	store := ctx.KVStore(k.storeKey)
	store.Delete([]byte(types.WhoisPrefix + key))
}

//
// Functions used by querier  为用户查询的函数
//

// 查询whois的集合
func listWhois(ctx sdk.Context, k Keeper) ([]byte, error) {
	var whoisList []types.Whois
	store := ctx.KVStore(k.storeKey)
	// 根据前缀创建循环迭代器, 遍历所有包含此前缀字段的key对应的value
	// KVStorePrefixIterator 按升序迭代所有带有特定前缀的键
	iterator := sdk.KVStorePrefixIterator(store, []byte(types.WhoisPrefix))
	// 遍历
	for ; iterator.Valid(); iterator.Next() {
		var whois types.Whois
		// 1. 迭代器获取包含特定前缀的完整key
		// 2. 获取whois（[]byte）进行解码
		// 3. 赋值给whois
		k.cdc.MustUnmarshalBinaryLengthPrefixed(store.Get(iterator.Key()), &whois)
		// 添加到集合中
		whoisList = append(whoisList, whois)
	}
	// 再将整个list解码/序列化成字节数组，
	// MustMarshalJSONIndent有Must代表不返回其错误直接Panic处理， 即使有err的话，没有的话返回可能的错误
	res := codec.MustMarshalJSONIndent(k.cdc, whoisList)
	return res, nil
}

// 查询单个Whois
func getWhois(ctx sdk.Context, path []string, k Keeper) (res []byte, sdkError error) {
	// 获取key, path的第一个参数(用户命令行输入)
	key := path[0]
	// 调用keeper的基本方法GetWhois(见上方)
	whois, err := k.GetWhois(ctx, key)
	if err != nil {
		return nil, err
	}

	// 编码/序列化为字节数组,
	res, err = codec.MarshalJSONIndent(k.cdc, whois)
	if err != nil {
		return nil, sdkerrors.Wrap(sdkerrors.ErrJSONMarshal, err.Error())
	}

	return res, nil
}

// Resolves a name, returns the value
// 解析域名对应的值,也就是whois中的字段value
func resolveName(ctx sdk.Context, path []string, keeper Keeper) ([]byte, error) {
	// 直接调用keeper的解析函数(见下方), key是path[0]
	value := keeper.ResolveName(ctx, path[0])

	if value == "" {
		return []byte{}, sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, "could not resolve name")
	}

	// 编码/序列化为字节数组
	// QueryResResolve是types/querier文件下的函数, QueryResResolve就是解析值的一个结构体
	// 因为MarshalJSONIndent解析需要一个结构体， 所以创建了这样的QueryResResolve结构体以赋值
	res, err := codec.MarshalJSONIndent(keeper.cdc, types.QueryResResolve{Value: value})
	if err != nil {
		return nil, sdkerrors.Wrap(sdkerrors.ErrJSONMarshal, err.Error())
	}

	return res, nil
}

// Get creator of the item
// 获取域名的创建者
func (k Keeper) GetCreator(ctx sdk.Context, key string) sdk.AccAddress {
	whois, _ := k.GetWhois(ctx, key)
	return whois.Creator
}

// Check if the key exists in the store
// 检查当前域名key/name是否存在
func (k Keeper) Exists(ctx sdk.Context, key string) bool {
	store := ctx.KVStore(k.storeKey)
	return store.Has([]byte(types.WhoisPrefix + key))
}

// ResolveName - returns the string that the name resolves to
// 获取域名的解析值
func (k Keeper) ResolveName(ctx sdk.Context, name string) string {
	whois, _ := k.GetWhois(ctx, name)
	return whois.Value
}

// SetName - sets the value string that a name resolves to
// 设置域名对应的解析值
func (k Keeper) SetName(ctx sdk.Context, name string, value string) {
	whois, _ := k.GetWhois(ctx, name)
	whois.Value = value
	k.SetWhois(ctx, name, whois)
}

// HasOwner - returns whether or not the name already has an owner
// 检查当前域名是否有创建人
func (k Keeper) HasCreator(ctx sdk.Context, name string) bool {
	whois, _ := k.GetWhois(ctx, name)
	return !whois.Creator.Empty()
}

// SetOwner - sets the current owner of a name
// 设置域名的当前拥有者
func (k Keeper) SetCreator(ctx sdk.Context, name string, creator sdk.AccAddress) {
	whois, _ := k.GetWhois(ctx, name)
	whois.Creator = creator
	k.SetWhois(ctx, name, whois)
}

// GetPrice - gets the current price of a name
// 获取域名的价格
func (k Keeper) GetPrice(ctx sdk.Context, name string) sdk.Coins {
	whois, _ := k.GetWhois(ctx, name)
	return whois.Price
}

// SetPrice - sets the current price of a name
// 设置域名的价格
func (k Keeper) SetPrice(ctx sdk.Context, name string, price sdk.Coins) {
	whois, _ := k.GetWhois(ctx, name)
	whois.Price = price
	k.SetWhois(ctx, name, whois)
}

// Check if the name is present in the store or not
// 检查当前的name参数是否存在于store中，注意于Exists函数的区别：没有加前缀
func (k Keeper) IsNamePresent(ctx sdk.Context, name string) bool {
	store := ctx.KVStore(k.storeKey)
	return store.Has([]byte(name))
}

// Get an iterator over all names in which the keys are the names and the values are the whois
// 获取固定前缀的迭代器
func (k Keeper) GetNamesIterator(ctx sdk.Context) sdk.Iterator {
	store := ctx.KVStore(k.storeKey)
	return sdk.KVStorePrefixIterator(store, []byte(types.WhoisPrefix))
}

// Get creator of the item
// 根据key = id获取whois的创建者， 但是可能会返回err
func (k Keeper) GetWhoisOwner(ctx sdk.Context, key string) sdk.AccAddress {
	whois, err := k.GetWhois(ctx, key)
	if err != nil {
		return nil
	}
	return whois.Creator
}

// Check if the key exists in the store
// 根绝key查询是否存在， key是域名结构体中的id
func (k Keeper) WhoisExists(ctx sdk.Context, key string) bool {
	store := ctx.KVStore(k.storeKey)
	return store.Has([]byte(types.WhoisPrefix + key))
}
```

# 8.Msgs and Handlers

## Msgs

msgs触发状态交易,Msg包含在Tx接口中,Tx就是网络中用户提交的交易

Cosmos sdk中的源码:

```go
// Tx defines the interface a transaction must fulfill.
	Tx interface {
		// Gets the all the transaction's messages.
		GetMsgs() []Msg

		// ValidateBasic does a simple and lightweight validation check that doesn't
		// require access to any other information.
		ValidateBasic() error
	}
```

作为开发者只需要定义Msg即可,Msg必须要满足以下接口:

`./x/nameservice/types/msg.go`

```go
package types

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
)

// Transactions messages must fulfill the Msg
// 实现接口必须实现所有的函数
type Msg interface {
	// Return the message type.
	// Must be alphanumeric or empty.
	// 返回消息类型， 必须是字母或者空
	Type() string

	// Returns a human-readable string for the message, intended for utilization
	// within tags
	// 返回消息的可读string,用于在标签中使用
	Route() string

	// ValidateBasic does a simple validation check that
	// doesn't require access to any other information.
	// 做一些基本的验证, 不需要访问其他信息
	ValidateBasic() error

	// Get the canonical byte representation of the Msg.
	// 获取Msg的规范字节表示即字节数组
	GetSignBytes() []byte

	// Signers returns the addrs of signers that must sign.
	// CONTRACT: All signatures must be present to be valid.
	// CONTRACT: Returns addrs in some deterministic order.
	// 返回必须签名的签名者地址集合， 所有的签名必须在当下还有效， 返回的签名集合会以某种确定的顺序
	GetSigners() []sdk.AccAddress
}
```

## Handlers

可看作为controller

定义了当接受到Msg时, 哪些数据存储需要更新,在什么样的环境下更新等

# 9.Msgs

现在开始编写本项目场景需要的Msg

## SetName

### MsgSetSetName

首先实现`SetName`, 这个消息Msg允许域名的所有者在解析器中设置该域名的返回值

Start by renaming the `./x/nameservice/types/MsgSetWhois.go` file to `./x/nameservice/types/MsgSetName.go`.

```go
package types

import (
  sdk "github.com/cosmos/cosmos-sdk/types"
  sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)

//const RouterKey = ModuleName // this was defined in your key.go file

// MsgSetName defines a SetName message
// 定义MsgSetName的结构
type MsgSetName struct {
  Name  string         `json:"name"`    // 目标域名
  Value string         `json:"value"`   // 对应的值/解析值
  Owner sdk.AccAddress `json:"owner"`   // 拥有者
}

// NewMsgSetName is a constructor function for MsgSetName
// MsgSetName构造函数
func NewMsgSetName(name string, value string, owner sdk.AccAddress) MsgSetName {
  return MsgSetName{
    Name:  name,
    Value: value,
    Owner: owner,
  }
}

// Route should return the name of the module
// 返回路由消息的键， 这里就是模块名
func (msg MsgSetName) Route() string { return RouterKey }

// Type should return the action
// 操作名
func (msg MsgSetName) Type() string { return "set_name" }

// ValidateBasic runs stateless checks on the message
// 基本参数检测
func (msg MsgSetName) ValidateBasic() error {
  if msg.Owner.Empty() {
    return sdkerrors.Wrap(sdkerrors.ErrInvalidAddress, msg.Owner.String())
  }
  if len(msg.Name) == 0 || len(msg.Value) == 0 {
    return sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, "Name and/or Value cannot be empty")
  }
  return nil
}

// GetSignBytes encodes the message for signing
// GetSignBytes对整个MsgSetName消息本身进行编码、排序以进行后续的签名
func (msg MsgSetName) GetSignBytes() []byte {
  // MustMarshalJSON序列化msg为字节切片
  // MustSortJSON返回根据key排序的json
  return sdk.MustSortJSON(ModuleCdc.MustMarshalJSON(msg))
}

// GetSigners defines whose signature is required
// 定义该需要谁的签名
func (msg MsgSetName) GetSigners() []sdk.AccAddress {
  return []sdk.AccAddress{msg.Owner}
}
```

The `MsgSetName` has the three attributes needed to set the value for a name:

- `name` - The name trying to be set. 域名
- `value` - What the name resolves to. 要解析的值
- `owner` - The owner of that name. 域名的使用者

**==SDK使用上述函数将消息路由到适当的模块进行处理==**。它们还将人类可读的操作名称添加到用于索引的数据库标签中。

`GetSignBytes`定义了消息Msg怎样对自身进行编码以方便签名, 一般都是序列化+排序, 一般不需要改动.

`GetSigners`定义了在Tx中那些人的签名是必须的,这样才能保证有效.

在此案例中, 当需要重新设置域名的解析时,`MsgSetName`需要Owner签名交易才能生效.

### Handler

目前`MsgSetName`已经规定好了, **==当接收到了消息后`Handler`负责定义接下来的行动.==** 

`NewHandler`本质上是一个子路由器，它将进入该模块的消息指示给适当的处理程序。目前，只有一个`Msg/Handler`。

Let's rename our `x/nameservice/handlerMsgSetWhois.go` to `x/nameservice/handlerMsgSetName.go`.

`mv x/nameservice/handlerMsgSetWhois.go x/nameservice/handlerMsgSetName.go`

修改如下:

```go
package nameservice

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
	"github.com/user/nameservice/x/nameservice/keeper"
	"github.com/user/nameservice/x/nameservice/types"
)

// Handle a message to set name
// 接受到msg， 进一步的处理，相当于controller层， 操作keeper层
func handleMsgSetName(ctx sdk.Context, keeper keeper.Keeper, msg types.MsgSetName) (*sdk.Result, error) {
	// 检查MsgSetName的提供者是否为想要设置域名的域名拥有者， 即验证身份
	if !msg.Owner.Equals(keeper.GetWhoisOwner(ctx, msg.Name)) { // Checks if the the msg sender is the same as the current owner
		return nil, sdkerrors.Wrap(sdkerrors.ErrUnauthorized, "Incorrect Owner") // If not, throw an error
	}
	// 如果是本人的话设置所规定的值
	// 调用keeper进行域名的解析值设定
	keeper.SetName(ctx, msg.Name, msg.Value) // If so, set the name to the value specified in the msg.
	return &sdk.Result{}, nil                // return
}
```

In the file (`./x/nameservice/handler.go`) make sure to replace the `types.MsgSetWhois` with the following code:

==**`handler.go`是各个handlerxxxx.go的总路由,  所以每一个对应的msg都需要在这里注册, 下面是修改完整的`handler.go`:**==

```go
package nameservice

import (
	"fmt"
	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
	"github.com/user/nameservice/x/nameservice/keeper"
	"github.com/user/nameservice/x/nameservice/types"
)

// NewHandler returns a handler for "nameservice" type messages.
// 返回一个操作nameservice各类消息的Handler对象
func NewHandler(keeper keeper.Keeper) sdk.Handler {
	return func(ctx sdk.Context, msg sdk.Msg) (*sdk.Result, error) {
		//.(type)获取接口实例实际的类型指针, 以此调用实例所有可调用的方法，包括接口方法及自有方法。
		//需要注意的是该写法必须与switch case联合使用，case中列出实现该接口的类型。
		switch msg := msg.(type) {
		// 添加操作类型
		case types.MsgSetName:
			return handleMsgSetName(ctx, keeper, msg)
		case types.MsgBuyName:
			return handleMsgBuyName(ctx, keeper, msg)
		case types.MsgDeleteName:
			return handleMsgDeleteName(ctx, keeper, msg)
		default:
			return nil, sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, fmt.Sprintf("Unrecognized nameservice Msg type: %v", msg.Type()))
		}
	}
}
```

现在, 拥有者可以设置域名的解析了, 但是如果是一个域名不存在拥有者的情况呢?

现在模块需要提供一种方式让用户去购买域名, 下面就定义`BuyName`消息

## Buy Name 

### MsgBuyName

`./x/nameservice/types/MsgBuyName.go`很类似于setName

We can replace the file `MsgCreateWhois.go`, as these two files are similar in nature, and we won't be using `MsgCreateWhois`.

`mv x/nameservice/types/MsgCreateWhois.go x/nameservice/types/MsgBuyName.go`

Replace `handlerMsgCreateWhois` by `handlerMsgBuyName`:

`mv x/nameservice/handlerMsgCreateWhois.go x/nameservice/handlerMsgBuyName.go`

**MsgBuyName.go:**

```go
package types

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)

// Originally, this file was named MsgCreateWhois, and has been modified using search-and-replace to our Msg needs.
// 根据MsgCreateWhois文件改写
// MsgBuyName defines the BuyName message
type MsgBuyName struct {
	Name  string         `json:"name"`		// 想购买的域名
	Bid   sdk.Coins      `json:"bid"`		// 出价
	Buyer sdk.AccAddress `json:"buyer"`		// 购买者
}

// NewMsgBuyName is the constructor function for MsgBuyName
// MsgBuyName构造函数
func NewMsgBuyName(name string, bid sdk.Coins, buyer sdk.AccAddress) MsgBuyName {
	return MsgBuyName{
		Name:  name,
		Bid:   bid,
		Buyer: buyer,
	}
}

// Route should return the name of the module
// 路由返回模块名nameService
func (msg MsgBuyName) Route() string { return RouterKey }

// Type should return the action
// 操作类型
func (msg MsgBuyName) Type() string { return "buy_name" }

// ValidateBasic runs stateless checks on the message
// 基本的检查
func (msg MsgBuyName) ValidateBasic() error {
	if msg.Buyer.Empty() {
		return sdkerrors.Wrap(sdkerrors.ErrInvalidAddress, msg.Buyer.String())
	}
	if len(msg.Name) == 0 {
		return sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, "Name cannot be empty")
	}
	if !msg.Bid.IsAllPositive() {
		return sdkerrors.ErrInsufficientFunds
	}
	return nil
}

// GetSignBytes encodes the message for signing
// 返回消息的编码后格式[]byte
func (msg MsgBuyName) GetSignBytes() []byte {
	return sdk.MustSortJSON(ModuleCdc.MustMarshalJSON(msg))
}

// GetSigners defines whose signature is required
// 要求签名的对象
func (msg MsgBuyName) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Buyer}
}
```

最后，更新执行由消息触发的状态转换的BuyName处理程序函数。请记住，此时消息已经运行了ValidateBasic函数，因此已经进行了一些输入验证。但是，==ValidateBasic不能查询应用程序状态。**依赖于网络状态(如账户余额)的验证**逻辑应该在handler函数中执行==

Let's rename `handlerMsgCreateWhois.go` to `handlerMsgBuyName.go`

`mv x/nameservice/handlerMsgCreateWhois.go x/nameservice/handlerMsgBuyName.go`

Go to `./x/nameservice/handlerMsgBuyName.go`

**`handlerMsgBuyName.go`:**

```go
package nameservice

import (
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"

	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/user/nameservice/x/nameservice/keeper"
	"github.com/user/nameservice/x/nameservice/types"
)

// Handle a message to buy name
func handleMsgBuyName(ctx sdk.Context, k keeper.Keeper, msg types.MsgBuyName) (*sdk.Result, error) {
	// Checks if the the bid price is greater than the price paid by the current owner
	// 1.检查当前出价是否高于目前的价格, 注意Msg本身的检查只是简单的检查，这里需要额外数据的检查就只能在Handler中做
	// GetPrice返回coin类型对象，其IsAllGT函数是比较大小（逐个字母比较，全部大于返回true）
	// 当需要的价格 > bid那么就返回错误
	if k.GetPrice(ctx, msg.Name).IsAllGT(msg.Bid) {
		return nil, sdkerrors.Wrap(sdkerrors.ErrInsufficientFunds, "Bid not high enough") // If not, throw an error
	}
	// 2.检查当前域名是否已经有拥有者了
	// 不论是已拥有或者没有人拥有， 如果购买者支付出价出现错误，那么都会造成资金的回滚
	if k.HasCreator(ctx, msg.Name) {
		// 如果已经是别人拥有的，那么购买者支付对应的出价给域名原来的拥有者
		// coin转移方向： msg.Buyer => Creator
		// 金额： Bid
		err := k.CoinKeeper.SendCoins(ctx, msg.Buyer, k.GetCreator(ctx, msg.Name), msg.Bid)
		if err != nil {
			return nil, err
		}
	} else {
		// 如果没有，那么从购买者处减去出价金额, 发送给一个不可回收的地址（burns）
		_, err := k.CoinKeeper.SubtractCoins(ctx, msg.Buyer, msg.Bid) // If so, deduct the Bid amount from the sender
		if err != nil {
			return nil, err
		}
	}
	// 分别为域名设置新的所有者与金额
	k.SetCreator(ctx, msg.Name, msg.Buyer)
	k.SetPrice(ctx, msg.Name, msg.Bid)
	return &sdk.Result{}, nil
}
```

这个处理程序使用来自`coinKeeper`的函数来执行货币操作。如果您的应用程序正在执行货币操作，您可能需要查看此模块的 [godocs for this module (opens new window)](https://godoc.org/github.com/cosmos/cosmos-sdk/x/bank#BaseKeeper)，看看它公开了哪些函数。

## Delete Name 

### MsgDeleteName

Now it is time to update the `Msg` for deleting names. Let's rename our `MsgDeleteWhois.go` to `MsgDeleteName.go`

`mv x/nameservice/types/MsgDeleteWhois.go x/nameservice/types/MsgDeleteName.go`

add it to the `./x/nameservice/types/MsgDeleteName.go` file.

Replace `MsgDeleteWhois` by `MsgDeleteName`:

`mv x/nameservice/types/MsgDeleteWhois.go x/nameservice/types/MsgDeleteName.go`

**`MsgDeleteName.go`:**

```go
package types

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)

var _ sdk.Msg = &MsgDeleteName{}

type MsgDeleteName struct {
	ID      string         `json:"id" yaml:"id"`				
	Creator sdk.AccAddress `json:"creator" yaml:"creator"`		// 创建者
}

func NewMsgDeleteName(id string, creator sdk.AccAddress) MsgDeleteName {
	return MsgDeleteName{
		ID:      id,
		Creator: creator,
	}
}

func (msg MsgDeleteName) Route() string {
	return RouterKey
}

func (msg MsgDeleteName) Type() string {
	return "DeleteName"
}

func (msg MsgDeleteName) GetSigners() []sdk.AccAddress {
	return []sdk.AccAddress{msg.Creator}
}

func (msg MsgDeleteName) GetSignBytes() []byte {
	bz := ModuleCdc.MustMarshalJSON(msg)
	return sdk.MustSortJSON(bz)
}

func (msg MsgDeleteName) ValidateBasic() error {
	if msg.Creator.Empty() {
		return sdkerrors.Wrap(sdkerrors.ErrInvalidAddress, "creator can't be empty")
	}
	return nil
}

```

Replace `handlerMsgDeleteWhois` by `handlerMsgDeleteName`:

`mv x/nameservice/handlerMsgDeleteWhois.go x/nameservice/handlerMsgDeleteName.go `

**`handlerMsgDeleteName.go `:**

```go
package nameservice

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"

	"github.com/user/nameservice/x/nameservice/keeper"
	"github.com/user/nameservice/x/nameservice/types"
)

// Handle a message to delete name
// 删除
func handleMsgDeleteName(ctx sdk.Context, k keeper.Keeper, msg types.MsgDeleteName) (*sdk.Result, error) {
	// 使用id检查是否存在该域名
	if !k.WhoisExists(ctx, msg.ID) {
		// replace with ErrKeyNotFound for 0.39+
		return nil, sdkerrors.Wrap(sdkerrors.ErrInvalidRequest, msg.ID)
	}
	// 检查是否是本人
	if !msg.Creator.Equals(k.GetWhoisOwner(ctx, msg.ID)) {
		return nil, sdkerrors.Wrap(sdkerrors.ErrUnauthorized, "Incorrect Owner")
	}
	k.DeleteWhois(ctx, msg.ID)
	return &sdk.Result{}, nil
}

```

Afterwards, we'll follow the same steps as earlier and add the `MsgDeleteName`handler to the module router in `./x/nameservice/handler.go`:

同样的要在Handler.go函数中注册这个子函数(在上面已经全部注册完毕了)

```go
package nameservice

import (
	"fmt"
	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
	"github.com/user/nameservice/x/nameservice/keeper"
	"github.com/user/nameservice/x/nameservice/types"
)

// NewHandler returns a handler for "nameservice" type messages.
func NewHandler(keeper keeper.Keeper) sdk.Handler {
	return func(ctx sdk.Context, msg sdk.Msg) (*sdk.Result, error) {
		switch msg := msg.(type) {
		// 添加操作类型
		case types.MsgSetName:
			return handleMsgSetName(ctx, keeper, msg)
		case types.MsgBuyName:
			return handleMsgBuyName(ctx, keeper, msg)
		case types.MsgDeleteName:
			return handleMsgDeleteName(ctx, keeper, msg)
		default:
			return nil, sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, fmt.Sprintf("Unrecognized nameservice Msg type: %v", msg.Type()))
		}
	}
}
```

# 10.Queriers

## Query Types

Start by navigating to `./x/nameservice/types/querier.go` file. This is where you will define your querier types

查询模块.

```go
package types

import "strings"

// 查询常量，对应客户端输入的参数
const QueryListWhois = "list-whois"
const QueryGetWhois = "get-whois"
const QueryResolveName = "resolve-name"

// QueryResResolve Queries Result Payload for a resolve query
// 查询域名解析结果结构体函数
// 因为MarshalJSONIndent解析需要一个结构体， 所以创建了这样的QueryResResolve结构体以赋值
type QueryResResolve struct {
	Value string `json:"value"`
}

// implement fmt.Stringer
// 重写string方法
func (r QueryResResolve) String() string {
	return r.Value
}

// QueryResNames Queries Result Payload for a names query
// 查询域名群集合的解析结果，返回结果的切片
type QueryResNames []string

// implement fmt.Stringer
// 格式化输出解析结果切片的数据
func (n QueryResNames) String() string {
	return strings.Join(n[:], "\n")
}
```

## Querier

Now you can navigate to the `./x/nameservice/keeper/querier.go` file.

在这里可以定义**针对应用程序状态用户可以进行哪些查询**

Your `nameservice` module will expose three queries:

- `resolveName`: This takes a `name` and returns the `value` that is stored by the `nameservice`. This is similar to a DNS query.

  **解析值**

- `getWhois`: This takes a `name` and returns the `price`, `value`, and `owner` of the name. Used for figuring out how much names cost when you want to buy them.

  **查询域名的所有相关信息**

- `listWhois` : This does not take a parameter, it returns all the names stored in the `nameservice` store.

  **查询所有的已存在域名**

你需要修改switch语句的用例(它们不能从query .Route()函数中取出):

```go
package keeper

import (
	// this line is used by starport scaffolding # 1
	"github.com/user/nameservice/x/nameservice/types"

	abci "github.com/tendermint/tendermint/abci/types"

	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
)

// NewQuerier creates a new querier for nameservice clients.
// 创建了一个keeper层的查询对象给客户端
func NewQuerier(k Keeper) sdk.Querier {
	return func(ctx sdk.Context, path []string, req abci.RequestQuery) ([]byte, error) {
		switch path[0] { // 根据客户端输入的路径的第一个变量，确定查询的类型
		// this line is used by starport scaffolding # 2
		// 客户端输入的内容在types/querier.go中定义了常量作为路由
		case types.QueryResolveName:
			return resolveName(ctx, path[1:], k)
		case types.QueryListWhois:
			return listWhois(ctx, k)
		case types.QueryGetWhois:
			return getWhois(ctx, path[1:], k)
		default:
			return nil, sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, "unknown nameservice query endpoint")
		}
	}
}
```

Now that the router is defined, we can verify that our querier functions in `./x/nameservice/keeper/whois.go` looks like this:

**现在我们可以在上述文件中找到对应路由的实现:**

**之前在`./x/nameservice/keeper/whois.go`中可以找到的Functions used by querier部分(不是keeper方法的函数都是querier)**

```go
//
// Functions used by querier
//

func listWhois(ctx sdk.Context, k Keeper) ([]byte, error) {
	var whoisList []types.Whois
	store := ctx.KVStore(k.storeKey)
	iterator := sdk.KVStorePrefixIterator(store, []byte(types.WhoisPrefix))
	for ; iterator.Valid(); iterator.Next() {
		var whois types.Whois
		k.cdc.MustUnmarshalBinaryLengthPrefixed(store.Get(iterator.Key()), &whois)
		whoisList = append(whoisList, whois)
	}
	res := codec.MustMarshalJSONIndent(k.cdc, whoisList)
	return res, nil
}

func getWhois(ctx sdk.Context, path []string, k Keeper) (res []byte, sdkError error) {
	key := path[0]
	whois, err := k.GetWhois(ctx, key)
	if err != nil {
		return nil, err
	}

	res, err = codec.MarshalJSONIndent(k.cdc, whois)
	if err != nil {
		return nil, sdkerrors.Wrap(sdkerrors.ErrJSONMarshal, err.Error())
	}

	return res, nil
}

// Resolves a name, returns the value
func resolveName(ctx sdk.Context, path []string, keeper Keeper) ([]byte, error) {
	value := keeper.ResolveName(ctx, path[0])

	if value == "" {
		return []byte{}, sdkerrors.Wrap(sdkerrors.ErrUnknownRequest, "could not resolve name")
	}

	res, err := codec.MarshalJSONIndent(keeper.cdc, types.QueryResResolve{Value: value})
	if err != nil {
		return nil, sdkerrors.Wrap(sdkerrors.ErrJSONMarshal, err.Error())
	}

	return res, nil
}
```

Note that `listWhois` and `getWhois` should already be defined, so you would only need to add `resolveName`.

* 在这里，你的Keeper的getter和setter被大量使用。当构建任何其他使用此模块的应用程序时，您可能需要返回并定义更多的getter /setter来访问您需要的状态片段。

* **按照约定，==每个输出类型都应该是JSON可编组的和字符串可编的==(实现了Golang fmt接口)**。**返回的字节应该是输出结果的JSON编码。**
* 因此，对于resolve的输出类型，我们**将解析字符串包装在一个名为QueryResResolve的结构中，该结构既可用于JSON编组，又有一个. string()方法。**
  
* 对于Whois的输出，<u>正常的Whois结构已经是可以JSON编组的，但是我们需要在其上添加一个. string()方法。</u>
  * 对于names查询的输出也是一样的，<u>[]字符串已经是本机可编组的，但是我们想在其上添加一个. string()方法</u>。
  
* 类型Whois没有在`./x/nameservice/types/`查询器中定义。因为它是在`./x/nameservice/types/TypeWhois. go`文件中创建的.go文件。

# 11.Codec File

there is a bit of code that needs to be placed in `./x/nameservice/types/codec.go`. Any interface you create and any struct that implements an interface needs to be declared in the `RegisterCodec` function

一系列代码需要修改,许多实现的接口等函数和结构体都需要在RegisterCodec函数中申明

 In this module the three `Msg`implementations (`SetName`, `BuyName` and `DeleteName`) have been registered, but your `Whois` query return type needs to be registered.

在此案例中,我们创建了三个Msg结构的实现:(`SetName`, `BuyName` and `DeleteName`) ,这些都需要被注册

```go
package types

import (
	"github.com/cosmos/cosmos-sdk/codec"
)

// RegisterCodec registers concrete types on codec
// codec上注册具体的类型
func RegisterCodec(cdc *codec.Codec) {
	// this line is used by starport scaffolding # 1
	cdc.RegisterConcrete(MsgBuyName{}, "nameservice/BuyName", nil)
	cdc.RegisterConcrete(MsgSetName{}, "nameservice/SetName", nil)
	cdc.RegisterConcrete(MsgDeleteName{}, "nameservice/DeleteName", nil)
}

// ModuleCdc defines the module codec
var ModuleCdc *codec.Codec

func init() {
	// 创建实例codec
	ModuleCdc = codec.New()
	RegisterCodec(ModuleCdc)
	// Register the go-crypto to the codec
	// 注册加密函数信息
	codec.RegisterCrypto(ModuleCdc)
	// 封装
	ModuleCdc.Seal()
}
```

```go
// This function should be used to register concrete types that will appear in
// interface fields/elements to be encoded/decoded by go-amino.
// Usage:
// `amino.RegisterConcrete(MyStruct1{}, "com.tendermint/MyStruct1", nil)`
//这个函数应该用来注册将出现在go-amino编码/解码的接口字段/元素中的具体类型。
func (cdc *Codec) RegisterConcrete(o interface{}, name string, copts *ConcreteOptions) {...}
```

# 12.Nameservice Module CLI

cosmos sdk使用了 [`cobra` (opens new window)](https://github.com/spf13/cobra)客户端工具

This library makes it easy for each module to expose its own commands. The `type` command should have scaffolded the following files for us -

- `./x/nameservice/client/cli/queryWhois.go`
- `./x/nameservice/client/cli/txWhois.go`

## Queries

Start in `queryWhois.go`. Here, define `cobra.Command`s for each of your modules `Queriers` (`resolve`, and `whois`):

实现查询的客户端命令逻辑:

```go
package cli

import (
	"fmt"

	"github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/spf13/cobra"
	"github.com/user/nameservice/x/nameservice/types"
)

// 获取所有的whois
func GetCmdListWhois(queryRoute string, cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "list-whois",		// 使用的命令
		Short: "list all whois",	// 介绍
		// 运行的内容
		RunE: func(cmd *cobra.Command, args []string) error {
			cliCtx := context.NewCLIContext().WithCodec(cdc)
			res, _, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/%s", queryRoute, types.QueryListWhois), nil)
			if err != nil {
				fmt.Printf("could not list Whois\n%s\n", err.Error())
				return nil
			}
			var out []types.Whois
			// 解码结果放到out中
			cdc.MustUnmarshalJSON(res, &out)
			return cliCtx.PrintOutput(out)
		},
	}
}

// 获取一个whois
func GetCmdGetWhois(queryRoute string, cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "get-whois [key]",
		Short: "Query a whois by key",
		Args:  cobra.ExactArgs(1),		// 参数对应key
		RunE: func(cmd *cobra.Command, args []string) error {
			cliCtx := context.NewCLIContext().WithCodec(cdc)
			key := args[0]			// 获取命令中的key
			res, _, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/%s/%s", queryRoute, types.QueryGetWhois, key), nil)
			if err != nil {
				fmt.Printf("could not resolve whois %s \n%s\n", key, err.Error())
				return nil
			}

			var out types.Whois
			cdc.MustUnmarshalJSON(res, &out)
			return cliCtx.PrintOutput(out)
		},
	}
}

// GetCmdResolveName queries information about a name
// 查询域名的解析值
func GetCmdResolveName(queryRoute string, cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "resolve [name]",
		Short: "resolve name",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {
			cliCtx := context.NewCLIContext().WithCodec(cdc)
			name := args[0]
			res, _, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/%s/%s", queryRoute, types.QueryResolveName, name), nil)
			if err != nil {
				fmt.Printf("could not resolve name - %s \n", name)
				return nil
			}

			var out types.QueryResResolve
			cdc.MustUnmarshalJSON(res, &out)
			return cliCtx.PrintOutput(out)
		},
	}
}

```

* cli创建了一个新的`CLIContext`, 它携带了有关cli交互所需要的用户输入和应用程序配置的数据

* `cliCtx.QueryWithdata()`函数所需的路径直接映射到查询路由器中的名称。
  *  路径的第一部分用于区分SDK应用程序可能使用的查询类型:`custom` is for `Queriers`.
  * 第二个部分是要查询路由的模块的名字
  * 最后，模块中将有一个特定的查询器，该查询器将被调用
  * 在本例中，第四部分是查询。这是因为查询参数是一个简单的字符串。要启用更复杂的查询输入，您需要使用querywithdata()函数的第二个参数来传递数据。有关此示例，请参阅Staking模块中的查询器.

在`cli/query.go`中添加子命令:

==**这个文件官方没有写,但是需要改动!**==

 **==注意一定添加`GetCmdResolveName(queryRoute, cdc),`, 不然的话使用命令行测试时resolve命令无法解析!!!==**

```go
package cli

import (
	"fmt"
	// "strings"

	"github.com/spf13/cobra"

	"github.com/cosmos/cosmos-sdk/client"
	"github.com/cosmos/cosmos-sdk/client/flags"

	// "github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/codec"
	// sdk "github.com/cosmos/cosmos-sdk/types"

	"github.com/user/nameservice/x/nameservice/types"
)

// GetQueryCmd returns the cli query commands for this module
func GetQueryCmd(queryRoute string, cdc *codec.Codec) *cobra.Command {
	// Group nameservice queries under a subcommand
	nameserviceQueryCmd := &cobra.Command{
		Use:                        types.ModuleName,
		Short:                      fmt.Sprintf("Querying commands for the %s module", types.ModuleName),
		DisableFlagParsing:         true,
		SuggestionsMinimumDistance: 2,
		RunE:                       client.ValidateCmd,
	}

	nameserviceQueryCmd.AddCommand(
		flags.GetCommands(
      // this line is used by starport scaffolding # 1
			GetCmdListWhois(queryRoute, cdc),
			GetCmdGetWhois(queryRoute, cdc),
      // 注意一定添加这个命令, 不然的话使用命令行测试时resolve命令无法解析!!!
			GetCmdResolveName(queryRoute, cdc),
		)...,
	)
	return nameserviceQueryCmd
}
```

## Transactions

实现交易命令的客户端

Now that the query interactions are defined, it is time to move on to transaction generation in `txWhois.go`:

**NOTE**: Your application needs to import the code you just wrote. Here the import path is set to this repository (`github.com/cosmos/sdk-tutorials/nameservice/x/nameservice`). If you are following along in your own repo you will need to change the import path to reflect that (`github.com/{ .Username }/{ .Project.Repo }/x/nameservice`). 

如果你用的是自己的仓库记得改路径

```go
package cli

import (
	"bufio"

	"github.com/spf13/cobra"

	"github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/cosmos/cosmos-sdk/x/auth"
	"github.com/cosmos/cosmos-sdk/x/auth/client/utils"
	"github.com/user/nameservice/x/nameservice/types"
)

// 购买新域名
func GetCmdBuyName(cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "buy-name [name] [price]",
		Short: "Buys a new name",
		Args:  cobra.ExactArgs(2),
		RunE: func(cmd *cobra.Command, args []string) error {
			argsName := string(args[0])		//获取购买的名字name

			cliCtx := context.NewCLIContext().WithCodec(cdc)
			// 获取标准读入
			inBuf := bufio.NewReader(cmd.InOrStdin())
			// 创建交易创建器
			txBldr := auth.NewTxBuilderFromCLI(inBuf).WithTxEncoder(utils.GetTxEncoder(cdc))

			coins, err := sdk.ParseCoins(args[1])	//解析出价
			if err != nil {
				return err
			}
			// 构建NewMsgBuyName实例
			msg := types.NewMsgBuyName(argsName, coins, cliCtx.GetFromAddress())
			// 做基本的验证
			err = msg.ValidateBasic()
			if err != nil {
				return err
			}
			// 生成或广播交易
			return utils.GenerateOrBroadcastMsgs(cliCtx, txBldr, []sdk.Msg{msg})
		},
	}
}

// 设置域名解析
func GetCmdSetWhois(cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "set-name [value] [name]",
		Short: "Set a new name",
		Args:  cobra.ExactArgs(2),
		RunE: func(cmd *cobra.Command, args []string) error {
			argsValue := args[0]
			argsName := args[1]

			cliCtx := context.NewCLIContext().WithCodec(cdc)
			inBuf := bufio.NewReader(cmd.InOrStdin())
			txBldr := auth.NewTxBuilderFromCLI(inBuf).WithTxEncoder(utils.GetTxEncoder(cdc))
			msg := types.NewMsgSetName(argsName, argsValue, cliCtx.GetFromAddress())
			err := msg.ValidateBasic()
			if err != nil {
				return err
			}
			return utils.GenerateOrBroadcastMsgs(cliCtx, txBldr, []sdk.Msg{msg})
		},
	}
}

// 删除一个域名
func GetCmdDeleteWhois(cdc *codec.Codec) *cobra.Command {
	return &cobra.Command{
		Use:   "delete-name [id]",
		Short: "Delete a new name by ID",
		Args:  cobra.ExactArgs(1),
		RunE: func(cmd *cobra.Command, args []string) error {

			cliCtx := context.NewCLIContext().WithCodec(cdc)
			inBuf := bufio.NewReader(cmd.InOrStdin())
			txBldr := auth.NewTxBuilderFromCLI(inBuf).WithTxEncoder(utils.GetTxEncoder(cdc))

			msg := types.NewMsgDeleteName(args[0], cliCtx.GetFromAddress())
			err := msg.ValidateBasic()
			if err != nil {
				return err
			}
			return utils.GenerateOrBroadcastMsgs(cliCtx, txBldr, []sdk.Msg{msg})
		},
	}
}
```

We also need to add the commands to our `tx` command in

`x/nameservice/client/cli/tx.go` file:

`./x/nameservice/client/cli/tx.go`

```go
package cli

import (
	"fmt"

	"github.com/spf13/cobra"

	"github.com/cosmos/cosmos-sdk/client"
	"github.com/cosmos/cosmos-sdk/client/flags"
	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/user/nameservice/x/nameservice/types"
)

// GetTxCmd returns the transaction commands for this module
// 返回模块的交易命令
func GetTxCmd(cdc *codec.Codec) *cobra.Command {
	nameserviceTxCmd := &cobra.Command{
		Use:                        types.ModuleName,
		Short:                      fmt.Sprintf("%s transactions subcommands", types.ModuleName),
		DisableFlagParsing:         true,
		SuggestionsMinimumDistance: 2,
		RunE:                       client.ValidateCmd,
	}

		// 把txWhois.go中的命令都添加
	nameserviceTxCmd.AddCommand(flags.PostCommands(
		// this line is used by starport scaffolding
		GetCmdBuyName(cdc),
		GetCmdSetWhois(cdc),
		GetCmdDeleteWhois(cdc),
	)...)

	return nameserviceTxCmd
}
```

这里使用的是authcmd包。它提供对CLI控制的帐户的访问，并方便签名。

# 13.NameService Module Rest Interface

你的模型也可以使用REST接口实现命令行客户端

To get started navigate to `./x/nameservice/client/rest/rest.go` where HTTP handlers are held.

## RegisterRoutes

首先定义REST接口在`RegisterRoutes`函数中

使路由全部以您的模块名称开头，以防止名称空间与其他模块的路由冲突

**`./x/nameservice/client/rest/rest.go`：**

```go
package rest

import (
	"github.com/gorilla/mux"

	"github.com/cosmos/cosmos-sdk/client/context"
)

// RegisterRoutes registers nameservice-related REST handlers to a router
func RegisterRoutes(cliCtx context.CLIContext, r *mux.Router) {
	// this line is used by starport scaffolding
	r.HandleFunc("/nameservice/whois", buyNameHandler(cliCtx)).Methods("POST")
	r.HandleFunc("/nameservice/whois", listWhoisHandler(cliCtx, "nameservice")).Methods("GET")
	r.HandleFunc("/nameservice/whois/{key}", getWhoisHandler(cliCtx, "nameservice")).Methods("GET")
	r.HandleFunc("/nameservice/whois/{key}/resolve", resolveNameHandler(cliCtx, "nameservice")).Methods("GET")
	r.HandleFunc("/nameservice/whois", setWhoisHandler(cliCtx)).Methods("PUT")
	r.HandleFunc("/nameservice/whois", deleteWhoisHandler(cliCtx)).Methods("DELETE")
}
```

## Query Handlers

Next, its time to define the handlers mentioned above in `queryWhois.go`. These will be very similar to the CLI methods defined earlier. `listWhoisHandler` and `getWhoisHandler` should already be defined, and you can use `getWhois` as a template to write the `resolveNameHandler` function.

**查询接口函数的编写(类似于cli的编写): `queryWhois.go`**

```go
package rest

import (
	"fmt"
	"net/http"

	"github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/types/rest"
	"github.com/gorilla/mux"
	"github.com/user/nameservice/x/nameservice/types"
)

// 查询所有的whois
func listWhoisHandler(cliCtx context.CLIContext, storeName string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		res, _, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/%s", storeName, types.QueryListWhois), nil)
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusNotFound, err.Error())
			return
		}
		rest.PostProcessResponse(w, cliCtx, res)
	}
}

// 获取一个域名
func getWhoisHandler(cliCtx context.CLIContext, storeName string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// 获取参数
		vars := mux.Vars(r)
		key := vars["key"]

		res, _, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/%s/%s", storeName, types.QueryGetWhois, key), nil)
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusNotFound, err.Error())
			return
		}
		rest.PostProcessResponse(w, cliCtx, res)
	}
}

// 解析域名
func resolveNameHandler(cliCtx context.CLIContext, storeName string) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		vars := mux.Vars(r)
		paramType := vars["key"]

		res, _, err := cliCtx.QueryWithData(fmt.Sprintf("custom/%s/%s/%s", storeName, types.QueryResolveName, paramType), nil)
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusNotFound, err.Error())
			return
		}

		rest.PostProcessResponse(w, cliCtx, res)
	}
}
```

Notes on the above code:

- Notice we are using the same `cliCtx.QueryWithData` function to fetch the data
- These functions are almost the same as the corresponding CLI functionality

同样使用了QueryWithData函数,编写类似于cli

## Tx Handlers

Now define the `buyName`, `setName` and `deleteName` transaction routes in `txWhois.go` - you can replace the existing handlers that were generated by `starport type`. 

请注意，这些实际上并没有执行购买，设置和删除名称的交易，因为在一般情况下，这需要某种形式的身份验证。取而代之的是，这些端点构建并返回每个特定的交易，然后可以以安全的方式对其进行签名，然后使用诸如`/txs`

```go
package rest

import (
	"fmt"
	"net/http"

	"github.com/cosmos/cosmos-sdk/client/context"
	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/cosmos/cosmos-sdk/types/rest"
	"github.com/cosmos/cosmos-sdk/x/auth/client/utils"
	"github.com/user/nameservice/x/nameservice/types"
)

type buyNameRequest struct {
	BaseReq rest.BaseReq `json:"base_req"`		// 包含了创建交易的基本的请求字段
	Buyer   string       `json:"buyer"`
	Name    string       `json:"name"`
	Price   string       `json:"price"`
}

// 购买域名
func buyNameHandler(cliCtx context.CLIContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req buyNameRequest
		// 读取请求
		if !rest.ReadRESTReq(w, r, cliCtx.Codec, &req) {
			rest.WriteErrorResponse(w, http.StatusBadRequest, "failed to parse request")
			return
		}
		baseReq := req.BaseReq.Sanitize()
		// 基本的验证
		if !baseReq.ValidateBasic(w) {
			return
		}
		// AccAddressFromBech32转换string为32位地址的方法
		addr, err := sdk.AccAddressFromBech32(req.Buyer)
		fmt.Println(addr)
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusBadRequest, err.Error())
			return
		}
		// 解析金额
		// ParseCoins 将字符串转为coin
		coins, err := sdk.ParseCoins(req.Price)
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusBadRequest, err.Error())
			return
		}
		// 创建NewMsgBuyName对象
		msg := types.NewMsgBuyName(req.Name, coins, addr)
		// 简单的验证
		err = msg.ValidateBasic()
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusBadRequest, err.Error())
			return
		}
		// 返回响应
		utils.WriteGenerateStdTxResponse(w, cliCtx, baseReq, []sdk.Msg{msg})
	}
}

type setWhoisRequest struct {
	BaseReq rest.BaseReq `json:"base_req"`
	Name    string       `json:"name"`
	Value   string       `json:"value"`
	Creator string       `json:"creator"`
}

// 设置解析值
func setWhoisHandler(cliCtx context.CLIContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req setWhoisRequest
		if !rest.ReadRESTReq(w, r, cliCtx.Codec, &req) {
			rest.WriteErrorResponse(w, http.StatusBadRequest, "failed to parse request")
			return
		}
		baseReq := req.BaseReq.Sanitize()
		if !baseReq.ValidateBasic(w) {
			return
		}
		addr, err := sdk.AccAddressFromBech32(req.Creator)
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusBadRequest, err.Error())
			return
		}
		msg := types.NewMsgSetName(req.Name, req.Value, addr)

		err = msg.ValidateBasic()
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusBadRequest, err.Error())
			return
		}

		utils.WriteGenerateStdTxResponse(w, cliCtx, baseReq, []sdk.Msg{msg})
	}
}

type deleteWhoisRequest struct {
	BaseReq rest.BaseReq `json:"base_req"`
	Owner   string       `json:"owner"`
	Name    string       `json:"name"`
}

// 删除消息
func deleteWhoisHandler(cliCtx context.CLIContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req deleteWhoisRequest
		if !rest.ReadRESTReq(w, r, cliCtx.Codec, &req) {
			rest.WriteErrorResponse(w, http.StatusBadRequest, "failed to parse request")
			return
		}
		baseReq := req.BaseReq.Sanitize()
		if !baseReq.ValidateBasic(w) {
			return
		}
		addr, err := sdk.AccAddressFromBech32(req.Owner)
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusBadRequest, err.Error())
			return
		}
		msg := types.NewMsgDeleteName(req.Name, addr)

		err = msg.ValidateBasic()
		if err != nil {
			rest.WriteErrorResponse(w, http.StatusBadRequest, err.Error())
			return
		}

		utils.WriteGenerateStdTxResponse(w, cliCtx, baseReq, []sdk.Msg{msg})
	}
}
```

注意点:

*  [`BaseReq` ](https://godoc.org/github.com/cosmos/cosmos-sdk/client/utils#BaseReq)包含了创建交易的基本的请求字段 (which key to use, how to decode it, which chain you are on, etc...)并且设计成嵌入

* `baseReq.ValidateBasic` handles setting the response code for you and therefore you don't need to worry about handling errors or successes when using those functions.

  **`baseReq.ValidateBasic`为你设置了响应代码**

# 14.AppModule Interface

The Cosmos SDK provides a standard interface for modules. This [`AppModule` (opens new window)](https://github.com/cosmos/cosmos-sdk/blob/master/types/module.go)interface requires modules to provide a set of methods used by the `ModuleBasicsManager` to incorporate them into your application.

We should already have a `module.go` file in `./nameservice`, and we don't need to change anything, but it should look like this.

Cosmos SDK为模块提供了一个标准接口。这个AppModule接口要求模块提供一组ModuleBasicsManager使用的方法，以便将它们合并到你的应用程序中。

我们应该已经有一个模块了。进入`./nameservice`文件，我们不需要修改任何东西，但它应该是这样的。

**`x/nameservice/module.go`:**

```go
package nameservice

import (
	"encoding/json"

	"github.com/gorilla/mux"
	"github.com/spf13/cobra"

	abci "github.com/tendermint/tendermint/abci/types"

	"github.com/cosmos/cosmos-sdk/client/context"
	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/cosmos/cosmos-sdk/types/module"
	"github.com/cosmos/cosmos-sdk/x/bank"
	"github.com/user/nameservice/x/nameservice/client/cli"
	"github.com/user/nameservice/x/nameservice/client/rest"
	"github.com/user/nameservice/x/nameservice/keeper"
	"github.com/user/nameservice/x/nameservice/types"
)

// Type check to ensure the interface is properly implemented
var (
	_ module.AppModule      = AppModule{}
	_ module.AppModuleBasic = AppModuleBasic{}
)

// AppModuleBasic defines the basic application module used by the nameservice module.
type AppModuleBasic struct{}

// Name returns the nameservice module's name.
func (AppModuleBasic) Name() string {
	return types.ModuleName
}

// RegisterCodec registers the nameservice module's types for the given codec.
func (AppModuleBasic) RegisterCodec(cdc *codec.Codec) {
	types.RegisterCodec(cdc)
}

// DefaultGenesis returns default genesis state as raw bytes for the nameservice
// module.
func (AppModuleBasic) DefaultGenesis() json.RawMessage {
	return types.ModuleCdc.MustMarshalJSON(types.DefaultGenesisState())
}

// ValidateGenesis performs genesis state validation for the nameservice module.
func (AppModuleBasic) ValidateGenesis(bz json.RawMessage) error {
	var data types.GenesisState
	err := types.ModuleCdc.UnmarshalJSON(bz, &data)
	if err != nil {
		return err
	}
	return types.ValidateGenesis(data)
}

// RegisterRESTRoutes registers the REST routes for the nameservice module.
func (AppModuleBasic) RegisterRESTRoutes(ctx context.CLIContext, rtr *mux.Router) {
	rest.RegisterRoutes(ctx, rtr)
}

// GetTxCmd returns the root tx command for the nameservice module.
func (AppModuleBasic) GetTxCmd(cdc *codec.Codec) *cobra.Command {
	return cli.GetTxCmd(cdc)
}

// GetQueryCmd returns no root query command for the nameservice module.
func (AppModuleBasic) GetQueryCmd(cdc *codec.Codec) *cobra.Command {
	return cli.GetQueryCmd(types.StoreKey, cdc)
}

//____________________________________________________________________________

// AppModule implements an application module for the nameservice module.
type AppModule struct {
	AppModuleBasic

	keeper     keeper.Keeper
	coinKeeper bank.Keeper
	// TODO: Add keepers that your application depends on

}

// NewAppModule creates a new AppModule object
func NewAppModule(k keeper.Keeper, bankKeeper bank.Keeper) AppModule {
	return AppModule{
		AppModuleBasic: AppModuleBasic{},
		keeper:         k,
		coinKeeper:     bankKeeper,
		// TODO: Add keepers that your application depends on
	}
}

// Name returns the nameservice module's name.
func (AppModule) Name() string {
	return types.ModuleName
}

// RegisterInvariants registers the nameservice module invariants.
func (am AppModule) RegisterInvariants(_ sdk.InvariantRegistry) {}

// Route returns the message routing key for the nameservice module.
func (AppModule) Route() string {
	return types.RouterKey
}

// NewHandler returns an sdk.Handler for the nameservice module.
func (am AppModule) NewHandler() sdk.Handler {
	return NewHandler(am.keeper)
}

// QuerierRoute returns the nameservice module's querier route name.
func (AppModule) QuerierRoute() string {
	return types.QuerierRoute
}

// NewQuerierHandler returns the nameservice module sdk.Querier.
func (am AppModule) NewQuerierHandler() sdk.Querier {
	return keeper.NewQuerier(am.keeper)
}

// InitGenesis performs genesis initialization for the nameservice module. It returns
// no validator updates.
func (am AppModule) InitGenesis(ctx sdk.Context, data json.RawMessage) []abci.ValidatorUpdate {
	var genesisState types.GenesisState
	types.ModuleCdc.MustUnmarshalJSON(data, &genesisState)
	InitGenesis(ctx, am.keeper, genesisState)
	return []abci.ValidatorUpdate{}
}

// ExportGenesis returns the exported genesis state as raw bytes for the nameservice
// module.
func (am AppModule) ExportGenesis(ctx sdk.Context) json.RawMessage {
	gs := ExportGenesis(ctx, am.keeper)
	return types.ModuleCdc.MustMarshalJSON(gs)
}

// BeginBlock returns the begin blocker for the nameservice module.
func (am AppModule) BeginBlock(ctx sdk.Context, req abci.RequestBeginBlock) {
	BeginBlocker(ctx, req, am.keeper)
}

// EndBlock returns the end blocker for the nameservice module. It returns no validator
// updates.
func (AppModule) EndBlock(_ sdk.Context, _ abci.RequestEndBlock) []abci.ValidatorUpdate {
	return []abci.ValidatorUpdate{}
}
```

# 15.Genesis

AppModule接口包含了许多用于初始化和导出初始化状态的区块链函数。当启动、停止或导出链时，ModuleBasicManager在每个模块上调用这些函数。下面是一个非常基本的实现，您可以对其进行扩展。

在 `x/nameservice/types/genesis.go`。我们会定义初始状态是什么，默认的初始状态以及验证它的方法这样我们就不会遇到任何错误当我们以预先存在的状态开始链的时候。

```go
package types

import "fmt"

// GenesisState - all nameservice state that must be provided at genesis
type GenesisState struct {
	// TODO: Fill out what is needed by the module for genesis
	WhoisRecords []Whois `json:"whois_records"`
}

// NewGenesisState creates a new GenesisState object
func NewGenesisState( /* TODO: Fill out with what is needed for genesis state */ ) GenesisState {
	return GenesisState{
		// TODO: Fill out according to your genesis state
		WhoisRecords: nil,
	}
}

// DefaultGenesisState - default GenesisState used by Cosmos Hub
func DefaultGenesisState() GenesisState {
	return GenesisState{
		WhoisRecords: []Whois{},
	}
}

// ValidateGenesis validates the nameservice genesis parameters
func ValidateGenesis(data GenesisState) error {
	// TODO: Create a sanity check to make sure the state conforms to the modules needs
	for _, record := range data.WhoisRecords {
		if record.Creator == nil {
			return fmt.Errorf("invalid WhoisRecord: Creator: %s. Error: Missing Creator", record.Creator)
		}
		if record.Value == "" {
			return fmt.Errorf("invalid WhoisRecord: Value: %s. Error: Missing Value", record.Value)
		}
		if record.Price == nil {
			return fmt.Errorf("invalid WhoisRecord: Price: %s. Error: Missing Price", record.Price)
		}
	}
	return nil
}
```

Next we can update our `x/nameservice/genesis.go` file, and modify the functions `InitGenesis` and `ExportGenesis`

```go
package nameservice

import (
	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/user/nameservice/x/nameservice/keeper"
	"github.com/user/nameservice/x/nameservice/types"
	// abci "github.com/tendermint/tendermint/abci/types"
)

// InitGenesis initialize default parameters
// and the keeper's address to pubkey map
func InitGenesis(ctx sdk.Context, keeper keeper.Keeper, data types.GenesisState) {
	for _, record := range data.WhoisRecords {
		keeper.SetWhois(ctx, record.Value, record)
	}
}

// ExportGenesis writes the current store values
// to a genesis file, which can be imported again
// with InitGenesis
func ExportGenesis(ctx sdk.Context, k keeper.Keeper) types.GenesisState {
	var records []types.Whois
	iterator := k.GetNamesIterator(ctx)
	for ; iterator.Valid(); iterator.Next() {

		name := string(iterator.Key())
		whois, _ := k.GetWhois(ctx, name)
		records = append(records, whois)

	}
	return types.GenesisState{WhoisRecords: records}
}
```

关于上述代码的一些注意事项:

* `ValidateGenesis()`验证提供的生成状态，以确保所期望的不变量保持不变
* `DefaultGenesisState()`主要用于测试。这提供了一个最小的起源状态。
* 在链启动时调用`InitGenesis()`，该函数将生成状态导入到keeper中。
* `ExportGenesis()`在停止链后被调用，此函数将应用程序状态加载到GenesisState结构中，以便稍后导出到`genesis.json`以及来自其他模块的数据。

**==用来配置区块数据保存/再记载的逻辑==**

# 16 Complete App

When you used the `starport type` command, your application has already been incorporated in the `.app/app.go` file.

在`app/app.go`文件，它做了以下改变:

* 从每个需要的模块实例化所需的保存器。

* 生成每个管理员Keeper所需的storekey。

* 从每个模块注册处理程序。baseapp路由器的AddRoute()方法用于此目的。

* 从每个模块注册查询器。baseapp的queryRouter中的AddRoute()方法用于此目的。

* 将KVStores挂载到baseApp多存储区中提供的密钥。

* 设置initChainer以定义初始应用程序状态。

因此，该文件应该如下所示

```go
package app

import (
	"encoding/json"
	"io"
	"os"

	abci "github.com/tendermint/tendermint/abci/types"
	"github.com/tendermint/tendermint/libs/log"
	tmos "github.com/tendermint/tendermint/libs/os"
	dbm "github.com/tendermint/tm-db"

	bam "github.com/cosmos/cosmos-sdk/baseapp"
	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/cosmos/cosmos-sdk/simapp"
	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/cosmos/cosmos-sdk/types/module"
	"github.com/cosmos/cosmos-sdk/version"
	"github.com/cosmos/cosmos-sdk/x/auth"
	"github.com/cosmos/cosmos-sdk/x/bank"
	"github.com/cosmos/cosmos-sdk/x/genutil"
	"github.com/cosmos/cosmos-sdk/x/params"
	"github.com/cosmos/cosmos-sdk/x/staking"
	"github.com/cosmos/cosmos-sdk/x/supply"
	"github.com/user/nameservice/x/nameservice"
	nameservicekeeper "github.com/user/nameservice/x/nameservice/keeper"
	nameservicetypes "github.com/user/nameservice/x/nameservice/types"
	// this line is used by starport scaffolding
)

const appName = "nameservice"

var (
	// 在用户目录创建两个命令文件
	// 客户端执行文件
	DefaultCLIHome  = os.ExpandEnv("$HOME/.nameservicecli")
	// 节点执行文件
	DefaultNodeHome = os.ExpandEnv("$HOME/.nameserviced")
	// 模型的基本模块引入
	ModuleBasics    = module.NewBasicManager(
		genutil.AppModuleBasic{},
		auth.AppModuleBasic{},
		bank.AppModuleBasic{},
		staking.AppModuleBasic{},
		params.AppModuleBasic{},
		supply.AppModuleBasic{},
		nameservice.AppModuleBasic{},
		// this line is used by starport scaffolding # 2
	)

	maccPerms = map[string][]string{
		auth.FeeCollectorName:     nil,
		staking.BondedPoolName:    {supply.Burner, supply.Staking},
		staking.NotBondedPoolName: {supply.Burner, supply.Staking},
	}
)

// 创建codec
func MakeCodec() *codec.Codec {
	var cdc = codec.New()

	ModuleBasics.RegisterCodec(cdc)
	sdk.RegisterCodec(cdc)
	codec.RegisterCrypto(cdc)

	return cdc.Seal()
}

// 新的app结构体
type NewApp struct {
	*bam.BaseApp
	cdc *codec.Codec

	invCheckPeriod uint

	keys  map[string]*sdk.KVStoreKey
	tKeys map[string]*sdk.TransientStoreKey

	subspaces map[string]params.Subspace

	accountKeeper     auth.AccountKeeper
	bankKeeper        bank.Keeper
	stakingKeeper     staking.Keeper
	supplyKeeper      supply.Keeper
	paramsKeeper      params.Keeper
	nameserviceKeeper nameservicekeeper.Keeper
	// this line is used by starport scaffolding # 3
	mm *module.Manager

	sm *module.SimulationManager
}

var _ simapp.App = (*NewApp)(nil)

// 初始化app
func NewInitApp(
	logger log.Logger, db dbm.DB, traceStore io.Writer, loadLatest bool,
	invCheckPeriod uint, baseAppOptions ...func(*bam.BaseApp),
) *NewApp {
	cdc := MakeCodec()

	bApp := bam.NewBaseApp(appName, logger, db, auth.DefaultTxDecoder(cdc), baseAppOptions...)
	bApp.SetCommitMultiStoreTracer(traceStore)
	bApp.SetAppVersion(version.Version)

	keys := sdk.NewKVStoreKeys(
		bam.MainStoreKey,
		auth.StoreKey,
		staking.StoreKey,
		supply.StoreKey,
		params.StoreKey,
		nameservicetypes.StoreKey,
		// this line is used by starport scaffolding # 5
	)

	tKeys := sdk.NewTransientStoreKeys(staking.TStoreKey, params.TStoreKey)

	var app = &NewApp{
		BaseApp:        bApp,
		cdc:            cdc,
		invCheckPeriod: invCheckPeriod,
		keys:           keys,
		tKeys:          tKeys,
		subspaces:      make(map[string]params.Subspace),
	}

	app.paramsKeeper = params.NewKeeper(app.cdc, keys[params.StoreKey], tKeys[params.TStoreKey])
	app.subspaces[auth.ModuleName] = app.paramsKeeper.Subspace(auth.DefaultParamspace)
	app.subspaces[bank.ModuleName] = app.paramsKeeper.Subspace(bank.DefaultParamspace)
	app.subspaces[staking.ModuleName] = app.paramsKeeper.Subspace(staking.DefaultParamspace)

	app.accountKeeper = auth.NewAccountKeeper(
		app.cdc,
		keys[auth.StoreKey],
		app.subspaces[auth.ModuleName],
		auth.ProtoBaseAccount,
	)

	app.bankKeeper = bank.NewBaseKeeper(
		app.accountKeeper,
		app.subspaces[bank.ModuleName],
		app.ModuleAccountAddrs(),
	)

	app.supplyKeeper = supply.NewKeeper(
		app.cdc,
		keys[supply.StoreKey],
		app.accountKeeper,
		app.bankKeeper,
		maccPerms,
	)

	stakingKeeper := staking.NewKeeper(
		app.cdc,
		keys[staking.StoreKey],
		app.supplyKeeper,
		app.subspaces[staking.ModuleName],
	)

	app.stakingKeeper = *stakingKeeper.SetHooks(
		staking.NewMultiStakingHooks(),
	)

	app.nameserviceKeeper = nameservicekeeper.NewKeeper(
		app.bankKeeper,
		app.cdc,
		keys[nameservicetypes.StoreKey],
	)

	// this line is used by starport scaffolding # 4

	app.mm = module.NewManager(
		genutil.NewAppModule(app.accountKeeper, app.stakingKeeper, app.BaseApp.DeliverTx),
		auth.NewAppModule(app.accountKeeper),
		bank.NewAppModule(app.bankKeeper, app.accountKeeper),
		supply.NewAppModule(app.supplyKeeper, app.accountKeeper),
		nameservice.NewAppModule(app.nameserviceKeeper, app.bankKeeper),
		staking.NewAppModule(app.stakingKeeper, app.accountKeeper, app.supplyKeeper),
		// this line is used by starport scaffolding # 6
	)

	app.mm.SetOrderEndBlockers(staking.ModuleName)

	app.mm.SetOrderInitGenesis(
		staking.ModuleName,
		auth.ModuleName,
		bank.ModuleName,
		nameservicetypes.ModuleName,
		supply.ModuleName,
		genutil.ModuleName,
		// this line is used by starport scaffolding # 7
	)

	app.mm.RegisterRoutes(app.Router(), app.QueryRouter())

	app.SetInitChainer(app.InitChainer)
	app.SetBeginBlocker(app.BeginBlocker)
	app.SetEndBlocker(app.EndBlocker)

	app.SetAnteHandler(
		auth.NewAnteHandler(
			app.accountKeeper,
			app.supplyKeeper,
			auth.DefaultSigVerificationGasConsumer,
		),
	)

	app.MountKVStores(keys)
	app.MountTransientStores(tKeys)

	if loadLatest {
		err := app.LoadLatestVersion(app.keys[bam.MainStoreKey])
		if err != nil {
			tmos.Exit(err.Error())
		}
	}

	return app
}

type GenesisState map[string]json.RawMessage

func NewDefaultGenesisState() GenesisState {
	return ModuleBasics.DefaultGenesis()
}

func (app *NewApp) InitChainer(ctx sdk.Context, req abci.RequestInitChain) abci.ResponseInitChain {
	var genesisState simapp.GenesisState

	app.cdc.MustUnmarshalJSON(req.AppStateBytes, &genesisState)

	return app.mm.InitGenesis(ctx, genesisState)
}

func (app *NewApp) BeginBlocker(ctx sdk.Context, req abci.RequestBeginBlock) abci.ResponseBeginBlock {
	return app.mm.BeginBlock(ctx, req)
}

func (app *NewApp) EndBlocker(ctx sdk.Context, req abci.RequestEndBlock) abci.ResponseEndBlock {
	return app.mm.EndBlock(ctx, req)
}

func (app *NewApp) LoadHeight(height int64) error {
	return app.LoadVersion(height, app.keys[bam.MainStoreKey])
}

func (app *NewApp) ModuleAccountAddrs() map[string]bool {
	modAccAddrs := make(map[string]bool)
	for acc := range maccPerms {
		modAccAddrs[supply.NewModuleAddress(acc).String()] = true
	}

	return modAccAddrs
}

func (app *NewApp) Codec() *codec.Codec {
	return app.cdc
}

func (app *NewApp) SimulationManager() *module.SimulationManager {
	return app.sm
}

func GetMaccPerms() map[string][]string {
	modAccPerms := make(map[string][]string)
	for k, v := range maccPerms {
		modAccPerms[k] = v
	}
	return modAccPerms
}
```

上面提到的TransientStore是KVStore的内存中实现，用于不持久的状态。

注意**模块的启动方式**：顺序很重要！在这里，**序列是Auth-> Bank-> Feecollection-> Stake-> Distribution-> Slashing，**然后为stake模块设置了钩子。这是因为其中一些模块在使用之前就依赖于其他现有模块

您将注意到文件末尾有几个函数。**initChainer在生成中定义帐户**。json在初始链启动时映射到应用程序状态。**ExportAppStateAndValidators函数帮助引导应用程序的初始状态。****BeginBlocker和EndBlocker是可选的方法，开发者可以在他们的模块中实现**。当从底层共识引擎接收到BeginBlock和EndBlock ABCI消息时，它们将分别在每个块的开始和结束被触发。

# 17.Entry points

In Golang the convention is to place files that compile to a binary in the `./cmd`folder of a project. For your application there are 2 binaries that you want to create:

根据golang的规则, 需要编译可执行程序到cmd目录下, 这个项目需要编译创建两个二进制文件

nameservicecli:这个二进制文件提供了==允许用户与应用程序交互的命令==。

nameserviced:这个二进制文件与bitcoind或其他加密货币守护进程类似，==因为它维护p2p连接、传播事务、处理本地存储并提供RPC接口来与网络交互==。在这种情况下，使用Tendermint进行联网和交易排序。

我们应该已经为我们搭建了以下两个文件:

- `./cmd/nameserviced/main.go`
- `./cmd/nameservicecli/main.go`

## nameserviced

```go
package main

import (
	"encoding/json"
	"io"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"

	abci "github.com/tendermint/tendermint/abci/types"
	"github.com/tendermint/tendermint/libs/cli"
	"github.com/tendermint/tendermint/libs/log"
	tmtypes "github.com/tendermint/tendermint/types"
	dbm "github.com/tendermint/tm-db"

	"github.com/user/nameservice/app"

	"github.com/cosmos/cosmos-sdk/baseapp"
	"github.com/cosmos/cosmos-sdk/client/debug"
	"github.com/cosmos/cosmos-sdk/client/flags"
	"github.com/cosmos/cosmos-sdk/server"
	"github.com/cosmos/cosmos-sdk/store"
	sdk "github.com/cosmos/cosmos-sdk/types"
	"github.com/cosmos/cosmos-sdk/x/auth"
	genutilcli "github.com/cosmos/cosmos-sdk/x/genutil/client/cli"
	"github.com/cosmos/cosmos-sdk/x/staking"
)

const flagInvCheckPeriod = "inv-check-period"

var invCheckPeriod uint

func main() {
	cdc := app.MakeCodec()

	app.SetConfig()

	ctx := server.NewDefaultContext()
	cobra.EnableCommandSorting = false
	rootCmd := &cobra.Command{
		Use:               "nameserviced",
		Short:             "app Daemon (server)",
		PersistentPreRunE: server.PersistentPreRunEFn(ctx),
	}

	rootCmd.AddCommand(genutilcli.InitCmd(ctx, cdc, app.ModuleBasics, app.DefaultNodeHome))
	rootCmd.AddCommand(genutilcli.CollectGenTxsCmd(ctx, cdc, auth.GenesisAccountIterator{}, app.DefaultNodeHome))
	rootCmd.AddCommand(genutilcli.MigrateGenesisCmd(ctx, cdc))
	rootCmd.AddCommand(
		genutilcli.GenTxCmd(
			ctx, cdc, app.ModuleBasics, staking.AppModuleBasic{},
			auth.GenesisAccountIterator{}, app.DefaultNodeHome, app.DefaultCLIHome,
		),
	)
	rootCmd.AddCommand(genutilcli.ValidateGenesisCmd(ctx, cdc, app.ModuleBasics))
	rootCmd.AddCommand(AddGenesisAccountCmd(ctx, cdc, app.DefaultNodeHome, app.DefaultCLIHome))
	rootCmd.AddCommand(flags.NewCompletionCmd(rootCmd, true))
	rootCmd.AddCommand(debug.Cmd(cdc))

	server.AddCommands(ctx, cdc, rootCmd, newApp, exportAppStateAndTMValidators)

	// prepare and add flags
	executor := cli.PrepareBaseCmd(rootCmd, "AU", app.DefaultNodeHome)
	rootCmd.PersistentFlags().UintVar(&invCheckPeriod, flagInvCheckPeriod,
		0, "Assert registered invariants every N blocks")
	err := executor.Execute()
	if err != nil {
		panic(err)
	}
}

func newApp(logger log.Logger, db dbm.DB, traceStore io.Writer) abci.Application {
	var cache sdk.MultiStorePersistentCache

	if viper.GetBool(server.FlagInterBlockCache) {
		cache = store.NewCommitKVStoreCacheManager()
	}
	pruningOpts, err := server.GetPruningOptionsFromFlags()
	if err != nil {
		panic(err)
	}
	return app.NewInitApp(
		logger, db, traceStore, true, invCheckPeriod,
		baseapp.SetPruning(pruningOpts),
		baseapp.SetMinGasPrices(viper.GetString(server.FlagMinGasPrices)),
		baseapp.SetHaltHeight(viper.GetUint64(server.FlagHaltHeight)),
		baseapp.SetHaltTime(viper.GetUint64(server.FlagHaltTime)),
		baseapp.SetInterBlockCache(cache),
	)
}

func exportAppStateAndTMValidators(
	logger log.Logger, db dbm.DB, traceStore io.Writer, height int64, forZeroHeight bool, jailWhiteList []string,
) (json.RawMessage, []tmtypes.GenesisValidator, error) {

	if height != -1 {
		aApp := app.NewInitApp(logger, db, traceStore, false, uint(1))
		err := aApp.LoadHeight(height)
		if err != nil {
			return nil, nil, err
		}
		return aApp.ExportAppStateAndValidators(forZeroHeight, jailWhiteList)
	}

	aApp := app.NewInitApp(logger, db, traceStore, true, uint(1))

	return aApp.ExportAppStateAndValidators(forZeroHeight, jailWhiteList)
}
```

•上面的大多数代码结合了Tendermint的CLI命令，•Cosmos-SDK和您的Nameservice模块

## nameservicecli

Finish up by confirming your `nameservicecli` command looks as follows:

```go
package main

import (
	"fmt"
	"os"
	"path"

	"github.com/cosmos/cosmos-sdk/client"
	"github.com/cosmos/cosmos-sdk/client/flags"
	"github.com/cosmos/cosmos-sdk/client/keys"
	"github.com/cosmos/cosmos-sdk/client/lcd"
	"github.com/cosmos/cosmos-sdk/client/rpc"
	"github.com/cosmos/cosmos-sdk/version"
	"github.com/cosmos/cosmos-sdk/x/auth"
	authcmd "github.com/cosmos/cosmos-sdk/x/auth/client/cli"
	authrest "github.com/cosmos/cosmos-sdk/x/auth/client/rest"
	"github.com/cosmos/cosmos-sdk/x/bank"
	bankcmd "github.com/cosmos/cosmos-sdk/x/bank/client/cli"

	"github.com/spf13/cobra"
	"github.com/spf13/viper"

	"github.com/tendermint/go-amino"
	"github.com/tendermint/tendermint/libs/cli"

	"github.com/user/nameservice/app"
	// this line is used by starport scaffolding
)

func main() {
	// Configure cobra to sort commands
	cobra.EnableCommandSorting = false

	// Instantiate the codec for the command line application
	cdc := app.MakeCodec()

	app.SetConfig()

	// TODO: setup keybase, viper object, etc. to be passed into
	// the below functions and eliminate global vars, like we do
	// with the cdc

	rootCmd := &cobra.Command{
		Use:   "nameservicecli",
		Short: "Command line interface for interacting with nameserviced",
	}

	// Add --chain-id to persistent flags and mark it required
	rootCmd.PersistentFlags().String(flags.FlagChainID, "", "Chain ID of tendermint node")
	rootCmd.PersistentPreRunE = func(_ *cobra.Command, _ []string) error {
		return initConfig(rootCmd)
	}

	// Construct Root Command
	rootCmd.AddCommand(
		rpc.StatusCommand(),
		client.ConfigCmd(app.DefaultCLIHome),
		queryCmd(cdc),
		txCmd(cdc),
		flags.LineBreak,
		lcd.ServeCommand(cdc, registerRoutes),
		flags.LineBreak,
		keys.Commands(),
		flags.LineBreak,
		version.Cmd,
		flags.NewCompletionCmd(rootCmd, true),
	)

	// Add flags and prefix all env exposed with AA
	executor := cli.PrepareMainCmd(rootCmd, "AA", app.DefaultCLIHome)

	err := executor.Execute()
	if err != nil {
		fmt.Printf("Failed executing CLI command: %s, exiting...\n", err)
		os.Exit(1)
	}
}

func queryCmd(cdc *amino.Codec) *cobra.Command {
	queryCmd := &cobra.Command{
		Use:     "query",
		Aliases: []string{"q"},
		Short:   "Querying subcommands",
	}

	queryCmd.AddCommand(
		authcmd.GetAccountCmd(cdc),
		flags.LineBreak,
		rpc.ValidatorCommand(cdc),
		rpc.BlockCommand(),
		authcmd.QueryTxsByEventsCmd(cdc),
		authcmd.QueryTxCmd(cdc),
		flags.LineBreak,
	)

	// add modules' query commands
	app.ModuleBasics.AddQueryCommands(queryCmd, cdc)

	return queryCmd
}

func txCmd(cdc *amino.Codec) *cobra.Command {
	txCmd := &cobra.Command{
		Use:   "tx",
		Short: "Transactions subcommands",
	}

	txCmd.AddCommand(
		bankcmd.SendTxCmd(cdc),
		flags.LineBreak,
		authcmd.GetSignCommand(cdc),
		authcmd.GetMultiSignCommand(cdc),
		flags.LineBreak,
		authcmd.GetBroadcastCommand(cdc),
		authcmd.GetEncodeCommand(cdc),
		authcmd.GetDecodeCommand(cdc),
		flags.LineBreak,
	)

	// add modules' tx commands
	app.ModuleBasics.AddTxCommands(txCmd, cdc)

	// remove auth and bank commands as they're mounted under the root tx command
	var cmdsToRemove []*cobra.Command

	for _, cmd := range txCmd.Commands() {
		if cmd.Use == auth.ModuleName || cmd.Use == bank.ModuleName {
			cmdsToRemove = append(cmdsToRemove, cmd)
		}
	}

	txCmd.RemoveCommand(cmdsToRemove...)

	return txCmd
}

// registerRoutes registers the routes from the different modules for the LCD.
// NOTE: details on the routes added for each module are in the module documentation
// NOTE: If making updates here you also need to update the test helper in client/lcd/test_helper.go
func registerRoutes(rs *lcd.RestServer) {
	client.RegisterRoutes(rs.CliCtx, rs.Mux)
	authrest.RegisterTxRoutes(rs.CliCtx, rs.Mux)
	app.ModuleBasics.RegisterRESTRoutes(rs.CliCtx, rs.Mux)
	// this line is used by starport scaffolding # 2
}

func initConfig(cmd *cobra.Command) error {
	home, err := cmd.PersistentFlags().GetString(cli.HomeFlag)
	if err != nil {
		return err
	}

	cfgFile := path.Join(home, "config", "config.toml")
	if _, err := os.Stat(cfgFile); err == nil {
		viper.SetConfigFile(cfgFile)

		if err := viper.ReadInConfig(); err != nil {
			return err
		}
	}
	if err := viper.BindPFlag(flags.FlagChainID, cmd.PersistentFlags().Lookup(flags.FlagChainID)); err != nil {
		return err
	}
	if err := viper.BindPFlag(cli.EncodingFlag, cmd.PersistentFlags().Lookup(cli.EncodingFlag)); err != nil {
		return err
	}
	return viper.BindPFlag(cli.OutputFlag, cmd.PersistentFlags().Lookup(cli.OutputFlag))
}
```

注意:

代码结合了来自Tendermint、Cosmos-SDK和Nameservice模块的CLI命令。
cobra CLI文档(打开新窗口)将有助于理解上述代码。
您可以在这里看到早些时候定义的ModuleClient。
注意这些路由是如何包含在registerRoutes函数中的。

# 18.go.mod and Makefile

## Starport serve

Having bootstrapped your application with starport you can use the Starport utility `starport serve` to get your blockchain running. Make sure to include your `nametoken` into your `config.yml` file

```go
version: 1
accounts:
  - name: user1
    coins: ["1000token", "100000000stake", "10000nametoken"]
  - name: user2
    coins: ["500token", "1000nametoken"]
validator:
  name: user1
  staked: "100000000stake"
```

## go.mod and Makefile

### Makefile

Help users build your application by writing a `./Makefile` in the root directory that includes common commands. The scaffolding tool has created a generic makefile that you will be able to use:

通过在包含常用命令的根目录中编写./Makefile来帮助用户构建应用程序。 脚手架工具已经创建了一个通用的makefile，您将可以使用：

**NOTE**: The below Makefile contains some of same commands as the Cosmos SDK and Tendermint Makefiles.

下面的Makefile包含与Cosmos SDK和Tendermint Makefile相同的命令。

```go
PACKAGES=$(shell go list ./... | grep -v '/simulation')

VERSION := $(shell echo $(shell git describe --tags) | sed 's/^v//')
COMMIT := $(shell git log -1 --format='%H')

ldflags = -X github.com/cosmos/cosmos-sdk/version.Name=NameService \
	-X github.com/cosmos/cosmos-sdk/version.ServerName=nameserviced \
	-X github.com/cosmos/cosmos-sdk/version.ClientName=nameservicecli \
	-X github.com/cosmos/cosmos-sdk/version.Version=$(VERSION) \
	-X github.com/cosmos/cosmos-sdk/version.Commit=$(COMMIT) 

BUILD_FLAGS := -ldflags '$(ldflags)'

all: install

install: go.sum
		@echo "--> Installing nameserviced & nameservicecli"
		@go install -mod=readonly $(BUILD_FLAGS) ./cmd/nameserviced
		@go install -mod=readonly $(BUILD_FLAGS) ./cmd/nameservicecli

go.sum: go.mod
		@echo "--> Ensure dependencies have not been modified"
		GO111MODULE=on go mod verify

test:
	@go test -mod=readonly $(PACKAGES)
```

#### How about including Ledger Nano S support?

This requires a few small changes:

- Create a file `Makefile.ledger` with the following content:

  `404: Not Found`

- Add `include Makefile.ledger` at the beginning of the Makefile:

  ```go
  BUILD_FLAGS := -ldflags '$(ldflags)'
  
  include Makefile.ledger
  all: install
  ```


### go.mod

Cosmos SDK apps currently depend on specific versions of some libraries. The below manifest contains all the necessary versions. To get started replace the contents of the `./go.mod` file with the `constraints` and `overrides` below:

Golang有一些依赖管理工具。 在本教程中，您将使用Go模块。 Go Modules在存储库的根目录中使用go.mod文件来定义应用程序需要的依赖项。 Cosmos SDK应用程序当前取决于某些库的特定版本。 以下清单包含所有必需的版本。 首先，将./go.mod文件的内容替换为以下约束和替代：

- You will have to run `go get ./...` to get all the modules the application is using. This command will get the dependency version stated in the `go.mod` file.
- If you would like to use a specific version of a dependency then you have to run `go get github.com/<github_org>/<repo_name>@<version>`

#### Building the app

```go
# Install the app into your $GOBIN
make install

# Now you should be able to run the following commands:
nameserviced help
nameservicecli help
```

**mac M1 Big Sur 11.2.1 make install 会报错:** 

![kGFHvi](http://xwjpics.gumptlu.work/qinniu_uPic/kGFHvi.png)

unbutu服务器环境没问题:

![uLBvnx](http://xwjpics.gumptlu.work/qinniu_uPic/uLBvnx.png)

![W4IGk9](http://xwjpics.gumptlu.work/qinniu_uPic/W4IGk9.png)

![1ZdPsS](http://xwjpics.gumptlu.work/qinniu_uPic/1ZdPsS.png)

# 19 Build and run the app

## Building the `nameservice` application

This repo contains a complete `nameservice` application, scaffolded with starport. You should be able to run the application using `starport serve` in the home directory:

```go
$ starport serve

📦 Installing dependencies...
🛠️  Building the app...
🙂 Created an account. Password (mnemonic): insane flash movie sketch saddle antique mean season damp thunder tag reunion quantum sock cube early glimpse cabbage smile photo hill relax couch sweet
🙂 Created an account. Password (mnemonic): whip bone crane flag lesson mule valley soup faith include october monkey volume iron mushroom cry acid case village clog abstract antenna wife eyebrow
🌍 Running a Cosmos 'nameservice' app with Tendermint at http://localhost:26657.
🌍 Running a server at http://localhost:1317 (LCD)

🚀 Get started: http://localhost:12345/
```

![fRB7hY](http://xwjpics.gumptlu.work/qinniu_uPic/fRB7hY.png)

Now, you can install and run the application.

If you have not completed the tutorial then you can follow the below cloning instructions:

tutorial的所有项目完整代码:

```go
# Clone the source of the tutorial repository
git clone https://github.com/cosmos/sdk-tutorials.git
cd sdk-tutorials
cd nameservice/nameservice
starport serve
```

注意: 如果你有用于ledger的Cosmos应用，你想使用它，当你用nameservicecli键创建键时，只需在末尾添加——ledger即可。这就是你所需要的。当您签名时，user1将被识别为分类帐密钥，并将需要一个设备。

After you have generated a genesis transaction, you will have to input the genTx into the genesis file, so that your nameservice chain is aware of the validators. To do so, run:

在生成了一个创世交易之后，您必须将genTx输入到生成文件中，这样您的**namesservice链才能知道验证器**。为此，运行:

```
nameserviced collect-gentxs
```

![7lmvlW](http://xwjpics.gumptlu.work/qinniu_uPic/7lmvlW.png)

格式化以后内容为:

```json
{
	"app_message": {
		"auth": {
			"accounts": [{
				"type": "cosmos-sdk/Account",
				"value": {
					"account_number": "0",
					"address": "cosmos108egxhu7u7c63erhkxqr23zuydh07zhuner794",
					"coins": [{
						"amount": "10000",
						"denom": "nametoken"
					}, {
						"amount": "100000000",
						"denom": "stake"
					}, {
						"amount": "1000",
						"denom": "token"
					}],
					"public_key": null,
					"sequence": "0"
				}
			}, {
				"type": "cosmos-sdk/Account",
				"value": {
					"account_number": "0",
					"address": "cosmos1ynhnxn39c5w00p4qh3647c35k4gjgq4cprgvza",
					"coins": [{
						"amount": "1000",
						"denom": "nametoken"
					}, {
						"amount": "500",
						"denom": "token"
					}],
					"public_key": null,
					"sequence": "0"
				}
			}],
			"params": {
				"max_memo_characters": "256",
				"sig_verify_cost_ed25519": "590",
				"sig_verify_cost_secp256k1": "1000",
				"tx_sig_limit": "7",
				"tx_size_cost_per_byte": "10"
			}
		},
		"bank": {
			"send_enabled": true
		},
		"genutil": {
			"gentxs": [{
				"type": "cosmos-sdk/StdTx",
				"value": {
					"fee": {
						"amount": [],
						"gas": "200000"
					},
					"memo": "70dfc95d3f9869145d79387c201eae58399074d8@172.17.59.2:26656",
					"msg": [{
						"type": "cosmos-sdk/MsgCreateValidator",
						"value": {
							"commission": {
								"max_change_rate": "0.010000000000000000",
								"max_rate": "0.200000000000000000",
								"rate": "0.100000000000000000"
							},
							"delegator_address": "cosmos108egxhu7u7c63erhkxqr23zuydh07zhuner794",
							"description": {
								"details": "",
								"identity": "",
								"moniker": "mynode",
								"security_contact": "",
								"website": ""
							},
							"min_self_delegation": "1",
							"pubkey": "cosmosvalconspub1zcjduepq2ndyr7spzqr0u3pufhdwmqlxkcg9cww3ywesywvn376h4aheqdfqqgjsue",
							"validator_address": "cosmosvaloper108egxhu7u7c63erhkxqr23zuydh07zhukdhtfx",
							"value": {
								"amount": "100000000",
								"denom": "stake"
							}
						}
					}],
					"signatures": [{
						"pub_key": {
							"type": "tendermint/PubKeySecp256k1",
							"value": "A0ETGI578auv3BEwLbbDJwtLF3A+2MnWLtjexi1f/r5G"
						},
						"signature": "WtBOWmg6ZI5cywhQQFbJ8bBNUODESALzwnXXXRmeb8U672UkX1y6xNSl4L8tj+mxLgDcpndprqhgTVK5gE+wKA=="
					}]
				}
			}]
		},
		"nameservice": {
			"whois_records": []
		},
		"params": null,
		"staking": {
			"delegations": null,
			"exported": false,
			"last_total_power": "0",
			"last_validator_powers": null,
			"params": {
				"bond_denom": "stake",
				"historical_entries": 0,
				"max_entries": 7,
				"max_validators": 100,
				"unbonding_time": "1814400000000000"
			},
			"redelegations": null,
			"unbonding_delegations": null,
			"validators": null
		},
		"supply": {
			"supply": []
		}
	},
	"chain_id": "nameservice",
	"gentxs_dir": "/root/.nameserviced/config/gentx",
	"moniker": "mynode",
	"node_id": "70dfc95d3f9869145d79387c201eae58399074d8"
}
```

and to make sure your genesis file is correct, run:

并且确定你的创世文件是正确的

```
nameserviced validate-genesis
```

![KRO6CI](http://xwjpics.gumptlu.work/qinniu_uPic/KRO6CI.png)

You can now start `nameserviced` by calling `nameserviced start`. You will see logs begin streaming that represent blocks being produced, this will take a couple of seconds.

你现在可以通过调用' nameserviced start '来启动' nameserviced '。您将看到表示正在生成的块的日志流开始，这需要几秒钟的时间。

**==注意官方有命令(set-name)写反了~!!!==**

```shell
# First check the accounts to ensure they have funds
# 检查资金余额
nameservicecli query account $(nameservicecli keys show user1 -a)
nameservicecli query account $(nameservicecli keys show user2 -a)

# Buy your first name using your coins from the genesis file
# 使用coins购买域名
# 发起购买域名交易
nameservicecli tx nameservice buy-name user1.id 5nametoken --from user1

# Set the value for the name you just bought
# 设置域名的解析值
# 设置域名解析值
# 注意官方写反了!!!!!!
nameservicecli tx nameservice set-name 8.8.8.8 user1.id --from user1

# Try out a resolve query against the name you registered
# 针对您注册的名称尝试一个resolve查询
nameservicecli query nameservice resolve user1.id
# > 8.8.8.8

# Try out a whois query against the name you just registered
nameservicecli query nameservice get-whois user1.id
# whois查询
# > {"value":"8.8.8.8","owner":"cosmos1l7k5tdt2qam0zecxrx78yuw447ga54dsmtpk2s","price":[{"denom":"nametoken","amount":"5"}]}

# user2 buys name from user1
# user2买user1的域名
nameservicecli tx nameservice buy-name user1.id 10nametoken --from user2

# user2 decides to delete the name she just bought from user1
# use2删除域名
nameservicecli tx nameservice delete-name user1.id --from user2

# Try out a whois query against the name you just deleted
nameservicecli query nameservice get-whois user1.id
# > {"value":"","owner":"","price":[{"denom":"nametoken","amount":"1"}]}

# 列出所有的nameservice域名
nameservicecli query nameservice list-whois
```

![E8GyGu](http://xwjpics.gumptlu.work/qinniu_uPic/E8GyGu.png)

![E4LXXw](http://xwjpics.gumptlu.work/qinniu_uPic/E4LXXw.png)

![HvLCWY](http://xwjpics.gumptlu.work/qinniu_uPic/HvLCWY.png)

![6NfJan](http://xwjpics.gumptlu.work/qinniu_uPic/6NfJan.png)

![s8W6Lp](http://xwjpics.gumptlu.work/qinniu_uPic/s8W6Lp.png)

![OgInYy](http://xwjpics.gumptlu.work/qinniu_uPic/OgInYy.png)

![npCWzq](http://xwjpics.gumptlu.work/qinniu_uPic/npCWzq.png)

**检查一下user2是否成功购买:**

![mxpNfQ](http://xwjpics.gumptlu.work/qinniu_uPic/mxpNfQ.png)

余额检查:

* 购买前: user1: 9990, user2: 1000

*  购买后: user1: 9990 - 5 + 50 = 10035 , user2: 1000 - 50 = 950

![MWdcH3](http://xwjpics.gumptlu.work/qinniu_uPic/MWdcH3.png)

![TOzYHX](http://xwjpics.gumptlu.work/qinniu_uPic/TOzYHX.png)

![V7Jmds](http://xwjpics.gumptlu.work/qinniu_uPic/V7Jmds.png)

## Run second node on another machine (Optional)

Open terminal to run commands against that just created to install nameserviced and nameservicecli

打开终端，对刚刚创建的用于安装nameserviced和nameservicecli的终端运行命令

复制项目, 进入目录:

```shell
make install
```

### init use another moniker(绰号) and same namechain

```go
nameserviced init <moniker-2> --chain-id namechain
```

### overwrite ~/.nameserviced/config/genesis.json with first node's genesis.json

### change persistent_peers

```shell
vim /.nameserviced/config/config.toml
persistent_peers = "id@first_node_ip:26656"
```

To find the node id of the first machine, run the following command on that machine:

`nameserviced tendermint show-node-id`

### start this second node

`nameserviced start`

# 20 Run REST routes

==**对于Rest接口，发送需要签名的交易需要本地签名，所以只有查询接口能够立即看到效果，其他接口都要等待签名**==

==**我的做法是使用前端cosmosjs：https://github.com/cosmostation/cosmosjs==**

Now that you tested your CLI queries and transactions, time to test same things in the REST server. Leave the `nameserviced` that you had running earlier and start by gathering your addresses:

现在您已经测试了CLI查询和事务，现在可以在REST服务器中测试相同的内容了。离开你之前运行的' nameserviced '，开始收集你的地址:

```shell
# 查看用户的完整信息
$ nameservicecli keys show 用户名
# 查看所有的账户信息
nameservicecli keys list
```

```shell
$ nameservicecli keys show user1 --address
$ nameservicecli keys show user2 --address
```

Now its time to start the `rest-server` in another terminal window:

开启rest服务在另一个客户端:

**==注意,如果你的app已经开启了(即starport serve),那么默认的1317端口的rest服务已经打开,所以不需要使用以下命令.==**

```shell
$ nameservicecli rest-server --chain-id namechain --trust-node
```

![i6Skw9](http://xwjpics.gumptlu.work/qinniu_uPic/i6Skw9.png)

Then you can construct and run the following queries:

接下来就可以开始查询:

> NOTE: Be sure to substitute your password and buyer/owner addresses for the ones listed below!
>
> 请务必将您的密码和buyer/owner地址替换为下面列出的那些!

==**注意:此处官方文档的测试参数也有错误!!!, 参数amount要换成price**==

```shell
# Get the sequence and account numbers for user1 to construct the below requests
# 获取user1的序列和账号来构造下面的请求
curl -s http://localhost:1317/auth/accounts/$(nameservicecli keys show user1 -a)
# > {"type":"auth/Account","value":{"address":"cosmos127qa40nmq56hu27ae263zvfk3ey0tkapwk0gq6","coins":[{"denom":"jackCoin","amount":"1000"},{"denom":"nametoken","amount":"1010"}],"public_key":{"type":"tendermint/PubKeySecp256k1","value":"A9YxyEbSWzLr+IdK/PuMUYmYToKYQ3P/pM8SI1Bxx3wu"},"account_number":"0","sequence":"1"}}

# Get the sequence and account numbers for user2 to construct the below requests
curl -s http://localhost:1317/auth/accounts/$(nameservicecli keys show user2 -a)
# > {"type":"auth/Account","value":{"address":"cosmos1h7ztnf2zkf4558hdxv5kpemdrg3tf94hnpvgsl","coins":[{"denom":"aliceCoin","amount":"1000"},{"denom":"nametoken","amount":"980"}],"public_key":{"type":"tendermint/PubKeySecp256k1","value":"Avc7qwecLHz5qb1EKDuSTLJfVOjBQezk0KSPDNybLONJ"},"account_number":"1","sequence":"2"}}

# Buy another name for user1, first create the raw transaction
# NOTE: Be sure to specialize this request for your specific environment, also the "buyer" and "from" should be the same address
# 为user1购买另一个名称，首先创建原始交易
# 注意:请确保针对您的特定环境专门处理此请求，而且“buyer”和“from”应该是相同的地址
# 注意此处不是官方的amount,而是price, 在官网有错
curl -X POST -s http://localhost:1317/nameservice/whois --data-binary '{"base_req":{"from":"'$(nameservicecli keys show user1 -a)'","chain_id":"namechain"},"name":"user1.id","price":"5nametoken","buyer":"'$(nameservicecli keys show user1 -a)'"}' > unsignedTx.json

# Then sign this transaction
# NOTE: In a real environment the raw transaction should be signed on the client side. Also the sequence needs to be adjusted, depending on what the query of user2's account has shown.
# 然后签署该事务
# 注意:在真实环境中，原始交易应该在客户端签名。此外，还需要根据user2帐户的查询显示的内容调整序列。
# 注意. 这里的sequence、account-number都是第一步查询序号和账户号的得到的,下面的命令自行替换
nameservicecli tx sign unsignedTx.json --from user1 --offline --chain-id namechain --sequence 1 --account-number 0 > signedTx.json

# 不使用离线模式进行签名
nameservicecli tx sign unsignedTx.json --from user1 --chain-id namechain > signedTx.json



# And finally broadcast the signed transaction
nameservicecli tx broadcast signedTx.json
# > { "height": "266", "txhash": "C041AF0CE32FBAE5A4DD6545E4B1F2CB786879F75E2D62C79D690DAE163470BC", "logs": [  {   "msg_index": "0",   "success": true,   "log": ""  } ],"gas_wanted":"200000", "gas_used": "41510", "tags": [  {   "key": "action",   "value": "buy_name"  } ]}

# Set the data for that name that user1 just bought
# NOTE: Be sure to specialize this request for your specific environment, also the "owner" and "from" should be the same address
# 为user1刚刚购买的名称设置数据
# 注意:请确保为您的特定环境专门化此请求，而且“owner”和“from”应该是相同的地址
curl -X PUT -s http://localhost:1317/nameservice/whois --data-binary '{"base_req":{"from":"'$(nameservicecli keys show user1 -a)'","chain_id":"namechain","sequence": "1","account_number": "2"},"name":"user1.id","value":"8.8.4.4","owner":"'$(nameservicecli keys show user1 -a)'"}' > unsignedTx.json
# > {"check_tx":{"gasWanted":"200000","gasUsed":"1242"},"deliver_tx":{"log":"Msg 0: ","gasWanted":"200000","gasUsed":"1352","tags":[{"key":"YWN0aW9u","value":"c2V0X25hbWU="}]},"hash":"B4DF0105D57380D60524664A2E818428321A0DCA1B6B2F091FB3BEC54D68FAD7","height":"26"}

# Again we need to sign and broadcast
nameservicecli tx sign unsignedTx.json --from user1 --offline --chain-id namechain --sequence 2 --account-number 0 > signedTx.json
nameservicecli tx broadcast signedTx.json

# Query the value for the name user1 just set
$ curl -s http://localhost:1317/nameservice/whois/user1.id/resolve
# 8.8.4.4

# Query whois for the name user1 just bought
$ curl -s http://localhost:1317/nameservice/whois/user1.id
# > {"value":"8.8.8.8","owner":"cosmos127qa40nmq56hu27ae263zvfk3ey0tkapwk0gq6","price":[{"denom":"STAKE","amount":"10"}]}

# user2 buys name from user1
$ curl -X POST -s http://localhost:1317/nameservice/whois --data-binary '{"base_req":{"from":"'$(nameservicecli keys show user2 -a)'","chain_id":"namechain"},"name":"user1.id","price":"10nametoken","buyer":"'$(nameservicecli keys show user2 -a)'"}' > unsignedTx.json

# Again we need to sign and broadcast
# NOTE: The account number has changed to 1 and the sequence is now 2, according to the query of user2's account
nameservicecli tx sign unsignedTx.json --from user2 --offline --chain-id namechain --sequence 2 --account-number 1 > signedTx.json
nameservicecli tx broadcast signedTx.json
# > { "height": "1515", "txhash": "C9DCC423E10E7E5E40A549057A4AA060DA6D6A885A394F6ED5C0E40AEE984A77", "logs": [  {   "msg_index": "0",   "success": true,   "log": ""  } ],"gas_wanted": "200000", "gas_used": "42375", "tags": [  {   "key": "action",   "value": "buy_name"  } ]}

# Now, user2 no longer needs the name she bought from jack and hence deletes it
# NOTE: Only the owner can delete the name. Since she is one, she can delete the name she bought from jack
$ curl -XDELETE -s http://localhost:1317/nameservice/names --data-binary '{"base_req":{"from":"'$(nameservicecli keys show user2 -a)'","chain_id":"namechain"},"name":"user1.id","owner":"'$(nameservicecli keys show user2 -a)'"}' > unsignedTx.json

# And a final time sign and broadcast
# NOTE: The account number is still 1, but the sequence is changed to 3, according to the query of user2's account
nameservicecli tx sign unsignedTx.json --from user2 --offline --chain-id namechain --sequence 3 --account-number 1 > signedTx.json
nameservicecli tx broadcast signedTx.json

# Query whois for the name user2 just deleted
$ curl -s http://localhost:1317/nameservice/names/user1.id/whois
# > {"value":"","owner":"","price":[{"denom":"STAKE","amount":"1"}]}
```

## 1.开启rest服务

**如果已经开启整个项目(即starport serve)那么此步骤可以不需要做**

```shell
nameservicecli rest-server --chain-id namechain --trust-node
```

![CjF66E](http://xwjpics.gumptlu.work/qinniu_uPic/CjF66E.png)

## 2.查询账户信息

```shell
# Get the sequence and account numbers for user1 to construct the below requests
# 获取user1的序列和账号来构造下面的请求
curl -s http://localhost:1317/auth/accounts/$(nameservicecli keys show user1 -a)

# Get the sequence and account numbers for user2 to construct the below requests
curl -s http://localhost:1317/auth/accounts/$(nameservicecli keys show user2 -a)
```

这些信息用于完成下面的请求参数

![1NI6dR](http://xwjpics.gumptlu.work/qinniu_uPic/1NI6dR.png)

psotman:

![90EpOV](http://xwjpics.gumptlu.work/qinniu_uPic/90EpOV.png)

## 3.购买域名

user1购买域名

```shell
curl -X POST -s http://localhost:1317/nameservice/whois --data-binary '{"base_req":{"from":"'$(nameservicecli keys show user1 -a)'","chain_id":"namechain"},"name":"user1.id","price":"5nametoken","buyer":"'$(nameservicecli keys show user1 -a)'"}' > unsignedTx.json
```

![KkukAm](http://xwjpics.gumptlu.work/qinniu_uPic/KkukAm.png)

![OITmtI](http://xwjpics.gumptlu.work/qinniu_uPic/OITmtI.png)

```json
{
    "type": "cosmos-sdk/StdTx",
    "value": {
        "msg": [
            {
                "type": "nameservice/BuyName",
                "value": {
                    "name": "www.baidu.com",
                    "bid": [
                        {
                            "denom": "nametoken",
                            "amount": "5"
                        }
                    ],
                    "buyer": "cosmos170ca57fje9tcjapjeaeyk36xzzdfatlru06ju0"
                }
            }
        ],
        "fee": {
            "amount": [],
            "gas": "200000"
        },
        "signatures": null,
        "memo": ""
    }
}
```

## 4.本地进行签名

```shell
# 然后签署该事务
# 注意:在真实环境中，原始交易应该在客户端签名。此外，还需要根据user2帐户的查询显示的内容调整序列。
nameservicecli tx sign unsignedTx.json --from user1 --offline --chain-id namechain --sequence 1 --account-number 0 > signedTx.json
```

signedTx.json:

```json
{
  2   "type": "cosmos-sdk/StdTx",
  3   "value": {
  4     "msg": [
  5       {
  6         "type": "nameservice/BuyName",
  7         "value": {
  8           "name": "user1.id",
  9           "bid": [
 10             {
 11               "denom": "nametoken",
 12               "amount": "5"
 13             }
 14           ],
 15           "buyer": "cosmos128su4wssj7dcmcsa2mr00pu4c0520fq5lcsd5m"
 16         }
 17       }
 18     ],
 19     "fee": {
 20       "amount": [],
 21       "gas": "200000"
22     },
 23     "signatures": [
 24       {
 25         "pub_key": {
 26           "type": "tendermint/PubKeySecp256k1",
 27           "value": "A7vz4G+HsxMmbd/19PAzIOfZro6xgRjgBZyb04H1tZRJ"
 28         },
 29         "signature": "k6Cq6DF1bbZRxUqFYeDteUt889HN1DwDULxQUS9QuTlXoFpl04U+Ge87jqjh6fv8mDrJLASu56rRt6vv7NtoPA=="
 30       }
 31     ],
 32     "memo": ""
 33   }
 34 }
```

## 5.广播交易

```shell
# And finally broadcast the signed transaction
nameservicecli tx broadcast signedTx.json
```

