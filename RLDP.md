## RLDP

RLDP - Reliable Large Datagram Protocol 是一種運行在 ADNL UDP 上的協議，用於傳輸大型資料，並包含正向錯誤更正（FEC）算法以替代對方的資料包確認。

這可以更有效地（雖然消耗更多的流量）在網絡組件之間傳輸資料。

RLDP 幾乎廣泛地用於 TON 的基礎設施中。

例如，用於從其他節點下載塊和傳輸資料，用於 TON 網站的請求以及在 TON 存儲中。

協議
為了進行 RLDP 資料交換，使用以下 TL 結構：

#### 協議

為了進行 RLDP 資料交換，使用以下 TL 結構：
```
fec.raptorQ data_size:int symbol_size:int symbols_count:int = fec.Type;
fec.roundRobin data_size:int symbol_size:int symbols_count:int = fec.Type;
fec.online data_size:int symbol_size:int symbols_count:int = fec.Type;

rldp.messagePart transfer_id:int256 fec_type:fec.Type part:int total_size:long seqno:int data:bytes = rldp.MessagePart;
rldp.confirm transfer_id:int256 part:int seqno:int = rldp.MessagePart;
rldp.complete transfer_id:int256 part:int = rldp.MessagePart;

rldp.message id:int256 data:bytes = rldp.Message;
rldp.query query_id:int256 max_answer_size:long timeout:int data:bytes = rldp.Message;
rldp.answer query_id:int256 data:bytes = rldp.Message;
```
序列化的結構包裝在 TL 方案 `adnl.message.custom` 中，並通過 ADNL UDP 發送。

要傳輸大型資料，使用轉移，生成隨機 `transfer_id`，並使用 FEC 算法處理資料。

將接收到的碎片包裝在結構 `rldp.messagePart` 中，並發送給接收器，直到接收器向我們發送 `rldp.complete` 為止。

當接收器收集到需要組合完整消息的所有 `rldp.messagePart` 片段時，它將它們全部連接在一起，使用 FEC 解碼並將接收到的 Byte 陣列反序列化為 `rldp.query` 或 `rldp.answer` 中的一個結構，具體取決於類型（tl id 前綴）

### FEC

當使用 RLDP 時，可使用的前向錯誤更正算法包括 RoundRobin、Online 和 RaptorQ 。

