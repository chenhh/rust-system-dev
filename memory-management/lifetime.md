# 生命週期(lifetime)

## 簡介

借出一個指向他人擁有的資源的引用是很複雜的。 舉例來說，想像一下以下的操作行為：

1. 我獲得一個某種資源的控制代碼（handle）。
2. 我把指向這個資源的引用借給你。
3. 我決定不再使用這個資源，釋放它，但你仍還有你那邊的引用。
4. 你決定要使用此資源(但引用指向的資源已經無效)。

你的引用指向了一個無效的資源。 如果資源是記憶體時，我們叫dangling pointer或「釋放後使用」。

 <mark style="color:red;">所有的變數都有生命週期 (lifetime)，而參考型別帶來最困難的問題，就是如何避免懸置參考 (dangling reference)，亦即參考所指向的物件已經消失，但是參考本身仍然可以存取的現象</mark>。

要修正這個問題，我們必須確保上述第四步不會在第三步之後發生。 Rust 的所有權系統透過一個叫做生命週期（lifetimes）的概念達成這點，它會描述引用的有效範圍。

**Rust 中的每一個引用(reference)都有其 生命週期（lifetime）**，也就是引用保持有效的作用域。大部分時候生命週期是隱含並可以推斷的，正如大部分時候類型也是可以推斷的一樣。類似於當因為有多種可能類型的時候必須註明類型，也會出現引用的生命週期以一些不同方式相關聯的情況，所以 Rust 需要我們使用泛型生命週期參數來註明他們的關系，這樣就能確保運行時實際使用的引用絕對是有效的。

### 生命週期要點

* 對於生命週期<mark style="color:red;">全都是關於引用（references）的</mark>，與其他變數無關。
* 生命週期參數是一種標記，類似於類型限定。
* 生命週期幫助編譯器執行一個簡單的規則：<mark style="background-color:red;">**引用不應該活得比所指對象長**</mark><mark style="background-color:red;">(no reference should outlive its referent)</mark>。
* 生命週期參數的功能是向編譯器**告知(讓編譯器檢查是否合法)**引用之間的drop關係，而非擴展或延長引用的存在時間，因此在編譯過程中，編譯器會根據生命週期參數檢查傳入的參數是否合法。
* 為了能夠進行借用規則檢查，編譯器需要知道所有引用的生命週期。<mark style="color:blue;">在很多情況下，編譯器能夠自己推導出生命週期，但是有些情況它無法完成，這就需要開發者手動的對生命週期進行標注(註：只有在編譯器無法推論生命週期時，才須手動標註)</mark>。

## 作用域（Scope）

所有權的規則，是依賴於作用域的。

隱式作用域就是在當前作用域中由let開啟的作用域。在Rust中，也有一些特殊的巨集，比如`println!()`，也會產生一個預設的作用域，並且會隱式借用變數。除此之外，更明顯的作用域 範圍則是函式，也就是說，一個函式本身，就是一個顯式的作用域。你也可以使用一對花括號（`{}`）來建立顯式的作用域。

除此之外，一個函式本身就顯式的開闢了一個獨立的作用域。

```rust
// 參數的所有權move，但a,b為基礎類別有實作copy trait,
// 因此最後是copy
fn sum(a: u32, b: u32) -> u32 {
      a + b
 }

 fn main(){
     let a = 1;
     let b = 2;
     // a和b當作引數傳遞過去，此時就會發生所有權move的行為，
     // 但是因為a和b都是基本資料型別，實現了Copy Trait，
     // 所以它們的所有權沒有被轉移。
     sum(a, b);
 }
```

<mark style="background-color:blue;">作用域在Rust中的作用就是製造一個邊界，這個邊界是所有權的邊界。變數走出其所在作用域，所有權會move。</mark>如果不想讓所有權move，則可以使用“引用”來“出借”變數，而此時作用域的作用就是保證被“借用”的變數準確歸還。

引用非常方便我們使用，但是如果濫用的話，會引起安全問題，比如懸垂指標。

