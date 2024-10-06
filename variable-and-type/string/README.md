# 字串

## 簡介

Rust的字元固定使用4 bytes儲存unicode。

&#x20;為了處理所有權，Rust的字串涉及兩種類型，一種是`&str(slice類型)`，另外一種是`String`，為Rust 標准庫中集合（collections）的非常有用的資料結構。

字串切片`str`，它通常是以借用的形式存在`&str`，是一個針對存在某處的 UTF-8 編碼資料的參考。舉例來說，字串字面值（String literals）就儲存在程式的執行檔中，因此就是字串切片。

`String` 型別是 Rust 標準函式庫所提供的型別，並不是核心語言內建的型別，它是可增長的、可變的、可擁有所有權的 UTF-8 編碼字串型別。

<mark style="background-color:red;">如果將字串變數傳參數給函數時，\&str的字度固定只有16 bytes，而String類型會將字串複製後再傳入，較浪費時間和記憶體</mark>。

## 字串切片(\&str)：字串常值

\&str是Rust的內置類型，<mark style="color:red;">不可修改指向的字串</mark>。\&str是對str的借用。

那麼這麼字串(str)的所有者是誰？<mark style="color:blue;">字串常值是引用自“預分配文字(preallocated text)”的字串切片</mark>，這個預分配文字存儲在可執行程式的唯讀記憶體中。換句話說，這是裝載我們程式的記憶體並且不依賴於在堆積上分配的緩沖區。

<mark style="color:red;">**Rust的字串內部預設是使用utf-8編碼格式的。而內置的char類型是4位元組長度的，存儲的內容是Unicode Scalar Value**</mark>。所以，Rust裡面的字串不能視為char類型的陣列，而更接近u8類型(無號8位元整數而非utf-8)的陣列。

<figure><img src="../../.gitbook/assets/image (13).png" alt="" width="177"><figcaption><p>字串切片&#x26;str記錄字串的記憶體位址</p></figcaption></figure>

<mark style="color:red;">字串切片是一個胖指標（fat pointer），內容為指向一個記憶體位址以及指向內容(編譯時已放進程式中，無法修改)的長度</mark>。

```rust
use std::mem::size_of;
fn main() {
    // 8 bytes, 只儲存記憶體位址
    println!("size of raw pointer: {}", size_of::<*const ()>()); 
    // 16 bytes，儲存記憶體位址和字串長度
    println!("size of string slice: {}", size_of::<&str>()); 
}
```

如果字串不宣告資料類型時，預設類型為`&str`。

```rust
use std::mem;

fn print_type_of<T>(_: &T) {
    println!("{}", std::any::type_name::<T>());
}

fn main() {
    let s = "Hello, world!"; // &str，string slice，字串常值無法變更。
    println!("{s}, length:{}", s.len()); // Hello, world!, length:13
    print_type_of(&s); // &str
    println!("size: {}", mem::size_of_val(&s)); // 16

    let s2 = String::from(s);
    println!("{s2}, length:{}", s2.len()); // Hello, world!, length:13
    print_type_of(&s2); // alloc::string::String
    println!("size: {}", mem::size_of_val(&s2)); // 24
}
```

Rust的String是一個`Vec<u8>`的型態，也有是一個儲存格式良好（well formed）的向量，存放在堆積(heap)。

String, \&str 都可有一個`.len()`的方法，這個方法回傳的不是字元數而是bytes的長度（記憶體用量）。

由於\&str是個指標，所以\&str本身是無法修改的，必須是要String才能處理。

### String和\&str的互相轉換

```rust
fn main(){
    // 由str建構String
    let mut s = String::from("world");
    s += "hi hi ";
    println!("{s}"); // worldhi hi 
    
    // 由str建構String, 必須指定s的類型
    let s: String = "also this".into();
    println!("{s}");    // also this
    
    // 由str轉為String, s不可變
    let s = "initial contents".to_string();
    println!("{s}"); // initial contents
    
    // 由String建構&str
    let cs = s.as_str();
    let cs2 = s.to_string();
    let cs3 = &s;
    println!("{cs}, {cs2}, {cs3}");    
}
```

