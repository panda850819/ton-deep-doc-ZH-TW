
## ADNL TCP + Liteserver

ADNL TCP + Liteserver 是 TON 網路中構建所有互動的底層協議，它可以在任何協議上運作，但通常在 TCP 和 UDP 協議上運作。UDP 用於節點之間的通訊，而 TCP 用於與輕型伺服器通訊。

現在我們將介紹在 TCP 上運作的 ADNL，並學習如何直接與輕型伺服器進行互動。

在 ADNL 的 TCP 版本中，網路節點使用公鑰 ed25519 作為地址，並使用橢圓曲線 Diffie-Hellman 程序獲取的共享密鑰建立連接。

### 封包結構

除了握手機制（Handshake）封包之外，每個 ADNL TCP 包都具有以下結構：

* 4 個位元組的 little endian 格式的封包大小（N）
* 32 個位元組的 nonce（以防校驗和攻擊）
* (N-64) 位元組的有效資料
* 32 個位元組的 SHA256 校驗用於 nonce 和有效資料

整個封包，包括大小，都使用 AES-CTR 加密。在解密後，必須驗證校驗和是否與資料相符。為了進行驗證，只需自行計算校驗和並將結果與包中的校驗和進行比較。

握手包是個例外，它以部分開放的形式傳遞，並在下一節中進行描述。

整個封包，包括尺寸，都是使用 **AES-CTR** 加密的。在解密後，必須確認檢查和是否與資料相符，檢查和可以自行計算並與封包中的結果進行比較以進行確認。

握手封包是一個例外，它是以部分開放的方式傳輸的，並在下一節中進行了描述。

### 建立連接
要建立連接，我們需要知道伺服器的 IP、端口和公鑰，並生成自己的 Ed25519 私鑰和公鑰。

