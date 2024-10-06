# 基本資料類型

## bool

布林類型（bool）代表的是“是”和“否”的二值邏輯。它有兩個值：`true`和`false`。一般用在邏輯運算式中，可以執行AND、OR、NOT等運算。

#### 佔用一位元組

```rust
fn main() {
    // bool 佔用1 byte的空間
    println!("size of bool {}", std::mem::size_of::<bool>()); // 1
}
```

#### 基本計算

```rust
fn main() {
    let x = true;
    let y: bool = !x; // not運算
    
    // false, logical and,帶短路功能
    let z = x && y; 
    println!("{z}");
    
    // true, logical or,帶短路功能
    let z = x || y; 
    println!("{z}");
    
    // false, bitwise and,不帶短路功能
    let z = x & y; 
    println!("{z}");
    
    // true, bitwise or,不帶短路功能
    let z = x | y; 
    println!("{z}");
    
    // true, bitwise xor,不帶短路功能
    let z = x ^ y; 
    println!("{z}");

    // 一些比較運算運算式的類型就是bool類型
    let z: bool = 2 < 3;
    println!("{z}");    // true
}
```

```rust
fn main() {
    // 整數相加
    println!("1 + 2 = {}", 1u32 + 2);

    // 整數相减
    println!("1 - 2 = {}", 1i32 - 2);

    // 短路求值
    println!("true AND false is {}", true && false); // false
    println!("true OR false is {}", true || false); // true
    println!("NOT true is {}", !true); // false

    // 位元計算
    println!("0011 AND 0101 is {:04b}", 0b0011u32 & 0b0101); // 0001
    println!("0011 OR 0101 is {:04b}", 0b0011u32 | 0b0101);  // 01111
    println!("0011 XOR 0101 is {:04b}", 0b0011u32 ^ 0b0101); // 0110
    println!("1 << 5 is {}", 1u32 << 5); // 32
    println!("0x80 >> 2 is 0x{:x}", 0x80u32 >> 2); //0x20

    // 使用下劃線改善可讀性
    println!("One million is written as {}", 1_000_000u32);
}
```

#### 條件判斷式

bool類型運算式可以用在if/while等運算式中，作為條件判斷式。

```rust
if a >= b {
  ...
} else {
  ...
}
```

## char

字元類型由char表示(<mark style="color:red;">固定為4 bytes</mark>)。它可以描述任何一個符合unicode標準的字元值。在程式碼中，單個的字元字面量用單引號包圍。

