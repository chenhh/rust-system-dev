# 語句與運算式

## 語句\(statement\)

一個Rust程式，是從main函數開始執行的。而函數體內，則是由一條條語句組成的。Rust程式裡，運算式（Expression）和語句（Statement）是完成流程控制、計算求值的主要工具。

在Rust程式裡面，運算式可以是語句的一部分，反過來，語句也可以是運算式的一部分。

* 一個運算式總是會產生一個值，因此它必然有類型；
* 語句不產生值，它的類型永遠是（）。
* 如果把一個運算式加上分號，那麼它就變成了一個語句；
* 如果把語句放到一個語句塊中包起來，那麼它就可以被當成一個運算式使用。

## 運算式\(expression\)

“運算式”在Rust程式中佔據著重要位置，運算式的功能非常強大。Rust中的運算式語法具有非常好的“一致性”，每種運算式都可以嵌入到另外一種運算式中，組成更強大的運算式。

Rust的運算式包括**字面量運算式、方法調用運算式、陣列運算式、索引運算式、單目運算子運算式、雙目運算子運算式**等。

Rust運算式又可以分為“左值”（lvalue）和“右值”（rvalue）兩類。

* 所謂左值，意思是這個運算式可以表達一個記憶體位址。因此，它們可以放到設定運算子左邊使用。
* 其他的都是右值。

### 算數運算式

Rust的算術運算子包括：加（+）、減（-）、乘（\*）、除（/）、求餘數（%）。

```rust
fn main() {
    let x = 100;
    let y = 10;
    // 算數運算式計算後會回傳值
    println!("{} {} {} {} {}", x + y, x - y, x * y, x / y, x % y);
}
```

x+y、x-y這些都是算數運算式，它們都有自己的值和類型。常見的整數、浮點數類型都支援這幾種運算式。它們還可以被重載，讓自訂的類型也支援這幾種運算式。

Rust的比較運算子包括：等於（==）、不等於（！=）、小於（&lt;）、大於（&gt;）、小於等於（&lt;=）、大於等於（&gt;=）。比較運算子的兩邊必須是同類型的，並滿足PartialEq約束。比較運算式的類型是bool。另外，Rust禁止連續比較。

```rust
fn main() {
    let (a, b, c) = (true, true, false);
    // error: comparison operators cannot be chained
    //   a == b == c;
    a == b && b == c;
}
```



### bitwise運算子

```rust
fn main() {
    let num1: u8 = 0b_1010_1010;
    let num2: u8 = 0b_1111_0000;
    println!("{:08b}", !num1);       //01010101
    println!("{:08b}", num1 & num2); //10100000
    println!("{:08b}", num1 | num2); //11111010
    println!("{:08b}", num1 ^ num2); //01011010
    println!("{:08b}", num1 << 4);   //10100000
    println!("{:08b}", num1 >> 4);   //00001010
}
```

### logical運算子

取反運算子\(!\)既支援“邏輯取反”也支援“按位元取反”，它們是同一個運算子，根據類型決定執行哪個操作。如果被運算元是bool類型，那麼就是邏輯取反；如果被運算元是其他數字類型，那麼就是按位元取反。

bool類型既支援“邏輯AND”、“邏輯OR”，也支援“按位元AND”、“按位元OR”。它們的區別在於，“邏輯AND”、“邏輯OR”具備“短路”功能。

Rust裡面的運算子優先順序與C語言裡面的運算子優先順序設置是不一樣的，有些細微的差別。不過這並不是很重要。不論在哪種程式設計語言中，建議如果碰到複雜一點的運算式，儘量用小括弧明確表達計算順序，避免依賴語言預設的運算子優先順序。因為不同知識背景的程式師對運算子優先順序順序的記憶是不同的。