可以從[配置文件](https://ton-blockchain.github.io/global.config.json)中獲得伺服器的公開資訊，例如 IP、端口和鑰匙。在配置文件中，IP 是以數字形式表示的，可以使用像[這樣的工具](https://www.browserling.com/tools/dec-to-ip)將其轉換為正常形式。公鑰以 Base64 格式在配置文件中提供。

客戶端生成 160 個隨機位元組，其中一部分將用作 AES 加密的基礎。從中建立了 2 個永久 AES-CTR 加密，這將在握手後由雙方用於加密 / 解密訊息。

* 加密 A - 金鑰 0-31 位元組，iv 64-79 位元組
* 加密 B - 金鑰 32-63 位元組，iv 80-95 位元組


這些加密按以下順序應用：

* 伺服器使用加密 A 加密發送的消息。
* 客戶端使用加密 A 解密收到的消息。
* 客戶端使用加密 B 加密發送的消息。
* 伺服器使用加密 B 解密收到的消息。
為了建立連接，客戶端必須發送一個握手封包，其中包含：

* [32 字節] 伺服器金鑰 ID [更多資訊](#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%B0%D0%B9%D0%B4%D0%B8-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0)
* [32 字節] 我們的 ed25519 公鑰
* [32 字節] 我們 160 字節的 SHA256 哈希值
* [160 字節] 我們的 160 字節加密後的數據  [更多資訊](#%D1%88%D0%B8%D1%84%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85-handshake-%D0%BF%D0%B0%D0%BA%D0%B5%D1%82%D0%B0)

接收到握手包後，伺服器在自己端進行同樣的操作，獲取 ECDH 金鑰，解密 160 個字節並創建 2 個永久的金鑰。如果一切順利，伺服器將回傳一個空的 ADNL 包，沒有有效數據，為了解密它（以及後續的數據），需要使用其中一個永久加密。

從這一刻開始，可以認為連接已經建立。

## 數據交換

在建立連接後，可以開始接收訊息，使用 TL 語言進行數據序列化。
[更多關於 TL 的資訊]((/TL.md))

### Ping & Pong

Ping 封包最好每 5 秒左右發送一次，以維持連接，如果在交換數據之前沒有發送，則伺服器將斷開連接。

Ping 包和其他包一樣，遵循 [上面](#структура-пакетов) 描述的標準方案，其有效數據包括 ID 和請求 ID。

可以在 [這裡](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L35) 找到 ping 請求的正確方案，並計算方案 ID，如下所示：`crc32_IEEEE("tcp.ping random_id:long = tcp.Pong")`。在使用小端序轉換為字節時，ID 為 **9a2b084d**。

因此，我們的 ADNL ping 包將如下所示：

* 4 字節的小端序包大小 -> 64 + (4+8) = 76
* 32 字節的 nonce -> 32 個隨機字節
* 4 字節的 TL 方案 ID -> **9a2b084d**
* 8 字節的請求 ID -> 隨機 uint64 數字
* SHA256 摘要的 32 個字節，是由 nonce 和有用的數據計算而得

我們發送我們的包，然後等待回覆的 [tcp.pong](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L23)，其中的 `random_id` 將等於我們發送的 ping 的值。

### 從輕量級服務器獲取訊息

所有向區塊鏈獲取訊息的請求都被封裝在 [LiteServer Query](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L83) 方案中，該方案本身又被封裝在 [ADNL Query](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L2) 方案中。

LiteQuery:
`liteServer.query data:bytes = Object`, ID 是 **df068c79**

ADNLQuery:
`adnl.message.query query_id:int256 query:bytes = adnl.Message`, ID 是 **7af98bb4**

LiteQuery 被傳遞到 ADNLQuery 內部作為 `query:bytes`，最終請求作為 `data:bytes` 在 LiteQuery 內部傳遞。

[TL 中的字節編碼解析](/TL.md)

#### 獲取有用數據

現在，由於我們已經知道如何為 Lite API 構建 TL 包，我們可以請求有關 TON 主控鏈當前區塊的資訊。主控鏈區塊在許多後續請求中都被用作輸入參數，用於指示我們需要的訊息狀態（時間點）。

##### getMasterchainInfo
尋找我們需要的 [TL 方案](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L60)，計算其 ID 並構建封包：

* 4個字節的封包大小（little endian）-> 64 + (4+32+(1+4+(1+4+3)+3)) = **116**
* 32 個字節的 nonce -> 隨機的 32 個字節
* 4個字節的 ADNLQuery 方案ID -> **7af98bb4**
* 32個字節的 `query_id:int256` -> 隨機的32個字節
* 1 個字節的數組大小 -> **12**
* 4 個字節的 LiteQuery 方案 ID -> **df068c79**
* 1個字節的數組大小 -> **4**
* 4 個字節的 getMasterchainInfo 方案 ID -> **2ee6b589**
* 3 個零填充字節（按 8 字節對齊）
* 3 個零填充字節（按 16 字節對齊）
* 32 個字節 SHA256 摘要的校驗和，由 nonce 和有用的數據計算而得。

封包的十六進位範例
```
74000000                                                             -> 封包大小 (116)
5fb13e11977cb5cff0fbf7f23f674d734cb7c4bf01322c5e6b928c5d8ea09cfd     -> nonce
  7af98bb4                                                           -> ADNLQuery
  77c1545b96fa136b8e01cc08338bec47e8a43215492dda6d4d7e286382bb00c4   -> query_id
    0c                                                               -> 陣列大小
    df068c79                                                         -> LiteQuery
      04                                                             -> 陣列大小
      2ee6b589                                                       -> getMasterchainInfo
      000000                                                         -> 3 個填充字節
    000000                                                           -> 3 個填充字節
ac2253594c86bd308ed631d57a63db4ab21279e9382e416128b58ee95897e164     -> sha256
```

我們期望收到的回覆是由 [liteServer.masterchainInfo](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L30) 構成的，其中包含 last:[ton.blockIdExt](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/tonlib_api.tl#L51) state_root_hash:int256 和 init:[tonNode](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/ton_api.tl#L359).zeroStateIdExt。

獲得的封包與發送的封包以相同的方式反序列化 - 使用相同的算法，只是反過來，唯一的區別在於回覆僅被封裝在 [ADNLAnswer](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L23) 內。

После расшифровки ответа, получаем пакет вида:
```
20010000                                                                  -> 封包大小 (288)
5558b3227092e39782bd4ff9ef74bee875ab2b0661cf17efdfcd4da4e53e78e6          -> nonce
  1684ac0f                                                                -> ADNLAnswer
  77c1545b96fa136b8e01cc08338bec47e8a43215492dda6d4d7e286382bb00c4        -> query_id (與請求相同)
    b8                                                                    -> 數組大小
    81288385                                                              -> liteServer.masterchainInfo
                                                                          last:tonNode.blockIdExt
        ffffffff                                                          -> workchain:int
        0000000000000080                                                  -> shard:long
        27405801                                                          -> seqno:int   
        e585a47bd5978f6a4fb2b56aa2082ec9deac33aaae19e78241b97522e1fb43d4  -> root_hash:int256
        876851b60521311853f59c002d46b0bd80054af4bce340787a00bd04e0123517  -> file_hash:int256
      8b4d3b38b06bb484015faf9821c3ba1c609a25b74f30e1e585b8c8e820ef0976    -> state_root_hash:int256
                                                                          init:tonNode.zeroStateIdExt 
        ffffffff                                                          -> workchain:int
        17a3a92992aabea785a7a090985a265cd31f323d849da51239737e321fb05569  -> root_hash:int256      
        5e994fcf4d425c0a6ce6a792594b7173205f740a39cd56f537defd28b48a0f6e  -> file_hash:int256
    000000                                                                -> 3 個填充字節
520c46d1ea4daccdf27ae21750ff4982d59a30672b3ce8674195e8a23e270d21          -> sha256
```

##### runSmcMethod

現在我們已經知道如何獲取主控鏈區塊，所以我們可以調用任何一個 Lite API 的方法。
讓我們來看看 **runSmcMethod**，它是一個調用智能合約函數並返回結果的方法。在這裡，我們需要理解一些新的數據類型，如 [TL-B](/TL-B.md)、[Cell](https://ton.org/docs/learn/overviews/Cells) 和 [BoC](/Cells-BoC.md#bag-of-cells)。

執行智能合約方法，我們需要使用 TL schema 發送請求：
`liteServer.runSmcMethod mode:# id:tonNode.blockIdExt account:liteServer.accountId method_id:long params:bytes = liteServer.RunMethodResult`

然後等待以下格式的回應：
`liteServer.runMethodResult mode:# id:tonNode.blockIdExt shardblk:tonNode.blockIdExt shard_proof:mode.0?bytes proof:mode.0?bytes state_proof:mode.1?bytes init_c7:mode.3?bytes lib_extras:mode.4?bytes exit_code:int result:mode.2?bytes = liteServer.RunMethodResult;`

在請求中，我們看到以下字段：

1. mode:# - uint32 位元遮罩，表示我們要在回應中看到什麼。例如，只有當索引為 2 的位元等於 1 時，result:mode.2?bytes 才會出現在回應中。
2. id:tonNode.blockIdExt - 我們在前一章中收到的主區塊狀態。
3. account:[liteServer.accountId](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L27) - 智能合約地址的 workchain 和資料。
4. method_id:long - 8 個位元組，其中寫入了從被調用方法的名稱計算的 crc16 和設置了第 17 個位元 [計算](https://github.com/xssnick/tonutils-go/blob/88f83bc3554ca78453dd1a42e9e9ea82554e3dd2/ton/runmethod.go#L16)
5. params:bytes - 序列化為 BoC 的 [Stack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L783) сериализованый в [BoC](/Cells-BoC.md#bag-of-cells)，包含呼叫方法的引數。[實作範例](https://github.com/xssnick/tonutils-go/blob/88f83bc3554ca78453dd1a42e9e9ea82554e3dd2/tlb/stack.go)

例如，我們只需要 `result:mode.2?bytes`，那麼我們的 mode 將等於 0b100，即 4。在回應中，我們將收到：

1. mode:# - 我們發送的值 - 4。
2. id:tonNode.blockIdExt - 我們的主區塊，相對於該區塊執行方
3. shardblk:tonNode.blockIdExt - 合約帳戶所在的 shard 區塊
4. exit_code:int - 執行方法時的 4 個字節的退出程式碼。如果一切順利，它將是 0，如果不是，它將等於異常程式碼
5. result:mode.2?bytes - 序列化為 [BoC](/Cells-BoC.md#bag-of-cells) 的 [Stack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L783)，包含方法返回的值。


讓我們看一下從 `EQBL2_3lMiyywU17g-or8N7v9hDmPCpttzBPE2isF2GTzpK4` 合約的 `a2` 方法調用和結果檢索：

FunC 中的方法程式碼：
```
(cell, cell) a2() method_id {
  cell a = begin_cell().store_uint(0xAABBCC8, 32).end_cell();
  cell b = begin_cell().store_uint(0xCCFFCC1, 32).end_cell();
  return (a, b);
}
```

我們填寫我們的請求：
* `mode` = 4, 我們只需要結果  -> `04000000`
* `id` = getMasterchainInfo 執行的結果
* `account` = 工作鏈 0 (4 字節 `00000000`), 以及從我們[合約地址](/Address.md#сериализация)獲取的 int256，即 32 字節 
`4bdbfde5322cb2c14d7b83ea2bf0deeff610e63c2a6db7304f1368ac176193ce`
* `method_id` = 從 `a2` [計算](https://github.com/xssnick/tonutils-go/blob/88f83bc3554ca78453dd1a42e9e9ea82554e3dd2/ton/runmethod.go#L16) 出的 id -> `0a2e010000000000`
* `params：bytes` = 我們的方法不需要任何輸入參數，因此我們需要傳遞一個序列化為 BoC 的空堆疊（`000000`，3 字節單元格 - 深度為 0 的堆疊）-> `b5ee9c72010101010005000006000000`-> 序列化為字節並獲取 `10b5ee9c72410101010005000006000000000000` 0x10 - 大小，末尾 3 個字節 - 填充。

我們得到的回覆如下：

* `mode:#` -> 不重要
* `id:tonNode.blockIdExt` -> 不重要
* `shardblk:tonNode.blockIdExt` -> 不重要
* `exit_code:int` -> 如果一切正常則為 0
* `result:mode.2?bytes` -> 包含以 BoC 格式返回的方法數據的 [Stack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L783)，我們將對其進行解壓縮。

在 `result` 中，我們得到了 `b5ee9c7201010501001b000208000002030102020203030400080ccffcc1000000080aabbcc8`，這是一個包含數據堆棧的 [BoC](/Cells-BoC.md#bag-of-cells)。

當我們對其進行反序列化時，我們將得到一個單元格：
```
32[00000203] -> {
  8[03] -> {
    0[],
    32[0AABBCC8]
  },
  32[0CCFFCC1]
}
```
如果我們對其進行解析，就會得到我們的 FunC 方法返回的 2 個 cell 值。根單元格的前 3 個字節 `000002` 是堆棧深度，即為 2。
這意味著該方法返回了 2 個值。

繼續解析，下一個 8 個位元組（1 個字節）是當前堆棧層級上的值類型。

對於某些類型，它可能佔用 2 個字節。可以在 [模式](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L766) 中查看可能的選項。在這種情況下，我們得到了 `03`，這表示：


```
vm_stk_cell#03 cell:^Cell = VmStackValue;
```
這意味著值的類型為 cell，根據模式，它儲存值本身作為引用。
但是，如果我們查看堆棧元素的儲存模式：
```
vm_stk_cons#_ {n:#} rest:^(VmStackList n) tos:VmStackValue = VmStackList (n + 1);
```

我們可以看到，第一個引用 `rest:^(VmStackList n)` 是堆棧中下一個值的單元格，而我們的值 `tos:VmStackValue` 是第二個值，這意味著要獲取值，我們需要讀取第二個引用，即 `32[0CCFFCC1]`，這是由合約返回的

現在我們可以深入瞭解並獲取堆棧的第二個元素，通過第一個引用繼續，現在我們有：
```
8[03] -> {
    0[],
    32[0AABBCC8]
  }
```

我們重複同樣的過程。前 8 個位元組為 03，即為 cell。第二個引用是值 `32[0AABBCC8]`，由於堆棧深度為 2，我們完成了遍歷。因此，我們得到了合約返回的 2 個值：`32[0CCFFCC1]` 和 `32[0AABBCC8]`。

請注意，它們是以相反的順序出現的。當調用函數時，也必須以與 FunC 程式碼中所見相反的順序傳遞引數。

[實現範例](https://github.com/xssnick/tonutils-go/blob/master/ton/runmethod.go#L24)

##### getAccountState

要獲取帳戶狀態的數據，例如餘額、程式碼和儲存的數據，我們可以使用 [getAccountState](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L68) 方法。我們需要提供一個最新的主區塊和帳戶地址進行請求。

作為回覆，我們將獲得 TL 結構  [AccountState](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/lite_api.tl#L38)。

讓我們看一下 AccountState TL 模式：
```
liteServer.accountState id:tonNode.blockIdExt shardblk:tonNode.blockIdExt shard_proof:bytes proof:bytes state:bytes = liteServer.AccountState;
```
1. `id` - 我們相對於該主區塊獲取數據。
2. `shardblk` - 關於我們獲取數據的工作鏈的區塊，其中包含我們的帳戶。
3. `shard_proof` - 區塊的 Merkle 證明。
4. `proof` - 帳戶狀態的 Merkle 证明。
5. `state` - [BoC](/Cells-BoC.md#bag-of-cells) TLB  [帳戶狀態模式](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L232)。

在所有這些數據中，我們需要的數據位於 `state` 中，讓我們來分析它。

例如，讓我們獲取 TF 帳戶 `EQAhE3sLxHZpsyZ_HecMuwzvXHKLjYx4kEUehhOy2JmCcHCT` 的狀態，回覆中的 state 將是：

```
b5ee9c720102350100051e000277c0021137b0bc47669b3267f1de70cbb0cef5c728b8d8c7890451e8613b2d899827026a886043179d3f6000006e233be8722201d7d239dba7d818134001020114ff00f4a413f4bcf2c80b0d021d0000000105036248628d00000000e003040201cb05060013a03128bb16000000002002012007080043d218d748bc4d4f4ff93481fd41c39945d5587b8e2aa2d8a35eaf99eee92d9ba96004020120090a0201200b0c00432c915453c736b7692b5b4c76f3a90e6aeec7a02de9876c8a5eee589c104723a18020004307776cd691fbe13e891ed6dbd15461c098b1b95c822af605be8dc331e7d45571002000433817dc8de305734b0c8a3ad05264e9765a04a39dbe03dd9973aa612a61f766d7c02000431f8c67147ceba1700d3503e54c0820f965f4f82e5210e9a3224a776c8f3fad1840200201200e0f020148101104daf220c7008e8330db3ce08308d71820f90101d307db3c22c00013a1537178f40e6fa1f29fdb3c541abaf910f2a006f40420f90101d31f5118baf2aad33f705301f00a01c20801830abcb1f26853158040f40e6fa120980ea420c20af2670edff823aa1f5340b9f2615423a3534e2a2d2b2c0202cc12130201201819020120141502016616170003d1840223f2980bc7a0737d0986d9e52ed9e013c7a21c2b2f002d00a908b5d244a824c8b5d2a5c0b5007404fc02ba1b04a0004f085ba44c78081ba44c3800740835d2b0c026b500bc02f21633c5b332781c75c8f20073c5bd0032600201201a1b02012020210115bbed96d5034705520db3c8340201481c1d0201201e1f0173b11d7420c235c6083e404074c1e08075313b50f614c81e3d039be87ca7f5c2ffd78c7e443ca82b807d01085ba4d6dc4cb83e405636cf0069006031003daeda80e800e800fa02017a0211fc8080fc80dd794ff805e47a0000e78b64c00015ae19574100d56676a1ec40020120222302014824250151b7255b678626466a4610081e81cdf431c24d845a4000331a61e62e005ae0261c0b6fee1c0b77746e102d0185b5599b6786abe06fedb1c68a2270081e8f8df4a411c4605a400031c34410021ae424bae064f613990039e2ca840090081e886052261c52261c52265c4036625ccd88302d02012026270203993828290111ac1a6d9e2f81b609402d0015adf94100cc9576a1ec1840010da936cf0557c1602d0015addc2ce0806ab33b50f6200220db3c02f265f8005043714313db3ced542d34000ad3ffd3073004a0db3c2fae5320b0f26212b102a425b3531cb9b0258100e1aa23a028bcb0f269820186a0f8010597021110023e3e308e8d11101fdb3c40d778f44310bd05e254165b5473e7561053dcdb3c54710a547abc2e2f32300020ed44d0d31fd307d307d33ff404f404d10048018e1a30d20001f2a3d307d3075003d70120f90105f90115baf2a45003e06c2170542013000c01c8cbffcb0704d6db3ced54f80f70256e5389beb198106e102d50c75f078f1b30542403504ddb3c5055a046501049103a4b0953b9db3c5054167fe2f800078325a18e2c268040f4966fa52094305303b9de208e1638393908d2000197d3073016f007059130e27f080705926c31e2b3e63006343132330060708e2903d08308d718d307f40430531678f40e6fa1f2a5d70bff544544f910f2a6ae5220b15203bd14a1236ee66c2232007e5230be8e205f03f8009322d74a9802d307d402fb0002e83270c8ca0040148040f44302f0078e1771c8cb0014cb0712cb0758cf0158cf1640138040f44301e201208e8a104510344300db3ced54925f06e234001cc8cb1fcb07cb07cb3ff400f400c9
```

我們來解析這個 [BoC](/Cells-BoC.md#bag-of-cells) 並獲取其中的值。

<details>
  <summary>大單元格</summary>
  
  ```
  473[C0021137B0BC47669B3267F1DE70CBB0CEF5C728B8D8C7890451E8613B2D899827026A886043179D3F6000006E233BE8722201D7D239DBA7D818130_] -> {
    80[FF00F4A413F4BCF2C80B] -> {
      2[0_] -> {
        4[4_] -> {
          8[CC] -> {
            2[0_] -> {
              13[D180],
              141[F2980BC7A0737D0986D9E52ED9E013C7A218] -> {
                40[D3FFD30730],
                48[01C8CBFFCB07]
              }
            },
            6[64] -> {
              178[00A908B5D244A824C8B5D2A5C0B5007404FC02BA1B048_],
              314[085BA44C78081BA44C3800740835D2B0C026B500BC02F21633C5B332781C75C8F20073C5BD00324_]
            }
          },
          2[0_] -> {
            2[0_] -> {
              84[BBED96D5034705520DB3C_] -> {
                112[C8CB1FCB07CB07CB3FF400F400C9]
              },
              4[4_] -> {
                2[0_] -> {
                  241[AEDA80E800E800FA02017A0211FC8080FC80DD794FF805E47A0000E78B648_],
                  81[AE19574100D56676A1EC0_]
                },
                458[B11D7420C235C6083E404074C1E08075313B50F614C81E3D039BE87CA7F5C2FFD78C7E443CA82B807D01085BA4D6DC4CB83E405636CF0069004_] -> {
                  384[708E2903D08308D718D307F40430531678F40E6FA1F2A5D70BFF544544F910F2A6AE5220B15203BD14A1236EE66C2232]
                }
              }
            },
            2[0_] -> {
              2[0_] -> {
                323[B7255B678626466A4610081E81CDF431C24D845A4000331A61E62E005AE0261C0B6FEE1C0B77746E0_] -> {
                  128[ED44D0D31FD307D307D33FF404F404D1]
                },
                531[B5599B6786ABE06FEDB1C68A2270081E8F8DF4A411C4605A400031C34410021AE424BAE064F613990039E2CA840090081E886052261C52261C52265C4036625CCD882_] -> {
                  128[ED44D0D31FD307D307D33FF404F404D1]
                }
              },
              4[4_] -> {
                2[0_] -> {
                  65[AC1A6D9E2F81B6090_] -> {
                    128[ED44D0D31FD307D307D33FF404F404D1]
                  },
                  81[ADF94100CC9576A1EC180_]
                },
                12[993_] -> {
                  50[A936CF0557C14_] -> {
                    128[ED44D0D31FD307D307D33FF404F404D1]
                  },
                  82[ADDC2CE0806AB33B50F60_]
                }
              }
            }
          }
        },
        872[F220C7008E8330DB3CE08308D71820F90101D307DB3C22C00013A1537178F40E6FA1F29FDB3C541ABAF910F2A006F40420F90101D31F5118BAF2AAD33F705301F00A01C20801830ABCB1F26853158040F40E6FA120980EA420C20AF2670EDFF823AA1F5340B9F2615423A3534E] -> {
          128[DB3C02F265F8005043714313DB3CED54] -> {
            128[ED44D0D31FD307D307D33FF404F404D1],
            112[C8CB1FCB07CB07CB3FF400F400C9]
          },
          128[ED44D0D31FD307D307D33FF404F404D1],
          40[D3FFD30730],
          640[DB3C2FAE5320B0F26212B102A425B3531CB9B0258100E1AA23A028BCB0F269820186A0F8010597021110023E3E308E8D11101FDB3C40D778F44310BD05E254165B5473E7561053DCDB3C54710A547ABC] -> {
            288[018E1A30D20001F2A3D307D3075003D70120F90105F90115BAF2A45003E06C2170542013],
            48[01C8CBFFCB07],
            504[5230BE8E205F03F8009322D74A9802D307D402FB0002E83270C8CA0040148040F44302F0078E1771C8CB0014CB0712CB0758CF0158CF1640138040F44301E2],
            856[DB3CED54F80F70256E5389BEB198106E102D50C75F078F1B30542403504DDB3C5055A046501049103A4B0953B9DB3C5054167FE2F800078325A18E2C268040F4966FA52094305303B9DE208E1638393908D2000197D3073016F007059130E27F080705926C31E2B3E63006] -> {
              112[C8CB1FCB07CB07CB3FF400F400C9],
              384[708E2903D08308D718D307F40430531678F40E6FA1F2A5D70BFF544544F910F2A6AE5220B15203BD14A1236EE66C2232],
              504[5230BE8E205F03F8009322D74A9802D307D402FB0002E83270C8CA0040148040F44302F0078E1771C8CB0014CB0712CB0758CF0158CF1640138040F44301E2],
              128[8E8A104510344300DB3CED54925F06E2] -> {
                112[C8CB1FCB07CB07CB3FF400F400C9]
              }
            }
          }
        }
      }
    },
    114[0000000105036248628D00000000C_] -> {
      7[CA] -> {
        2[0_] -> {
          2[0_] -> {
            266[2C915453C736B7692B5B4C76F3A90E6AEEC7A02DE9876C8A5EEE589C104723A1800_],
            266[07776CD691FBE13E891ED6DBD15461C098B1B95C822AF605BE8DC331E7D45571000_]
          },
          2[0_] -> {
            266[3817DC8DE305734B0C8A3AD05264E9765A04A39DBE03DD9973AA612A61F766D7C00_],
            266[1F8C67147CEBA1700D3503E54C0820F965F4F82E5210E9A3224A776C8F3FAD18400_]
          }
        },
        269[D218D748BC4D4F4FF93481FD41C39945D5587B8E2AA2D8A35EAF99EEE92D9BA96000]
      },
      74[A03128BB16000000000_]
    }
  }
  ```

</details>

現在我們需要根據 TL-B 結構解析單元格：
```
account_none$0 = Account;

account$1 addr:MsgAddressInt storage_stat:StorageInfo
          storage:AccountStorage = Account;
```
我們的結構引用了其他結構，例如：
```
anycast_info$_ depth:(#<= 30) { depth >= 1 } rewrite_pfx:(bits depth) = Anycast;
addr_std$10 anycast:(Maybe Anycast) workchain_id:int8 address:bits256  = MsgAddressInt;
addr_var$11 anycast:(Maybe Anycast) addr_len:(## 9) workchain_id:int32 address:(bits addr_len) = MsgAddressInt;
   
storage_info$_ used:StorageUsed last_paid:uint32 due_payment:(Maybe Grams) = StorageInfo;
storage_used$_ cells:(VarUInteger 7) bits:(VarUInteger 7) public_cells:(VarUInteger 7) = StorageUsed;
  
account_storage$_ last_trans_lt:uint64 balance:CurrencyCollection state:AccountState = AccountStorage;

currencies$_ grams:Grams other:ExtraCurrencyCollection = CurrencyCollection;
           
var_uint$_ {n:#} len:(#< n) value:(uint (len * 8)) = VarUInteger n;
var_int$_ {n:#} len:(#< n) value:(int (len * 8)) = VarInteger n;
nanograms$_ amount:(VarUInteger 16) = Grams;  
           
account_uninit$00 = AccountState;
account_active$1 _:StateInit = AccountState;
account_frozen$01 state_hash:bits256 = AccountState;
```

正如我們所看到的，該單元格包含很多數據，但我們將涵蓋主要的情況和獲取餘額，您可以通過類似的方式解析其餘部分。

讓我們開始解析。在根單元格的數據中，我們有：
```
C0021137B0BC47669B3267F1DE70CBB0CEF5C728B8D8C7890451E8613B2D899827026A886043179D3F6000006E233BE8722201D7D239DBA7D818130_
```
讓我們將其轉換為二進制並獲取：
```
11000000000000100001000100110111101100001011110001000111011001101001101100110010011001111111000111011110011100001100101110110000110011101111010111000111001010001011100011011000110001111000100100000100010100011110100001100001001110110010110110001001100110000010011100000010011010101000100001100000010000110001011110011101001111110110000000000000000000000110111000100011001110111110100001110010001000100000000111010111110100100011100111011011101001111101100000011000000100110
```
讓我們看看我們的主要 TL-B 結構，我們看到我們有 2 個選項可以選擇 - `account_none$0` 或 `account$1`。 我們可以通過閱讀 $ 符號後聲明的前綴來了解我們有哪個選項，在我們的例子中它是 1 位。 如果有 0，則我們有 `account_none`，如果有 1，則有 `account`。

上面數據的第一位 = 1，所以我們正在使用 account$1 並將使用該方案：
```
帳戶 $1 地址：MsgAddressInt storage_stat：StorageInfo
           儲存：AccountStorage = 帳戶；
```
接下來我們有 addr:MsgAddressInt ，我們看到對於 MsgAddressInt 我們還有幾個選項：
```
addr_std$10 任播：（可能是任播）workchain_id：int8 地址：bits256 = MsgAddressInt；
addr_var$11 任播：（可能是任播）addr_len：（## 9）workchain_id：int32 地址：（位addr_len）= MsgAddressInt；
```
為了了解使用哪一個，我們像上次一樣讀取前綴位，這次我們讀取 2 位。 

我們切斷了已經讀取的位，"1000000..." 仍然存在，我們讀取前 2 位並得到 "10"，這意味著我們正在使用“addr_std$10”。

接下來我們需要解析`anycast:(Maybe Anycast)`，Maybe 表示應該讀取 1 位，如果有則讀取 Anycast，否則跳過。 

我們剩下的位是 `00000...`，讀取 1 位，它是 0，所以我們跳過 Anycast。

接下來，我們有 `workchain_id: int8`，這裡一切都很簡單，我們讀取 8 位，這將是工作鏈 ID。 我們讀取接下來的 8 位，全為 0，因此工作鍊為 0。

接下來，我們讀取 address:bits256，這是地址的 256 位，與 workchain_id 相同。 閱讀時，我們得到十六進製表示形式的“21137B0BC47669B3267F1DE70CBB0CEF5C728B8D8C7890451E8613B2D8998270”。



我們讀取地址`addr:MsgAddressInt`，然後我們從主結構中得到`storage_stat:StorageInfo`，它的方案是：
```
storage_info$_ used:StorageUsed last_paid:uint32 due_payment:(Maybe Grams) = StorageInfo;
```
首先是 "used:StorageUsed"，其架構為：
```
storage_used$_ cells:(VarUInteger 7) bits:(VarUInteger 7) public_cells:(VarUInteger 7) = StorageUsed;
```
這是用於儲存帳戶數據的單元格和位數。 

每個字段都定義為 "VarUInteger 7"，這意味著一個動態大小的單位，但最多 7 位。 

你可以根據架構來理解它是如何排列的：
```
var_uint$_ {n:#} len:(#< n) value:(uint (len * 8)) = VarUInteger n;
```
在我們的例子中，n 將等於 7。

在 len 中我們將有 `(#< 7)`，這意味著可以容納最多 7 個數字的位數。

您可以通過將 7-1=6 轉換為來確定它二進制形式 - `110`，我們得到 3 位，所以長度 len = 3 位。 值是`(uint (len * 8))`。 

為了確定它，我們需要讀取長度的 3 位，得到一個數字並乘以 8，這將是 value 的大小，即需要讀取的位數才能獲得 VarUInteger 的值.

讀取 `cells:(VarUInteger 7)`，從根單元格中取出我們接下來的位，看接下來的 16 位就明白了，這是 `0010011010101000`。 
我們讀取 len 的前 3 位，這是 `001`，即 1，我們得到大小 (uint (1 * 8))，我們得到 uint 8，我們讀取 8 位，它將是 `cells`，`00110101 `，即十進制形式的 53。 

我們對 `bits` 和 `public_cells` 做同樣的事情。

我們成功讀取了 used:StorageUsed ，接下來我們有 last_paid:uint32 ，這裡一切都很簡單，我們讀取 32 位。 

一切都與 `due_payment:(Maybe Grams)` 一樣簡單，這裡可能為 0，因此我們跳過 Grams。 

但是，如果 maybe 為 1，我們可以查看 Grams `amount:(VarUInteger 16) = Grams` 方案並立即明白我們已經知道如何使用它。 

和上次一樣，只是我們有 16 個而不是 7 個。

接下來我們有一個帶有模式的 `storage:AccountStorage`：
```
account_storage$_ last_trans_lt:uint64 balance:CurrencyCollection state:AccountState = AccountStorage;
```
我們讀到 `last_trans_lt:uint64`，這是 64 位，儲存最後一個賬戶交易的 lt。 最後，由圖表表示的餘額：
```
currencys$_grams:Grams 其他:ExtraCurrencyCollection = CurrencyCollection;
```
從這裡我們將讀取“grams:Grams”，這將是納米色調的賬戶餘額。
`grams:Grams`是`VarUInteger 16`，儲存16（二進制形式`10000`，減1得到`1111`），然後我們讀取前4位，並將結果值乘以8，然後我們讀取接收到的位數，這將是我們的餘額。

讓我們分析一下我們數據的剩餘部分：
```
100000000111010111110100100011100111011011101001111101100000011000000100110
```
我們讀取前4位-`1000`，這是8。8*8=64，我們讀取接下來的64位=`00000111010111111010010001110011101101110100111110110000001100000` 1011011101001`1110000000等於, 531223439883591776`，並從 nano 轉換為 tone，我們得到`531223439.883591776`。

到此為止，因為我們已經分析了所有的主要情況，其餘的可以通過與我們分析的類似的方式獲得。 此外，有關解析 TL-B 的其他訊息可以在 [官方文檔](/TL-B.md) 中找到

##### 其他方法
現在，在研究了所有訊息後，您也可以調用和處理來自其他簡化服務器方法的響應。 同樣的原則:)

## 握手的其他技術細節

#### 獲取 ID 密鑰
<!-- 對應[32 字節] 伺服器金鑰 ID  -->
密鑰 id 是序列化 TL 模式的 SHA256 哈希。

最常用的 TL 方案是：
```
pub.ed25519 key:int256 = PublicKey -- ID c6b41348
pub.aes key:int256 = PublicKey -- ID d4adbc2d
pub.overlay name:bytes = PublicKey -- ID cb45ba34
pub.unenc 數據：字節 = PublicKey -- ID 0a451fb6
pk.aes 密鑰：int256 = 私鑰 -- ID 3751e8a5
```

例如，對於像 ED25519 這樣用於握手的密鑰，密鑰 ID 將是來自
**[0xC6, 0xB4, 0x13, 0x48]** 和 **public key**，(36 byte array, prefix + key)

[程式碼範例](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/crypto.go#L16)

#### 握手包數據加密

握手包以半開放形式發送，僅加密 160 字節，包含永久密碼訊息。

要加密它們，我們需要一個 AES-CTR 密碼，要獲得它，我們需要一個 160 字節的 SHA256 哈希和 [ECDH 共享密鑰](#%D0%BF%D0%BE%D0%BB%D1%83%D1%87%D0%B5%D0%BD%D0%B8%D0%B5-%D0%BE%D0%B1%D1%89%D0%B5%D0%B3%D0%BE-%D0%BA%D0%BB%D1%8E%D1%87%D0%B0-%D0%BF%D0%BE-ecdh)

密碼是這樣構建的：
* key = (0 - 15 bytes of public key) + (16 - 31 bytes of hash)
* iv = (0 - 3 個散列字節) + (20 - 31 個公鑰字節)

密碼組裝好後，我們用它加密我們的 160 個字節。

[程式碼範例](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/connection.go#L361)

#### 通過 ECDH 獲取共享密鑰
要計算共享密鑰，我們需要我們的私鑰和服務器的公鑰。

DH 的本質是在不洩露隱私訊息的情況下獲得一個共享密鑰。 我將以最簡化的形式舉例說明這是如何發生的。

假設我們需要在我們和服務器之間生成一個共享密鑰，過程將是這樣的：
1. 我們生成秘密和公共數字，如 **6** 和 **7**
2、服務器生成密號和公號，如 **5** 和 **15**
3、我們和服務器交換公眾號，發送 **7** 給服務器，它給我們發送**15**。
4. 我們計算：**7^6 mod 15 = 4**
5. 服務器計算：**7^5 mod 15 = 7**
6. 我們交換收到的號碼，我們給服務器 **4**，它給我們 **7**
7. 我們計算 **7^6 mod 15 = 4**
8. 服務器計算：**4^5 mod 15 = 4**
9. 共享密鑰 = **4**

為了簡單起見，將省略 ECDH 本身的細節。 

它是使用 2 個密鑰（私有密鑰和公共密鑰）通過在曲線上找到一個公共點來計算的。 如果有興趣，最好單獨閱讀。

[程式碼範例](https://github.com/xssnick/tonutils-go/blob/2b5e5a0e6ceaf3f28309b0833cb45de81c580acc/liteclient/crypto.go#L32)
