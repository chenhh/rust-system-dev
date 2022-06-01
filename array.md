---
description: 數組
---

# 陣列

## 簡介

* 陣列是一個容器，它在一塊**連續空間記憶體**中，存儲了一系列的**同樣類型**的資料。
* 陣列中元素的佔用空間大小必須是**編譯期確定**的。
* 陣列本身所容納的元素個數也必須是編譯期確定的，執行階段不可變。如
* 果需要使用變長的容器，可以使用標準庫中的Vec/LinkedList等。
* 陣列類型的表示方式為**\[T:n\]**。其中T代表元素類型；n代表元素個數；它必須是編譯期常量整數；中間用分號隔開。
* 對陣列內部元素的訪問，可以使用中括弧索引的方式。Rust支援usize類型的索引的陣列，**索引從0開始計數**。

```rust
fn main() {
    // 固定長度的陣列
    let xs: [i32; 5] = [1, 2, 3, 4, 5];
    println!("{:?}", xs);
    // 所有的元素,可使用以下語法初始為同個的值
    // 類別可省略由編譯器自動推導
    let ys = [2; 10];
    println!("{:?}", ys);
}
```

在Rust中，對於兩個陣列類型，只有元素類型和元素個數都完全相同，這兩個陣列才是同類型的。陣列與指標之間不能隱式轉換。同類型的陣列之間可以互相賦值。

```rust
fn main() {
    let mut xs: [i32; 5] = [1, 2, 3, 4, 5];
    let ys: [i32; 5] = [6, 7, 8, 9, 10];
    xs = ys;    // 所有權轉移
    println!("new array {:?}", xs);
    
    let zs = [0; 6];
    // xs = zs; // error, 長度不同的陣列不可賦值
    
    let ds = [0_f32; 5];
    // xs = ds; // error, 類別不同的陣列不可賦值
}
```

### 陣列比較

```rust
fn main() {
    let v1 = [1, 2, 3];
    let v2 = [1, 2, 4];
    // element-wise comparison
    println!("{:?}", v1 < v2);
}
```

### 遍歷操作

```rust
fn main() {
    let v = [0_i32; 10];
    for i in &v {
        println!("{:?}", i);
    }
}
```

## 多維陣列

```rust
fn main() {
    // 3 * 2 array
    let v: [[i32; 2]; 3] = [[0, 0], [0, 0], [0, 0]];
    for i in &v {
        println!("{:?}", i);
    }
}
```

## 陣列切片\(slice\)

**對陣列取借用borrow操作，可以生成一個“陣列切片”（Slice）**。

陣列切片對陣列沒有“所有權”，我們可以把陣列切片看作專門用於指向陣列的指標，是對陣列的另外一個“視圖” \(view\)。比如，我們有一個陣列`[T:n]`，它的借用指標的類型就是`&[T;n]`。它可以通過編譯器內部魔法轉換為陣列切片類型`&[T]`。陣列切片實質上還是指標，它不過是在類型系統中丟棄了編譯階段定長陣列類型的長度資訊，而將此長度資訊存儲為運行期的值。

```rust
fn main() {
    fn mut_array(a: &mut [i32]) {
        a[2] = 5;
    }
    println!("size of &[i32; 3] : {:?}", std::mem::size_of::<&[i32; 3]>()); //8
    // fat pointer佔用了兩個pointer的空間 (64-bit OS pointer為8 bytes)
    println!("size of &[i32] : {:?}", std::mem::size_of::<&[i32]>());   // 16
    println!("size of i32 : {:?}", std::mem::size_of::<i32>());   // 4
    let mut v: [i32; 3] = [1, 2, 3];
    {
        let s: &mut [i32; 3] = &mut v;
        mut_array(s);
    }
    println!("{:?}", v);    // [1, 2, 5]
}
```

## DST與胖指標

Slice與普通的指標是不同的，它有一個非常形象的名字：胖指標（fat pointer）。與這個概念相對應的概念是“動態大小類型”（Dynamic Sized Type, DST）。

**所謂的DST指的是編譯階段無法確定佔用空間大小的類型。為了安全性，指向DST的指標一般是胖指標**。比如：對於不定長陣列類型`[T]`，有對應的胖指標`&[T]`類型；對於不定長字串`str`類型，有對應的胖指標`&str`類型；以及在後文中會出現的Trait Object；等等。