```rust
fn f1() -> bool {
    println!("Call f1");
    true
}
fn f2() -> bool {
    println!("Call f2");
    false
}
fn main() {
    // call f2, call f1, Bit AND: false
    println!("Bit AND: {}\n", f2() & f1()); 
    
    // call f2, Logic AND: false (f1因為短路沒有被執行)
    println!("Logic AND: {}\n", f2() && f1()); 
    
    // call f1, call f2, bit OR: true
    println!("Bit OR: {}\n", f1() | f2());
    
    // call f1, logic OR: true (f2因為短路沒有被執行)
    println!("Logic OR: {}\n", f1() || f2());
}
```

### 賦值運算式

一個左值運算式、設定運算子（=）和右值運算式，可以構成一個賦值運算式。

```rust
// 聲明區域變數,帶 mut 修飾
let mut x : i32 = 1;
// x 是 mut 綁定,所以可以為它重新賦值
x = 2;
```

x=2是一個賦值運算式，它末尾加上分號，才能組成一個語句。賦值運算式具有“副作用”：當它執行的時候，會把右邊運算式的值“複製或者移動”（copy or move）到左邊的運算式中。

賦值號左右兩邊運算式的類型必須一致，否則會產生編譯錯誤。

賦值運算式也有對應的類型和值。這裡不是說賦值運算式左運算元或右運算元的類型和值，而是說整個運算式的類型和值。**Rust規定，賦值運算式的類型為unit，即空的tuple（）**。

```rust
fn main() {
    let x = 1;
    let mut y = 2;
    // 注意這裡專門用括弧括起來了
    // 賦值表達式必回傳unit ()
    let z = (y = x);
    println!("{:?}", z); // ()
}
```

**Rust這麼設計可以防止連續賦值**。如果你有x：i32、y：i32以及z：i32，那麼運算式z=y=x會發生編譯錯誤。因為變數z的類型是i32但是卻用（）對它初始化了，編譯器是不允許通過的。

C語言允許連續賦值，但這個設計沒有帶來任何性能提升，反而在某些場景下給用戶帶來了程式碼不夠清晰直觀的麻煩。

**這個設計同樣可以防止把==寫成=的錯誤**。比如，Rust規定，在if運算式中，它的條件運算式類型必須是bool類型，所以if x=y{}這樣的程式碼是無論如何都編譯不過的，哪怕x和y的類型都是bool也不行。賦值運算式的類型永遠是（），它無法用於if條件運算式中。

Rust也支援組合賦值運算式，+、-、\*、/、%、&、\|、^、&lt;&lt;、&gt;&gt;這幾個運算子可以和設定運算子組合成賦值運算式。

LEFT OP=RIGHT這種寫法，含義等同於LEFT=LEFT OP RIGHT。所以，y+=x的意義相當於y=y+x，依此類推。

Rust不支援++、--運算子，改使用+=1、-=1替代。

```rust
fn main() {
    let x = 2;
    let mut y = 4;
    y += x; // 6
    y *= x; // 12
    println!("{} {}", x, y);
}
```

### 語句塊運算式

語句和運算式的區分方式是後面帶不帶分號（；）。

* 如果帶了分號，意味著這是一條語句，它的類型是（）；
* 如果不帶分號，它的類型就是運算式的類型，因此有回傳值。
* 不加分法直接回傳這種寫法與return 100；語句的效果是一樣的，相較於return語句來說沒有什麼區別，但是更加簡潔。特別是用在閉包closure中，這樣寫就方便輕量得多。

```rust
fn main() {
    // 語句塊可以是運算式,注意後面有分號結尾,x的類型是()
    let x: () = {
        println!("Hello.");
    };
    // Rust將按循序執行語句塊內的語句,並將最後一個運算式類型返回,y的類型是 i32
    let y: i32 = {
        println!("Hello.");
        5
    };
}

// 在函數中，我們也可以利用這樣的特點來寫返回值
fn my_func() -> i32 {
    // ... blablabla 各種語句
    100
}
```

## if-else

if-else運算式的構成方式為：以if關鍵字開頭，後面跟上條件運算式，後續是結果語句塊，最後是可選的else塊。條件運算式的類型必須是bool。

