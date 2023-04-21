# 覆蓋網路（Overlay Network）

TON 的架構使得可以同時存在許多獨立的鏈 - 它們可以是私有的，也可以是公共的。節點可以選擇儲存和處理哪些分片和鏈的數據。同時，由於其通用性，數據交換協議保持不變。DHT、RLDP 和 overlay 等技術實現了這一目標。我們已經熟悉前兩個，現在讓我們來了解一下覆蓋網路。

覆蓋網路負責將單一網路分成額外的子網路。覆蓋網路可以是公共的，任何人都可以連接，也可以是私有的，需要附加的數據，只有特定人士才知道。

TON 中的所有鏈，包括主鏈，都使用它們自己的覆蓋網路進行數據交換。要加入覆蓋網路，需要找到已經加入該網路的節點，並與它們進行數據交換。可以使用 DHT 來查找這些節點。

## 與其他覆蓋網路的節點互動

我們已經在 DHT 的文章中介紹了查找覆蓋網路節點的示例，請參閱查找[儲存區塊鏈狀態的節點](https://github.com/xssnick/ton-deep-doc/blob/master/DHT.md#%D0%BF%D0%BE%D0%B8%D1%81%D0%BA-%D0%BD%D0%BE%D0%B4-%D1%85%D1%80%D0%B0%D0%BD%D1%8F%D1%89%D0%B8%D1%85-%D1%81%D0%BE%D1%81%D1%82%D0%BE%D1%8F%D0%BD%D0%B8%D0%B5-%D0%B1%D0%BB%D0%BE%D0%BA%D1%87%D0%B5%D0%B8%D0%BD%D0%B0)。在這一節中，我們將集中討論與這些節點的互動。

通過向 DHT 發送請求，我們可以獲取覆蓋網路節點的地址，這些節點可以使用 [overlay.getRandomPeers](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L237) 來查找屬於該覆蓋網路的其他節點的地址。在與足夠數量的節點建立連接之後，我們可以從它們那裡獲取有關塊和鏈的所有訊息，還可以將我們的交易發送到它們以進行處理。

### 讓我們找到更多的鄰居

讓我們來看一個關於在 Overlay 獲取節點的例子。

為此，我們將發送一個 `overlay.getRandomPeers` 請求，序列化 TL 模式：


```
overlay.node id:PublicKey overlay:int256 version:int signature:bytes = overlay.Node;
overlay.nodes nodes:(vector overlay.node) = overlay.Nodes;

overlay.getRandomPeers peers:overlay.nodes = overlay.Nodes;
```
`peers` 應包含我們知道的對等方。在我們想要參與 Overlay 的情況下，例如為了處理廣播，它還應包含我們的已簽名地址。但由於我們還不知道任何人且只是“詢問”，`peers.nodes` 將是一個空的數組。

`overlay.getRandomPeers` 請求是一種特殊的 ping，Overlay 成員會互相發送它，交換已知節點並檢查彼此的可用性。

為了成為有效的鄰居，需要將您的地址 `overlay.node` 保持在最新狀態，通過不斷使用當前的 unix timestamp 更新 `version` 字段。如果您的版本已過期超過 10 分鐘，您可能會被從鄰居列表中刪除。

Overlay 內的每個請求都應該具有 TL 模式作為前綴：
```
overlay.query overlay:int256 = True;
```
overlay 應該是 Overlay 的 ID - 來自 tonNode.ShardPublicOverlayId 的鍵的 ID，就像我們在 DHT 中查找時使用的那個。

我們需要將兩個序列化的模式合併，只需連接兩個位元組陣列，`overlay.query` 將是第一個，`overlay.getRandomPeers` 將是第二個。

我們將獲得的數組包裝在 `adnl.message.query` 模式中並通過 ADNL 發送。我們等待 `overlay.nodes` 的回覆 - 這將是我們可以連接到的覆蓋層節點列表，並且如果需要，可以重複相同的請求以獲得足夠數量的連接。

### 功能性請求

建立連接後，我們可以使用 `tonNode.* ` 向覆蓋層節點發出[請求](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L413)。

此類查詢使用 RLDP 協議。重要的是不要忘記前綴 `overlay.query` - 它應該用於覆蓋層中每個查詢。

在查詢本身中沒有什麼特別之處，它們非常類似於我們在有關 [ADNL TCP](https://github.com/xssnick/ton-deep-doc/blob/master/ADNL-TCP-Liteserver.md#getmasterchaininfo) 的文章中所做的內容。

例如，在 `downloadBlockFull` 查詢中已經使用了我們熟悉的塊 ID：
```
tonNode.downloadBlockFull block:tonNode.blockIdExt = tonNode.DataFull;
```

傳遞它後，我們就能夠下載有關區塊完整訊息，並收到以下回覆：

```
tonNode.dataFull id:tonNode.blockIdExt proof:bytes block:bytes is_link:Bool = tonNode.DataFull;
```

或者
```
tonNode.dataFullEmpty = tonNode.DataFull;
```
如果存在，則字段 block 中將包含 TL-B 格式數據。

因此，我們可以直接從節點獲取訊息。

## 廣播 - 在網路上傳播訊息

在某些網路中，由於網路狀態不穩定且可能會發生變化，使用廣播協議。它負責將數據傳輸到相鄰的對等方，每個接收和處理廣播訊息的對等方都會轉發該訊息給其最多五個鄰居。因此，通過向幾個對等方傳遞初始訊息，它可以在整個網路中傳播。

廣播協議在 ADNL UDP 之上工作，在 overlay 上下文中使用[自定義訊息](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L133)，並且在 shardchain 網路中被廣泛用於從驗證器向所有節點分發新塊以及從網路節點向驗證器分發外部訊息。

##### 廣播可分爲幾種類型：

* [普通](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L236) - 使用此類型的訊息可在整個網路上傳輸小數據量（例如來自用戶的外部訊息）。通常不需要特殊權限。

* [**FEC**](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L237)- 廣波訊息類型，用於分佈大量數據（例如新塊）。按 RLDP 原則工作，可能由許多部分組成，並使用 RaptorQ 算法對數據進行打包，就像在 RLDP 中一樣。
* 
* 但是，在接收完成後，我們需要回覆 [overlay.fec.completed](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L220)。
 
* [**FEC Short**](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L239) - 類似於上一個選項，但不包含數據本身，僅傳輸其哈希值。它可以優化網路負載，因為數據不會反覆來回傳輸。如果對等方還不知道所收到的哈希值，則會向發送訊息的節點請求數據。實際上目前還沒有使用此功能。

### 廣播權限

Overlay 可以具有不同的權限，並且只能允許特定節點（例如驗證器）啟動廣播。每個網路成員都事先知道可信公鑰列表（例如，在 shardchain 中 - 可信 ID 金鑰儲存在網路配置文件中，在 32-37 參數中）。

這些是以前、當前和下一個驗證器的列表。主要清單每次更新時都會更新，並且不能同時存在所有配置金鑰。

每當參與者接收到廣播訊息時，都會對其進行驗證。

要確定來源，使用 `src:PublicKey` 和 `certificate:overlay.Certificate` 字段。

首先檢查可信列表中是否存在來自 `src` 金鑰的 ID。如果 ID 在列表中 - 則該金鑰被視為有效。如果沒有此金鑰 - 則我們檢查`證書`。

要將 `PublicKey` 轉換爲 ID 金鑰，我們需要從其序列化方案中取出 [sha256 哈希值](https://github.com/xssnick/ton-deep-doc/blob/master/ADNL-TCP-Liteserver.md#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B0%D0%B9%D0%B4%D0%B8-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0)。

#### 廣播證書檢查

證書可以有多種類型：
* [**overlay.certificate**](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L228) - 標準的證書類型，沒有設置，允許所有廣播類型，但可能通過 `max_size` 參數限制消息大小。

* [**overlay.certificateV2**](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L229) - 擴展的證書類型，允許限制廣播類型（禁止 FEC）並控制信任級別（trusted/need check），因為它具有 `flags` 參數。

* [**overlay.emptyCertificate**](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L230) - 缺少證書。如果主要廣播密鑰驗證未通過-則將視為無效並被忽略。

要檢查證書，我們首先需要檢查誰發布了該證書。

為此，我們讀取 `issued_by:PublicKey` 字段的證書，並檢查是否在可信列表中存在該鍵 ID。

如果是，則我們通過根據[類型](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L232)將其序列化為模式來檢查證書籤名，在節點中指定發出證書的鍵 ID 作為 node 即 `src`：從廣播消息中獲取的 PublicKey ID。

如果簽名匹配，則可以將證書視為可信並將其歸屬於 src 密鑰，並考慮在證書中指定的限制。

#### 廣播簽名驗證

在密鑰和證書經過驗證後，我們需要驗證廣播消息本身與 `src` 字段中的密鑰是否匹配。為此，我們首先需要計算廣播消息的 ID，它將是所需[廣播類型的 broadcast.Id 模式的 ID](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L222-L223)。

在 src 字段中，我們需要指定從廣播本身 `src` 計算出來的密鑰 ID。對於 `FEC` 類型還存在 `type` 字段 - 這是來自廣播方案 `fec` 的 `id` 字段。填充結構後，我們獲取其 ID（哈希），這就是我們廣播消息的 ID。

如果我們要發送 FEC 類型，則還需要計算其部分標識符，並序列化 [overlay.broadcastFec.partId](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L224) 結構，在 `data_hash` 字段中使用從接收到的廣播部分數據獲得 sha256 哈希值。

然後，我們需要序列化 [overlay.broadcast.toSign](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L226) 模式，在 `hash` 參數中傳遞其部分標識符（在上面計算）以進行 FEC 或僅對數據域進行散列處理以進行普通廣播。

在 `data` 字段中記錄從消息獲取到時間戳記。

在序列化模式之後，唯一剩下要做的就是相對於密鑰比較簽名。

如果簽名有效，則可以將該廣播視為已接受並發送給鄰居並在本地處理。
#### Shardchain 節點內廣告內容

新的區塊（主鏈和基礎鏈）和用戶外部消息通過 shardchain 節點傳遞。

[使用方案](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L392-L397)

例如，在 `tonNode.blockBroadcast` 結構內部，在 data 字段中包含一個經過序列化的區塊單元 [TL-B 模式](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L446)，其處理可能已經取決於特定目標。

廣播處理取決於具體目標，但重要的是要記住不僅僅讀取它們是不夠的，還**需要將它們傳播給鄰居以確保網絡穩定性**。
