# 借用和引用(borrow and reference)

## 借用(borrow)

<mark style="background-color:blue;">註：&符號在說明記憶體的所有權時的概念稱為借用(borrow)，而在說明變數的型別時稱為引用(reference)</mark>。

如果我們必須在每個函式都交還所有權，這將會非常煩人。 當我們想處理更多所有權的時候會變得更糟。Rust 提供一個功能，借用（borrowing），可以幫助我們解決這個問題。

借用指標的語法使用**`&`符號(唯讀)**或者**`&mut`符號(可讀寫)**表示。前者表示唯讀借用，後者表示可讀寫借用。**借用指標（borrow pointer）也可以稱作“引用”（reference）**。

&#x20;我們把 `&T` 型別稱為「參照」（reference），它借用了所有權，而非掌握所有權。 一個借用東西的綁定不會在離開有效範圍時把資源釋放掉。 這代表在呼叫函數完之後，我們仍可再度使用我們原始的變數綁定。

<mark style="color:blue;">借用指標與普通指標的內部資料是一模一樣的，唯一的區別是語義層面上的。它的作用是告訴編譯器，它對指向的這塊記憶體區域</mark><mark style="color:red;">沒有所有權</mark>。

```rust
fn main() {
    let n = 5;
 
    // Get the reference of n
    // nref變數存放n的記憶體地址
    let n_ref = &n;  // &i32
 
    println!("{:p}", &n);       // 0x7fff24bd3e9c
    println!("{:p}", n_ref);    // 0x7fff24bd3e9c
    println!("{:p}", &n_ref);   // 0x7fff24bd3ea0
    // Dereference n_ref to get n
    println!("{}", *n_ref);     // 5
}
```

### 唯讀的借用

* `&`符號在let表達式中，一定是放在`=`符號的右側，表示對記憶體的引用，而左側的變數只是借用，沒有所有權。
* `&`符號在函數中，一定是放在變數型別前，表示傳入的是引用型別。
* 變數傳入函數時，要明確寫出`&`符號。

```rust
fn foo(_v1: &Vec<i32>, _v2: &Vec<i32>) -> i32 {
    // 變數以reference傳入，所有權為borrow而非move
    println!("borrow: {:p}, {:p}", _v1, _v2);
    // 0x7ffce11a8da0, 0x7ffce11a8db8, 相同的記憶體地址
    42
}

fn main() {
    let v1 = vec![1, 2, 3]; // v1有vec的所有權
    let v2 = vec![1, 2, 3]; // v2有vec的所有權
    println!("{:p}, {:p}", &v1, &v2);   // 0x7ffce11a8da0, 0x7ffce11a8db8
    let answer = foo(&v1, &v2); // 以reference傳入函數
    println!("v1: {:?}, v2: {:?}", v1, v2); // OK, 所有權仍在v1, v2
    println!("answer: {}", answer); // 42
}
```

### 可變的借用

* `&mut`符號在let表達式中，一定是放在`=`符號的右側，表示對記憶體可變的引用。
* `&mut`符號在函數中，一定是放在變數型別前。
* 變數傳入函數時，要明確寫出`&mut`符號，<mark style="color:red;">而且只有\&mut的變數可以傳入</mark>。

```rust
// 我們需要“可變的”借用指標,因此函數簽名需要改變
fn foo(v: &mut Vec<i32>) {
    v.push(5);
}
fn main() {
    // 我們需要這個動態陣列本身是“可變的”,才能獲得它的“可變借用指標”
    let mut v = vec![];
    // 在函式呼叫的時候,同時也要顯示獲取它的“可變借用指標”
    foo(&mut v);
    // 列印結果,可以看到v已經被改變
    println!("{:?}", v);
}
```

對於`&mut`型指標，要注意不要混淆它與變數綁定之間的語法：

* 如果`mut`修飾的是變數名，那麼它代表這個變數可以被重新綁定；(但指向的物件不能被修改)
* 如果`mut`修飾的是“借用指標&”，那麼它代表的是被指向的物件可以被修改。(但變數不能被修改)

## 可變借用與綁定的組合

| let a: \&T = value;             | a與value都不能修改                                  | const T\* const a |
| ------------------------------- | --------------------------------------------- | ----------------- |
| let mut a: \&T = \&value;       | a可變，但value不可變，即不能透過a修改指向(value)的內容，但a可指向新的地址。 | const T\* a       |
| let a: \&mut T = \&mut value;   | a不可變，但value可變，即可透過a修改value的內容，但a不可指向新的地址。     | T\* const a       |
| let mut a:\&mut T =\&mut value; | a與value都能修改                                   | T\* a             |