```rust
fn main() {
    let r;
    {
        let a = 1;
        // a離開作用域就會被釋放
        // r在離開作用域後，會指向已釋放的物件
        r = &a; //  ^^ borrowed value does not live long enough
    }
    println!("{r}");
}
```

## 生命週期(lifetime)

一個變數的生命週期就是它從創建到銷毀的整個過程。<mark style="color:red;">生命週期在必要時可以顯式標記(通常在函式傳參數時)</mark>。

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5]; // --> v 的生命週期開始
    {
        let center = v[2];       // --> center 的生命週期開始
        println!("{}", center);
    }                            // <-- center 的生命週期結束
    println!("{:?}", v);
}                                // <-- v 的生命週期結束
```

同一個函式作用域下，編譯器可以識別出生命週期的問題，但是當我們在函式之間傳遞引用的時候，編譯器就很難自動識別出這些問題了，所以Rust要求我們為這些引用顯式的指定生命週期標記，如果你不指定生命週期標記，那麼編譯器將會“鞭策”你。

## 生命週期標記

對一個函數內部的生命週期進行分析，Rust編譯器可以很好地解決。但是，當生命週期跨函數的時候，就需要一種特殊的生命週期標記符號了。

生命週期是用`'`符號，和泛型一樣都是放在`<>`中。

```rust
struct Foo<'a> {
// 加上生命週期標記以後，
// 編譯器中的借用檢查器就會幫助我們自動比對引數變數的作用域長度，
// 確保記憶體安全。
    x: &'a i32,
}

fn main() {
    // y這個借用被傳遞到了 let f = Foo { x: y }所在作用域中。
    // 所以需要確保借用y活得比Foo結構體例項長才行，
    // 否則，如果借用y被提前釋放，Foo結構體例項就會造成懸垂指標了。
    let y = &5;
    let f = Foo { x: y };

    println!("{}", f.x);
}
```

## 函數的生命週期標記

```rust
struct T {
    member: i32,
}

// 生命週期的標註是在引用符號後面
fn test<'a>(arg: &'a T) -> &'a i32 {
    &arg.member
}

fn main() {
    let t = T { member: 0 }; //----- 't -|
    let x = test(&t);        //-- 'x ---| |
    println!("{:?}", x);     // | |
}
```

生命週期符號使用單引號開頭，後面跟一個合法的名字。生命週期標記和泛型類型參數是一樣的，都需要先聲明後使用。在上面這段程式碼中，尖括弧裡面的`'a`是聲明一個生命週期參數，它在後面的參數和返回值中被使用。

借用指標類型都有一個生命週期泛型參數，它們的完整寫法應該是`&'a T&'a mut T`，只不過<mark style="background-color:blue;">在做區域變數的時候，生命週期參數是可以省略的</mark>。

生命週期之間有重要的包含關係。如果生命週期`'a`比`'b`更長或相等，則記為`'a:'b`，意思是`'a`至少不會比`'b`短，英語讀做“lifetime a outlives lifetime b”。對於借用指標類型來說，如果`&'a`是合法的，那麼`'b`作為`'a`的一部分，`&'b`也一定是合法的。

<mark style="background-color:orange;">註：可用trait的繼承類比，</mark><mark style="background-color:orange;">`trait Derived: Base{}`</mark><mark style="background-color:orange;">，可把</mark><mark style="background-color:orange;">`'a:'b`</mark><mark style="background-color:orange;">中，</mark><mark style="background-color:orange;">`'a`</mark><mark style="background-color:orange;">做為</mark><mark style="background-color:orange;">`'b`</mark><mark style="background-color:orange;">的延伸，因此生命週期更長一點</mark>。

生命週期的省略規則：在涉及到函數時，編譯器不進行生命週期的推導。而某些固定形式的函數簽名所需要標注的生命週期參數是確定的。所以當函數簽名是給定的形式時，Rust允許編寫者按照省略規則，隱式的省略生命週期參數。

## static生命週期