### 字串常值與字串的所有權

Rust 中只有一種字串原生類型：`str`，而字串切片，它通常以被借用的形式出現，`&str`，因為是唯讀借用，所以**沒有所有權且不可變**。

&#x20;String 的類型是由標准庫提供的，而沒有寫進核心語言部分，它是可增長的、可變的、**有所有權**的UTF-8 編碼的字串類型，String 是一個 `Vec<u8>` 的封裝。

<mark style="color:red;">當 Rustacean 們談到 Rust 的 “字串”時，它們通常指的是</mark> <mark style="color:red;">`String`</mark> <mark style="color:red;">和字串</mark> <mark style="color:red;">`slice &str`</mark> <mark style="color:red;">類型，而不僅僅是其中之一</mark>。<mark style="background-color:red;">String 和字串 slice 都是 UTF-8 編碼的</mark>。

如果用C++來對比，Rust的`String`類型類似於`std::string`，而Rust的`&str`類型類似於`std::string_view`。

但是一般地，保守來講，如果我們正在構建的API不需要擁有或者修改使用的文字，那麼應該使用`&str`而不是`String`。

Rust 標准庫中還包含一系列其他字串類型，比如 OsString、OsStr、CString 和 CStr。對應著它們提供的所有權和可借用的字串變體。

```rust
// 此處的u8為8 bits(1 byte)的無號整數類型
pub struct String {
    vec: Vec<u8>,
}
```

## 字元和字串

| 名稱   | 範例       | 類別                       |
| ---- | -------- | ------------------------ |
| 字元   | 'H'      | unicode(UTF-16), 4 bytes |
| 位元組  | b'H'     | ASCII, 1byte             |
| 字串常值 | "hello"  | u8 array                 |
| 位元字串 | b"hello" | ASCII array              |

```rust
use std::mem::size_of_val;
fn print_type_of<T>(_: &T) {
    println!("{}", std::any::type_name::<T>());
}

fn main() {
    // 儲存unicode
    let c = 'H';
    print_type_of(&c); // char
    println!("sizeof: {}", size_of_val(&c)); // 4

    // 儲存byte char
    let c = b'H';
    print_type_of(&c); // u8
    println!("sizeof: {}", size_of_val(&c)); // 1

    // 字串常數
    let s = "hello";
    print_type_of(&s); // &str
    println!("sizeof: {}", size_of_val(&s)); // 16

    // 位元切片
    let s = b"hello";
    print_type_of(&s); // &[u8; 5]
    println!("sizeof: {}", size_of_val(&s)); // 8
}
```



## 字串切片(\&str)：截取部份字串

只有ascii字串常值可以使用下標方式取切片，unicode字串要特別處理。

```rust
fn main() {
    // 因為字串是 UTF-8 串流，所以並不支援使用索引來存取
    let s: &str = "hello";
    println!("The first letter of s is {}", s[0]); // ERROR!!!
}

fn main() {
    // unicode字串取切片比較麻煩
    let s = "今天天氣很好";
    // 要先用take，再用skip，此時index才會和切片的[start, end)一致
    // 如果是用先用skip再用take，則take的index要修改為end-start
    let substr: String = s.chars().take(4).skip(1).collect();

    // ascii可直接取切片
    let s2 = "hello world";
    let substr2 = &s2[1..4];

    println!("{substr}"); // 天天氣
    println!("{substr2}"); // ell
}
```

實際上str類型有一種方法：`fn as_ptr（&self）->*const u8`。它內部無須做任何計算，只需做一個強制類型轉換即可。

