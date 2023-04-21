## Cell 單元格

Cell 是一個包含資料的容器，最多可儲存 1023 個位元組並具有最多 4 個指向其他 Cell 的連結。
在 TON 中，所有東西都是由 Cell 構成，包括合約程式碼、儲存的資料、區塊等等。

透過這樣的方法達到通用性。

## Bag of Cells
Bag of Cells 是一種將 Cell 序列化為位元組數組的格式，並以 TL-B 方式描述。
## Cell 的序列化

讓我們來看一下下面的 Cell：

```
1[8_] -> {
  24[0AAAAA],
  7[FE] -> {
    24[0AAAAA]
  }
}
``` 

這裡我們有一個 1 個位元組大小的根 Cell，它有 2 個連結：第一個是 24 個位元組大小的 Cell，第二個是 7 個位元組大小的 Cell，它有一個連結指向一個大小為 24 個位元組的 Cell。

我們需要將 Cell 序列化成位元組，首先我們需要只保留獨特的 Cell，它們中有 3 個是唯一的，而第四個是重複的。

我們得到：
```
1[8_]
24[0AAAAA]
7[FE]
```
現在我們按照這樣的順序排列它們，以便父 Cell 不會指向後面的 Cell。
被其他 Cell 指向的 Cell 應該在指向它們的 Cell 後面。

我們得到：

```
1[8_]      -> 索引 0 (根單元格)
7[FE]      -> 索引 1
24[0AAAAA] -> 索引 2
```

對每個 cell 計算其描述符號。

描述符號由 2 個位元組組成，儲存有關資料長度和引用數量的符號。

在當前分析中，標誌將被省略，它們幾乎總是為 0。第一個字節包含 5 位符號和 3 位引用數量。

2 個位元組是完整的 4 位元組的長度（但如果不為空，則至少為 1）。

得到：

```
1[8_]      -> 0201 -> 2 個引用, 長度為 1 
7[FE]      -> 0101 -> 1 個引用, 長度為 1
24[0AAAAA] -> 0006 -> 0 個引用, 長度為 6
```
對於具有不完整的 4 位元組的資料，會在結尾添加 1 位，表示位元組的結束位，用於確定不完整位元組的實際大小。

添加它：
```
1[8_]      -> C0     -> 0b10000000->0b11000000
7[FE]      -> FF     -> 0b11111110->0b11111111
24[0AAAAA] -> 0AAAAA -> 不變（完整的組）
```

現在添加引用 Index：
```
0 1[8_]      -> 0201 -> 引用索引為 2 的兩個 cell
1 7[FE]      -> 02 -> 引用索引為 2 的 cell
2 24[0AAAAA] -> 沒有引用
```

把它們組合起來：
```
0201 C0     0201  
0101 AA     02
0006 0AAAAA 
```

並將它們拼湊成位元組：`0201c002010101ff0200060aaaaa`，大小為 14 個位元組。

