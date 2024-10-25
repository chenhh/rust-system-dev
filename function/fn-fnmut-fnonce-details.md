# Fn, FnMut, FnOnce的區別

## 簡介

在Rust裡，閉包被分為了三種類型，列舉如下

* [Fn(\&self)](https://doc.rust-lang.org/std/ops/trait.Fn.html)。代表不可變借用的閉包，可重複執行。
* [FnMut(\&mut self)](https://doc.rust-lang.org/std/ops/trait.FnMut.html)。FnMut 代表閉包可變引用修改了變數，可重複執行。
* [FnOnce(self)](https://doc.rust-lang.org/std/ops/trait.FnOnce.html)。只能執行一次，一旦呼叫閉包將喪失所有權了。

## 標準庫的定義

```rust
pub trait Fn<Args>: FnMut<Args>
where
    Args: Tuple,
{
    // Required method
    extern "rust-call" fn call(&self, args: Args) -> Self::Output;
}

pub trait FnMut<Args> : FnOnce<Args> {
    extern "rust-call" fn call_mut(&mut self, args: Args) -> Self::Output;
}

pub trait FnOnce<Args> {
    type Output;

    extern "rust-call" fn call_once(self, args: Args) -> Self::Output;
}
```

上面是標準庫中，Fn, FnMut, FnOnce 的實現。可以看到 Fn 繼承自 FnMut, FnMut 繼承自 FnOnce。

## 繼承關係

<mark style="color:red;">由於 Fn 是繼承自 FnMut，那麼可以把實現 Fn 的閉包傳給 FnMut</mark>。

```rust
fn test<T>(mut f: T)
where
    T: FnMut(),
{
    f();
    f();
}

fn main() {
    let s = String::from("hello wrold");
    let f = || {
        println!("{s}");
    };
    test(f);
}
```

## move語句的判定

Fn/FnMut/FnOnce三父子是閉包，所謂閉包是有一塊記憶體區域，儲存著捕獲變數，捕獲變數是按不可變引用，還是按可變引用，或者所有權轉移的方式，對閉包的特性起著很大的決定性作用。

然後，複製語義和移動語義的所有權轉移的影響是不同的，有時候轉移了所有權閉包反而更自由了。最後，閉包捕獲變數儲存區的特性是確定了，就看閉包行為如何使用這些閉包捕獲變數了。

* 如果是**不可變引用**的方式捕獲的，那肯定是Fn了。
* 如果是**可變引用**捕獲的，可能是FnMut，也可能是Fn，得再看閉包行為：
  * 如果閉包行為只是“不可變引用”式的使用捕獲變數，那還是Fn（話說回來，這就退化成不可變引用捕獲了）。
  * 如果閉包行為改變了捕獲變數，那就是FnMut。
*   如果是**所有權轉移**捕獲的，可能是FnOnce，也可能是FnMut，也可能是Fn：

    * 如果捕獲的是複製語義的變數，是Fn。
    * 如果捕獲的是移動語義的變數，在看閉包行為：
      * 如果閉包行為沒消費轉移走所有權，那就還是Fn/FnMut。
      * 如果閉包行為消費轉移走了所有權，那才是FnOnce。

    <mark style="color:red;">所有權的捕獲方式，並非move關鍵字就能決定的，這是迷惑之一。沒move關鍵字，閉包怎麼個捕獲的規則，也是迷惑之一</mark>。

#### 若沒有move關鍵字，自動判定變數的捕獲方式根據如下規則：

1. 若捕獲變數是複製語義，即實現了Copy，比如i32，則按引用捕獲，可變引用還是不可變引用，看閉包對該變數的使用行為。
2.  若捕獲變數是移動語義，即沒實現Copy，再看閉包行為：

    2.1  若是沒消費變數所有權，就算是改變了變數，都按引用捕獲。\
    2.2. 若是消費了變數所有權，則按所有權捕獲。

從規則1和2.1看，<mark style="color:red;">沒move關鍵字，不輕易捕獲所有權，這是編譯器預設行</mark>為。

### 誤解一：認為沒move關鍵字，就一定是引用捕獲

本身捕獲也是隱式捕獲，如果沒用move關鍵字，那就是引用捕獲？那可不一定，move關鍵字在閉包前的真正作用是強制採用所有權捕獲，關鍵在強制二字，相當於顯式地告知要move；反過來講，如果沒有move關鍵字，也有可能是所有權捕獲的。

```rust
let s = "hello".to_string();
let c = || {
    println!("s is {:?}", s);
    s // 有這行後，主動對s的所有權進行了消費轉移，就對s進行了所有權捕獲；
      // 沒這行則會變成不可變引用捕獲
};ㄐㄧㄐㄧㄧ
```

### 誤解二：認為有了move關鍵字，所有權捕獲型的，就一定是FnOnce

不一定，還得看捕獲的類型，複製語義和移動語義的大不相同：

* 如果move的是一個i32，那閉包裡存個i32，複製語義的，多次呼叫也沒問題，儼然一個Fn。
* 如果move的本身是一個引用，那就看閉包行為會不會改變它，改變它了，是一個FnMut。

```rust
let mut x = 5;
// 有move卻是複製捕獲，閉包更自由了，能多次呼叫還是隨便Copy，Fn
let mut c1 = move || x += 1;  
// 沒move，默認可變引用捕獲，FnMut
let mut c2 = || x += 1; 
```

如果move的是一個所有權對象，如String，那也得看閉包行為怎麼用這個對象，是唯讀式的，還是消費掉所有權。

```rust
let s = "hello".to_string();
// 雖然是所有權捕獲，但依然可以多次呼叫，Fn
let c = move || println!("s is {:?}", s); 
c();
c();
```

### 誤解三：對閉包變數使用mut修飾，就認為是FnMut

```rust
let s = "hello".to_string();
// 除了編譯器會提供一個多餘的mut告警，c還是Fn
let mut c = || println!("s is {:?}", s);  
```

### 誤解四：認為Fn/FnMut/FnOnce跟Copy/Clone有關係

Fn/FnMut/FnOnce只是限定了可否重複呼叫，以何種方式呼叫，即呼叫時是否改變了閉包儲存區的狀態。至於能不能Copy/Clone，是另外一回事。

Fn/FnMut/FnOnce是否實現了Copy/Clone，完全看閉包捕獲變數儲存區的特徵，把它當成一個結構體，就清楚了。

```rust
let s = "hello".to_string();
// Fn，但沒有實現Copy，因閉包捕獲變數儲存區有所有權
let c1 = move || println!("{s}");  

let mut x = 5;
// FnMut，但沒實現Copy，因為可變引用，要滿足**可變不共享**規則
let mut c = || x += 1; 
```

### 誤解五：認為能夠滿足FnOnce限定的就一定是FnOnce

有時函數參數要求是一個FnOnce的閉包，一看能傳遞進去當參數，就以為該閉包是FnOnce，其實並不是。

本文一開始稱Fn/FnMut/FnOnce為三父子，就是他們之間的“繼承”關係：Fn 繼承於 FuMut 繼承於 FnOnce，所以：

任何閉包，都可以滿足FnOnce限定，因為Fn/FnMut都可以呼叫多次，更不怕FnOnce只呼叫一次了。只是以FnOnce呼叫一次，會消耗掉閉包所有權，只有Fn閉包必須是具備Copy/Clone特質的才可能沒丟掉所有權！

```rust
// 以FnOnce的方式，呼叫FnMut的內部實現模擬
fn call_once(self, ...) {
    call_mut(&mut self, ...);
}
```

### 多變數捕獲

如果捕獲多個變數，還使用move關鍵字，那所有變數都被轉移了所有權，情況正慢慢地變得不可控。。但又只想一部分變數轉移所有權進閉包，一部分是可變引用捕獲，一部分是不可變引用捕獲，又該如何?

解決方案：顯式地先聲明好變數的引用，再move進去這些引用，可保變數不全給轉移所有權進去。

```rust
let s1 = "hello ".to_string();
let s2 = "world".to_string();

// 關鍵一句，不像C++ lambda可以主動選擇怎麼捕獲，只能在閉包前先聲明好了
let s2_ref = &s2;  
let c = move || s1 + s2_ref;
```



## 函數指標與閉包

```rust
fn call(f: fn()) {
    // function pointer
    f();
}

fn main() {
    let a = 1;

    let f = || println!("abc"); // anonymous function
    let c = || println!("{}", &a); // closure

    call(f);
    // call(c);
}

```





## 參考資料

* [https://dengjianping.github.io/2019/03/05/%E8%B0%88%E4%B8%80%E8%B0%88Fn,-FnMut,-FnOnce%E7%9A%84%E5%8C%BA%E5%88%AB.html](https://dengjianping.github.io/2019/03/05/%E8%B0%88%E4%B8%80%E8%B0%88Fn,-FnMut,-FnOnce%E7%9A%84%E5%8C%BA%E5%88%AB.html)
* [https://rustcc.cn/article?id=8b6c5e63-c1e0-4110-8ae8-a3ce1d3e03b9](https://rustcc.cn/article?id=8b6c5e63-c1e0-4110-8ae8-a3ce1d3e03b9)