這樣設計有一個缺點，就是不能支援O(1）時間複雜度的索引操作。如果我們要找一個字串s內部的第n個字元，不能直接通過s\[n]得到，這一點跟其他許多語言不一樣。在Rust中，這樣的需求可以通過下面的語句實現：

```rust
use std::mem;
use std::mem::size_of_val;
fn print_type_of<T>(_: &T) {
    println!("{}", std::any::type_name::<T>());
}

fn main() {
    let hachiko: &str = "忠犬ハチ公";

    // 轉成bytes, 每個位元為8bytes
    for b in hachiko.as_bytes() {
        println!("{b}, size:{}", mem::size_of_val(&b));
        print_type_of(&b);  // &u8, fat pointer
    }
    // 229, 191, 160, 231, 138, 172, 227, 131, 143, 227, 131, 129, 229, 133, 172,
    println!("");

    // 轉成chars iterator，每個字元為4 bytes
    for c in hachiko.chars() {
        println!("{c}, sizeof:{}", mem::size_of_val(&c));
        print_type_of(&c);  // char
    }
    // 忠, 犬, ハ, チ, 公,

    println!("");

    // 可以使用以下的方法達到類似於索引的功能
    let dog = hachiko.chars().nth(1); // kinda like hachiko[1]
    println!("{dog:?}"); // Some('犬')
    
    // 將字串以單字切割後，再收集成為vector的元素
    let data: Vec<_> = hachiko.chars().collect();
    println!("{data:?}");
}
```

它的時間複雜度是O(n），因為utf-8是變長編碼，如果我們不從頭開始過一遍，根本不知道第n個字元的位址在什麼地方。但是，綜合來看，選擇utf-8作為內部預設編碼格式是缺陷最少的一種方式了。相比其他的編碼格式，它有相當多的優點。

\[T]是DST類型，對應的str是DST類型。&\[T]是陣列切片類型，對應的\&str是字串切片類型：

```rust
fn main() {
    let hachiko: &str = "hello";
    // substr中取range可行是因為hackiko中存放的都是ascii字元
    // 如果存放的是utf8的字串的話，range取法會有問題, 
    // 因為range是取bytes而不是chars
    let substr : &str = &hachiko[2..];
    println!("{}", substr);
}
```

### \&str轉換為String

```rust
fn main() {
  let my_str = "Pascal";
  // 使用to_string將&str轉為String
  greet_str(my_str);
  greet_string(&my_str.to_string());
  
  // String傳入&str函數，只要用&即可
  let s2 = my_str.to_string();
  greet_str(&s2);
  
}

fn greet_str(name: &str) {
  println!("str: Hello, {}!", name);
}

fn greet_string(name: &String) {
  println!("string Hello, {}!", name);
}
```

## 字串(String)

模組[std::string](https://doc.rust-lang.org/std/string/index.html)。

String類型，這個型別管理被分配到**堆積**上的資料，所以能夠儲存在編譯時未知大小的文字。

<mark style="background-color:red;">它跟\&str類型的主要區別是，它有管理記憶體空間的權力</mark>。<mark style="color:red;">**\&str類型是對一塊字串區間的借用，它對所指向的記憶體空間沒有所有權，哪怕\&mut str也一樣**</mark>。

### 新建字串

* 使用`String::new()`創建一個空的字串，無法在new中放入參數。
* 使用`String::from("...")`創建一個字串。
* 在實現了Display特性的物件上調用`to_string`方法得到一個字串，或者用字串常值調用`into()`方法。

```rust
fn main(){
    // 直接新建字串, new()不用能參數傳入
    let mut s = String::new(); 
    // String可用+新增內容
    s += "hello world";
    println!("{s}");    // hello world
    
    // 由str建構String
    let mut s = String::from("world");
    s += "hi hi ";
    println!("{s}"); // worldhi hi 
    
    // 由str建構String, 必須指定s的類型, s不可變
    let s: String = "also this".into();
    println!("{s}");    // also this
    
    // 由str轉為String, s不可變
    let s = "initial contents".to_string();
    println!("{s}"); // initial contents
}
```

```rust
fn main() {
    // String類型在堆上動態申請了一塊記憶體空間，
    // 它有權對這塊記憶體空間進行擴容，
    // 內部實現類似於std::Vec<u8>類型。
    // 所以我們可以把這個類型作為容納字串的容器使用。
    let mut s = String::from("Hello");
    s.push(' ');
    s.push_str("World."); // push_str() 在字串後追加字面值
    println!("{}", s);
}
```

### 更新字串

String 的大小可以增加，其內容也可以改變，可以方便的使用 `+` 運算符或 `format!` 巨集來拼接 String 值。

```rust
fn main() {
    // 必須是mut才可修改
    let mut s = "hello".to_string();
    s += " world";
    println!("{s}");
}
```

### 串接多個字串

規則如下：

* String類型可以串接\&str，不可以連接String，但可以連接\&String(會轉形為\&str)。
* \&str不可以連接\&str、String，必須先使用to\_owned轉成String且複製\&str內容。

使用+符號，且將String手動轉為\&str。

<mark style="background-color:red;">規則很複雜，建議全部以format!或是String陣列方式，再用push方式處理</mark>。

```rust
fn main() {
    let s1 = String::from("tic");
    let s2 = "tac";
    let s3 = "toe".to_string();

    let s = s1 + "-" + s2 + "-" + &s3;
    println!("{s}"); // tic-tac-toe
    
    // to_owned: 複製&str且轉為String
    let s = s2.to_owned() + "-" + &s2;
    println!("{s}"); // tac-tac
}
```

### 使用format!巨集

類似python的f-string，可處理不同類型的資料，不須轉型，且語法與println!相同。

```rust
fn main() {
    let s1 = String::from("tic");
    let s2 = "tac";
    let s3 = "toe".to_string();

    let s = format!("{s1}-{s2}-{s3}");
    println!("{s}"); // tic-tac-toe
}
```

### 使用 push\_str 和 push 附加字串

push是附加字元，push\_str是附加字串。

```rust
fn main(){
    let mut s = String::from("hello");
    s.push(' ');    // 使用單引號附加字元
    s.push_str("world");
    println!("{s}");
    
    let mut s = String::from("hello");
    let s2 = String::from("world");
    s.push(' ');    // 使用單引號附加字元
    s.push_str(&s2);    // 自動解引用轉為&str類別
    println!("{s}");
}
```

### 將String重新變成\&str

```rust
fn use_str(s: &str) {
    println!("I am: {}", s);
}
fn main() {
    let s = "Hello".to_string();
    // 先對String進行Deref,也就是*操作。
    // 因單獨的str是不能在Rust裡直接存在的，
    // 我們需要先給他進行&操作取得&str這個結果。
    use_str(&*s);
}
```

也可將`&*s`簡寫為`&s`，這其實是一個編譯器的特性，因為Rust的編譯器會在&後面插入足夠多的\*來儘可能滿足Deref這個特性。這個特性會在某些情況下失效。

### 自動解引用

String類型實現了`Deref<Target=str>`的trait。所以在很多情況下，`&String`類型可以被編譯器自動轉換為`&str`類型。

```rust
fn capitalize(substr: &mut str) {
    substr.make_ascii_uppercase();
}
fn main() {
    // s的類型為mut String
    let mut s = String::from("Hello World");
    // 傳進函數時， mut String自動轉為&mut str
    capitalize(&mut s);
    println!("{}", s);    // HELLO WORL
```

Rust的記憶體管理方式和C++有很大的相似之處。如果用C++來對比，Rust的String類型類似於std::string，而Rust的\&str類型類似於std::string\_view。

### 索引字串

在 Rust 中，如果你嘗試使用索引語法訪問 String 的一部分，會出現一個錯誤，<mark style="color:blue;">因為String是UTF-8陣列，每一個字元是變動長度，而非固定長度，因此一個字串字節值的索引並不總是對應一個有效的 Unicode 標量值</mark>。

另一個 Rust 不允許使用索引獲取 String 字元的原因是，<mark style="color:blue;">索引操作預期總是需要常數時間 (O(1))。但是對於 String 不可能保證這樣的效能，因為 Rust 必須從開頭到索引位置遍歷來確定有多少有效的字元</mark>。

解法：操作字串每一部分的最好的方法是明確表示需要字元還是字節。對於單獨的 Unicode 標量值使用 chars 方法。

```rust
fn main() {
    let s = "世界怎麼跟的上台灣".to_string();
    let s2 = &s[..3];   // 與預期不符，結果為世, 因為剛好為合法的utf-8字元，所以可切
    // let s2p = &s[..5];  //error, 沒有切在char boundary
    // let s3 = s[0];  // error, 不可用index
    let len = s.len();
    // 先轉為chars()後，再用nth取unicode字元，傳回的是Option，所以還要用unwrap取值
    println!("index");
    for idx in 0..len {
        print!("{} ", s.chars().nth(idx).unwrap());    
    }
    // note idx走訪完成後，iter已經到底了，要重新設定回起點
    
    // 也可直接走訪chars()的元素
    println!("iter");
    for c in s.chars(){
          print!("{} ", c);    
    }
    println!("direct");
    
    println!("{s}, {s2}");
}
```

如果頻繁調用`.chars().nth(n)` ，那麼效能將會降低到無法忍受的地步，此時我們可以考慮先整體解析一遍，儲存一個Vec，這樣我們就不需要.`chars().nth(n)`了。

```rust
fn main() {
    let mystring = "ABCD".to_string();
    // 先用vec將strings的字元儲存後再索引
    let mychars: Vec<char> = mystring.chars().collect(); // ['A', 'B', 'C', 'D']
    let mychar = mychars[1]; // 'B'
    println!("{:?}, {}", mychars, mychar);

    // 可以使用from_iter將vec轉回string
    let mystring = String::from_iter(mychars.iter()); // String "ABCD"，保留mychars
    let mystring2 = String::from_iter(mychars); // String "ABCD"，不保留mychars
    println!("{mystring}, {mystring2}");
}
```

### String和Vec\<u8>

如果你確信你要處理的String只有ASCII字元沒有擴展字元的話，那可以按字節來解析，或者你打算手寫UTF-8的解析，那你可以使用bytes系列的函數來產生一個Vec。

```rust
fn main() {
    let mystring = "ABCD".to_string();
    let mychars = mystring.into_bytes(); // Vec[b'A', b'B', b'C', b'D']
    println!("{:?}", mychars); // [65, 66, 67, 68]
}
```

bytes系列函數有：as\_bytes、bytes、into\_bytes，這三個函數各自特點如下：

* as\_bytes：借用內部Vec，返回&\[u8] 。
* bytes：借用內部Vec，返回Bytes（按字節迭代的迭代器）。
* into\_bytes：消耗String產生一個Vec\<u8, Global>。

從&\[u8]、Vec構造String的示例如下：

```rust
fn main() {
    // 從Vec<u8>構造
    let mystring = "ABCD".to_string();
    let mybytes = mystring.into_bytes(); // Vec[b'A', b'B', b'C', b'D']
    let mystring = String::from_utf8(mybytes).unwrap();
    // 從Bytes構造，其實就是構造Vec<u8>再調用from_utf8
    let mystring = "ABCD".to_string();
    let mybytes = mystring.bytes();
    let mystring = String::from_utf8(mybytes.collect()).unwrap();
    // 從&[u8]構造，其實就是構造Vec<u8>再調用from_utf8
    let mystring = "ABCD".to_string();
    let mybytes = mystring.as_bytes(); // &[b'A', b'B', b'C', b'D']
    let mystring = String::from_utf8(mybytes.into()).unwrap();
}
```

以上是從UTF-8編碼來構造String，其實還有from\_utf16、from\_utf16\_lossy。 從UTF-8編碼來構造也有別的方法：from\_utf8\_lossy、from\_utf8\_unchecked()。

## 字串切片(string slice)

字串 slice（string slice）是字串中一部分值的引用，沒有所有權。它不是對整個字串的引用，而是對部分字串的引用。

有兩種情況我們需要使用字串切片：

* 要麼創建一個對子字串的引用，
* 或者我們使用字串字面量(string literals)。

```rust
fn main() {
    let s = String::from("hello world");

    let hello = &s[0..5];    //hello 
    let world = &s[6..11];  // world
    println!("{hello}-{world}");
}
```

hello 是一個部分字串的引用，由一個額外的`[0..5]`部分指定。可以使用一個由中括號中的 \[starting\_index..ending\_index] 指定的 range 創建一個切片。

![hello world slice](../../.gitbook/assets/hello\_world\_slice-min.png)

## String，＆str，Vec 和＆\[u8]的慣用轉換

| From     | To       | Method                                                         |
| -------- | -------- | -------------------------------------------------------------- |
| \&str    | String   | String::from(s) or s.to\_string() or s.to\_owned()             |
| \&str    | &\[u8]   | s.as\_bytes()                                                  |
| \&str    | Vec\<u8> | s.as\_bytes().to\_vec() or s.as\_bytes().to\_owned()           |
| String   | \&str    | \&s if possible\* else s.as\_str()                             |
| String   | &\[u8]   | s.to\_vec() or s.to\_owned()                                   |
| String   | Vec\<u8> | s.into\_bytes()                                                |
| &\[u8]   | \&str    | s.to\_vec() or s.to\_owned()                                   |
| &\[u8]   | String   | std::str::from\_utf8(s).unwrap(), but don't\*\* (確定有資料再unwrap) |
| &\[u8]   | Vec\<u8> | String::from\_utf8(s).unwrap(), but don't\*\*                  |
| Vec\<u8> | \&str    | \&s if possible\* else s.as\_slice()                           |
| Vec\<u8> | String   | std::str::from\_utf8(\&s).unwrap(), but don't\*\*              |
| Vec\<u8> | &\[u8]   | String::from\_utf8(s).unwrap(), but don't\*\*                  |

## String與\&str的選用

設計函數時，需要考慮獲取所有權或是借用。

### 不需所有權

<mark style="color:red;">如果你不需要獲取資料的所有權，那麼將參數設定成引用類型</mark>，接下來的問題就變成了：是用`&str`還是`&String`？答案通常是`&str`。

```rust
// 在任何地方你需要一個&str，那麼你也可以直接無障礙的直接使用&String，
// 但反過來就不行了，如果你將函數的定義改成接收&String類型的變數，
// 那麼當你傳遞&str類型參數的時候編譯器就會報錯

fn print_str(msg: &str) {
    // &str可接受&String, 因為會自動dereference
    println!("{}", msg);
}

fn print_string(msg: &String) {
    // &String只能接受&String，不能使用&str
    println!("{}", msg);
}

fn main() {
    let string_slice = "String slice assigned to variable";
    let real_string = "Genuine String".to_owned();
    let my_string = "hello world".to_string();
    
    print_str(string_slice);
    print_str("Literal slice");
    print_str(&real_string);
    print_str(&my_string);
    
    
    // print_string(string_slice); // error
    // print_string("Literal slice"); // error
    print_string(&real_string);
    print_string(&my_string);
}
```

### 需要所有權

如果你需要一個有所有權的String，那麼需要考慮以下幾方面：

* 手動將\&str轉化為String類型是件麻煩事，這是設計函數的大障礙。
* 應該讓介面的使用者決定如何建立有所有權的資料，你不要簡單地到處接收\&str類型的變數然後自己去轉換它們。
