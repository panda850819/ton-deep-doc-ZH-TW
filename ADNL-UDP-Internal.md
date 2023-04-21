## ADNL UDP

ADNL over UDP 用於節點和 TON 組件之間的通信。這是一種底層協議，高層協議 TON，如 DHT 和 RLDP，運行在其上。

在本文中，我們將討論 ADNL over UDP 在節點之間的基本數據交換中的工作原理，而隧道和流量匿名化將在另一篇文章中討論。

與 ADNL over TCP 不同，UDP 實現中的握手過程以不同的方式進行，並使用通道作為附加層，但其他原則相似：金鑰也是基於我們的私鑰和已知的服務器公鑰生成的，該公鑰從配置文件中預先知道或從網絡中的其他節點獲得。

在 UDP 版本的 ADNL 中，當客戶端收到第一個數據時，同時建立連接，計算金鑰並確認通道的創建，如果啟動方發送了一條關於創建通道的訊息。

在一個連接中可以打開多個通道，它們用於隔離數據。每個通道都有自己的 ID 和加密金鑰。但通常，基本交互只使用一個通道，該通道在第一個請求中創建。

### 封包結構和訊息交換

##### 第一次數據交換
讓我們研究如何初始化與 DHT 節點的連接以及獲取簽章地址列表，以了解數據交換的工作原理。

