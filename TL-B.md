# TL-B

本節中將單獨討論複雜和不明顯的TL-B結構。建議首先閱讀[基礎文檔](https://github.com/tonstack/ton-docs/tree/main/TL-B)。

### Unary
Unary 通常用於定義動態大小的結構，例如 [hml_short](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L29)。


Unary 有兩個變體：
```
unary_zero$0 = Unary ~0;
unary_succ$1 {n:#} x:(Unary ~n) = Unary ~(n + 1);
```

對於 `unary_zero`，非常簡單：如果第一位是0，那麼所尋找的數字就是0。

`unary_succ` 則更有趣，它是遞歸加載，其值為 ~(n + 1)。這意味著它會不斷調用自身，直到遇到 `unary_zero` 為止。換句話說，所尋找的值將等於連續出現的1的數量。

舉個例子，讓我們解析 `1110`。

調用鏈應該是這樣的：
`unary_succ$1 -> unary_succ$1 -> unary_succ$1 -> unary_zero$0`

一旦我們到達 `unary_zero`，返回值就會像在函數的遞歸調用中一樣向上返回。
現在，為了理解結果，我們來看看返回值的路徑，也就是從最後開始：

```0 -> ~(0 + 1) -> ~(1 + 1) -> ~(2 + 1) -> 3```

### Either
```
left$0 {X:Type} {Y:Type} value:X = Either X Y;
right$1 {X:Type} {Y:Type} value:Y = Either X Y;
```
當兩個類型都可能出現時，如果前綴的位是 0，則序列化左側的類型，如果是 1，則序列化右側的類型。

例如，在序列化消息時，當 body 可以是主要單元的一部分或是一個引用時，就可以使用此方法。


### Maybe
```
nothing$0 {X:Type} = Maybe X;
just$1 {X:Type} value:X = Maybe X;
```
用於可選值，如果第一個位元為 0，則該值不會被序列化（跳過），如果為 1，則會被序列化。

### Both
```
pair$_ {X:Type} {Y:Type} first:X second:Y = Both X Y;
```
普通的配對 - 兩個類型都會被序列化，沒有條件限制。

### Hashmap

以 `Hashmap` 為例，這是一個用於存儲智能合約 FunC 代碼中的 `dict` 的結構，可以展示如何處理複雜的 TL-B 結構。

以下是使用固定長度金鑰序列化 Hashmap 所使用的 TL-B 結構：

```
hm_edge#_ {n:#} {X:Type} {l:#} {m:#} label:(HmLabel ~l n) 
          {n = (~m) + l} node:(HashmapNode m X) = Hashmap n X;

hmn_leaf#_ {X:Type} value:X = HashmapNode 0 X;
hmn_fork#_ {n:#} {X:Type} left:^(Hashmap n X) 
           right:^(Hashmap n X) = HashmapNode (n + 1) X;

hml_short$0 {m:#} {n:#} len:(Unary ~n) {n <= m} s:(n * Bit) = HmLabel ~n m;
hml_long$10 {m:#} n:(#<= m) s:(n * Bit) = HmLabel ~n m;
hml_same$11 {m:#} v:Bit n:(#<= m) = HmLabel ~n m;

unary_zero$0 = Unary ~0;
unary_succ$1 {n:#} x:(Unary ~n) = Unary ~(n + 1);

hme_empty$0 {n:#} {X:Type} = HashmapE n X;
hme_root$1 {n:#} {X:Type} root:^(Hashmap n X) = HashmapE n X;
```

其中根結構是 `HashmapE`。它可以有以下兩種狀態之一：`hme_empty` 或 `hme_root`。
#### Hashmap 解析範例

讓我們以以下二進位形式的 Cell 為例進行解析：
```
1[1] -> {
  2[00] -> {
    7[1001000] -> {
      25[1010000010000001100001001],
      25[1010000010000000001101111]
    },
    28[1011100000000000001100001001]
  }
}
```

此 Cell 是大小為 8 位的 `HashmapE`，其值是 `uint16` 數字，即 `HashmapE 8 uint16`。它有 3 個金鑰：

```
1 = 777
17 = 111
128 = 777
```

要解析它，我們需要預先知道要使用哪個結構，`hme_empty` 還是 `hme_root`。我們可以根據前綴來確定，對於 hme_empty，其前綴的第一位為 `0（hme_empty$0）`，對於 hme_root，其前綴的第一位為 1（`hme_root$1`）。我們讀取第一位，它等於 `1（1[1]）`，因此我們面前的是 `hme_root`。


我們使用已知的值填充結構變量，得到：
`hme_root$1 {n:#} {X:Type} root:^(Hashmap 8 uint16) = HashmapE 8 uint16;`

前綴的第一位已經被讀取，括號內的內容是條件，不需要被讀取。 (`{n:#}` 表示 n 是任何 uint32 數字，`{X:Type}` 表示 X 是任何類型。) 我們需要讀取的下一個內容是 `root:^(Hashmap 8 uint16)`，其中 `^` 表示引用，我們加載它。我們得到：

```
2[00] -> {
    7[1001000] -> {
      25[1010000010000001100001001],
      25[1010000010000000001101111]
    },
    28[1011100000000000001100001001]
  }
```

##### 開始解析哈希表的分支

根據我們的 TL-B 結構，這是一個 `Hashmap 8 uint16` 的結構。填寫我們知道的值，我們得到：
```
hm_edge#_ {n:#} {X:Type} {l:#} {m:#} label:(HmLabel ~l 8) 
          {8 = (~m) + l} node:(HashmapNode m uint16) = Hashmap 8 uint16;
```

正如我們所看到的，出現了條件變量 `{l:#}` 和 `{m:#}`，它們的值是我們不知道的。此外，在讀取了 `label` 後，n 在等式 `{n = (~m) + l}` 中參與其中，在這種情況下，我們可以計算出 `l` 和 `m`，這就是 `~` 的含義。

為了確定 `l` 的值，我們需要加載 `label:(HmLabel ~l uint16)`。`HmLabel` 有三種結構：

```
hml_short$0 {m:#} {n:#} len:(Unary ~n) {n <= m} s:(n * Bit) = HmLabel ~n m;
hml_long$10 {m:#} n:(#<= m) s:(n * Bit) = HmLabel ~n m;
hml_same$11 {m:#} v:Bit n:(#<= m) = HmLabel ~n m;
```
結構的選擇基於其前綴。在我們目前的根元素中，有 2 個零位元`（2[00]）`。我們只有一個選擇，該選擇的前綴以 0 開始。因此，我們面前的結構是 `hml_short$0`。

填寫 `hml_short` 中已知的值：
```
hml_short$0 {m:#} {n:#} len:(Unary ~n) {n <= 8} s:(n * Bit) = HmLabel ~n 8
```

我們不知道 `n` 的值，但由於它有符號 `~`，我們可以計算出它。

為此，我們加載 `len:(Unary ~n)`，[關於 Unary 的更多信息](#unary)。

在我們的情況下，我們有 `2[00]`，在確定 `HmLabel` 類型後，我們剩下了兩個位中的 1 位，請加載它並查看其是否為 0，則意味著我們的選項是 `unary_zero$0`。

因此，在 `HmLabel` 中得到 n 等於零。

通過計算得到 `hml_short`：

```
hml_short$0 {m:#} {n:#} len:0 {n <= 8} s:(0 * Bit) = HmLabel 0 8
```
這給了我們一個空白的 `HmLabel` 和 s=0, 因此沒有什麼可加載。

回到上面，並用計算出來的 `l` 補充完整結構：

```
hm_edge#_ {n:#} {X:Type} {l:0} {m:#} label:(HmLabel 0 8) 
          {8 = (~m) + 0} node:(HashmapNode m uint16) = Hashmap 8 uint16;
```

現在當我們計算出 `l` 時，就可以使用方程式 `n=(~ m)+ 0` 來計算出 `m` ，即 `m=n-0` ，所以 m=n=8。

現在已經解決所有未知問題，並且可以加載 `node：(HashmapNode 8 uint16)` 。 HashmapNode 具有以下變體：

```
hmn_leaf#_ {X:Type} value:X = HashmapNode 0 X;
hmn_fork#_ {n:#} {X:Type} left:^(Hashmap n X) 
           right:^(Hashmap n X) = HashmapNode (n + 1) X;
```

在這種情況下，我們根據參數而不是前綴來確定變體。如果 n=0，則為 `hmn_leaf`，否則為 `hmn_fork`。

在我們的情況下，n=8，因此為 `hmn_fork`。

獲取它並填寫已知值：
```
hmn_fork#_ {n:#} {X:uint16} left:^(Hashmap n uint16) 
           right:^(Hashmap n uint16) = HashmapNode (n + 1) uint16;
```
這裡不是很明顯，我們有一個 `HashmapNode (n+1)uint16`。

這意味著結果中的 n 必須等於我們的參數即 8. 要計算本地 n ，需要計算它：`n=(local_n+1)` -> `local_n=(n-1)` -> `local_n=(8-1)`-> `local_n=7`。

```
hmn_fork#_ {n:#} {X:uint16} left:^(Hashmap 7 uint16) 
           right:^(Hashmap 7 uint16) = HashmapNode (7 + 1) uint16;
```

現在一切都很簡單，我們加載左右分支，對於每個分支重複這個過程。

##### 讓我們來看看值的加載
繼續上一個例子，讓我們來看看右側分支的加載，即 `28[1011100000000000001100001001]` 

再次出現了 `hm_edge`，在其中填入已知的值：

```
hm_edge#_ {n:#} {X:Type} {l:#} {m:#} label:(HmLabel ~l 7) 
          {7 = (~m) + l} node:(HashmapNode m uint16) = Hashmap 7 uint16;
```

加載 `HmLabel`，在這種情況下是 `hml_long`，因為前綴等於 `10`。

```
hml_long$10 {m:#} n:(#<= m) s:(n * Bit) = HmLabel ~n m;
```
填寫它：
```
hml_long$10 {m:#} n:(#<= 7) s:(n * Bit) = HmLabel ~n 7;
```

新構造 - `n:(#<= 7)`，表示可以容納數字 7 大小的數字。
實際上這是 log2 數量 +1。
但是為了簡單起見，請計算需要寫入數字 "7" 的位數。
二進制中的７為 `111`，因此我們需要 3 個位元組，`n = 3`。

```
hml_long$10 {m:#} n:(## 3) s:(n * Bit) = HmLabel ~n 7;
```

加載 `n`，得到 `111`，即 7 (匹配)。 
然後加載 `s`，7 位 - `0000000`。 `s` 是密鑰的一部分。

返回頂部並填寫所獲取的 `l`：

```
hm_edge#_ {n:#} {X:Type} {l:#} {m:#} label:(HmLabel 7 7) 
          {7 = (~m) + 7} node:(HashmapNode m uint16) = Hashmap 7 uint16;
```

計算`m`, `m = 7 - 7`, `m = 0`.

由於 `m = 0`，我們已經到達值，並且可以使用以下結構作為 HashmapNode：
```
hmn_leaf#_ {X:Type} value:X = HashmapNode 0 X;
```

這很簡單：將我們的 uint16 類型代入並加載值。
剩下的十六位數字 `0000001100001001` 以十進制表示為 777，它是我們要找的值。

現在恢復密鑰。將所有在上面走過我們頭頂上方 (沿著相同路徑向上移動)，並添加每個分支一個比特位數來組合金鑰。如果我們按右側分支行進，則添加 1；如果按左側行進，則添加 0。如果在我們上面的位置不是空的 HmLabel，則將其位添加到密鑰中。

在我們的例子中，從 HmLabel 中取出 7 個位元 `0000000`，並在它們之前添加 1，因為我們通過右側分支找到了值。
我們得到 8 個位元 `10000000`，這是我們的金鑰 - 數字 `128`。

#### 其他類型的 Hashmap
還有其他類型的字典，它們都具有相同的本質，在理解主要 Hashmap 加載後，您可以按照相同原則加載其他類型。

##### HashmapAugE
```
ahm_edge#_ {n:#} {X:Type} {Y:Type} {l:#} {m:#} 
  label:(HmLabel ~l n) {n = (~m) + l} 
  node:(HashmapAugNode m X Y) = HashmapAug n X Y;
  
ahmn_leaf#_ {X:Type} {Y:Type} extra:Y value:X = HashmapAugNode 0 X Y;

ahmn_fork#_ {n:#} {X:Type} {Y:Type} left:^(HashmapAug n X Y)
  right:^(HashmapAug n X Y) extra:Y = HashmapAugNode (n + 1) X Y;

ahme_empty$0 {n:#} {X:Type} {Y:Type} extra:Y 
          = HashmapAugE n X Y;
          
ahme_root$1 {n:#} {X:Type} {Y:Type} root:^(HashmapAug n X Y) 
  extra:Y = HashmapAugE n X Y;
```

與通常的 Hashmap 的不同之處在於每個節點中存在一個 `extra:Y` 字段（不僅在具有值的葉子中）。

##### PfxHashmap
```
phm_edge#_ {n:#} {X:Type} {l:#} {m:#} label:(HmLabel ~l n) 
           {n = (~m) + l} node:(PfxHashmapNode m X) 
           = PfxHashmap n X;

phmn_leaf$0 {n:#} {X:Type} value:X = PfxHashmapNode n X;
phmn_fork$1 {n:#} {X:Type} left:^(PfxHashmap n X) 
            right:^(PfxHashmap n X) = PfxHashmapNode (n + 1) X;

phme_empty$0 {n:#} {X:Type} = PfxHashmapE n X;
phme_root$1 {n:#} {X:Type} root:^(PfxHashmap n X) 
            = PfxHashmapE n X;
```
由於 `phmn_leaf$0`, `phmn_fork$1` 節點的標記，與通常的 Hashmap 的不同之處在於能夠儲存不同長度的金鑰。

##### VarHashmap
```
vhm_edge#_ {n:#} {X:Type} {l:#} {m:#} label:(HmLabel ~l n) 
           {n = (~m) + l} node:(VarHashmapNode m X) 
           = VarHashmap n X;
vhmn_leaf$00 {n:#} {X:Type} value:X = VarHashmapNode n X;
vhmn_fork$01 {n:#} {X:Type} left:^(VarHashmap n X) 
             right:^(VarHashmap n X) value:(Maybe X) 
             = VarHashmapNode (n + 1) X;
vhmn_cont$1 {n:#} {X:Type} branch:Bit child:^(VarHashmap n X) 
            value:X = VarHashmapNode (n + 1) X;

// nothing$0 {X:Type} = Maybe X;
// just$1 {X:Type} value:X = Maybe X;

vhme_empty$0 {n:#} {X:Type} = VarHashmapE n X;
vhme_root$1 {n:#} {X:Type} root:^(VarHashmap n X) 
            = VarHashmapE n X;
```
由於 `vhmn_leaf$00`、`vhmn_fork$01` 節點的標記，與通常的 Hashmap 的不同之處在於能夠儲存不同長度的金鑰。 

同時，`VarHashmap` 能夠形成一個公共值前綴（子映射），代價是`vhmn_cont$1`。

##### BinTree
```
bta_leaf$0 {X:Type} {Y:Type} extra:Y leaf:X = BinTreeAug X Y;
bta_fork$1 {X:Type} {Y:Type} left:^(BinTreeAug X Y) 
           right:^(BinTreeAug X Y) extra:Y = BinTreeAug X Y;
```
一棵簡單的二元樹：金鑰生成的原理，和 Hashmap 一樣，但是沒有標籤，只統計分支前綴。

### VmTuple

```
vm_tupref_nil$_ = VmTupleRef 0;
vm_tupref_single$_ entry:^VmStackValue = VmTupleRef 1;
vm_tupref_any$_ {n:#} ref:^(VmTuple (n + 2)) = VmTupleRef (n + 2);
vm_tuple_nil$_ = VmTuple 0;
vm_tuple_tcons$_ {n:#} head:(VmTupleRef n) tail:^VmStackValue = VmTuple (n + 1);
vm_stk_tuple#07 len:(## 16) data:(VmTuple len) = VmStackValue;
```

Tuple 是元素的元組，在 FunC 中寫成 `[a, b, c]`，VmTuple 在[VmStack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L833) 序列化從合約方法返回的元組。

讓我們看一下 3 元組 [44, Cell{00B5BD3EB0}, -1] 的序列化。

在其最終形式中，它將如下所示：

```
24[070003] -> {
   0[] -> {
      72[01000000000000002C],
      8[03] -> {
         40[00B5BD3EB0]
      }
   },
   2[01FFFFFFFFFFFFFFFF]
}
```

讓我們更詳細地解釋一下，根單元是 `vm_stk_tuple`，因為前綴 = 0x07，接下來的 2 個位元組是 `len:(## 16)`，它們等於 `0003` == 3。

然後我們有 `data:(VmTuple 3)`，對於 VmTuple，我們有兩種序列化選項- `VmTuple（n+1）`和 `VmTuple 0`，因為我們的 n> 0，所以我們使用了 `VmTuple(n+1)`。

第一個元素是 `head:(VmTupleRef 2 =(3-1))`，對於 VmTupleRef，我們有三種序列化選項：0、1 和 n+2。 

VmTupleRef 的 n 為 2，則選擇第三個選項。

對於 VmTupleRef n+2 的唯一元素是 `ref:^(VmTuple(n +2))`，即 `ref:^(VMTuple 2)`。

對於 VMtuple 而言, 我們的 N 值為 2, 因此選擇 (n+1) 變量. 閱讀 head: (VMtuple Ref n)-> head:(VMtuple Ref(1=(2-1)). 

對於 VMtuple Ref(1)，我們只有一個字段 - `entry:^VmStackValue`，它實際上就是該值本身 -` 01 000000000000002C`，棧中的 OX01 表示 int，並且 8 個位元組表示值 = 44.

現在已經讀取了 `head` 並到達末尾，請向上移動並閱讀 `tail:^VmStackValue`。第一個 tail 是 `03`，並帶有引用到單元格。

棧中的 0x03 表示單元格，我們只需讀取此引用並將其保存為值 - `00B5BD3EB0`。

向上移動一個級別並再次閱讀 `tail`，這是 `01 FFFFFFFFFFFFFFFF`，即等於 -1 的 int。

完成解析後，請按照獲得它們的順序將所有獲取到的元素放入數組中。