由於不定長陣列類型`[T]`在編譯階段是無法判斷該類型佔用空間的大小的，目前我們不能在堆疊上聲明一個不定長大小陣列的變數實例，也不能用它作為函數的參數、返回值。但是，指**向不定長陣列的胖指標的大小是確定的**，`&[T]`類型可以用做變數實例、函數參數、返回值。

胖指標內部的資料既包含了指向原始陣列的位址，又包含了該切片的長度。

```rust
fn raw_slice(arr: &[i32]) {
    unsafe {
        // 強制類型轉換, 以usize去切記憶體
        let (val1, val2): (usize, usize) = std::mem::transmute(arr);
        println!("Value in raw pointer:");
        // fat pointer第一個值存原始pointer的地址
        println!("value1: {:x}", val1); // 7ffe95900c9c
        // fat pointer第二個值存原始pointer的長度
        println!("value2: {:x}", val2); // 5
    }
}
fn main() {
    let arr: [i32; 5] = [1, 2, 3, 4, 5];
    let address: &[i32; 5] = &arr;
    println!("Address of arr: {:p}", address);  // 0x7ffe95900c9c
    // 轉型為fat pointer
    raw_slice(address as &[i32]);
}
```

對於DST類型，Rust有如下限制：

* 只能通過指標來間接創建和操作DST類型，`&[T]Box<[T]>`可以，`[T]`不可以；
* 區域變數和函數參數的類型不能是DST類型，因為區域變數和函數參數必須在編譯階段知道它的大小，因為目前unsized rvalue功能還沒有實現；
* enum中不能包含DST類型，struct中只有最後一個元素可以是DST，其他地方不行，如果包含有DST類型，那麼這個結構體也就成了DST類型。

Rust設計出DST類型，使得類型暫時系統更完善，也有助於消除一些C/C++中容易出現的bug。這一設計的好處有：·

* 首先，DST類型雖然有一些限制條件，但我們依然可以把它當成合法的類型看待，比如，可以為這樣的類型實現trait、添加方法、用在泛型參數中等；
* 胖指標的設計，避免了陣列類型作為參數傳遞時自動退化為裸指標類型，丟失了長度資訊的問題，保證了類型安全；
* 這一設計依然保持了與“所有權”“生命週期”等概念相容的特點。陣列切片不只是提供了“陣列到指標”的安全轉換，配合上Range功能，它還能提供陣列的局部切片功能。

## Range

Rust中的Range代表一個“區間”，一個“範圍”，它有內置的語法支援，就是兩個小數點..。

```rust
fn main() {
    // r是一個Range<i32>,中間是兩個點,代表[1,10)這個區間
    let r = 1..10;
    // 等價寫法
    // let r = Range { start: 1, end: 10 };
    for i in r {
        print!("{:?}\t", i);
    }
}
```

