### 地址
在 TON 中，地址是由 StateInit 的 TL-B 結構的 SHA-256 Hash 生成的。

即使合約尚未部署到網路中，也可以計算出該地址。
##### 序列化
標準地址的格式為 `EQBL2_3lMiyywU17g-or8N7v9hDmPCpttzBPE2isF2GTzpK4`，它是經過 base64 uri 編碼的位元組序列。

該序列的長度為 36 個位元組，其中最後兩個位元組是使用 XMODEM 表進行 crc16 校驗和第一個位元組是標誌位（Flag），第二個位元組是工作鏈（WorkChain）。

中間的 32 個位元組是地址本身的資料，通常被表示為 int256。

[程式碼範例](https://github.com/xssnick/tonutils-go/blob/3d9ee052689376061bf7e4a22037ff131183afad/address/addr.go#L156)