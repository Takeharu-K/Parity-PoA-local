---
title: 【Parity】PoAのプライベートネットをローカル環境で構築する方法
tags: parity Blockchain Ethereum
author: tktktktk
slide: false
---
# 概要
Ethereumのプライベートネットをローカル環境で構築します．

一般的に，プライベートネットを構築するにはgo言語で実装されたEthereumクライアントソフトウェアであるgethを使った場合が多いです．
しかし，今回はRustで実装されたクライアントソフトウェアであるParityを用いてPoAのプライベイートネットを構築します．

最小の構成を実現するために，今回は2ノードをバリデータとし，ユーザーアカウントを1つ作成します．最後に，構築したプライベートネットにトランザクションを送信してブロックに取り込まれたことを確認します．

# 手順
1. Parityのインストール
2. プライベートネットの設定
3. ノードの設定
4. アカウント作成
5. ノードの起動
6. ノード間の接続
7. トランザクションの送信

## 1. Parityのインストール
###Ubuntu or Mac（Homebrewインストール済み）
```
$ bash <(curl https://get.parity.io -L)
```
###Linux or Mac or Windows
[ここからダウンロード](https://github.com/paritytech/parity-ethereum/releases)
ダウンロードしてきたプログラムに実行権限を与える

## 2. プライベートネットの設定
プライベートネットの設定ファイルを作成します．

```json:demo-spec.json
{
    "name": "DemoPoA",
    "engine": {
        "authorityRound": {
            "params": {
                "stepDuration": "5",
                "validators" : {
                    "list": []
                }
            }
        }
    },
    "params": {
        "gasLimitBoundDivisor": "0x400",
        "maximumExtraDataSize": "0x20",
        "minGasLimit": "0x1388",
        "networkID" : "0x2323",
        "eip155Transition": 0,
        "validateChainIdTransition": 0,
        "eip140Transition": 0,
        "eip211Transition": 0,
        "eip214Transition": 0,
        "eip658Transition": 0
    },
    "genesis": {
        "seal": {
            "authorityRound": {
                "step": "0x0",
                "signature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
            }
        },
        "difficulty": "0x20000",
        "gasLimit": "0x5B8D80"
    },
    "accounts": {
        "0x0000000000000000000000000000000000000001": { "balance": "1", "builtin": { "name": "ecrecover", "pricing": { "linear": { "base": 3000, "word": 0 } } } },
        "0x0000000000000000000000000000000000000002": { "balance": "1", "builtin": { "name": "sha256", "pricing": { "linear": { "base": 60, "word": 12 } } } },
        "0x0000000000000000000000000000000000000003": { "balance": "1", "builtin": { "name": "ripemd160", "pricing": { "linear": { "base": 600, "word": 120 } } } },
        "0x0000000000000000000000000000000000000004": { "balance": "1", "builtin": { "name": "identity", "pricing": { "linear": { "base": 15, "word": 3 } } } }
    }
}
```
- `engine` : PoAのエンジンであるAura（Authority Round Algorithm）を指定
    - `stepDuration` : ブロック生成間隔は5秒に指定
    - `validators` : ブロック生成を行うノードのアカウントアドレスを記載する
- `params` : 標準のパラメータ設定
- `genesis` : PoAの標準値
- `accounts` : Ethereumの組み込みスマートコントラクトアドレスを設定．新規アカウントはここに追加する．

## 3. ノードの設定
2つのノードの設定ファイルを作成する．

```toml:node0.toml
[parity]
chain = "demo-spec.json"
base_path = "/tmp/parity0"
[network]
port = 30300
[rpc]
port = 8540
apis = ["web3", "eth", "net", "personal", "parity", "parity_set", "traces", "rpc", "parity_accounts"]
[websockets]
port = 8450
```

```toml:node1.toml
[parity]
chain = "demo-spec.json"
base_path = "/tmp/parity1"
[network]
port = 30301
[rpc]
port = 8541
apis = ["web3", "eth", "net", "personal", "parity", "parity_set", "traces", "rpc", "parity_accounts"]
[websockets]
port = 8451
[ipc]
disable = true
```
## 4. アカウントの作成
### 4.1. node0とnode1をそれぞれ別のターミナルで起動する．

```
$ parity --config node0.toml
```

```
$ parity --config node1.toml
```

### 4.2. 2つのバリデータアカウントと1つのユーザーアカウントを作成する．
- バリデータアカウント

```
$ curl --data '{"jsonrpc":"2.0","method":"parity_newAccountFromPhrase","params":["node0", "node0"],"id":0}' -H "Content-Type: application/json" -X POST localhost:8540
```

アドレス : `0x00Bd138aBD70e2F00903268F3Db08f2D25677C9e`

```
$ curl --data '{"jsonrpc":"2.0","method":"parity_newAccountFromPhrase","params":["node1", "node1"],"id":0}' -H "Content-Type: application/json" -X POST localhost:8541
```

アドレス : `0x00Aa39d30F0D20FF03a22cCfc30B7EfbFca597C2`

- ユーザアカウント

```
$ curl --data '{"jsonrpc":"2.0","method":"parity_newAccountFromPhrase","params":["user", "user"],"id":0}' -H "Content-Type: application/json" -X POST localhost:8540
```

アドレス : `0x004ec07d2329997267Ec62b4166639513386F32E`

アドレスを作成できたらノードを停止する．

### 4.3. プライベートネットの設定ファイルにアカウントを追加

```json:demo-spec.json
{
    "name": "DemoPoA",
    "engine": {
        "authorityRound": {
            "params": {
                "stepDuration": "5",
                "validators" : {
                    "list": [
						"0x00bd138abd70e2f00903268f3db08f2d25677c9e",
						"0x00aa39d30f0d20ff03a22ccfc30b7efbfca597c2"
					]
                }
            }
        }
    },
    "params": {
        "gasLimitBoundDivisor": "0x400",
        "maximumExtraDataSize": "0x20",
        "minGasLimit": "0x1388",
        "networkID" : "0x2323",
        "eip155Transition": 0,
        "validateChainIdTransition": 0,
        "eip140Transition": 0,
        "eip211Transition": 0,
        "eip214Transition": 0,
        "eip658Transition": 0
    },
    "genesis": {
        "seal": {
            "authorityRound": {
                "step": "0x0",
                "signature": "0x0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
            }
        },
        "difficulty": "0x20000",
        "gasLimit": "0x5B8D80"
    },
    "accounts": {
        "0x0000000000000000000000000000000000000001": { "balance": "1", "builtin": { "name": "ecrecover", "pricing": { "linear": { "base": 3000, "word": 0 } } } },
        "0x0000000000000000000000000000000000000002": { "balance": "1", "builtin": { "name": "sha256", "pricing": { "linear": { "base": 60, "word": 12 } } } },
        "0x0000000000000000000000000000000000000003": { "balance": "1", "builtin": { "name": "ripemd160", "pricing": { "linear": { "base": 600, "word": 120 } } } },
		"0x0000000000000000000000000000000000000004": { "balance": "1", "builtin": { "name": "identity", "pricing": { "linear": { "base": 15, "word": 3 } } } },
		"0x004ec07d2329997267ec62b4166639513386f32e": { "balance": "10000000000000000000000" }
    }
}
```

## 5. ノードの起動
###5.1. パスワードファイルの作成とノード情報の追加

```pwds:node.pwds
node0
node1
```

```toml:node0.toml
[parity]
chain = "demo-spec.json"
base_path = "/tmp/parity0"
[network]
port = 30300
[rpc]
port = 8540
apis = ["web3", "eth", "net", "personal", "parity", "parity_set", "traces", "rpc", "parity_accounts"]
[websockets]
port = 8450
[account]
password = ["node.pwds"]
[mining]
engine_signer = "0x00Bd138aBD70e2F00903268F3Db08f2D25677C9e"
reseal_on_txs = "none"
```

```toml:node1.toml
[parity]
chain = "demo-spec.json"
base_path = "/tmp/parity1"
[network]
port = 30301
[rpc]
port = 8541
apis = ["web3", "eth", "net", "personal", "parity", "parity_set", "traces", "rpc", "parity_accounts"]
[websockets]
port = 8451
[ipc]
disable = true
[account]
password = ["node.pwds"]
[mining]
engine_signer = "0x00Aa39d30F0D20FF03a22cCfc30B7EfbFca597C2"
reseal_on_txs = "none"
```

###5.2. 各ノードの起動
```
$ parity --config node0.toml
```

```
$ parity --config node1.toml
```

## 6. ノード間の接続
###6.1. node0のenodeアドレスを取得

```
$ curl --data '{"jsonrpc":"2.0","method":"parity_enode","params":[],"id":0}' -H "Content-Type: application/json" -X POST localhost:8540
```
###6.2. 獲得したenodeアドレスを登録
`enode://RESULT`に上で確認したアドレスを入れる．

```
curl --data '{"jsonrpc":"2.0","method":"parity_addReservedPeer","params":["enode://RESULT"],"id":0}' -H "Content-Type: application/json" -X POST localhost:8541
```

ピアの数が1/25になっていたら接続成功！

## 7. トランザクションの送信
###7.1. ユーザーアカウントからnode1にトークン送信
```
$ curl --data '{"jsonrpc":"2.0","method":"personal_sendTransaction","params":[{"from":"0x004ec07d2329997267Ec62b4166639513386F32E","to":"0x00Bd138aBD70e2F00903268F3Db08f2D25677C9e","value":"0xde0b6b3a7640000"}, "user"],"id":0}' -H "Content-Type: application/json" -X POST localhost:8540
```

###7.2. node1のトークン残高確認
```
$ curl --data '{"jsonrpc":"2.0","method":"eth_getBalance","params":["0x00Bd138aBD70e2F00903268F3Db08f2D25677C9e", "latest"],"id":1}' -H "Content-Type: application/json" -X POST localhost:8540
```
残高が増えていたらトランザクションの送信に成功！！

# まとめ
Parityクライアントを用いてプライベートネットをローカル環境に構築することができた．

今後は，リモートのサーバーで環境を構築してコンソーシアムチェーン感を出していきたい．

# 参考
[Parity 公式ドキュメント インストール手順](https://wiki.parity.io/Setup)
[Parity 公式ドキュメント PoA チュートリアル](https://wiki.parity.io/Demo-PoA-tutorial)

#Happy Hacking :sunglasses: !
