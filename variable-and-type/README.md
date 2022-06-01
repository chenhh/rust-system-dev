# 變數與類型

## 變數宣告

Rust的變數必須先聲明後才能使用，<mark style="color:red;">預設是唯讀不可寫入</mark>，與C/C++中變數預寫為可讀寫不同，定義可寫入的變數要用`let mut`宣告。

Rust 通過靜態類型確保類型安全。變量綁定可以在聲明時說明類型，不過在多數情況下， 編譯器能夠從上下文推導出變量的類型，從而大大減少了類型說明的工作。

* 使用 let 綁定操作可以將值（比如字面量）綁定（bind）到變量。
* 變量綁定默認是不可變的（immutable），但加上 mut 修飾語後變量(等號左邊)就可以改變。
*   可以先聲明（declare）變量綁定，後面才將它們初始化（initialize）。但是這種做法很少用，因為這樣可能導致使用未初始化的變量。





```rust
fn main() {
    let an_integer = 1u32;    // 將數字1綁定至唯讀變數an_integer
    let a_boolean = true;     // 將布林true綁定至唯護變數a_boolean 
    let unit = ();            // empty tuple

    // 将 `an_integer` 複製到 `copied_integer`，因為基本型別有實現copy trait
    let copied_integer = an_integer;

    println!("An integer: {:?}", copied_integer);
    println!("A boolean: {:?}", a_boolean);
    println!("Meet the unit value: {:?}", unit);

    // 編譯器會讓未使用的變數產生警告，可以變數加下劃線消除警告
    let _unused_variable = 3u32;

    let noisy_unused_variable = 2u32;
    // 改正 ^ 在变量名前加上下划线以消除警告
}
```

```rust
fn main() {
   let x: i32 = 100;  // type i32可省略，由編譯器自動推斷type
   //x = 80;          // error, x不可變
   println!("x={}", x);
}

fn main() {
    let x; // type i32可省略，由編譯器自動推斷type
    x = 100; // 此處為初始化，而非賦值
    //x = 80;  // error, x不可變
    println!("x={}", x);
}

/* Rust中，每個變數必須被合理初始化之後才能被使用。
 * 使用未初始化變數這樣的錯誤，在Rust中是不可能出現的。
 */
fn main() {
    let x: i32;        // 未初始化的變數
    println!("{}", x); // compile error
}
```

區域變數聲明一定是以關鍵字let開頭，類型一定是跟在冒號：的後面。語法歧義更少，語法分析器更容易編寫。

因為在變數聲明語句中，最重要的是變數本身，而類型其實是個附屬的額外描述，並非必不可少的部分。<mark style="background-color:blue;">**如果我們可以通過上下文環境由編譯器自動分析出這個變數的類型，那麼這個類型描述完全可以省略不寫**</mark>。Rust一開始的設計就考慮了類型自動推導功能，因此類型後置的語法更合適。

<mark style="background-color:red;">類型沒有“預設構造函數”，變數沒有“預設值”。</mark>對於`let x：i32`；如果沒有顯式賦值，它就沒有被初始化，不要想當然地以為它的值是0。

```rust
fn main() {
    let mut x = 100; // 關鍵字mut說明x可讀寫
    x = 80;          // 改變x之值, i32有實作copy trait，因此為copy而非move
    println!("x={}", x); //80
}

fn main() {
    let (mut a, mut b) = (1, 2); //pattern destructure
    a = 3;
    b = 4;
    println!("a={}, b={}", a, b); //a=3, b=4
}
```

如果我們需要讓變數是可寫的，那麼需要使用`mut`關鍵字，我們不能把let mut視為一個組合，而應該將`mut x`視為一個組合。

<mark style="background-color:red;">Rust內的型別如果沒有實作copy trait時(只有基礎型別可實作)，賦值運算子</mark><mark style="background-color:red;">`=`</mark><mark style="background-color:red;">預設為move語意</mark>。

Rust裡面的底線`_`是一個特殊的識別字，在編譯器內部它是被特殊處理的。它跟其他識別字有許多重要區別。

### 印出變數的記憶體位址

```rust
fn main() {
    let data: u32 = 42;
    let copy_data = data;
    let borrow_data = &data;

    // the {:p} mapping shows pointer values as hexadecimal memory addresses
    println!("Data address: {:p}", &data);
    println!("Copy data address: {:p}", &copy_data); // 與data不同，因為為copy而非move
    println!("Borrow data address: {:p}", &borrow_data); // borrow_data變數的地址
    println!("Borrow data storage address: {:p}", borrow_data); // borrow_data變數儲存之值的地址，與data相同
}
```

