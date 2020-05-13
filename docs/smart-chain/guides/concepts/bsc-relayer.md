# BSC Relayer
Relayers are responsible to submit Cross-Chain Communication Packages between the two blockchains. Due to the heterogeneous parallel chain structure, two different types of Relayers are created.

Relayers for BC-to-BSC communication referred to as **BSC Relayers** are a standalone process that can be run by anyone, and anywhere, except that Relayers must register themselves onto BSC and deposit a certain amount of BNB. Only relaying requests from the registered Relayers will be accepted by BSC.

## Monitor and Parse Cross Chain Event
As a BSC relayer, it must have proper configurations on the following three items:

| Name | Type | Description |
| ---- | ---- | ----------- |
|srcCrossChainID  |   uint16   |      CrossChainID of BC, the value is 1 for testnet       |
|destCrossChainID|uint16|CrossChainID of BSC, the value is 2 for testnet|
|destChainName|string|name of targe chain, for Binance Smart Chain, the value should be “bsc”|

A BSC relayer is required to parse all block results and pick out all events with event type “IBCPackage” from endBlock event table. This is an cross chain package event example:

```json
{ "type": "IBCPackage",
"attributes":
 [{    "key": "IBCPackageInfo",
       "value": "bsc::2::8::19"   }]
}

```

BSC relayer should iterate all the attributes and parse the attribute value:

1. Split the value with “::” and get a 4-length string array
2. Follow the following table to parse the 4 elements:

|IndexDesription|Type|Example Value|
| ---- | ---- | ----------- |
|0|destination chain name|string|bsc|
|1|CrossChainID of destination chain |int16|2|
|2|channel id|int8|8|
|3|sequenceu|int64|19|

3. Filter out all attributes with mismatched destCrossChainID and destChainName.

## Build Tendermint Header and Query Cross Chain Package

### Build Tendermint Header
```golang
import tmtypes "github.com/tendermint/tendermint/types"
type Header struct {
  tmtypes.SignedHeader
  ValidatorSet     *tmtypes.ValidatorSet `json:"validator_set"`
  NextValidatorSet *tmtypes.ValidatorSet `json:"next_validator_set"`
}
```

If a cross chain package event is found at height **H**, wait for block **H+1** and call the following rpc methods to build the above **Header** object:

|Name|Method|
| ---- | ---- |
|tmtypes.SignedHeader|{rpc}/commit?height=**H+1**|
|ValidatorSet|{rpc}/validators?height=**H+1**|
|NextValidatorSet|{rpc}/validators?height=**H+2**|

Encode the Header object to a byte array:

1. Add dependency on [go-amino v0.14.1](https://github.com/tendermint/go-amino/tree/v0.14.1)
2. Add dependency on [tendermint v0.32.3](https://github.com/tendermint/tendermint/tree/v0.32.3):
3. Example golang code to encode **Header**:
```golang

import (
  amino "github.com/tendermint/go-amino"
  tmtypes "github.com/tendermint/tendermint/types"
)

var cdc = amino.NewCodec()

func init() {
  tmtypes.RegisterBlockAmino(cdc)
}

func (h *Header) EncodeHeader() ([]byte, error) {
  bz, err := cdc.MarshalBinaryLengthPrefixed(h)
  if err != nil {
     return nil, err
  }
  return bz, nil
}

```

### Query Package With Proof
1. Specify the height as H
2. Store name: ibc
2. Query path: /store/ibc/key
3. Follow the table to build a 14-length byte array as query key:

|Name|Length|Value|
| ---- | ---- | ------ |
|prefix|1 bytes|0x00|
|souce chain CrossChainID|2 bytes|srcCrossChainID in bsc relayer configuration|
|destination chain CrossChainID|2 bytes|destCrossChainID in bsc relayer configuration|
|channelID|1 bytes|channelID in attribute value|
|sequence|8 bytes|sequence in attribute value|

4. The query result should contains non-empty package bytes and merkle proof byte.

## Call Build-In System Contract

### Sync BC Header
* function **syncTendermintHeader**(bytes calldata header, uint64 height)
Call syncTendermintHeader of TendermintLightClient contract to sync BC header. The contract address is 0x0000000000000000000000000000000000001003. The “header” should be the encoding result of Header struct and the height should be **H + 1**

### Deliver Cross Chain Package
Follow this table to get build-in system contract address and method name:

|ChannelID|Desription|contract address|Methods|
| ---- | ---- | ----------- |----|
|1|bind channel|0x0000000000000000000000000000000000001004|handleBindPackage|
|2|transfer channel|0x0000000000000000000000000000000000001004|handleTransferInPackage|
|3|refund channel|0x0000000000000000000000000000000000001004|handleRefundPackage|
|8|staking channel|0x0000000000000000000000000000000000001000|handlePackage|

All above methods shares the same parameter table:


|Parameter Name|Type|Value|
| ---- | ---- | ----------- |
|msgBytes|[]byte|package bytes|
|proof|[]byte|merkle proof bytes|
|height|uint64|**H + 1**|
|packageSequence|uint64|sequence from attribution value|

## Incentives Mechanism

[BSC Relayer Incentive Mechanism](incentives.md)