在begin..end這個語法中，前面是閉區間，後面是開區間。這個語法實際上生成的是一個[`std::ops::Range<_>`](https://doc.rust-lang.org/std/ops/struct.Range.html)類型的變數。

兩個小數點的語法僅僅是一個“語法糖”而已，用它構造出來的變數是Range類型。

```rust
pub struct Range<Idx> {
    /// The lower bound of the range (inclusive).
    pub start: Idx,
    /// The upper bound of the range (exclusive).
    pub end: Idx,
}
```

這個類型本身實現了Iterator trait，因此它可以直接應用到迴圈語句中。Range具有反覆運算器的全部功能，因此它能調用反覆運算器的成員方法。

```rust
fn main() {
    // 先用rev方法把這個區間反過來,然後用map方法把每個元素乘以10
    let r = (1i32..11).rev().map(|i| i * 10);
    for i in r {
        print!("{:?}\t", i);
    }
    // 100	90	80	70	60	50	40	30	20	10
}
```

在Rust中，還有其他的幾種Range，包括

* `std::ops::RangeFrom`代表只有起始沒有結束的範圍，語法為`start..`，含義是\[start，+∞）；
* `std::ops::RangeTo`代表沒有起始只有結束的範圍，語法為`..end`，對有符號數的含義是`（-∞，end）`，對無符號數的含義是`[0，end）`；
* `std::ops::RangeFull`代表沒有上下限制的範圍，語法為..，對有符號數的含義是`（-∞，+∞）`，對無符號數的含義是`[0，+∞）`。

```rust
fn print_slice(arr: &[i32]) {
    println!("Length: {}", arr.len());
    for item in arr {
        print!("{}\t", item);
    }
    println!("");
}
fn main() {
    let arr: [i32; 5] = [1, 2, 3, 4, 5];
    print_slice(&arr[..]); // full range
    let slice = &arr[2..]; // RangeFrom
    print_slice(slice);
    let slice2 = &slice[..2]; // RangeTo
    print_slice(slice2);
}
```

在許多時候，使用陣列的一部分切片作為被操作物件在函數間傳遞，既保證了效率（避免直接複製大陣列），又能保證將所需要執行的操作限制在一個可控制的範圍內（有長度資訊，有越界檢查），還能控制其讀寫許可權，非常有用。

雖然左閉右開區間是最常用的寫法，然而，在有些情況下，這種語法不足以處理邊界問題。比如，我們希望產生一個i32類型的從0到i32::MAX的範圍，就無法表示。因為按語法，我們應該寫0..（i32::MAX+1），然而（i32::MAX+1）已經溢出了。所以，Rust還提供了一種左閉右閉區間的語法，它使用這種語法來表示..=。

閉區間對應的標準庫中的類型是：

* `std::ops::RangeInclusive`，語法為`start..=end`，含義是`[start, end]`。
* `std::ops::RangeToInclusive`，語法為`..=end`，對有符號數的含義是`（-∞, end]`，對無符號數的含義是`[0, end]`。

## 邊界檢查\(boundary check\)

```rust
fn main() {
    let v = [10i32, 20, 30, 40, 50];
    // 索引值在執行時決定，可能越界
    let index: usize = std::env::args()
        .nth(1)
        .map(|x| x.parse().unwrap_or(0))
        .unwrap_or(0);
    println!("{:?}", v[index]);
}
```

在Rust中，“索引”操作也是一個通用的運算子，是可以自行擴展的。

* 如果希望某個類型可以執行“索引”讀操作，就需要該類型實現`std::ops::Index` trait；
* 如果希望某個類型可以執行“索引”寫操作，就需要該類型實現`std::ops::IndexMut` trait。

對於陣列類型，如果使用usize作為索引類型執行讀取操作，實際執行的是標準庫中的以下程式碼。如果index超過了陣列的真實長度範圍，會執行panic！操作，導致執行緒abort。使用Range等類型做Index操作的執行流程與此類似。

```rust
impl<T> ops::Index<usize> for [T] {
    type Output = T;
    // 每次從array取值都會檢查是否超出邊界
    fn index(&self, index: usize) -> &T {
        assert!(index < self.len());
        unsafe { self.get_unchecked(index) }
    }
}
```

為了防止索引操作導致程式崩潰，如果我們不確定使用的“索引”是否合法，應該使用get\(\)方法調用來獲取陣列中的元素，這個方法不會引起panic！，它的返回類型是Option&lt;T&gt;：

```rust
fn main() {
    let v = [10i32, 20, 30, 40, 50];
    let first = v.get(0);   // Some(10), 可用first.unwrap()取值
    let tenth = v.get(10);  // None
    println!("{:?} {:?}", first, tenth);
}
```

Rust宣稱的優點是“無GC的記憶體安全”，那麼陣列越界會直接導致程式崩潰這件事情是否意味著Rust不夠安全呢？不能這麼理解。Rust保證的“記憶體安全”，並非意味著“永不崩潰”。**Rust中關於陣列越界的行為，定義得非常清晰。相比於C/C++，Rust消除的是“未定義行為”（Undefined Behavior）。**

對於明顯的陣列越界行為，在Rust中可以通過lint檢查來發現。大家可以參考“clippy”這個項目，它可以檢查出這種明顯的常量索引越界的現象。然而，**總體來說，在Rust裡面，靠編譯階段靜態檢查是無法消除陣列越界的行為的**。

一般情況下，Rust不鼓勵大量使用“索引”操作。正常的“索引”操作都會執行一次“邊界檢查”。**從執行效率上來說，Rust比C/C++的陣列索引效率低一點，因為C/C++的索引操作是不執行任何安全性檢查的**，它們對應的Rust程式碼相當於調用get\_unchecked\(\)函數。在Rust中，更加地道的做法是儘量使用“迭代器”方法。

```rust
fn main() {
    let v = &[10i32, 20, 30, 40, 50];
    // 如果我們同時需要index和內部元素的值,調用enumerate()方法
    for (index, value) in v.iter().enumerate() {
        println!("{} {}", index, value);
    }
    // filter方法可以執行過濾,nth函數可以獲取第n個元素
    let item = v.iter().filter(|&x| *x % 2 == 0).nth(2);
    println!("{:?}", item);
}
```



