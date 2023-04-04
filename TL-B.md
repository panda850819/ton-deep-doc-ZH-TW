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

以下是使用固定長度鍵序列化 Hashmap 所使用的 TL-B 結構：

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

此 Cell 是大小為 8 位的 `HashmapE`，其值是 `uint16` 數字，即 `HashmapE 8 uint16`。它有 3 個鍵：

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

Нам не известен `n`, но так как он имеет символ `~`, мы можем его вычислить. Для этого загружаем `len:(Unary ~n)`, [[Подробнее про Unary]](#unary).
В нашем случае у нас было `2[00]`, после опеределения типа `HmLabel` у нас остался 1 бит из двух, загружаем его и видим, что это 0, значит наш вариант - это `unary_zero$0`. Получается n из `HmLabel` равен нулю.

Дополним `hml_short` посчитанным значением n:
```
hml_short$0 {m:#} {n:#} len:0 {n <= 8} s:(0 * Bit) = HmLabel 0 8
```
Получается у нас пустой `HmLabel`, s = 0, соответственно загружать нечего.

Возвращаемся еще выше и дополняем нашу структуру посчитанным значением `l`:
```
hm_edge#_ {n:#} {X:Type} {l:0} {m:#} label:(HmLabel 0 8) 
          {8 = (~m) + 0} node:(HashmapNode m uint16) = Hashmap 8 uint16;
```
Теперь, когда мы посчитали `l`, мы можем посчитать и `m`, используя уравнение `n = (~m) + 0`, т.е `m = n - 0`, m = n = 8.

Мы определили все неизвестные и теперь можем загружать `node:(HashmapNode 8 uint16)`. HashmapNode у нас имеет варианты:
```
hmn_leaf#_ {X:Type} value:X = HashmapNode 0 X;
hmn_fork#_ {n:#} {X:Type} left:^(Hashmap n X) 
           right:^(Hashmap n X) = HashmapNode (n + 1) X;
```
В данном случае вариант мы определяем не по префиксу, а по параметру, если n = 0, то у нас `hmn_leaf`, иначе `hmn_fork`. В нашем случае n = 8, у нас `hmn_fork`. Берем его и заполняем известные значения:
```
hmn_fork#_ {n:#} {X:uint16} left:^(Hashmap n uint16) 
           right:^(Hashmap n uint16) = HashmapNode (n + 1) uint16;
```
Здесь все не так очевидно, у нас `HashmapNode (n + 1) uint16`. Это значит,  что результирующий n должен быть равен нашему параметру, т.е 8. Чтобы вычислить локальный n, нужно посчитать его: `n = (n_local + 1)` -> `n_local = (n - 1)` -> `n_local = (8 - 1)` -> `n_local = 7`.
```
hmn_fork#_ {n:#} {X:uint16} left:^(Hashmap 7 uint16) 
           right:^(Hashmap 7 uint16) = HashmapNode (7 + 1) uint16;
```
Теперь все просто, загружаем левую и правую ветку, и для каждой ветки [повторяем процесс](#начинаем-парсинг-веток).

##### Разберем загрузку значений
Продолжая предыдущий пример, разберем загрузку правой ветки, т.е `28[1011100000000000001100001001]` 

Перед нами опять `hm_edge`, заполним его известными значениями:
```
hm_edge#_ {n:#} {X:Type} {l:#} {m:#} label:(HmLabel ~l 7) 
          {7 = (~m) + l} node:(HashmapNode m uint16) = Hashmap 7 uint16;
```

Загружаем `HmLabel`, в этот раз перед нами `hml_long`, так как префикс равен `10`.
```
hml_long$10 {m:#} n:(#<= m) s:(n * Bit) = HmLabel ~n m;
```
Заполним его:
```
hml_long$10 {m:#} n:(#<= 7) s:(n * Bit) = HmLabel ~n 7;
```
Перед нами новая конструкция - `n:(#<= 7)`, она обозначает число такого размера, который способен вместить 7, по сути это log2 от числа + 1. Но для простоты можно посчитать количество бит, которое нужно, чтобы записать число 7. 7 в бинарном виде - это `111`, значит нам нужно 3 бита, `n = 3`. 
```
hml_long$10 {m:#} n:(## 3) s:(n * Bit) = HmLabel ~n 7;
```
Загружаем `n`, получаем `111`, то есть 7 (совпадение). Далее загружаем `s`, 7 бит - `0000000`. `s` - это часть ключа. 

Возвращаемся наверх и заполняем полученный `l`:
```
hm_edge#_ {n:#} {X:Type} {l:#} {m:#} label:(HmLabel 7 7) 
          {7 = (~m) + 7} node:(HashmapNode m uint16) = Hashmap 7 uint16;
```
Вычисляем `m`, `m = 7 - 7`, `m = 0`.

Так как `m = 0`, мы добрались до значения, и в качестве HashmapNode применима структура:
```
hmn_leaf#_ {X:Type} value:X = HashmapNode 0 X;
```
Тут все просто: подставляем наш тип uint16 и загружаем значение. Оставшиеся 16 бит `0000001100001001` в десятичном виде - это 777, наше значение.

Теперь восстановим ключ. Собираем воедино все части ключей, идущие над нами (возвращаемся наверх по тому же пути), и добавляем по 1 биту за каждое ответвление. Если мы шли по правой ветке, то добавляем 1, если по левой - 0. Если над нами был не пустой HmLabel, то добавляем его биты в ключ.

В нашем случае берем 7 бит из HmLabel `0000000` и перед ними добавляем 1, засчет того, что мы добрались до значения по правой ветке. Получаем 8 бит `10000000`, наш ключ - это число `128`.

#### Другие виды Hashmap
Существуют также другие виды словарей, у всех них одинаковая суть и, разобравшись с загрузкой основного Hashmap, вы сможете по тому же принципу загрузить другие виды.

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
Отличие от обычного Hashmap - наличие `extra:Y` поля в каждой ноде (не только в листьях со значениями).

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
Отличие от обычного Hashmap - возможность хранить ключи разной длины, засчет тега у нод `phmn_leaf$0`, `phmn_fork$1`.

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
Отличие от обычного Hashmap - возможность хранить ключи разной длины, засчет тега у нод `vhmn_leaf$00`, `vhmn_fork$01`. При этом `VarHashmap` способен формировать общий префикс значений (дочерний мап), засчет `vhmn_cont$1`.

##### BinTree
```
bta_leaf$0 {X:Type} {Y:Type} extra:Y leaf:X = BinTreeAug X Y;
bta_fork$1 {X:Type} {Y:Type} left:^(BinTreeAug X Y) 
           right:^(BinTreeAug X Y) extra:Y = BinTreeAug X Y;
```
Простое бинарное дерево: принцип формирования ключа, как у Hashmap, но без label'ов, только засчет префиксов веток.

### VmTuple

```
vm_tupref_nil$_ = VmTupleRef 0;
vm_tupref_single$_ entry:^VmStackValue = VmTupleRef 1;
vm_tupref_any$_ {n:#} ref:^(VmTuple (n + 2)) = VmTupleRef (n + 2);
vm_tuple_nil$_ = VmTuple 0;
vm_tuple_tcons$_ {n:#} head:(VmTupleRef n) tail:^VmStackValue = VmTuple (n + 1);
vm_stk_tuple#07 len:(## 16) data:(VmTuple len) = VmStackValue;
```

Tuple является кортежем элементов и в FunC записывается, как `[a, b, c]`, VmTuple используется внутри [VmStack](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L833) для сериализации кортежей, возвращаемых из методов контрактов.

Разберем сериализацию кортежа из 3 элементов, [44, Cell{00B5BD3EB0}, -1]. 
В финальном виде она будет выглядеть так:
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

Разберем ее подробней, корневая ячейка - это `vm_stk_tuple`, так как префикс = 0x07, следующие 2 байта это `len:(## 16)`, которые равны `0003` == 3.

Далее у нас идет `data:(VmTuple 3)`, для VmTuple у нас есть 2 варианта сериализации - `VmTuple (n + 1)` и `VmTuple 0`, так как наше n > 0, мы используем `VmTuple (n + 1)`. 

Первым элементом является `head:(VmTupleRef 2=(3-1))`, для VmTupleRef у нас 3 варианта сериализации: 0, 1 и n+2. Наше n для VmTupleRef равно 2, мы берем третий вариант.

Первый и единственный элемент для VmTupleRef n+2 это `ref:^(VmTuple (n + 2))`, т.е `ref:^(VmTuple 2)`.

Наше n для VmTuple это 2, мы берем вариант (n + 1). Читаем `head:(VmTupleRef n)` -> `head:(VmTupleRef 1=(2-1))`. Для `VmTupleRef 1` у нас только одно поле - `entry:^VmStackValue`, которое по сути и является самим значением - `01 000000000000002C`, 0x01 у стека значит int, и 8 байт самого значения = 44.

Мы прочитали `head` и дошли до конца, теперь идем в обратную сторону и читаем `tail:^VmStackValue` поднимаясь вверх, первый tail у нас `03` со ссылкой на ячейку. 0x03  у стека значит ячейка, мы просто читаем эту ссылку и сохраняем, как значение - `00B5BD3EB0`. Поднимаемся на уровень выше и читаем еще один `tail`, это `01 FFFFFFFFFFFFFFFF`, т.е int равный -1.

После того, как мы дошли до конца, парсинг завершен, складываем все полученные элементы в массив в том порядке, в котором мы их получили.