目前，用於資料交換的前向錯誤更正算法是 [RaptorQ](https://www.qualcomm.com/media/documents/files/raptorq-technical-overview.pdf)。

##### RaptorQ

RaptorQ 的核心思想是將資料分成相同預先定義大小的區塊，這些區塊組成矩陣，並應用離散數學運算來生成幾乎無限數量的符號。

所有符號混合在一起，通過這種方式，可以恢復丟失的資料包，而無需向伺服器請求額外的資料，並且可以使用比按順序發送相同資料區塊少的資料包數來恢復資料。

生成的符號發送給接收方，直到接收方確認已經接收和恢復了所有資料為止，方法是應用相同的離散操作。

作者在 Golang 上實現了 [RaptorQ](https://github.com/xssnick/tonutils-go/tree/udp-rldp-2/adnl/rldp/raptorq)，您可以在這裡找到我的實現。

### RLDP-HTTP

在 TON 上與網站進行交互時，使用的是在 RLDP 中包裝的 HTTP 協議。主機將其網站放置在任何 HTTP Web 伺服器上，並在其旁邊啟動 RLDP-HTTP 代理。

來自 TON 網路的所有請求都通過 RLDP 協議發送到代理中，代理將請求重組為普通的 HTTP 請求並在本地調用 Web 伺服器。

使用者在本地（理想情況下）啟動代理，例如 [Tonutils Proxy](https://github.com/xssnick/TonUtils-Proxy)，並使用 `.ton` 網站，所有流量都被反向封裝，請求發送到本地代理，它再將它們通過 RLDP 發送到遠程 TON 網站。

在 RLDP 中實現 HTTP 使用 TL 結構。

```
http.header name:string value:string = http.Header;
http.payloadPart data:bytes trailer:(vector http.header) last:Bool = http.PayloadPart;
http.response http_version:string status_code:int reason:string headers:(vector http.header) no_payload:Bool = http.Response;

http.request id:int256 method:string url:string http_version:string headers:(vector http.header) = http.Response;
http.getNextPayloadPart id:int256 seqno:int max_chunk_size:int = http.PayloadPart;
```
這不是純文本的 HTTP，所有內容都包裝在二進制的 TL 中，在發送到 Web 伺服器或瀏覽器之前由代理解包裝。

其工作方式如下：
* 客戶端發送請求 `http.request`
* 當伺服器接收到請求時，檢查標頭 `Content-Length`
** 如果標頭中不是 0，則向客戶端發送請求 `http.getNextPayloadPart`
** 當客戶端收到請求時，它會發送 `http.payloadPart` - 根據 `seqno` 和 `max_chunk_size` 請求的正文部分.
** 伺服器重複請求，增加 `seqno`，直到從客戶端收到所有片段，直到最後接收片段的  `last:Bool` 為 true.
* 處理請求後，服務器發送 `http.response`，客戶端檢查標頭`Content-Length`
** 如果標頭中不是 0，則向服務器發送請求 `http.getNextPayloadPart`，並且操作與客戶端相同。

#### 訪問 TON 網站

為了理解 RLDP 的工作原理，讓我們以獲取 `foundation.ton` 網站的資料為例。

假設我們已經通過 NFT-DNS 合約的 Get 方法獲取了它的 ADNL 地址，使用 [DHT](/DHT.md) 確定了 RLDP 服務的地址和端口，並且通過 [ADNL UDP](/ADNL-UDP-Internal.md) 連接到它。

##### 像 `foundation.ton` 發送 GET 請求
為此，我們填充以下結構：

```
http.request id:int256 method:string url:string http_version:string headers:(vector http.header) = http.Response;
```
我們對 `http.request` 進行序列化，並填寫字段：

```
e191b161                                                           -- TL ID http.request      
116505dac8a9a3cdb464f9b5dd9af78594f23f1c295099a9b50c8245de471194   -- id           = {random}
03 474554                                                          -- method       = GET
16 687474703a2f2f666f756e646174696f6e2e746f6e2f 00                 -- url          = http://foundation.ton/
08 485454502f312e31 000000                                         -- http_version = HTTP/1.1
01000000                                                           -- headers (1)
   04 486f7374 000000                                              -- name         = Host
   0e 666f756e646174696f6e2e746f6e 00                              -- value        = foundation.ton
```

現在讓我們將序列化的 `http.request` 打包到 `rldp.query` 中並對其進行序列化：

```
694d798a                                                              -- TL ID rldp.query
184c01cb1a1e4dc9322e5cabe8aa2d2a0a4dd82011edaf59eb66f3d4d15b1c5c      -- query_id        = {random}
0004040000000000                                                      -- max_answer_size = 257 KB, 可以是任何足夠的大小
258f9063                                                              -- timeout (unix)  = 1670418213
34 e191b161116505dac8a9a3cdb464f9b5dd9af78594f23f1c295099a9b50c8245   -- data (http.request)
   de4711940347455416687474703a2f2f666f756e646174696f6e2e746f6e2f00
   08485454502f312e310000000100000004486f73740000000e666f756e646174
   696f6e2e746f6e00 000000
```

##### 編碼並發送資料包

現在我們的任務是將這些資料應用於 FEC 算法，具體來說是 RaptorQ。

讓我們創建一個編碼器，為此我們需要將收到的 Byte 陣列轉換為固定大小的符號。

在 TON 中，符號大小為 768 Byte。

為此，我們將陣列分成大小為 768 Byte 的塊。

如果最後一個塊的大小小於 768，則需要使用 0 Byte 填充它以達到所需的大小。

我們的陣列大小為 156  Byte ，因此只有 1 個塊，我們需要使用 612 個 0 Byte 將其填充到 768 的大小。

同樣，編碼器選擇的常數取決於資料和符號的大小，有關詳細信息，可以在RaptorQ的文檔中了解到，但是為了避免進入數學細節，- 我建議使用已經準備好的庫來實現此編碼。

[創建編碼器的範例](https://github.com/xssnick/tonutils-go/blob/udp-rldp-2/adnl/rldp/raptorq/encoder.go#L15) 和 [範例編碼符號](https://github.com/xssnick/tonutils-go/blob/be3411cf412f23e6889bf0b648904306a15936e7/adnl/rldp/raptorq/solver.go#L26)。

編碼和發送符號是循環進行的：最初，我們定義了一個 `seqno`，它等於 0 並且是編碼的參數，對於每個後續編碼的包，我們將其增加 1。

例如，如果我們有 2 個符號，那麼我們首先編碼並發送第一個符號，將 seqno 增加 1，然後編碼並發送第二個符號，再次增加 seqno 1，然後再編碼並發送第一個符號，這時 seqno 已經是 2 了，再增加1。

一直重複，直到收到接收器接收到資料的消息為止。

```
fec.raptorQ data_size:int symbol_size:int symbols_count:int = fec.Type;

rldp.messagePart transfer_id:int256 fec_type:fec.Type part:int total_size:long seqno:int data:bytes = rldp.MessagePart;
```

* `transfer_id` - 隨機的 int256，對於同一傳輸中的所有 messagePart 相同。
*  `fec_type` 將是 `fec.raptorQ`。
*  * `data_size` = 156
*  * `symbol_size` = 768
*  * `symbols_count` = 1
*  `part` 在我們的情況下始終為 0，可用於達到傳輸限制的傳輸。
*  `total_size` = 156. 我們傳輸的資料大小。
*  `seqno` - 對於第一個封包將為 0，對於每個後續封包都將增加 1，使用它進行編碼。
*  `data` - 我們編碼後的符號，大小為768字節。

在對 `rldp.messagePart` 進行序列化後，將其包裝在 `adnl.message.custom` 中並通過 ADNL UDP 發送。

我們循環發送封包，並持續增加 seqno，直到接收方發送 `rldp.complete` 消息，或達到超時。

在發送等於我們符號數量的封包後，我們可以減慢速度，例如每 10 毫秒或更少地發送一個附加封包。

附加封包用於在資料丟失的情況下進行恢復，因為 UDP 是快速但不可靠的協議。

[實現範例](https://github.com/xssnick/tonutils-go/blob/be3411cf412f23e6889bf0b648904306a15936e7/adnl/rldp/rldp.go#L249)

##### 解析來自 `foundation.ton` 的回應

在發送請求後（或在發送期間），我們可以等待服務器的回應，即在我們的情況下，我們等待包含 `http.response` 的 `rldp.answer`。

它像發送請求時一樣作為 RLDP 傳輸來發送。我們會收到包含 `rldp.messagePart` 的 `adnl.message.custom` 消息。

首先，我們需要從第一個接收到的傳輸消息中獲取有關 FEC 的信息，即 `fec.raptorQ` 結構的 messagePart 中的 `data_size`、`symbol_size` 和 `symbols_count` 參數。

它們對我們初始化 RaptorQ 解碼器非常重要。

[範例](https://github.com/xssnick/tonutils-go/blob/be3411cf412f23e6889bf0b648904306a15936e7/adnl/rldp/rldp.go#L137)

初始化後，我們將帶有其 seqno 的接收到的符號添加到解碼器中。當累積的符號數量達到 symbols_count 的最小要求時，我們可以嘗試解碼完整消息。成功後，我們將發送 `rldp.complete`。

[範例](https://github.com/xssnick/tonutils-go/blob/be3411cf412f23e6889bf0b648904306a15936e7/adnl/rldp/rldp.go#L168)

結果，我們將收到帶有相同 query_id 的 `rldp.answer`，該 ID 與我們發送的 `rldp.query` 相同。其中包含 `http.response` 的資料。

```
http.response http_version:string status_code:int reason:string headers:(vector http.header) no_payload:Bool = http.Response;
```

主要字段和 HTTP 中一樣，意義相同。

有趣的標誌是 `no_payload`，如果為 true，則意味著回應中沒有主體（`Content-Length` = 0）。

可以認為已經收到了服務器的回應。

如果 `no_payload` = false，則回應中有內容，我們需要獲取它。

為此，我們需要發送 TL 方案為 `http.getNextPayloadPart` 的請求，並將其包裝在 `rldp.query` 中。

```
http.getNextPayloadPart id:int256 seqno:int max_chunk_size:int = http.PayloadPart;
```

當你發送 `http.request` 後，id 應該和之前發送的相同，`seqno` 的值為 0，每個後續部分的值增加 1，`max_chunk_size` 表示每個部分的最大大小，通常為 128KB（131072 Byte）。

當收到響應時，你會得到以下 TL 方案的回應：
```
http.payloadPart data:bytes trailer:(vector http.header) last:Bool = http.PayloadPart;
```
如果 `last` 為 true，表示我們已經到達末尾，可以將所有部分結合起來，得到完整的回應主體，例如 HTML。