```rust
let x = value;
// x {binds immutably} to {immutable value}

let mut x = value;
// x {binds mutably} to {possibly mutable value}

let x = &value;
// x {binds immutably} to {a reference to} {immutable value}
// 不可變引用x指向不可變的value

let x = &mut value;
// x {binds immutably} to {a reference to} {mutable value}
// 可直接修改value之值，或是用*x修改指向值，但x不可指向其它變數

let mut x = &value;
// x {binds mutably} to {a reference to} {immutable value}
// value之值不可變，也不可透過*x修改值，但x可指向其它的變數

let mut x = &mut value;
// x {binds mutably} to {a reference to} {mutable value}
```

### 不可變借用的可變綁定

```rust
struct FullName {
    first_name: String,
    last_name: String,
}

fn main() {
    let obj1 = FullName {
        first_name: String::from("Jobs"),
        last_name: String::from("Steve"),
    };
    // mut a:& T, a可指向新的引用，但不可修改obj
    let mut a = &obj1;
    
    // a重新綁定到一個新的FullName的引用
    let obj2 = &FullName {
        first_name: String::from("Gates"),
        last_name: String::from("Bill"),
    };
    a = &obj2;
    //不允許對a指向的內容作出修改
    //a.first_name = String::from("Error");
    println!("{}:{}", a.last_name, a.first_name);
}
```

### 可變借用的不可變綁定

```rust
struct FullName {
    first_name: String,
    last_name: String,
}

fn main() {
    let mut obj1 = FullName {
        first_name: String::from("Jobs"),
        last_name: String::from("Steve"),
    };
    // a: &mut T，a不可指向新的引用，但可修改T
    let a = &mut obj1;
    
    let mut obj2 = FullName {
         first_name: String::from("Gates"),
         last_name: String::from("Bill"),
    };
    //a不允許重新綁定到一個新的FullName的引用
    // a = &mut obj2;
    
    //允許對a指向的內容作出修改
    a.first_name = String::from("Gates");
    println!("{}:{}", a.last_name, a.first_name);
}
```

## Rust沒有null指標

Rust 沒有 null pointer。在你拿一個合法物件的位址來初始化指標之前，任何操作指標的行為都會被編譯器阻止。因為 Rust 中不存在 null pointer，因此當你的函式接收到參考型別時，你可以假設它必然指向一個合法的物件，不需要進行額外檢查，編譯器也不會在執行時額外花時間去檢查參考是否合法，進而提昇執行效率。

<mark style="color:red;">但是Rust有Option\<T>的類型，可以用其中的None表示沒有任何值的概念</mark>。

## 資料競爭（data race）

類似於競態條件，它可由這三個行為(同時發生時)造成：

* 兩個或更多指標同時訪問同一資料(變數)。&#x20;
* 至少有一個指標被用來寫入資料。&#x20;
* 沒有同步資料訪問的機制。

資料競爭會導致未定義行為，難以在運行時追蹤，並且難以診斷和修復；Rust 避免了這種情況的發生。

借用規則
----

* <mark style="color:red;">引用必須總是有效的</mark>。
* <mark style="color:red;">借用指標不能比它指向的變數存在的時間更長</mark>。(註：編譯器無法判定時，會要求明確加入生命週期)。
  * **借用指標只能臨時地擁有對這個變數讀或寫的許可權，沒有義務管理這個變數的生命週期。**因此，借用指標的生命週期絕對不能大於它所引用的原來變數的生命週期，否則就是<mark style="color:red;">懸空指標(dangling pointer)</mark>，會導致記憶體不安全。
* 使用 `&` 取得變數的參考後，你只能透過參考讀取內容，而不能寫入資料。這樣的取址行為，Rust 稱之為不可變借用( immutable borrow)。
* 若你想要透過參考寫入資料，必需透過 `&mut` 來取得位址。這樣的取址稱為可變借用(mutable borrow)。
  * `&mut`型借用只能指向本身具有`mut`修飾的變數，對於唯讀變數，不可以有`&mut`型借用。
  * `&mut`型借用指標存在的時候，**被借用的變數本身會處於“凍結 (freeze)”狀態，即被借用的變數不可被存取**。
