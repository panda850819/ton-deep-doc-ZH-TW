# TL

TL (Type Language) - 是一種用於描述數據結構的語言。

在通訊中使用 [TL schemas](https://github.com/ton-blockchain/ton/tree/master/tl/generate/scheme) 來結構化有用的數據。

TL 操作以 32 位元塊為單位。因此，在 TL 中的數據大小必須為 4 的倍數。如果對象大小不是 4 的倍數，則需要添加所需數量的零位元組以實現對齊。

對於編碼數字，總是使用 Little Endian 順序。

有關詳細資訊，可以參閱 [Telegram 文檔](https://core.telegram.org/mtproto/TL)

### 在 TL 中對 bytes 進行編碼

為了編碼位元組陣列，我們需要首先確定其大小。如果它小於 254 個位元組，則使用一個位元組作為大小編碼。如果大於 254 個位元組，則將 0xFE 作為大型數組的指標寫入第一個位元組，然後跟隨著 3 個位元組大小。

例如，我們將 `[0xAA，0xBB]` 這個位元組陣列編碼，其大小為 2。我們使用 1 個位元組大小，然後編碼位元組陣列本身，得到 `[0x02，0xAA，0xBB]`，完成編碼。但是我們可以發現，最終大小為 3，並且不是 4 的倍數，因此，我們需要添加一個填充位元組以實現對齊，得到 `[0x02，0xAA，0xBB，0x00]`。

如果我們需要編碼大小為 396 的位元組陣列，我們可以執行以下操作：396 >= 254，所以我們使用 3 個位元組進行大小編碼和 1 個位元組作為大型數組指標，得到 `[0xFE，0x8C，0x01，0x00，array bytes]`，396+4 = 400，為 4 的倍數，不需要進行對齊。

### 無法明顯看出的序列化規則

在方案之前經常寫上 4 個位元組的前綴 - 它的 ID。方案的 ID 是使用 IEEEE 表格計算方案文字的 CRC32，並從中預先刪除 `;` 和括號 `()` 等字符。具有 ID 前綴的方案序列化稱為 **boxed**，這可以使解析器識別出其前面是哪個方案，如果有多種可能性。

如何確定序列化為 boxed 或非 boxed？如果我們的方案是另一個方案的一部分，則需要查看字段類型的指定方式。如果它是明確指定的，則序列化時不需要前綴；如果不明確指定（有許多此類型），則需要序列化為 boxed。例如：

```
pub.unenc data:bytes = PublicKey;
pub.ed25519 key:int256 = PublicKey;
pub.aes key:int256 = PublicKey;
pub.overlay name:bytes = PublicKey;
```

如果方案中指定了 `PublicKey`，例如 `adnl.node id:PublicKey addr_list:adnl.addressList = adnl.Node`，則未明確指定，因此需要使用 ID 前綴序列化（boxed）。如果方案中明確指定了如下：`adnl.node id:pub.ed25519 addr_list:adnl.addressList = adnl.Node`，那麼前綴將不需要。
