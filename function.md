# 函數

## 函數結構

Rust的函數使用關鍵字fn開頭。函數可以有一系列的輸入參數，還有一個返回類型。函數體包含一系列的語句（或者運算式）。函數返回可以使用return語句，也可以使用運算式。

```rust
// 函數名稱為add1, 參數t的類型為tuple (i32, i32)
// 函數的回傳類型為i32
// 函數內容最後一行為不加分號的表達式，為回傳值
fn add1(t: (i32, i32)) -> i32 {
    t.0 + t.1
}

// 參數也可在傳入時直接解構
fn add2((x, y): (i32, i32)) -> i32 {
    x + y
}

// Rust編寫的可執行程式的入口就是fn main()函數。
fn main() {
    let p = (1, 3);
    // 函數名稱可設定給變數使用
    // func 是一個區域變數
    let func = add2;
    // func 可以被當成普通函數一樣被調用
    println!("evaluation output {}", func(p)); // 4
}
```

函數體內部是一個運算式，這個運算式的值就是函數的返回值。也可以寫`return x+y；`這樣的語句作為返回值，效果是一樣的。

函數也可以不寫返回類型，在這種情況下，編譯器會認為返回類型是`unit()`。

## 函數簽名

在Rust中，每一個函數都具有自己單獨的類型，但是這個類型可以自動轉換到fn類型。雖然add1和add2有同樣的參數類型和同樣的返回數值型別，但它們是不同類型，所以這裡報錯了。

修復方案是讓func的類型為通用的fn類型即可。但是，我們不能在參數、返回數值型別不同\(參數與返回數值必須都是相同的類型\)的情況下作類型轉換。

```rust
fn add1(t: (i32, i32)) -> i32 {
    t.0 + t.1
}
fn add2((x, y): (i32, i32)) -> i32 {
    x + y
}
// 傳入參數不是tuple，不能轉換
fn add3(x: i32, y: i32) -> i32 {
    x + y
}
fn main() {
    // 原始寫法：先讓 func 指向 add1, 無法轉成add2
    // let mut func = add1;
    
     // 寫法一,用 as 類型轉換
    let mut func = add1 as fn((i32,i32))->i32;
    // 寫法二,用顯式類型標記
    // let mut func : fn((i32,i32))->i32 = add1;
    
    // 再重新賦值,讓 func 指向 add2, 只有寫法1,2能使用
    // different `fn` items always have unique types,
    // even if their signatures are the same
    func = add2;
    
    // 傳入的參數類型不相同，無法轉換
    //func = add3;
    
    println!("{}", func((1,2)));
}
```

## 函數內可定義元件

Rust的函數體內也允許定義其他item，包括靜態變數、常量、函數、trait、類型、模組等。

當你需要一些item僅在此函數內有用的時候，可以把它們直接定義到函數體內，以避免污染外部的命名空間。

```rust
fn test_inner() {
    // 定義靜態變數
    static INNER_STATIC: i64 = 42;
    // 函數內部定義的函數
    fn internal_incr(x: i64) -> i64 {
        x + 1
    }
    struct InnerTemp(i64);
    impl InnerTemp {
        fn incr(&mut self) {
            self.0 = internal_incr(self.0);
        }
    }
    // 函數體,執行語句
    let mut t = InnerTemp(INNER_STATIC);
    t.incr();
    println!("{}", t.0);
}
fn main() {
    test_inner();
}
```

## 發散函數\(diverging function\)

Rust支援一種特殊的發散函數（Diverging functions），它的返回類型是驚嘆號！。如果一個函數根本就不能正常返回，那麼它可以這樣寫：

```rust
// 若函數必定無法正常返回時，回傳值為發散函數
fn diverges() -> ! {
    panic!("This function never returns!");
}
```

因為panic！會直接導致堆疊展開，所以這個函式呼叫後面的程式碼都不會繼續執行，它的返回類型就是一個特殊的！符號，這種函數也叫作發散函數。

發散類型的最大特點就是，它可以被轉換為任意一個類型，因此可以存在於任何編譯器檢查類型需要匹配的語句，例如if-else區塊。

```rust
fn diverges() -> ! {
    panic!("This function never returns!");
}

fn main() {
    let x: bool = diverges();
    let y: String = diverges();
    let p = if x {
        panic!("error");
    } else {
        100
    };
}
```

在Rust中，有以下這些情況永遠不會返回，它們的類型就是！。

* panic！以及基於它實現的各種函數/巨集，比如unimplemented！、unreachable！；
* 無窮迴圈loop{}；
* 行程退出函數std::process::exit以及類似的libc中的exec一類函數。

## main函數

以C語言為例，主函數的原型一般允許定義成以下幾種形式：

```c
int main(void);
int main();
int main(int argc, char **argv);
int main(int argc, char *argv[]);
int main(int argc, char **argv, char **env);
```

Rust的設計稍微有點不一樣，傳遞參數和返回狀態碼都由單獨的API來完成：

```rust
fn main() {
    for arg in std::env::args() {
        println!("Arg: {}", arg);
    }
    std::process::exit(0);
}
```

編譯後，執行檔加上幾個參數可得到以下結果：

```bash
$ test -opt1 opt2 -- opt3
Arg: test
Arg: -opt1
Arg: opt2
Arg: --
Arg: opt3
```

每個被空格分開的字串都是一個參數。行程可以在任何時候調用`exit()`直接退出，退出時候的錯誤碼由`exit()`函數的參數指定。

### 讀取環境變數

如果要讀取環境變數，可以用[`std::env::var()`](https://doc.rust-lang.org/std/env/fn.var.html)以及`std::env::vars()`函數獲得。

var\(\)函數可以接受一個字串類型參數，用於查找當前環境變數中是否存在這個名字的環境變數，vars\(\)函數不攜帶參數，可以返回所有的環境變數。



```bash
fn main() {
    for arg in std::env::args() {
        match std::env::var(&arg) {
            Ok(val) => println!("{}: {:?}", &arg, val),
            Err(e) => println!("couldn't find environment {}, {}", &arg, e),
        }
    }
    println!("All environment varible count {}", std::env::vars().count());
}
```

## const fn

函數可以用const關鍵字修飾，這樣的函數可以在編譯階段被編譯器執行，返回值也被視為編譯期常量。

```rust
const fn cube(num: usize) -> usize {
    num * num * num
}
fn main() {
    // const fn才能用於初始化const
    const DIM: usize = cube(2);
    // 因為DIM在編譯期已決定，因此可用於初始化array
    const ARR: [i32; DIM] = [0; DIM];
    // [0, 0, 0, 0, 0, 0, 0, 0]
    println!("{:?}", ARR);
}
```

cube函數接受數位參數，它會返回一個數字，而且這個返回值本身可以用於給一個const常量做初始化，const常量又可以當成一個常量陣列的長度使用。

const函數是在編譯階段執行的，因此相比普通函數有許多限制，並非所有的運算式和語句都可以在其中使用。

## 遞迴函數\(recursive function\)

Rust允許函數遞迴呼叫。所謂遞迴呼叫，指的是函數直接或者間接調用自己。

當前版本的Rust暫時還不支持尾部遞迴優化，因此如果遞迴調用層次太多的話，是有可能撐爆堆疊空間的。

```rust
fn fib(index: u32) -> u64 {
    if index == 1 || index == 2 {
        1
    } else {
        fib(index - 1) + fib(index - 2)
    }
}
fn main() {
    let f8 = fib(8);
    println!("{}", f8);
}
```

