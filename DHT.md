# DHT

DHT 指 Decentalized Hash Table，本質上是一個分散式的 Key 值資料庫，每個網路參與者都可以保存一些訊息，例如有關自己的訊息。

在 TON 中，DHT 的實現類似於 Kademlia，它被用於 IPFS。

任何網路參與者都可以啟動 DHT 節點，生成密鑰並儲存資料。

為此，它需要生成一個隨機 ID，並通知其他節點自己的訊息。

為了確定在哪個節點上保存資料，使用一個算法來計算節點和密鑰之間的「距離」。

這個算法很簡單：取節點 ID 和密鑰 ID，進行 XOR 運算。

得到的值越小，節點越靠近密鑰。

任務是將密鑰保存在距離密鑰最近的節點上，以便其他網路參與者可以使用相同的算法找到能夠返回此 Key 的資料的節點。

### 根據 Key 查找值

讓我們來看一個查找 Key 的[例子](/ADNL-UDP-Internal.md#)

例如，我們想要查找與 TON 網站 `foundation.ton` 的 RLDP 節點連接的地址和資料。

假設我們已經通過 DNS 合約的 Get 方法獲取了該站點的 ADNL 地址。

以十六進制表示的 ADNL 地址是 `516618cf6cbe9004f6883e742c9a2e3ca53ed02e3e36f4cef62a98ee1e449174`。

現在我們的任務是查找具有此地址的節點的 IP、端口和公鑰。

為了做到這一點，我們需要獲取 DHT Key 的 ID，首先我們構建 DHT Key ：

```
dht.key id:int256 name:bytes idx:int = dht.Key
```

`name` 是 Key 的類型，對於 ADNL 地址，它使用 `address` 一詞，例如，對於查找 sharding node 的 Key ，使用 `nodes`。

但 Key 的類型可以是任何位元組，具體取決於所尋找的值。

填寫此模式，我們得到：
```
8fde67f6                                                           -- TL ID dht.key
516618cf6cbe9004f6883e742c9a2e3ca53ed02e3e36f4cef62a98ee1e449174   -- наш искомый ADNL адрес
07 61646472657373                                                  -- тип ключа, слово "address" в виде массива bytes
00000000                                                           -- индекс 0 т.к ключ всего 1
```

接下來 - 獲取 Key 的 ID，是序列化上面的位元組後的 sha256 Hash。
這將是 `b30af0538916421b46df4ce580bf3a29316831e0c3323a7f156df0236c5b2f75`。

現在，我們可以開始搜索了。

為此，我們需要執行具有[這個架構的查詢](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L197)：

```
dht.findValue key:int256 k:int = dht.ValueResult
```

`key` - DHT  Key 的 ID，`k` - 搜索的「寬度」，這個值越小，精確度越高，但尋找節點的數量也越少。

在 TON 中節點的最大 k 值為 10，我們可以使用 6。

填充此結構，對其進行序列化並使用 `adnl.message.query` 方案發送請求。

更多訊息請參閱[其他文章](/ADNL-UDP-Internal.md)。

我們可能會收到以下回應：
* `dht.valueNotFound` - 如果未找到值
* `dht.valueFound` - 如果值在此節點中找到

##### dht.valueNotFound

如果我們收到了 `dht.valueNotFound` 的回應，它將包含一個節點列表，這些節點已知於我們所查詢的節點，並且與我們所查詢的 Key 的列表中的節點非常接近。

在這種情況下，我們需要連接並將獲得的節點添加到我們已知的節點列表中。

之後，從我們所有已知的節點列表中選擇最接近、可用且尚未查詢的節點，然後對它進行相同的查詢。

這樣做，直到我們嘗試了所選定範圍內的所有節點或直到我們停止獲取新節點為止。

讓我們更詳細地了解回應中使用的字和架構：

```
adnl.address.udp ip:int port:int = adnl.Address;
adnl.addressList addrs:(vector adnl.Address) version:int reinit_date:int priority:int expire_at:int = adnl.AddressList;

dht.node id:PublicKey addr_list:adnl.addressList version:int signature:bytes = dht.Node;
dht.nodes nodes:(vector dht.node) = dht.Nodes;

dht.valueNotFound nodes:dht.nodes = dht.ValueResult;
```
`dht.nodes -> nodes` -  DHT 節點列表（陣列）。

每個節點都有一個 id，它是它的公鑰，通常是 pub.ed25519，用作通過 ADNL 連接到節點的服務器金鑰。

此外，每個節點都有一個地址列表 addr_list:adnl.addressList，版本和簽名。

我們必須始終檢查每個節點的簽名。

為此，我們讀取 `signature` 的值並將其清空（使其成為空位元組）。

接著，使用已清空的簽名對 TL 結構 `dht.node` 進行序列化並檢查 signature，這是在清空之前的簽名。

使用來自 `id` 的金鑰在序列化的位元組上進行檢查。[實現範例](https://github.com/xssnick/tonutils-go/blob/udp-rldp-2/adnl/dht/client.go#L91)

從 `addrs:(vector adnl.Address)` 列表中取出地址，並嘗試建立 [ADNL UDP](/ADNL-TCP-Liteserver) 連接，使用 `id` 作為公鑰作為服務器

##### dht.valueFound

該回應將包含值本身、 Key 的完整訊息以及可選的簽名。

讓我們更詳細地查看所使用的模式：

```
adnl.address.udp ip:int port:int = adnl.Address;
adnl.addressList addrs:(vector adnl.Address) version:int reinit_date:int priority:int expire_at:int = adnl.AddressList;

dht.key id:int256 name:bytes idx:int = dht.Key;

dht.updateRule.signature = dht.UpdateRule;
dht.updateRule.anybody = dht.UpdateRule;
dht.updateRule.overlayNodes = dht.UpdateRule;

dht.keyDescription key:dht.key id:PublicKey update_rule:dht.UpdateRule signature:bytes = dht.KeyDescription;

dht.value key:dht.keyDescription value:bytes ttl:int signature:bytes = dht.Value; 

dht.valueFound value:dht.Value = dht.ValueResult;
```

首先讓我們來看看 `key:dht.keyDescription`，它表示 Key 的完整描述，包括 Key 本身、關於誰以及如何更新值的訊息。

* `key:dht.key` - Key，必須完全與我們用於搜索的 Key 相匹配。
* `id:PublicKey` - 記錄擁有者的公鑰。
* `update_rule:dht.UpdateRule` - 更新記錄的規則
* * `dht.updateRule.signature` - 只有私鑰擁有者可以更新記錄，`signature`（包括 Key 和值的簽名）必須是有效的
* * `dht.updateRule.anybody` - 任何人都可以更新記錄，`signature` 為空並且不會進行檢查
* * `dht.updateRule.overlayNodes` - 只有來自同一重疊（workchain 中的分片）的節點可以更新 Key
###### dht.updateRule.signature

讀取完 Key 的描述後，我們會根據 `updateRule` 進行操作，對於查找 ADNL 地址的情況，類型始終為 `dht.updateRule.signature`。

我們以與上次相同的方式檢查 Key 的簽名，使簽名為空的位元組，序列化並進行檢查。

然後，我們對值執行相同的操作，即對整個 `dht.value` 對象進行檢查（此時 Key 的簽名已被還原）。

[實現範例](https://github.com/xssnick/tonutils-go/blob/udp-rldp-2/adnl/dht/client.go#L331)

###### dht.updateRule.overlayNodes

該用於包含有 網路中其他節點-分片的密鑰。該值總是包含 TL 結構 `overlay.nodes`。

值的簽名必須為空。

```
overlay.node id:PublicKey overlay:int256 version:int signature:bytes = overlay.Node;
overlay.nodes nodes:(vector overlay.node) = overlay.Nodes;
```
為了驗證其有效性，我們必須檢查所有的 `nodes`，並對每個節點檢查其 signature 是否符合其 id，並序列化 TL 結構：

```
overlay.node.toSign id:adnl.id.short overlay:int256 version:int = overlay.node.ToSign;
```

如我們所見，id 需要更換為 adnl.id.short，其為原始結構中的 id 的 Key。

序列化後，我們比較簽名和資料。

結果，我們得到了一個有效的節點列表，它們能夠提供我們所需的分片訊息。

###### dht.updateRule.anybody
沒有簽名，任何人都可以更新，但我沒有看到實際的用途。

##### 使用值

當所有驗證都通過且值的生存週期 `ttl:int` 沒有過期時，我們可以開始使用值本身，即 `value:bytes`。對於 ADNL 地址，其中應該包含 `adnl.addressList` 結構。

其中會列出與所請求的 ADNL 地址相對應的服務器的IP地址和端口。在我們的情況下，可能會有一個 RLDP-HTTP 服務器地址 `foundation.ton`。作為 Server Key，我們將使用 DHT 密鑰訊息中的公共金鑰 id:PublicKey。

建立連接後，我們可以使用 RLDP 協議請求網站頁面。

DHT 在此階段的任務已經完成。

### 在 DHT 中搜索存儲區塊鏈狀態的節點

DHT 也可用於查找存儲 vorkchain 和其分片資料的節點的訊息。該過程與查找任何密鑰時相同，唯一的區別在於密鑰本身的序列化和回答的驗證，這些問題我們將在本節中討論。

為了獲取例如主 workchain 及其分片的資料，我們需要填充 TL 結構：

```
tonNode.shardPublicOverlayId workchain:int shard:long zero_state_file_hash:int256 = tonNode.ShardPublicOverlayId;
```

對於主 `workchain` 的情況，workchain 將為 -1，其分片將為 -922337203685477580，並且零狀態（zero state）將為鏈的零狀態 Hash （file_hash），您可以從全局網路配置中獲取它以及其他資料，例如字段 "`validator`"

```json
"zero_state": {
  "workchain": -1,
  "shard": -9223372036854775808, 
  "seqno": 0,
  "root_hash": "F6OpKZKqvqeFp6CQmFomXNMfMj2EnaUSOXN+Mh+wVWk=",
  "file_hash": "XplPz01CXAps5qeSWUtxcyBfdAo5zVb1N979KLSKD24="
}
```

填寫完 `tonNode.shardPublicOverlayId` 後，我們對其進行序列化，並通過 Hash 從中獲取密鑰的 ID。

我們使用獲得的密鑰 ID 作為用於填充 `pub.overlay name:bytes = PublicKey` 結構的 `name`，將其包裝在 `bytes` 中。

然後序列化它，然後從中獲取密鑰 ID。

獲得的密鑰 ID 將是用於 `dht.findValue` 的密鑰，而 `name` 將是 `nodes` 一詞。

重複上一節中的主要過程，一切與之前一樣，但 `updateRule` 將是 [dht.updateRule.overlayNodes](#dhtupdateruleoverlaynodes)。

經過驗證後，我們將獲得擁有我們的 workchain 和分片訊息的節點公鑰（`id`）。

為了獲取這些節點的 ADNL 地址，我們需要將這些密鑰進行 Hash 處理以得到 ID，然後對每個 ADNL 地址重複上面描述的程序，就像處理 `foundation.ton `域名的 ADNL 地址一樣。

因此，我們將得到這些節點的地址，如果需要，就可以使用 [overlay.getRandomPeers](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L237) 來查詢這些鏈的其他節點的地址。
