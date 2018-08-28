---
id: loom-js-loom-provider-web3
title: Loom.js + Web3.js
sidebar_label: Loom.js + Web3.js
---
# 概要

`loom-js`には`LoomProvider`が備わっている。これはLoom DAppチェーン内で実行されるスマートコントラクトのデプロイ、及びスマートコントラクトへのトランザクションの送信、スマートコントラクトイベントのリッスンをイーサリアム開発者にとって可能にすると同時に、`Web3.js`をプロバイダとして接続できるようにする。さらなる詳細は[EVMページ](evm)をチェックしよう。

NPMで`loom-js`をインストール

```shell
yarn add loom-js
# or if you prefer...
npm install loom-js
```

# コントラクトのインスタンス化

## SimpleContract

あるSolidityコントラクトが、すでにコンパ入りされLoom DAppチェーン上にデプロイされているとしよう。

    pragma solidity ^0.4.22;
    
    contract SimpleStore {
      uint value;
    
      event NewValueSet(uint);
    
      function set(uint _value) public {
        value = _value;
        emit NewValueSet(value);
      }
    
      function get() public view returns (uint) {
        return value;
      }
    }
    

堅さのコンパイラでコンパイルされたバイナリを使って、次のステップではLoom Dappチェーン用の`genesis.json` を作成しよう。 (コンパイルされたバイナリに`location`を設定するのを忘れないように)

```Javascript
{
    "contracts": [
        {
            "vm": "EVM",
            "format": "hex",
            "name": "SimpleStore",
            "location": "/path_to_simple_store/SimpleStore.bin"
        }
    ]
}

```

コントラクトをコンパイルしたら、次のABIインターフェースが生成される:

```js
const ABI = [{
  "constant": false,
  "inputs": [{
    "name": "_value",
    "type": "uint256"
  }],
  "name": "set",
  "outputs": [],
  "payable": false,
  "stateMutability": "nonpayable",
  "type": "function"
}, {
  "constant": true,
  "inputs": [],
  "name": "get",
  "outputs": [{
    "name": "",
    "type": "uint256"
  }],
  "payable": false,
  "stateMutability": "view",
  "type": "function"
}, {
  "anonymous": false,
  "inputs": [{
    "indexed": false,
    "name": "",
    "type": "uint256"
  }],
  "name": "NewValueSet",
  "type": "event"
}]
```

インスタンス化および`LoomProvider`での`Web3`の使用は、イーサリアムのノードを使用するのに似ているが、まず`loom-js`クライアントを正しく初期化する必要がある。

```js
import {
  Client, Address, LocalAddress, CryptoUtils, LoomProvider, EvmContract
} from '../loom.umd'

import Web3 from 'web3'

// この関数はクライアントの初期化とリターンを行う
function getClient(privateKey, publicKey) {
  const client = new Client(
    'default',
    'ws://127.0.0.1:46658/websocket',
    'ws://127.0.0.1:46658/queryws',
  )

  return client
}

// キーのセットアップ
const privateKey = CryptoUtils.generatePrivateKey()
const publicKey = CryptoUtils.publicKeyFromPrivateKey(privateKey)

// クライアントを準備
const client = getClient(privateKey, publicKey)
```

クライアントの準備ができたので、今度は`Web3`をインスタンス化しよう。`Web3`を適切にインスタンス化するために、`client`と共に`LoomProvider`を渡そう。

```js
const web3 = new Web3(new LoomProvider(client, privateKey))
```

コントラクトをインスタンス化する準備ができた。

```js
// 公開鍵を基にアドレスを取得
const fromAddress = LocalAddress.fromPublicKey(publicKey).toString()

// コントラクトアドレスの取得 (アドレスは必要なく、genesis.json中での特定の名前だけで良い)
const loomContractAddress = await client.getContractAddressAsync('SimpleStore')

// Web3と互換性を持つようloom addressをhexaへ変換
const contractAddress = CryptoUtils.bytesToHexAddr(loomContractAddress.local.bytes)

// コントラクトのインスタンス化
const contract = new web3.eth.Contract(ABI, contractAddress, {from: fromAddress})
```

コントラクトはインスタンス化され、準備が整った。

# トランザクションと呼び出し

`Web3 Contract`のインスタンス化が終わったら、トランザクション(`send`) や呼び出し(`call`)のためのコントラクトメソッドを次のように使用できる:

```js
(async function () {
  // バリューを47に設定
  await contract.methods.set(47).send()

  // バリューの取得
  const result = await contract.methods.get().call()
  // 結果は47となる
})()
```

# イベント

コントラクトにイベントリスナーを追加することが可能だ。ただしフィルターはまだサポートしていない。

```js
(async function () {
  // Listen for new value set
  contract.events.NewValueSet({}, (err, newValueSet) {
    if (err) {
      console.error('error', err)
      return
    }

    console.log('New value set', newValueSet.returnValues)
  })
})()
```

## まとめ

全て準備が整ったので、DAppチェーンが稼働していることを確認してから、次のコードを実行してみよう。`Value: hello!`とコンソールにプリントされるはずだ。

```js
import {
  Client, Address, LocalAddress, CryptoUtils, LoomProvider
} from '../loom.umd'

import Web3 from 'web3'

// This function will initialize and return the client
function getClient(privateKey, publicKey) {
  const client = new Client(
    'default',
    'ws://127.0.0.1:46658/websocket',
    'ws://127.0.0.1:46658/queryws',
  )

  return client
}

// キーのセットアップ
const privateKey = CryptoUtils.generatePrivateKey()
const publicKey = CryptoUtils.publicKeyFromPrivateKey(privateKey)

// クライアントの用意
const client = getClient(privateKey, publicKey)

// web3を設定
const web3 = new Web3(new LoomProvider(client, privateKey))

;(async () => {
  // Set the contract ABI
  const ABI = [{"constant":false,"inputs":[{"name":"_value","type":"uint256"}],"name":"set","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"get","outputs":[{"name":"","type":"uint256"}],"payable":false,"stateMutability":"view","type":"function"}]

  // 公開鍵を元にアドレスを取得
  const fromAddress = LocalAddress.fromPublicKey(publicKey).toString()

  // コントラクトアドレスを取得 (これを行うにはgenesis.jsonにあるコントラクト名のみ必要となる)
  const loomContractAddress = await client.getContractAddressAsync('SimpleStore')

  // Web3との互換性を持つようLoomアドレスをhexaへ変換
  const contractAddress = CryptoUtils.bytesToHexAddr(loomContractAddress.local.bytes)

  // コントラクトのインスタンス化
  const contract = new web3.eth.Contract(ABI, contractAddress, {from: fromAddress})

  // 新規バリュー設定のリッスン
  contract.events.NewValueSet({}, (err, newValueSet) {
    if (err) {
      console.error('error', err)
      return
    }

    console.log('New value set', newValueSet.returnValues)
  })

  // 47のバリューを設定
  await contract.methods.set(47).send()

  // バリューの取得
  const result = await contract.methods.get().call()
  // result should be 47
})()

```