在[主網配置文件](https://ton-blockchain.github.io/global.config.json)的 `dht.nodes` 部分中找到一個喜歡的節點。例如：

```json
{
  "@type": "dht.node",
  "id": {
    "@type": "pub.ed25519",
    "key": "fZnkoIAxrTd4xeBgVpZFRm5SvVvSx7eN3Vbe8c83YMk="
  },
  "addr_list": {
    "@type": "adnl.addressList",
    "addrs": [
      {
        "@type": "adnl.address.udp",
        "ip": 1091897261,
        "port": 15813
      }
    ],
    "version": 0,
    "reinit_date": 0,
    "priority": 0,
    "expire_at": 0
  },
  "version": -1,
  "signature": "cmaMrV/9wuaHOOyXYjoxBnckJktJqrQZ2i+YaY3ehIyiL3LkW81OQ91vm8zzsx1kwwadGZNzgq4hI4PCB/U5Dw=="
}
```

1. 我們取她的 ED25519 金鑰 `fZnkoIAxrTd4xeBgVpZFRm5SvVvSx7eN3Vbe8c83YMk`，從 base64 解碼
2. 取她的 IP 地址 `1091897261` 並將其轉換為可讀格式，使用 [服務](https://www.browserling.com/tools/dec-to-ip)，得到 `65.21.7.173`
3. 結合端口號，得到 `65.21.7.173:15813` 並建立 UDP 連接。

我們想要打開一個通道與節點進行持續訊息交流，並且作為主要任務 - 從它那裡獲取已簽章地址列表。

為此，我們將生成兩條訊息，第一條是[創建通道](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L129)：
```
adnl.message.createChannel key:int256 date:int = adnl.Message
```
這裡有兩個參數 - 金鑰和日期。

對於日期，我們將指定當前 unix 時間戳。

而對於金鑰- 我們需要專門為該通道生成新的 ED25519 私鑰/公鑰對，它們將用於初始化[共享加密金鑰](/ADNL-TCP-Liteserver.md#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh)。

在訊息的 `key` 參數中，我們將指定我們生成的公鑰，而私鑰則暫時記住。

序列化填充好的 TL 結構體並得到：
```
bbc373e6                                                         -- TL ID adnl.message.createChannel 
d59d8e3991be20b54dde8b78b3af18b379a62fa30e64af361c75452f6af019d7 -- key
555c8763                                                         -- date
```

接下來轉向我們的主要請求 - [獲取地址列表](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L198)。

為了執行它，我們需要先序列化它的 TL 結構體：
```
dht.getSignedAddressList = dht.Node
```
其中沒有參數，因此只需對其進行序列化即可，得到 `ed4879a9`

接下來，由於這是一個更高級別的請求，即 DHT 協議，我們需要首先將其包裝在一個 `adnl.message.query` TL 結構中：
```
adnl.message.query query_id:int256 query:bytes = adnl.message
```
作為 `query_id`，我們生成隨機的 32 個位元組，作為 `query`，我們使用我們的主要查詢，[包裝為位元組數組](/TL.md#encoding-bytes-to-tl)。 

我們得到：
```
7af98bb4 -- TL ID adnl.message.query
d7be82afbc80516ebca39784b8e2209886a69601251571444514b7f17fcd8875 -- query_id
04 ed4879a9 000000 -- query
```
###### 構建封包

所有的數據交換都是使用封包進行的，封包的內容是 [TL 結構](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L81)：
```
adnl.packetContents
   rand1:bytes -- 隨機 7 或 15 個位元組
   flags:# -- 位標誌，用於確定下一個字段是否存在
   from:flags.0?PublicKey -- 發送者的公鑰
   from_short:flags.1?adnl.id.short -- 發件人 ID
   message:flags.2?adnl.Message -- 訊息（如果只有一個則使用）
   messages:flags.3?(vector adnl.Message) -- 訊息（如果 > 1）
   address:flags.4?adnl.addressList -- 我們的地址列表
   priority_address:flags.5?adnl.addressList -- 我們地址的優先級列表
   seqno:flags.6?long -- 包序號
   confirm_seqno:flags.7?long -- 最後接收到的數據包的序號
   recv_addr_list_version:flags.8?int -- 地址版本
   recv_priority_addr_list_version:flags.9?int -- 優先地址版本
   reinit_date:flags.10?int -- 連接重新初始化日期（計數器重置）
   dst_reinit_date:flags.10?int -- 從接收到的最後一個數據包開始的連接重新初始化日期
   signature:flags.11?bytes -- 簽章
   rand2:bytes -- 隨機 7 或 15 個位元組
         = adnl.PacketContents
        
```

一旦我們序列化了所有要發送的訊息，我們就可以開始構建封包了。要發送到通道的數據封包與通道初始化之前發送的數據封包在內容上有所不同。 

首先我們來分析一下主要的封包，它是用來初始化的。

初始交換信息時，在通道外，封包的序列化內容結構以服務器公鑰為前綴—— 32 位元組，我們的公鑰為 32 位元組，內容結構序列化 TL 的 sha256 哈希封包的 - 32 位元組。 

封包內容使用[共享金鑰](/ADNL-TCP-Liteserver.md#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh) 從我們的私鑰和服務器的公鑰中得到。

序列化我們的包內容結構，逐位元組解析：
```
89cd42d1 -- TL ID adnl.packetContents
0f 4e0e7dd6d0c5646c204573bc47e567 -- rand1, 15 (0f) 隨機位元組
d9050000 -- 標誌 (0x05d9) -> 0b0000010111011001
                                                                        -- 來自（存在是因為標誌位 0 = 1）
c6b41348 -- TL ID pub.ed25519
    afc46336dd352049b366c7fd3fc1b143a518f0d02d9faef896cb0155488915d6——金鑰：int256
                                                                        -- 訊息（存在是因為標誌的第 3 位 = 1）
02000000 -- 向量 adnl.Message，大小為 2 的訊息
    bbc373e6 -- TL ID adnl.message.createChannel
    d59d8e3991be20b54dde8b78b3af18b379a62fa30e64af361c75452f6af019d7——鑰匙
    555c8763 -- 日期（創建日期）
   
    7af98bb4 -- TL ID [adnl.message.query](/)
    d7be82afbc80516ebca39784b8e2209886a69601251571444514b7f17fcd8875 -- query_id
    04 ed4879a9 000000 -- 查詢（位元組大小 4，填充 3）
                                                                        -- 地址（存在是因為標誌的第 4 位 = 1），沒有 TL ID，因為明確指定
00000000 -- addrs（空向量，因為我們處於客戶端模式並且竊聽器上沒有地址）
555c8763 —— 版本（通常是初始化日期）
555c8763 -- reinit_date（通常是 reinit 日期）
00000000 —— 優先級
00000000 —— 過期時間

0100000000000000 -- seqno（存在是因為標誌位 6 = 1）
0000000000000000 -- confirm_seqno（存在是因為標誌位 7 = 1）
555c8763 -- recv_addr_list_version（存在於位 8 = 1，通常是初始化日期）
555c8763 -- reinit_date（存在是因為標誌位 10 = 1，通常是初始化日期）
00000000 -- dst_reinit_date（存在是因為標誌位 10 = 1）
0f 2b6a8c0509f85da9f3c7e11c86ba22 -- rand2, 15 (0f) 隨機位元組
```

序列化後 - 我們需要使用我們之前生成並記住的私有 ED25519 客戶端金鑰（而不是通道）對生成的位元組數組進行簽章。 當我們收到簽章（64 位元組大小）後，我們需要將其添加到包中，再次序列化，但將第 11 位添加到標誌中，這意味著簽章的存在：

```
89cd42d1 -- TL ID adnl.packetContents
0f 4e0e7dd6d0c5646c204573bc47e567 -- rand1, 15 (0f) 隨機位元組
d90d0000 -- 標誌 (0x0dd9) -> 0b0000110111011001
                                                                        -- 來自（存在是因為標誌位 0 = 1）
c6b41348 -- TL ID pub.ed25519
    afc46336dd352049b366c7fd3fc1b143a518f0d02d9faef896cb0155488915d6——金鑰：int256
                                                                        -- 訊息（存在是因為標誌的第 3 位 = 1）
02000000 -- 向量 adnl.Message，大小為 2 的訊息
    bbc373e6 -- TL ID adnl.message.createChannel
    d59d8e3991be20b54dde8b78b3af18b379a62fa30e64af361c75452f6af019d7——鑰匙
    555c8763 -- 日期（創建日期）
   
    7af98bb4 -- TL ID adnl.message.query
    d7be82afbc80516ebca39784b8e2209886a69601251571444514b7f17fcd8875 -- query_id
    04 ed4879a9 000000 -- 查詢（位元組大小 4，填充 3）
                                                                        -- 地址（存在是因為標誌的第 4 位 = 1），沒有 TL ID，因為明確指定
00000000 -- addrs（空向量，因為我們處於客戶端模式並且竊聽器上沒有地址）
555c876 -- 版本（通常是初始化日期）
555c8763 -- reinit_date（通常是 reinit 日期）
00000000 -- 優先級
00000000 -- 過期時間

0100000000000000 -- seqno（存在是因為標誌位 6 = 1）
0000000000000000 -- confirm_seqno（存在是因為標誌位 7 = 1）
555c8763 -- recv_addr_list_version（存在於位 8 = 1，通常是初始化日期）
555c8763 -- reinit_date（存在是因為標誌位 10 = 1，通常是初始化日期）
00000000 -- dst_reinit_date（存在是因為標誌位 10 = 1）
40 b453fbcbd8e884586b464290fe07475ee0da9df0b8d191e41e44f8f42a63a710 -- 簽章（存在是因為標誌位 11 = 1），（位元組大小 64，填充 3）
    341eefe8ffdc56de73db50a25989816dda17a4ac6c2f72f49804a97ff41df502 --
    000000 --
0f 2b6a8c0509f85da9f3c7e11c86ba22 -- rand2, 15 (0f) 隨機位元組
```
現在我們有一個組裝、簽章和序列化的封包，它是一個位元組數組。

為了讓收件人隨後驗證其完整性，我們需要計算其 sha256 哈希值。 

例如，讓它成為“408a2a4ed623b25a2e2ba8bbe92d01a3b5dbd22c97525092ac3203ce4044dcd2”。

現在讓我們使用[共享金鑰](/ADNL-TCP-Liteserver.md#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh)從我們的私鑰和服務器的公鑰（不是通道金鑰）中獲得。

我們幾乎準備好發送了，它仍然是[計算 ID](/ADNL-TCP-Liteserver.md#get-id-key) 服務器金鑰的 ED25519 並將所有內容連接在一起：
```
daa76538d99c79ea097a67086ec05acca12d1fefdbc9c96a76ab5a12e66c7ebb -- 服務器金鑰 ID
afc46336dd352049b366c7fd3fc1b143a518f0d02d9faef896cb0155488915d6——我們的公鑰
408a2a4ed623b25a2e2ba8bbe92d01a3b5dbd22c97525092ac3203ce4044dcd2 -- sha256 內容哈希（加密前）
... -- 包的加密內容
```
現在我們可以將接收到的數據封包通過 UDP 發送給服務器，等待響應。

作為響應，我們將收到一個結構相似但訊息不同的封包。 它將包括：
```

68426d4906bafbd5fe25baf9e0608cf24fffa7eca0aece70765d64f61f82f005 -- 我們金鑰的 ID
2d11e4a08031ad3778c5e060569645466e52bd1bd2c7b78ddd56def1cf3760c9 -- 服務器公鑰，用於共享金鑰
f32fa6286d8ae61c0588b5a03873a220a3163cad2293a5dace5f03f06681e88a -- sha256 內容哈希（加密前）
... -- 封包的加密內容
```

從服務端反序列化封包如下：
1. 我們從包裹中查看鑰匙的 ID，了解包裹是給我們的。
2. 使用包中服務器的公鑰和我們的私鑰，我們創建一個公鑰並解密包的內容
3. 將發送給我們的 sha256 哈希與從解密數據中接收到的哈希進行比較，它們必須匹配
4. 使用 `adnl.packetContents` TL 模式開始反序列化封包內容

封包的內容如下所示：
```
89cd42d1 -- TL ID adnl.packetContents
0f 985558683d58c9847b4013ec93ea28 -- rand1, 15 (0f) 隨機位元組
ca0d0000 -- 標誌 (0x0dca) -> 0b0000110111001010
daa76538d99c79ea097a67086ec05acca12d1fefdbc9c96a76ab5a12e66c7ebb -- from_short（因為標誌位 1 為 1）
02000000 -- 訊息（存在是因為第 3 個標誌位 = 1）
    691ddd60 -- TL ID adnl.message.confirmChannel
    db19d5f297b2b0d76ef79be91ad3ae01d8d9f80fab8981d8ed0c9d67b92be4e3 -- 金鑰（服務器通道公鑰）
    d59d8e3991be20b54dde8b78b3af18b379a62fa30e64af361c75452f6af019d7 -- peer_key（我們的公共頻道金鑰）
    94848863——日期
   
    1684ac0f -- TL ID adnl.message.answer
    d7be82afbc80516ebca39784b8e2209886a69601251571444514b7f17fcd8875 -- query_id
    90 48325384c6b413487d99e4a08031ad3778c5e060569645466e52bd5bd2c7b -- answer（我們要求的答案，我們會在DHT的文章中分析它的內容）
       78ddd56def1cf3760c901000000e7a60d67ad071541c53d0000ee354563ee --
       354563000000000000000009484886340d46cc50450661a205ad47bacd318c --
       65c8fd8e8f797a87884c1bad09a11c36669babb88f75eb83781c6957bc976 --
       6a234f65b9f6e7cc9b53500fbe2c44f3b3790f000000 --
       000000 --
0100000000000000 -- seqno（存在是因為標誌位 6 = 1）
0100000000000000 -- confirm_seqno（存在是因為標誌位 7 = 1）
94848863 -- recv_addr_list_version（存在是因為第 8 位 = 1，通常是初始化日期）
ee354563 -- reinit_date（存在是因為標誌位 10 = 1，通常是初始化日期）
94848863 -- dst_reinit_date（存在是因為標誌位 10 = 1）
40 5c26a2a05e584e9d20d11fb17538692137d1f7c0a1a3c97e609ee853ea9360ab6 -- 簽章（存在是因為標誌位 11 = 1），（位元組大小 64，填充 3）
    d84263630fe02dfd41efb5cd965ce6496ac57f0e51281ab0fdce06e809c7901 --
    000000 --
0f c3354d35749ffd088411599101deb2 -- rand2, 15 (0f) 隨機位元組
```

服務器用兩條訊息回答了我們：`adnl.message.confirmChannel` 和 `adnl.message.answer`。 

有了 adnl.message.answer 就簡單了，就是對我們請求的 dht.getSignedAddressList 的回答，我們會在 DHT 的文章中分析。

讓我們關注 `adnl.message.confirmChannel`，這意味著服務器已經確認頻道的創建並向我們發送了它的公共頻道金鑰。 現在，有了我們的私人頻道金鑰和服務器的公共頻道金鑰，我們可以計算[共享金鑰](/ADNL-TCP-Liteserver.md#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh)。

現在我們已經計算出共享通道金鑰，我們需要從中生成 2 個金鑰 - 一個用於加密傳出訊息，另一個用於解密傳入訊息。 

製作 2 把鑰匙很簡單，第二把鑰匙等於普通鑰匙倒序寫的。 

例子：
```
共享金鑰：AABB2233

第一把鑰匙：AABB2233
第二把鑰匙：3322BBAA
```
仍然需要確定將哪個金鑰用於什麼，我們可以通過將我們的公共頻道金鑰的 ID 與服務器頻道的公共金鑰的 ID 進行比較，將它們轉換為數字形式 - uint256 來完成此操作。 

此方法用於確保服務器和客戶端確定將哪個金鑰用於什麼。 
如果服務器使用第一個金鑰進行加密，那麼使用這種方法，客戶端將始終使用它進行解密。

使用條款如下：
```
服務器 id 小於我們的 id：
加密：第一把鑰匙
解密：第二把鑰匙

服務器 ID 大於我們的 ID：
加密：第二個金鑰
解密：第一把鑰匙

如果 ID 相等（幾乎不可能）：
加密：第一把鑰匙
解密：第一把鑰匙
```
[實現範例](https://github.com/xssnick/tonutils-go/blob/udp-rldp-2/adnl/adnl.go#L502)

##### 通道中的數據交換

所有後續數據封包交換都將在通道內進行，並且將使用新金鑰進行加密。 
讓我們在新創建的通道內發送相同的 `dht.getSignedAddressList` 請求以查看差異。

讓我們使用相同的 adnl.packetContents 結構收集頻道的包內容：
```
89cd42d1 -- TL ID adnl.packetContents
0f c1fbe8c4ab8f8e733de83abac17915 -- rand1, 15 (0f) 隨機位元組
c4000000 -- 標誌 (0x00c4) -> 0b0000000011000100
                                                                        -- 訊息（因為第 2 位 = 1）
7af98bb4 -- TL ID adnl.message.query
fe3c0f39a89917b7f393533d1d06b605b673ffae8bbfab210150fe9d29083c35 -- query_id
04 ed4879a9 000000 -- 查詢（我們的 dht.getSignedAddressList 用填充 3 位元組打包）
0200000000000000 -- seqno（因為標誌的第 6 位 = 1），2 tk 是我們的第二條訊息
0100000000000000 -- confirm_seqno (7th flag bit = 1), 1 tk 是從服務器收到的最後一個seqno
07 e4092842a8ae18 -- rand2, 7 (07) 隨機位元組
```

通道中的數據封包非常簡單，主要由序列 (seqno) 和訊息本身組成。

序列化後，和上次一樣，我們從內容中計算出 sha256 哈希值。 

然後我們使用用於通道傳出數據包的金鑰加密數據封包的內容。 
[計算](/ADNL-TCP-Liteserver.md#additional-technical-handshake-details) `pub.aes` 我們傳出訊息加密金鑰的 ID，並構建我們的封包：
```
bcd1cf47b9e657200ba21d94b822052cf553a548f51f539423c8139a83162180 -- 我們傳出訊息加密金鑰的 ID
6185385aeee5faae7992eb350f26ba253e8c7c5fa1e3e1879d9a0666b9bd6080 -- sha256 內容哈希（加密前）
... -- 封包的加密內容
```
我們通過 UDP 發送數據封包並等待響應。 

作為響應，我們將收到一個與我們發送的類型相同的數據包（相同的字段），但包含對我們請求“dht.getSignedAddressList”的回答。

#### 其他訊息類型
對於基本通信，我們使用了類似 `adnl.message.query` 和 `adnl.message.answer` 的訊息，我們在上面討論過，但在某些情況下也可能使用其他類型的訊息，我們將在本節中討論。

##### adnl.message.part
此訊息類型是其他可能訊息類型之一的一部分，例如 `adnl.message.answer`。 
當訊息太大而無法在單個 UDP 數據報中傳輸時，使用此傳輸方法。
```
adnl.message.part
hash:int256 -- 原始訊息的 sha256 哈希
total_size:int -- 原始訊息大小
offset:int -- 從原始訊息開始的偏移量
data:bytes -- 原始訊息的數據塊
    = adnl.message;
```
因此，為了組裝原始訊息，我們需要得到幾個部分，並根據偏移量將它們添加到單個位元組數中。 
然後將其作為訊息處理（根據此數組中的前綴）。

##### adnl.message 自定義
```
adnl.message.custom data:bytes = adnl.message;
```
當更高層的邏輯不符合請求-響應格式時使用此類訊息，這種類型的訊息允許您將處理完全移動到更高層，因為訊息只攜帶一個位元組數組，沒有 query_id 和其他領域。 

例如，在 RLDP 中使用這種類型的訊息，因為對許多請求只能有一個響應，並且此邏輯由 RLDP 本身控制。

##### 結論

進一步的數據交換是在本文討論的邏輯基礎上進行的，
但數據包的內容取決於更高級別的協議，例如 DHT 和 RLDP。