## 變數遮蔽(variable shadowing)

Rust允許在同一個程式碼塊中聲明同樣名字的變數。如果這樣做，後面聲明的變數會將前面聲明的變數“遮蔽”（Shadowing）起來。**但被遮蔽的變數的生命週期不受影響，即不會因為被遮蔽而提早結束**。

```rust
fn main() {
    let x = "hello";        //唯讀的變數, 型別為&str
    println!("x is {}", x); // hello
    let x = 5;              // shadowing而非賦值, 可視為重新宣告一個同名的變數
    println!("x is {}", x); // 5
}

/* 由可寫入變數變為唯讀變數 */
fn main() {
    let mut v = Vec::new(); // v 必須是mut修飾,因為我們需要對它寫入資料
    v.push(1);
    v.push(2);
    v.push(3);
    let v = v; // 從這裡往下,v成了唯讀變數,可讀寫變數v已經被遮蔽,無法再訪問
    for i in &v {
        println!("{}", i);
    }
}

/* 由唯讀變數變為可寫入變數 */
fn main() {
    let v = Vec::new();
    let mut v = v;  // 由唯讀變為可寫入
    v.push(1);
    println!("{:?}", v);
}
```

但是這兩個`x`代表的記憶體空間完全不同，類型也完全不同，它們實際上是兩個不同的變數。從第4行開始，一直到這個程式碼塊結束，我們沒有任何辦法再去訪問前一個`x`變數，因為它的名字已經被遮蔽了。

<mark style="background-color:blue;">變數遮蔽在某些情況下非常有用，比如，我們需要在同一個函數內部把一個變數轉換為另一個類型的變數，但又不想給它們起不同的名字</mark>。再比如，在同一個函數內部，需要修改一個變數綁定的可變性。

C/C++中也存在類似的功能，只不過它們只允許嵌套的區域內部的變數出現遮蔽。而Rust在這方面放得稍微寬一點，同一個語句塊內部聲明的變數也可以發生遮蔽。

### 凍結

<mark style="color:red;">當資料被相同的名稱不變地綁定時，它還會凍結（freeze）</mark>。在不可變綁定超出作用域之前，無法修改已凍結的資料。

```rust
fn main() {
    let mut _mutable_integer = 7i32;

    {
        // 被不可變的 `_mutable_integer` 遮蔽
        let _mutable_integer = _mutable_integer;

        // 報錯！`_mutable_integer` 在本作用域被凍結
        _mutable_integer = 50;
        // 改正 ^ 注釋掉上面這行

        // `_mutable_integer` 離開作用域
    }

    // 正常執行！ `_mutable_integer` 在這個作用域沒有凍結
    _mutable_integer = 3;
}
```

類型推導
----

Rust的類型推導功能是比較強大的。它不僅可以從變數聲明的當前語句中獲取資訊進行推導，而且還能通過上下文資訊進行推導。

自動類型推導和“動態類型系統”是兩碼事。**Rust依然是靜態類型的**。一個變數的類型必須在編譯階段確定，且無法更改，只是某些時候不需要在程式碼中顯式寫出來而已。

<mark style="color:red;">**Rust只允許“區域變數/全域變數”實現類型推導，而函數簽名等場景下是不允許的**</mark>，這是故意這樣設計的。這是因為區域變數只有局部的影響，全域變數必須當場初始化，而函數簽名具有全域性影響。函數簽名如果使用自動類型推導，可能導致某個調用的地方使用方式發生變化，它的參數、返回數值型別就發生了變化，進而導致遠處另一個地方的編譯錯誤，

```rust
fn main() {
    // 沒有明確標出變數的類型,但是通過字面量的尾碼,
    // 編譯器知道elem的類型為u8
    let elem = 5u8; // 也可寫5_u8, 底線不影響數值定義
                    // 創建一個動態陣列,陣列內包含的是什麼元素類型可以不寫
    let mut vec = Vec::new();
    vec.push(elem);
    // 到後面調用了push函數,通過elem變數的類型,
    // 編譯器可以推導出vec的實際類型是 Vec<u8>
    println!("{:?}", vec);
}
```

我們甚至還可以只寫一部分類型，剩下的部分讓編譯器去推導，比如下面的這個程式，我們只知道players變數是Vec動態陣列類型，但是裡面包含什麼元素類型並不清楚，可以在尖括弧中用底線來代替。