另外，`'static`是一個特殊的生命週期，它代表的是這個程式從開始到結束的整個階段，所以它比其他任何生命週期都長。這意味著，任意一個生命週期`'a`都滿足`'static：'a`。

```rust
// 函數標記x,y的生命週期一樣長
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
fn main() {
    let a = "hello";
    let result;
    {
        let b = String::from("world");
        // error[E0597]: `b` does not live long enough
        // 使用函數時, a的生命週期比b長，與函數的宣告不符
        result = longest(a, b.as_str());
    }
    println!("The longest string is {}", result);
}
```

* result是引用，但它可能「引用自」兩個生命週期不同的變數，要給result何種生命週期?
* 很明顯result的生命週期取決於longest的邏輯。然而一個函數的邏輯可能非常複雜，編譯器難以推斷返回值和傳入參數的關係。
* 所以Rust要求程式設計師顯示標記函數的返回值的生命週期與傳入的參數生命週期間的關係。

## 類型的生命週期標記

如果自訂類型中有成員包含生命週期參數，那麼這個自訂類型也必須有生命週期參數。

```rust
struct Test<'a> {
    member: &'a str,
}
```

在使用impl的時候，也需要先聲明再使用：

```rust
impl<'t> Test<'t> {
    fn test<'a>(&self, s: &'a str) {}
}
```

impl後面的那個`'t`是用於聲明生命週期參數的，後面的`Test<'t>`是在類型中使用這個參數。如果有必要的話，方法中還能繼續引入新的泛型參數。

如果在泛型約束中有`where T:'a`之類的條件，其意思是，類型`T`的所有生命週期參數必須大於等於`'a`。要特別說明的是，若是有`where T:'static`的約束，意思則是，類型`T`裡面不包含任何指向短生命週期的借用指標，意思是要麼完全不包含任何借用，要麼可以有指向`'static`的借用指標。

## 省略生命週期標記

在某些情況下，Rust允許我們在寫函數的時候省略掉顯式生命週期標記。在這種時候，編譯器會通過一定的固定規則為參數和返回值指定合適的生命週期，從而省略一些顯而易見的生命週期標記。

```rust
fn get_str(s: &String) -> &str {
    s.as_ref()
}

// 顯式標記生命週期
fn get_str<'a>(s: &'a String) -> &'a str {
    s.as_ref()
}
```

編譯器對於省略掉的生命週期，不是用的“自動推理”策略，而是用的幾個非常簡單的“固定規則”策略。這跟類型自動推導不一樣，當我們省略變數的類型時，編譯器會試圖通過變數的使用方式推導出變數的類型，這個過程叫“type inference”。

而對於省略掉的生命週期參數，編譯器的處理方式就簡單粗暴得多，它完全不管函數內部的實現，並不嘗試找到最合適的推理方案，只是應用幾個固定的規則而已，這些規則被稱為**“lifetime elision rules”**。以下就是省略的生命週期被自動補全的規則：

* 每個帶生命週期參數的輸入參數，每個對應不同的生命週期參數；
* 如果只有一個輸入參數帶生命週期參數，那麼返回值的生命週期被指定為這個參數；
* 如果有多個輸入參數帶生命週期參數，但其中有`&self`、`&mut self`，那麼返回值的生命週期被指定為這個參數；
* 以上都不滿足，就不能自動補全返回值的生命週期參數。

```rust
fn get_str(s: &String) -> &str {
    println!("call fn {}", s);
    "hello world"
}
// 編譯器會自動補全生命週期參數
fn get_str<'a>(s: &'a String) -> &'a str {
    println!("call fn {}", s);
    "hello world"
}
```

## 參考資料

* [\[知乎\] 如何理解Rust中的生命週期標注？](https://www.zhihu.com/question/435470652)
* [\[知乎\] 深入理解Rust中的生命週期](https://zhuanlan.zhihu.com/p/342405519)
* [\[知乎\] Rust中的生命週期](https://zhuanlan.zhihu.com/p/191457302)