* <mark style="color:red;">\[共享不可變] 如果作用域只有</mark><mark style="color:red;">`&`</mark><mark style="color:red;">型借用指標，那麼能同時存在多個</mark>；
* <mark style="color:red;">\[可變不共享] 如果作用域存在</mark><mark style="color:red;">`&mut`</mark><mark style="color:red;">型借用指標，那麼只能存在一個</mark>；
  * 如果同時有其他的`&`或者`&mut`型借用指針存在，那麼會出現編譯錯誤。
  * 有點類似、但不完全相同於資料競爭的定義：    當兩個或多個指標同時存取相同的記憶體，其中至少有一個正在寫入，且操作沒有同步時，會出現「資料競爭」（date race）。
* 原本宣告為 mutable 的變數，經過 & 取址後，會暫時變為 immutable，直到所有的 immutable borrow 消滅為止。
* 原本宣告為 mutable 的變數，經過 \&mut 取址後，會暫時無法存取，直到該 mutable borrow 消滅為止。

```rust
// 參數採用的“引用傳遞”,因此參數本身並未丟失對記憶體的管理權
fn borrow_semantics(v: &Vec<i32>) {
    // 列印參數佔用空間的大小,在64位元系統上,結果為8,
    // 表明該指針與普通裸指針的內部表示方法相同
    println!("size of param: {}", std::mem::size_of::<&Vec<i32>>());
    for item in v {
        print!("{} ", item);
    }
    println!("");
}
// 這裡的參數採用的“值傳遞”,
// 而Vec沒有實現Copy trait,意味著它將執行move語義
fn move_semantics(v: Vec<i32>) {
    // 列印參數佔用空間的大小,結果為24,
    // 表明參數中堆疊上分配的記憶體空間複製到了函數的參數中
    println!("size of param: {}", std::mem::size_of::<Vec<i32>>());
    for item in v {
        print!("{} ", item);
    }
    println!("");
}
fn main() {
    let array = vec![1, 2, 3];
    // 如果使用引用傳遞,不僅在函式宣告的地方需要使用&標記
    // 函式呼叫的地方同樣需要使用&標記,否則會出現語法錯誤
    // 這樣設計主要是為了顯眼,不用去閱讀該函數的簽名就知道這個函式呼叫的時候發生了什麼
    // 而小數點方式的成員函式呼叫,對於self參數,會“自動轉換”,不必顯式借用,這裡有個區別
    borrow_semantics(&array);
    // 在使用引用傳遞給上面的函數後,array本身依然有效,我們還能在下面的函數中使用
    move_semantics(array);
    // 在使用move語義傳遞後,array在這個函式呼叫後,它的生命週期已經完結
}
```

指標借用後，會被凍結：

```rust
fn main() {
    let mut x = 1_i32;
    let p = &mut x;
    // 任何借用指標的存在，都會導致原來的變數被“凍結”（Frozen）。
    // 因此x不可再被存取
    //x = 2; // compile error, 不可被修改
    // println!("value of x : {x}"); // 也不可被讀取
    println!("value of pointed : {p}");
}
```

可以使用大括號來創建一個新的作用域，以允許擁有多個可變引用，只是不能在同一作用域同時存在。

```rust
fn main() {
    let mut s = String::from("hello");
    let r2 = &s; // 沒問題
    //let r3 = &mut s; // 大問題
    {
        let r3 = &mut s;
    } // r3 在這裡離開了作用域，所以我們完全可以創建一個新的引用
    println!("s={}", s);
}
```

## 借用檢查

Rust的設計者們在一系列的“記憶體不安全”的問題中觀察到了這樣的一個結論：<mark style="color:red;">**Danger arises from Aliasing + Mutation**</mark> <mark style="color:red;"></mark><mark style="color:red;">(兩者同時發生時才會發生記憶體安全問題)</mark>。

Rust保證記憶體安全的一個重要原則就是，如果能保證alias和mutation不同時出現，那麼程式碼就一定是安全的。

Rust 希望達成的目標是，當你對某一塊記憶體寫入資料時，編譯器可以保證這塊記憶體沒有其它的別名 (alias)，從而避免對變數的寫入行為，會破壞所有指向該變數的參考問題。

簡單的做法就是<mark style="color:red;">「</mark><mark style="color:red;">**共享不可變，可變不共享**</mark><mark style="color:red;">」</mark>。

### alias (別名)

Alias的意思是“別名”。如果一個變數可以通過多種Path來訪問，那它們就可以互相看作alias。Alias意味著“共用”，我們可以通過多個入口訪問同一塊記憶體。例如指標或是reference。