```rust
fn main() {
    let player_scores = [("Jack", 20), ("Jane", 23), ("Jill", 18), ("John", 19)];
    // players 是動態陣列,內部成員的類型沒有指定,交給編譯器自動推導
    let players: Vec<_> = player_scores
        .iter()
        .map(|&(player, _score)| player)
        .collect();
    println!("{:?}", players);
}
```

## 類型別名(type alias)

可以用type關鍵字給同一個類型起個別名（type alias），可用在泛型上。

```rust
type Age = u32; // Age為u32的別名, 兩者等價

// 小括弧包圍的是一個 tuple, 
// 那麼以後使用Double<i32>的時候，就等同於（i32，Vec<i32>）
type Double<T> = (T, Vec<T>); 


fn grow(age: Age, year: u32) -> Age {
    age + year
}
fn main() {
    let x: Age = 20;
    println!("20 years later: {}", grow(x, 20));   
}
```

## 靜態變數

Rust中可以用static關鍵字聲明靜態變數。

靜態變數的生命週期(`'static`)從程式開始到程式結束，它佔用的記憶體空間也不會在執行程式過程中回收。<mark style="color:red;">這也是Rust中唯一的聲明全域變數的方法</mark>。

由於Rust非常注重記憶體安全，因此全域變數的使用有許多限制。這些限制都是為了防止程式設計師寫出不安全的程式碼：

* 全域變數必須在聲明的時候馬上初始化；
* 全域變數的初始化必須是編譯期可確定的常量，不能包括執行期才能確定的運算式、語句和函式呼叫；
* 帶有mut修飾的全域變數，在使用的時候必須使用unsafe關鍵字。

```rust
// 靜態變數必須顯式定義型別, 不可省略
// 且在宣告時必須馬上初始化
static GLOBAL: i32 = 0;
static mut MUT_GLOBAL: i32 = 0;
fn main() {
    println!("GLOBAL: {}", GLOBAL);
   
    unsafe{
        // 修改或讀取可變的全域變數必須在unsafe區間
        println!("MUT_GLOBAL:{}", MUT_GLOBAL);
        MUT_GLOBAL = 12;
        println!("MUT_GLOBAL:{}", MUT_GLOBAL);
    }
}

// 全域變數的記憶體不是分配在當前函數堆疊上,
// 函數退出的時候,並不會銷毀全域變數佔用的記憶體空間,
// 程式退出才會回收
```

Rust禁止在聲明static變數的時候調用普通函數，或者利用語句塊調用其他非const程式碼，而調用const fn是允許的。

```rust
fn main() {
    // 可確定的常量才可宣告為static
    static array: [i32; 3] = [1, 2, 3];
    // 這樣是不允許的，因為vec長度會變動
    //static vec : Vec<i32> = { let mut v = Vec::new(); v.push(1); v };
}
```

## 常數(const)

Rust 有兩種常量，可以在任意作用域聲明，包括全域性作用域。它們都需要顯式的類型聲明：

* <mark style="color:red;">const：不可改變的值（通常使用這種）。</mark>
* <mark style="color:red;">static：具有 'static 生命週期的，可以是可變的變量（須使用 static mut 關鍵字）</mark>。

使用const聲明的是常數，而不是變數。因此一定不允許使用mut關鍵字修飾這個變數綁定，這是語法錯誤。

常量的初始設定式也一定要是一個編譯期常數，不能是運行期的值。<mark style="color:red;">它與static變數的最大區別在於：</mark><mark style="color:red;">**編譯器並不一定會給const常量分配記憶體空間**</mark>，在編譯過程中，它很可能會被內聯優化。以const聲明一個常量，也不具備類似let語句的模式匹配功能。

```rust
fn main() {
    // 必須顯式指定型別，不可省略
    const GLOBAL: i32 = 0;
}
```

```rust
// 全域變數是在在所有其他作用域之外聲明的。
static LANGUAGE: &'static str = "Rust";
const  THRESHOLD: i32 = 10;

fn is_big(n: i32) -> bool {
    // 在一般函數中訪問常量
    n > THRESHOLD
}

fn main() {
    let n = 16;

    // 在 main 函數（主函數）中訪問常量
    println!("This is {}", LANGUAGE);
    println!("The threshold is {}", THRESHOLD);
    println!("{} is {}", n, if is_big(n) { "big" } else { "small" });

    // 報錯！不能修改一個 `const` 常量。
    // THRESHOLD = 5;
    // 改正 ^ 注釋掉此行
}
```

## 參考資料