```rust
fn main() {
    let n = 10;

    // 在if語句中，後續的結果語句塊要求一定要用大括弧包起來，不能省略，
    // 以便明確指出該if語句塊的作用範圍。
    // 條件表達式不需用括號, 編譯器會給warning
    // 所有的分支應有相同的回傳值
    if n < 0 {
        print!("{} is negative", n);
    } else if n > 0 {
        print!("{} is positive", n);
    } else {
        print!("{} is zero", n);
    }
}
```

if-else結構還可以當運算式使用，因此沒有C/C++中三元運算子?:。

```rust
fn main() {
    let n = 10;
    let x: i32 = if n >=5 { 1 } else { 10 };
    //--------------- ------^ -------- ^
    //------------------- 這兩個地方不要加分號
    println!("{}", x);
}
```

## loop \(無窮迴圈\)

在Rust中，使用loop表示一個無限迴圈。loop{}和while true{}迴圈的區別在於，相比於其他的許多語言，Rust語言要做更多的靜態分析。loop和while true語句在執行時沒有什麼區別，它們主要是會影響編譯器內部的靜態分析結果。

```rust
fn main() {
    let mut count = 0u32;
    println!("Let's count until infinity!");
    // 無限迴圈
    loop {
        count += 1;
        if count == 3 {
            println!("three");
            // 不再繼續執行後面的程式碼,跳轉到loop開頭繼續迴圈
            continue;
        }
        println!("{}", count);
        if count == 5 {
            println!("OK, that's enough");
            // 跳出迴圈
            break;
        }
    }
}
```

與if結構一樣，loop結構也可以作為運算式的一部分。

```rust
fn main() {
    let v = loop {
        break 10;
    };
    println!("{}", v);
}
```

在loop內部break的後面可以跟一個運算式，這個運算式就是最終的loop運算式的值。如果一個loop永遠不返回，那麼它的類型就是“發散類型”。

```rust
fn main() {
    let v = loop {};
    // warning: unreachable statement
    println!("{:?}", v);
}
```

### break, continue

* continue；語句表示本次迴圈內，後面的語句不再執行，直接進入下一輪迴圈。
* break；語句表示跳出迴圈，不再繼續。

break語句和continue語句還可以在多重迴圈中選擇跳出到哪一層的迴圈。

```rust
fn main() {
    // A counter variable
    let mut m = 1;
    let n = 1;
    // 使用lifetime label
    'a: loop {
        if m < 100 {
            m += 1;
        } else {
            // 使用lifetime label
            'b: loop {
                if m + n > 50 {
                    println!("break");
                    break 'a;
                } else {
                    continue 'a;
                }
            }
        }
    }
}
```

## while

while語句是帶條件判斷的迴圈語句。其語法是while關鍵字後跟條件判斷語句，最後是結果語句塊。如果條件滿足，則持續迴圈執行結果語句塊。

```rust
fn main() {
    // A counter variable
    let mut n = 1;
    // Loop while `n` is less than 101
    while n < 101 {
        if n % 15 == 0 {
            println!("fizzbuzz");
        } else if n % 3 == 0 {
            println!("fizz");
        } else if n % 5 == 0 {
            println!("buzz");
        } else {
            println!("{}", n);
        }
        // Increment counter
        n += 1;
    }
}
```

loop{}與while true{}的區別如下例所示。

```rust
fn main() {
    let x;
    loop {
        x = 1;
        break;
    }
    println!("{}", x); // 1

    // 以下會出現compile error
    // 因為while需要條件判斷，因此y可能沒有初始化
    let y;
    while true {
        y = 1;
        break;
    }
    println!("{}", y);
}
```

## for迴圈

Rust中的for迴圈實際上是許多其他語言中的for-each迴圈。Rust中沒有類似C/C++的三段式for迴圈語句。

```rust
fn main() {
    let array = [1, 2, 3, 4, 5];
    for i in array {
        println!("The number is {}", i);
        if i == 3{
            break;
        }
    }
}
```