### mutation (改變)

Mutation的意思是“改變”。如果我們通過某個變數修改了一塊記憶體，就是發生了mutation。Mutation意味著擁有“修改”許可權，我們可以寫入資料。

<mark style="color:blue;">`&mut`</mark><mark style="color:blue;">型借用也經常被稱為“獨佔指標”，</mark><mark style="color:blue;">`&`</mark><mark style="color:blue;">型借用也經常被稱為“共用指標</mark>”。

### 不可變的借用

```rust
fn main() {
    let i = 0;
    // p1, p2是i的別名，但不可寫入，
    // 所以是安全的
    let p1 = &i;
    let p2 = &i;
    // 三個變數指向同一塊記憶體, 但都是唯讀
    println!("{:p} {:p} {:p}", &i, p1, p2);
}
```

### 變數可變時不能唯讀借用

```rust
fn main() {
    let mut i = 0;
    // 編譯錯誤, 因為i可變, p1為別名時,
    // 違反共享不可變的原則
    let p1 = &i;
    i = 1;
}
```

### 變數可變時可使用可變借用

在p1存在的情況下，不可通過i寫變數。如果這種情況可以被允許，那就會出現多個入口可以同時訪問同一塊記憶體，且都具有寫許可權，這就違反了Rust的“共用不可變，可變不共用”的原則。

```rust
fn main() {
    let mut i = 0;
    // 可變借用時, 變數i被凍結
    // 不可透過i讀取或寫入變數
    // 因此沒有違反可變不共享原則
    let p1 = &mut i;
    *p1 = 2;
    // i  = 3;   // error
    println!("{p1}"); // 2
}
```

```rust
fn main() {
    let mut x = 5;
    let y = &mut x; // mutable borrow, 全域
 
    *y += 1;
    // 如果只有x傳給println是合法的操作
    println!("x={x}");
    // 若同時將borrow與mut borrow傳給println，
    // 會產生編譯錯誤
    println!("x={x}, y={y}");
}
```

* 上例這是因為我們違反了規則：我們有一個指向 x 的 `&mut T`，所以我們不被允許建立任何其他 `&T`。 必須要在兩者間做出選擇。&#x20;
* 解法：所以我們希望可變借用能在我們呼叫 println! 並建立不可變借用 之前 能結束掉。可變借用會在我們建立不可變借用前離開有效範圍。 有效範圍是個看清借用持續多久的關鍵。

```rust
fn main() {
    let mut x = 5;
    {
        // 將mutable borrow的範圍限縮在括號內
        let y = &mut x; // -+ &mut borrow starts here
        *y += 1; //  |
    } // -+ ... and ends here
      // mutable borrow生命周期已結束

    println!("x={}", x); //6
}
```



### &#xD;不可同時有多個可變借用

```rust
fn main() {
    let mut i = 0;
    // error, 同時有多個可變借用
    // 違反可變不共享原則
    let p1 = &mut i;
    let p2 = &mut i;
}
```

## 借用所預防的問題

### 疊代器失效（Iterator invalidation）

當你試圖改變一個正在疊代的集合（collection）時會發生。 Rust 的借用檢查器會預防這件事發生。避免迭代容器到一半時，容器改變而造成迭代子指向其它地方。

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    // 當我們疊代這個向量時，我們只會被給予其中元素的引用
    for i in &v {
        println!("{i}");
        // v 是一個不可變的借用，所以我們在疊代時不能更改它
        // v.push(34);
    }
}
```

### 在釋放之後使用（use after free）

References 不能存活得比所參考的資源還久。 Rust 會檢查你的引用的有效範圍來確保符合這個條件。



```rust
fn main() {
    let y: &i32;
    {
        let x = 5;
        y = &x; // x只在scope內存活
    }
    // x已經被釋放，y無法正確參照到x, 編譯錯誤
    println!("{}", y);
}
```

## 參考資料

* [\[知乎\] 理解 Rust 引用和借用](https://zhuanlan.zhihu.com/p/59998584)
* [\[stackoverflow\] What's the difference between placing "mut" before a variable name and after the ":"?](https://stackoverflow.com/questions/28587698/whats-the-difference-between-placing-mut-before-a-variable-name-and-after-the)
* [Rust Smart Pointers 最燒腦的智慧指標](https://weihanglo.tw/slides/rust-smart-pointers.html)