[std::mem::size\_of\_val](https://doc.rust-lang.org/std/mem/fn.size\_of\_val.html)可以查看傳入指標的佔用的記憶體空間位元數。

```rust
use std::mem;
fn main() {
    //char以單引號包住, 可以直接嵌入任何 unicode 字符
    //char佔用4 bytes的空間
    let love: char = '愛';
    println!("love={}, size={}", love, mem::size_of_val(&love));

    let c1: char = '\n'; // 分行符號, 佔用4bytes
    println!("{}, size={}", c1, mem::size_of_val(&c1));

    let c2 = '\x7f'; // 8 bit 字元變數, 佔用4bytes
    println!("{}, size={}", c2, mem::size_of_val(&c2));

    let c3 = '\u{7FFF}'; // unicode字元, 佔用4bytes
    println!("{}, size={}", c3, mem::size_of_val(&c3));

    /* 可以使用一個字母b在字元或者字串前面，
     * 代表這個字面量存儲在u8類型陣列中，
     * 這樣佔用空間比char型陣列要小一些。
     */
    let x: u8 = 1;
    println!("{}, size={}", x, mem::size_of_val(&x)); // 1 byte

    let y: u8 = b'A';
    println!("{}, size={}", y, mem::size_of_val(&y)); // 1 byte

    let s: &[u8; 5] = b"hello";
    println!("{:?}, size={}", s, mem::size_of_val(&s)); // 8 bytes

    let r: &[u8; 14] = br#"hello \n world"#;
    println!("{:?}, size={}", r, mem::size_of_val(&r)); // 8 bytes
}

```

## 整數類型

各種整數類型之間的主要區分特徵是：有符號/無符號與佔據空間大小，<mark style="color:red;">未指明整數類型時預設為i32</mark>。

| 整數類型         | 有符號   | 無符號   |
| ------------ | ----- | ----- |
| 8-bit        | i8    | u8    |
| 16- bit      | i16   | u16   |
| 32-bit       | i32   | u32   |
| 64-bit       | i64   | u64   |
| 128-bit      | i128  | u128  |
| pointer size | isize | usize |

關於各個整數類型所佔據的空間大小，在名字中就已經表現得很明確了，Rust原生支持了從8位元到128位元的整數。

**需要特別關注的是isize和usize類型。它們佔據的空間是不定的，與指標佔據的空間一致**，與所在的平臺相關。如果是32位元系統上，則是32位大小；如果是64位元系統上，則是64位大小。在C++中與它們相對應的類似類型是int\_ptr和uint\_ptr。

Rust的這一策略與C語言不同，C語言標準中對許多類型的大小並沒有做強制規定，比如int、long、double等類型，在不同平臺上都可能是不同的大小，這給許多程式師帶來了不必要的麻煩。相反，在語言標準中規定好各個類型的大小，讓編譯器針對不同平臺做適配，生成不同的程式碼，是更合理的選擇。

```rust
fn main() {
    let var1: i32 = 32;     // 十進位表示
    let var2: i32 = 0xFF;   // 以0x開頭代表十六進位表示
    let var3: i32 = 0o55;   // 以0o開頭代表八進制表示
    let var4: i32 = 0b1001; // 以0b開頭代表二進位表示
    // 32, 255, 45, 9
    println!("{}, {}, {}, {}", var1, var2, var3, var4);
    
    // 使用底線分割數字,不影響語義,但是極大地提升了閱讀體驗。
    let var5 = 0x_1234_ABCD; 
    println!("{}", var5); // 305441741
    
    // 純量後面可以跟尾碼，可代表該數字的具體類型，省略掉顯示類型標記
    let var6 = 123usize; // i6變數是usize類型
    let var7 = 0x_ff_u8; // i7變數是u8類型
    let var8 = 32;       // 不寫類型,預設為 i32 類型
    let an_integer   = 5i32; // 後缀說明
    println!("{}, {}, {}, {}", var6, var7, var8, an_integer);
}
```

在Rust中，我們可以為任何一個類型增加(實作)方法，整數也不例外。如std中對所有整數類型都有實作[pow](https://doc.rust-lang.org/std/primitive.i32.html#method.pow)方法。

```rust
fn main() {
    let x: i32 = 9;
    println!("9 power 3 = {}", x.pow(3));
    
    // 可以不使用變數，直接對整數調用函數
    println!("9 power 3 = {}", 9_i32.pow(3));
}
```

### 整數溢位

在C語言中，對於無符號類型，算數運算永遠不會overflow，如果超過表示範圍，則自動捨棄高位資料。對於有符號類型，如果發生了overflow，標準規定這是undefined behavior，也就是說隨便怎麼處理都可以。

未定義行為有利於編譯器做一些更激進的性能優化，但是這樣的規定有可能導致在某些極端場景下，產生詭異的bug。

Rust在這個問題上選擇的處理方式為：

* <mark style="color:red;">在debug模式下編譯器會自動插入整數溢出檢查，一旦發生溢出，則會引發panic</mark>；
* <mark style="color:red;">在release模式下，不檢查整數溢出，而是採用自動捨棄高位的方式</mark>。

```rust
fn arithmetic(m: i8, n: i8) {
    // 加法運算,有溢出風險, i8最大值為127
    println!("{}", m + n);
}
fn main() {
    let m: i8 = 120;
    let n: i8 = 120;
    arithmetic(m, n);
}
// debug模式會產生panic!
// release模式會使用自動截斷，輸出-16
```

Rust編譯器還提供了一個獨立的編譯開關供我們使用，通過這個開關，可以設置溢出時的處理策略。

```bash
$ rustc -C overflow-checks=no test.rs
```

如果在某些場景下，使用者確實需要更精細地自主控制整數溢出的行為，可以調用標準庫中的[checked\_\*](https://doc.rust-lang.org/std/primitive.i32.html#method.checked\_add)、[saturating\_\*](https://doc.rust-lang.org/std/primitive.i32.html#method.saturating\_add)和wrapping\_\*系列函數。

```rust
fn main() {
    let i = 100_i8;
    // checked_add在overflow時傳回None
    println!("checked {:?}", i.checked_add(i));
    // saturating_add在加法後若overflow會傳回類型最大值
    println!("saturating {:?}", i.saturating_add(i));
    //  直接拋棄已經溢出的最高位元，將剩下的部分返回。
    println!("wrapping {:?}", i.wrapping_add(i));
}
// checked None
// saturating 127
// wrapping -56
```

標準庫還提供了一個叫作[`std::num::Wrapping<T>`](https://doc.rust-lang.org/std/num/struct.Wrapping.html)的類型。它重載了基本的運算子，可以被當成普通整數使用。凡是被它包裹起來的整數，任何時候出現溢出都是截斷行為。

```rust
use std::num::Wrapping;
fn main() {
    let big = Wrapping(std::u32::MAX);
    let sum = big + Wrapping(2_u32);
    println!("{}", sum.0);
}
```

## 浮點數類型

Rust提供了基於IEEE 754-2008標準的浮點類型。按佔據空間大小區分，分別為f32和f64(<mark style="color:red;">未明確指定類型時，預設為f64</mark>)，其使用方法與整數差別不大。

```rust
fn main() {
    let f1 = 123.0f64; // type f64
    let f2 = 0.1f64;   // type f64
    let f3 = 0.1f32;   // type f32
    let f4 = 12E+99_f64; // type f64 科學記號
    let f5: f64 = 2.;  // type f64
}
```

浮點數的麻煩之處在於：**它不僅可以表達正常的數值，還可以表達不正常的數值**。在標準庫中，有一個[std::num::FpCategory](https://doc.rust-lang.org/std/num/enum.FpCategory.html)枚舉，表示了浮點數可能的狀態。

```rust
pub enum FpCategory {
    Nan,       // 不是數字(not a number)
    Infinite,  // 無窮大
    Zero,      // 數值為0
    Subnormal, // 浮點數計算式為(-1)^s*M*2^e
    Normal,    // 正常狀態的浮點數, (-1)^s*(1+M)*2^e
}
```

![IEEE754 單精度浮點數。](../.gitbook/assets/ieee754\_single-precision-min.png)

### 整數與浮點數四則運算需明確轉型

```rust
fn main() {
    let x = 32;
    let y = 2.0;
    // 必須明確轉型，否則會出現error
    // no implementation for `{integer} + {float}`
    let z = x as f64 + y;
    println!("z={}", z);
}
```

### normal狀態

在IEEE 754標準中，規定了浮點數的二進位表達方式：`x=（-1）^sign *（1+fraction）* 2^exponent`。其中sign(s)是符號位元，fraction(M)是分數(fraction)，exponent(e)是指數。分數M是一個\[0，1）範圍內的二進位表示的小數。

以32位浮點為例，如果只有normal形式的話，0表示為所有位數全0，則最小的非零正數將是尾數最後一位元為1的數字，就是（1+ 2^（-23））\*2^（-127），而次小的數字為（1+2^（-22））\*2^（-127），這兩個數字的差距為2^（-23）\*2^（-127）=2^（-150），然而最小的數字和0之間的差距有（1+2^（-23））\*2^（-127），約等於2^（-127），**也就是說，數字在漸漸減少到0的過程中突然降到了0。**

### **subnormal狀態**

為了減少0與最小數位和最小數位與次小數位之間步長的突然下跌，subnormal規定：當指數位全0的時候，指數表示為-126而不是-127（和指數為最低位為1一致）。然而公式改成（-1）^s\*M\*2^e，M不再+1，這樣最小的數字就變成2^（-23）\*2^（-126），次小的數字變成2^（-22）\*2^（-126），每兩個相鄰subnormal數字之差都是2^（-23）\*2^（-126），避免了突然降到0。在這種狀態下，這個浮點數就處於了Subnormal狀態，處於這種狀態下的浮點數表示精度比Normal狀態下的精度低一點。

```rust
fn main() {
    // 變數 small 初始化為一個非常小的浮點數
    let mut small = f32::EPSILON;
    // 不斷迴圈,讓 small 越來越趨近於 0,直到最後等於0的狀態
    // 浮數數狀態會由normal->subnormal-> zero
    while small > 0.0 {
        small = small / 2.0;
        // f32.classify()傳回浮點數的狀態
        println!("{} {:?}", small, small.classify());
    }
}
```

### infinite與Nan狀態

Infinite和Nan是帶來更多麻煩的特殊狀態。Infinite代表的是“無窮大”，Nan代表的是“不是數字”（not a number）。

非0数除以0值，得到的是inf，0除以0得到的是NaN。

```rust
fn main() {
    let x = 1.0f32 / 0.0; // infinity
    let y = 0.0f32 / 0.0; // Nan
    println!("{} {:?}", x, x.classify());
    println!("{} {:?}", y, y.classify());
}
```

對inf做一些數學運算的時候，它的結果可能與你期望的不一致。

```rust
fn main() {
    let inf = std::f32::INFINITY;
    // NaN 0 NaN
    println!("{} {} {}", inf * 0.0, 1.0 / inf, inf / inf);
}
```

NaN這個特殊值有個特殊的麻煩，主要問題還在於它不具備“全序”(total order)的特點。一個數字可以不等於自己。因為NaN的存在，浮點數是不具備“全序關係”（total order）的。關於“全序”和“偏序”的問題與 PartialOrd和Ord這兩個trait有關。

```rust
fn main() {
    let nan = std::f32::NAN;
    // false false false, 不滿足三一律
    println!("{} {} {}", nan < nan, nan > nan, nan == nan);
}
```

## 指標類型

Rust裡面也有指標類型，而且不止一種指標類型。常見的幾種指標如下：

| 類型名稱      | 說明                               |
| --------- | -------------------------------- |
| Box\<T>   | 指向類型T，具有所有權的指標，有權釋放記憶體。          |
| \&T       | 指向類型T的借用指標，也稱為引用，無權釋放記憶體，不可寫入資料。 |
| \&mut T   | 指向類型T的mut型借用指標，無權釋放記憶體，可寫入資料。    |
| \*const T | 指向類型T的唯讀裸指標，沒有生命週期資訊，不可寫入資料。     |
| \*mut T   | 指向類型T的唯讀裸指標，沒有生命週期資訊，可寫入資料。      |

除此之外，在標準庫中還有一種封裝起來的可以當作指標使用的類型，叫“智慧指標”（smart pointer）。

| 類型名稱       | 說明                                           |
| ---------- | -------------------------------------------- |
| Rc\<T>     | 指向類型T的引用計數指標，共享所有權，線程不安全。                    |
| Arc\<T>    | 指向類型T的原子型引用計數指標，共享所有權，線程安全。                  |
| Cow<'a, T> | Clone-on-write，寫入時複製指標。可能是借用指標，也可能是具有所有權的指標。 |

##