[序列化的範例](https://github.com/xssnick/tonutils-go/blob/3d9ee052689376061bf7e4a22037ff131183afad/tvm/cell/serialize.go#L205)

#### 打包成 BoC
讓我們把上一章的 cell 打包成 BoC。
我們已經把它序列化成了一個長度為 14 的位元組。

根據[方案](https://github.com/ton-blockchain/ton/blob/24dc184a2ea67f9c47042b4104bbb4d82289fac1/crypto/tl/boc.tlb#L25)構建檔頭。

```
b5ee9c72                      -> BoC 結構的 TL-B ID
01                            -> 標誌和大小 size:(## 3)，在我們的情況下標誌全部為 0，需要儲存 Cell 數量的位元組 -1。
                                 得到 -0b0_0_0_00_001
01                            -> 儲存序列化 cell 大小所需的位元組數
03                            -> cell 數量，1 個位元組（由標題的 size:(## 3) 確定），為 3。
01                            -> 根Cell（Root cell）數量 -1
00                            -> absent，總是為 0（在當前實現中）
0e                            -> 序列化 cell 大小，1 個位元組（大小由上面定義），為 14。
00                            -> 根Cell索引（Root Cell Index），大小為 1（由標題的 size:(## 3) 確定），總是為 0
0201c002010101ff0200060aaaaa  -> 序列化的 Cell
```

將上述內容結合為字節串，得到最終的 BoC：
`b5ee9c7201010301000e000201c002010101ff0200060aaaaa`

BoC 的實現範例：[序列化](https://github.com/xssnick/tonutils-go/blob/master/tvm/cell/serialize.go)， [反序列化](https://github.com/xssnick/tonutils-go/blob/master/tvm/cell/parse.go)

## 特殊的 Cell

Cell 分為兩種類型：普通（Ordinary）和特殊（Special）。

大多數用戶操作的 Cell 都是普通的，它們只是包含訊息的容器。

但是，對於網路內部功能，經常使用特殊 Cell，其處理方式與特殊 Cell 的子類型有關。

特殊 Cell 的類型由其資料的前 8 位確定，可以是以下之一：

```
0x01: PrunnedBranch
0x02: Library
0x03: MerkleProof
0x04: MerkleUpdate
```
在接下來的部分中，我們將更詳細地討論特殊類型。

### Merkle Proof

此類 Cell 是 Hash Tree，並包含了證明 Cell 資料部分屬於完整樹的證明，同時驗證方可能不知道完整樹的內容，只知道其 Hash 值。

該 Cell 本身包含對象的 Root Hash，以及以子 Cell 形式表示的其分支的 Hash 樹。

內部 Tree 按照原始對象的結構進行重複，而不是資料的證明，所有不屬於需要證明的資料部分的分支，以及不通往它的分支，均被替換為特殊 Cell 類型 `PrunnedBranch`，它們儲存其 Hash 值而不是資料。

因此，計算整棵樹的 Root Hash ，我們應該得到與 Merkle 證明類型的 Cell 資料中預先知道的 Hash 值相同的 Hash 。

例如，在我們向某人證明區塊中有一筆交易時，只要該人知道該區塊的 Hash 值，就可以使用這種方法進行證明。

##### getAccountState 的證明

這個對象是 [ShardStateUnsplit](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb#L399) 結構的證明。

但是，它只包含我們所要求的帳戶 - `EQCVRJ-RqeZWcDqgTzzcxUIrChFYs0SyKGUvye9kGOuEWndQ`。

所有與它不相關的和不導致它的分支都被替換為 `Pruned`，而指向它的分支是普通的 `Ordinary` 單元，包含區塊的真實資料。

用符號 * 標記了特殊的單元，根單元類型為 0x03 - MerkleProof，其他單元類型為 0x01 - Pruned。

<details>
  <summary><b>展開範例</b></summary>
  
```
280[03E42E9D59FE3B0900D185EA76AA5F64C6FA0F0C723FFD3CCA5F22B21354064C540219]* -> {
  362[9023AFE2FFFFFF110000000000000000000000000001E9AA7F0000000163C1322C00001F522060E3C10194CD420_] -> {
    288[0101CA91D55F8B903536211B4FB74EA37FD7930C3774D76FE64B45BCB1FE0E1E26380002]*,
    75[8209C398C55C7BDDF32_] -> {
      76[0104E1CC62AE3DEEF99_] -> {
        288[01011D8C7CB5BFB085D47FB85883C71A7A1DC5B4856A2B9C27DE05F63E994D4A557A0216]*,
        76[0101A1AB8F37E15D045_] -> {
          76[01010A762B5E3DB9BB3_] -> {
            76[01007896888C1BF9AFF_] -> {
              288[0101658DA3EABE6B20FA399239F51BF78EB7DB0D8AAE0572E682B986268E041F71590036]*,
              68[00F45169A7CA17BB3_] -> {
                68[00E1E486CCD6F2D24_] -> {
                  288[0101F2B70E32B323C3E1817C1780D7A2A49D08BA63B9B9D024B3C713AB936ECA3FD40028]*,
                  68[00E17E05DDDEF5882_] -> {
                    68[00E0EEBE3D59F5624_] -> {
                      288[0101447569C499386266F623BB34E72EF718158ACC2E76ED42F88C6D842D503CDF900020]*,
                      68[00E0B73C712EA7A3C_] -> {
                        68[00E026EB80E87B804_] -> {
                          288[01015DB34588AAB7AF2A5034E0C26168E51FAED17544C56405B0F80CD967A4EE5BD9001B]*,
                          68[00E023322BD069152_] -> {
                            68[00E0220B791895CE8_] -> {
                              68[00E0208438C7659B4_] -> {
                                68[00E0201CBDC2FA57E_] -> {
                                  288[010132D3C686F68E1E93D14D3DB79EE7D88FB582C87B35BB5B553D82AA25D09196130014]*,
                                  60[00C047CA5AA50C4_] -> {
                                    52[00BD53FE01932_] -> {
                                      52[00A0AC70E85E2_] -> {
                                        288[010178A421C00E63D315BAEB8086668F7D45632ACD4EF17B8A33884C1499856572E2000F]*,
                                        44[00894655318_] -> {
                                          44[0087FCFA2A6_] -> {
                                            44[00805E35524_] -> {
                                              288[010147E47997E4E71FC094644C589682DE513FB731A6B8E841EB2AB68DF4EF1BD92B0002]*,
                                              53[E0A04027728E60] -> {
                                                570[B991A9E656703AA04F3CDCC5422B0A1158B344B228652FC9EF6418EB845A001C09B2869397E6EAB3E5EC55401625FB84ED36424BB6EB0F344BC13953F74CC340000716D4326F30C_] -> {
                                                  288[0101BC95189D8B489591E527DFE6FC8787A3A2D382C1DCB6AA9153D7A538F9FFA3B90007]*
                                                },
                                                288[0101D3016CE07C229433613139886E3B2578C61C374FB3AB0C1889666D8588B5B0090008]*
                                              }
                                            },
                                            288[0101641854A854E746372DB4110640D8AA4AFFDCA2CE5F110980D7EB7CF4159652A7000B]*
                                          },
                                          288[01014C4C14F8F5E62FE6145EFBB10255D3F82082E492D8A8E6F17AC610C70A5B06A8000C]*
                                        }
                                      },
                                      288[0101DBC646BFEA5C4D04E6D76C3DCD521BECDD2062F493B9727D3466E0F5272F2FC90010]*
                                    },
                                    288[01017B7679129CCE779E8ADA70B51D8FD2A5B1FD8180E98FE35944399D3E583E49140013]*
                                  }
                                },
                                288[0101853D4CD469D08FBE03DA3DDA48568763B19BC5A711BB7887E9D7700D24314275001C]*
                              },
                              288[010156C1849FDD43EFEAD3B9FCFB246A5FC40D0D9AA4F4B0C877A9AEB5C4AFFF84850018]*
                            },
                            288[010157B0C46BA1C3726890924C2D7F3F4B755BF7B24363CC325B37C9F956DF569AD4001A]*
                          }
                        },
                        288[01014FD09DCDD654FD4C3758B75C755893092B5D7845356647C3CB5028AE4CF75E26001C]*
                      }
                    },
                    288[0101D3E6EF78A7FB3E14C066534F5172078FCC4382D2AD1512470C25E1504CB3BFE50025]*
                  }
                },
                288[0101CDF101C69F77450BAC9532A70318AB7F0DD902A4A3EE2CE8CDF5262547B5CB68002A]*,
                288[0101AAED7CCC3904836F362AE06EB234B71D64E02EB4BA6D6B7869197A9ED5C4B0B80001]*
              },
              288[0101AAED7CCC3904836F362AE06EB234B71D64E02EB4BA6D6B7869197A9ED5C4B0B80001]*
            },
            288[0101EE15491F5B3FFB71A0C759A476F99A84D7A1B5CF8A47B1DE5D1039882CD723710065]*,
            288[0101EC90A44EEE02BED840C10E88351163EE9E3613EB9DBE8DA760783DA449714E280001]*
          },
          288[01011AAC6151A41BCC89EEE855D29D087ACDD30BC4F2069B246BE0C69E3C228C55110067]*,
          288[0101BB06F3506745C5F6A6239D132A70B38439CB60FF95F62E45261BA12E844E889B0001]*
        },
        288[0101B3E9649D10CCB379368E81A3A7E8E49C8EB53F6ACC69B0BA2FFA80082F70EE390001]*
      },
      288[0101B3E9649D10CCB379368E81A3A7E8E49C8EB53F6ACC69B0BA2FFA80082F70EE390001]*
    },
    868[0000000000000000FFFFFFFFFFFFFFFF8270E631571EF77CCBBA0EF6D39D6343100001F5220425F440194CD434074F1CF0713270443B84FEA5C35142771F8C6895A2B36117743862354F4EBE767D25641E7D9D005816619EB807E1C18C943B86598038ADD33B65980DCB14EF2_] -> {
      288[0101B3E9649D10CCB379368E81A3A7E8E49C8EB53F6ACC69B0BA2FFA80082F70EE390001]*
    }
  }
}
```
  
</details>

