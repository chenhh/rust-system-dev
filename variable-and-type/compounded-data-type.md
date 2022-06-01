# 複合資料類型

## tuple(元組)

tuple指的是“元組”類型，它通過圓括號包含一組運算式構成。tuple內的元素沒有名字。tuple是把幾個相同或相異類型組合到一起的最簡單的方式。

```rust
fn main() {
    // 元組中包含兩個元素,第一個是i32類型,第二個是bool類型
    let a = (1i32, false);
    println!("{:?}", a);
    // 元組中包含兩個元素,第二個元素本身也是元組,它又包含了兩個元素
    let b = ("a", (1i32, 2i32));
    println!("{:?}", b);

    // 如果元組中只包含一個元素，應該在後面添加一個逗號
    let c = (0,); // c是一個元組,它有一個元素
    let d = (0); // d是一個括弧運算式,它是i32類型
    println!("{:?}", c);
    println!("{}", d);

    // 訪問元組內部元素有兩種方法，
    // 一種是“模式匹配”（pattern destructuring），
    // 另外一種是“數字索引
    let p = (1i32, 2i32);
    let (a, b) = p;
    let x = p.0;
    let y = p.1;
    println!("{} {} {} {}", a, b, x, y);
}
```

### unit(單元類型)

元組內部也可以一個元素都沒有。這個類型單獨有一個名字，叫unit（單元類型），不佔記憶體空間。

```rust
fn main() {
    // unit type
    let empty : () = ();
    
    // 空元組和空結構體struct Foo；一樣，都是佔用0記憶體空間。
    println!("size of i8 {}" , std::mem::size_of::<i8>());     // 1
    println!("size of char {}" , std::mem::size_of::<char>()); // 4
    println!("size of '()' {}" , std::mem::size_of::<()>());   // 0
}
```

在C++中，empty class佔用的是非零記憶體空間。

```cpp
class Empty {};
Empty emp;
assert(sizeof(emp) != 0);
```

## struct(結構體)

struct與tuple類似，也可以把多個類型組合到一起，作為新的類型。**區別在於，struct每個元素都有自己的名字**。

```rust
// struct宣告
// 每個元素之間採用逗號分開，最後一個逗號可以省略不寫。
// 類型在冒號後面，但是不能使用自動類型推導功能，必須顯式指定。
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    // struct類型的初始化語法類似於json的語法，
    // 使用“成員–冒號–值”的格
    let p = Point { x: 0, y: 0 };
    println!("Point is at {} {}", p.x, p.y);

    // 如果有區域變數名字和成員變數名字恰好一致，
    // 那麼可以省略掉重複的冒號初始化：
    // 剛好區域變數名字和結構體成員名字一致
    let x = 10;
    let y = 20;
    // 下麵是簡略寫法,等同於 Point { x: x, y: y },同名字的相對應
    let p = Point { x, y };
    println!("Point is at {} {}", p.x, p.y);

    // 訪問結構體內部的元素，也是使用“點”加變數名的方式。
    // 當然，也可以使用“模式匹配”功能
    // 聲明了px 和 py,分別綁定到成員 x 和成員 y
    let Point { x: px, y: py } = p;
    println!("Point is at {} {}", px, py);
    // 同理,在模式匹配的時候,如果新的變數名剛好和成員名字相同,可以使用簡寫方式
    let Point { x, y } = p;
    println!("Point is at {} {}", x, y);
}
```

Rust設計了一個語法糖，允許用一種簡化的語法賦值使用另外一個struct的部分成員。

```rust
#[derive(Debug)]
struct Point3d {
    x: i32,
    y: i32,
    z: i32,
}
fn default() -> Point3d {
    Point3d { x: 0, y: 0, z: 0 }
}
fn main() {
    // 可以使用default()函數初始化其他的元素
    // ..expr 這樣的語法,只能放在初始設定式中,所有成員的最後最多只能有一個
    let origin = Point3d { x: 5, ..default() };
    let point = Point3d {
        z: 1,
        x: 2,
        ..origin
    };

    println!("{:?}, {:?}", origin, point);
}
```

struct內部成員也可以是空：

```cpp
//以下三種語法都可以,內部可以沒有成員
struct Foo1;
struct Foo2();
struct Foo3{}
```

## tuple struct

tuple struct就像是tuple和struct的混合。區別在於，tuple struct有名字，而它們的成員沒有名字。

```rust
// tuple struct
#[derive(Debug)]
struct Color(i32, i32, i32);

#[derive(Debug)]
struct Point(i32, i32, i32);

// 等價的struct
// struct Color{
//     0: i32,
//     1: i32,
//     2: i32,
// }; 
// struct Point { 0: i32,
//     1: i32,
//     2: i32,
// };

fn main(){
    // tuple struct可用tuple方式初始化
    let v1 = Color(0, 1, 2);
    // tuple struct也可用struct方式初始化
    let v2 = Point{0: 10, 1: 12, 2: 13};
    println!("v1={:?}", v1);
    println!("v2={:?}", v2);
}
```

<mark style="background-color:red;">tuple struct有一個特別有用的場景，那就是當它只包含一個元素的時候，就是所謂的newtype idiom</mark>。因為它實際上讓我們非常方便地在一個類型的基礎上創建了一個新的類型。

<mark style="background-color:blue;">註：在實作中，常常會使用此方法限定資料的類別，比如說height, weight兩個變數都是浮點數，使用此方法可以替變數的類型取名，限定變數的類別</mark>。

```rust
fn main() {
    // tuple struct, Inches為i32的新類型
    struct Inches(i32);
    
    fn f1(value: Inches) {}
    fn f2(value: i32) {}
    let v: i32 = 0;
    // 因為Inches類型和i32是不同的類型，函式呼叫參數不匹配。
    // f1(v); // compile error, mismatched types'
    f2(v);
}
```

通過關鍵字type，我們可以創建一個新的類型名稱，但是這個類型不是全新的類型，而只是一個具體類型的別名。在編譯器看來，這個別名與原先的具體類型是一模一樣的。

而使用tuple struct做包裝，則是創造了一個全新的類型，它跟被包裝的類型不能發生隱式類型轉換，可以具有不同的方法，滿足不同的trait，完全按需求而定。

```rust
fn main() {
    // 使用type做alias明確指定I為i32的別名
    type I = i32;
    fn f1(v: I) {}
    fn f2(v: i32) {}
    let v: i32 = 0;
    // 編譯器知道I為i32的別名，可編譯成功
    f1(v);
    f2(v);
}
```

## enum

枚舉允許你通過列舉可能的 成員（variants） 來定義一個類型。

enum類型在Rust中代表的就是多個成員的OR關係（同時間只有一個成員可被使用），所<mark style="background-color:red;">佔的記憶體空間為佔空間最大的成員的位元組數</mark>。

在Rust中，enum和struct為內部成員創建了新的名字空間。如果要訪問內部成員，可以使用：：符號。因此，不同的enum中重名的元素也不會互相衝突。

與C/C++中的枚舉相比，Rust中的enum要強大得多，它可以為每個成員指定附屬的類型資訊。

```rust
// Number為整數或浮點數
enum Number {
    Int(i32),   // struct tuple
    Float(f32), // struct tuple
}

// enum的元素可為複合型別
enum Message {
    Quit,                       // 沒有資料
    ChangeColor(i32, i32, i32), // struct tuple
    Move { x: i32, y: i32 },    // struct
    Write(String),              // struct tuple
}

fn main() {
    let a: Number = Number::Int(10);
    let b = Number::Float(10.2);
    let m1 = Message::Quit;
    let m2 = Message::ChangeColor(10, 2, 2);
    let m3 = Message::Move { x: 10, y: 20 };
    let m4 = Message::Write("hello world".to_string());

    // 8 bytes, 4 byte保存標記，4 bypes為max(sizeof(i32), sizeof(f32))    
    println!("Size of Number: {}", std::mem::size_of::<Number>());
    // 32 bytes
    println!("Size of Message: {}", std::mem::size_of::<Message>());
}
```

enum常用在match表達式。

```rust
enum Number {
    Int(i32),
    Float(f32),
}

// match必須列出所有的enum分支，或者用_處理所有其它可能出現的分支，
// 否則編譯會出現 not covered錯誤
fn read_num(num: &Number) {
    match num {
        &Number::Int(value) => println!("Integer {}", value),
        &Number::Float(value) => println!("Float {}", value),
        // _ => println!("other type")
    }
}
fn main() {
    let n: Number = Number::Int(10);
    // 此處因為將變數傳入function後不需再返回，
    //  因此不使用borrow直接用move將所有權轉移也可以
    read_num(&n);
}
```

Rust的enum與C/C++的enum和union都不一樣。它是一種更安全的類型，可以被稱為“tagged union”。從C語言的視角來看Rust的enum類型，重寫上面這段程式碼，它的語義類似這樣：

```c
#include <stdio.h>
#include <stdint.h>
 // C 語言類比 Rust 的 enum
struct Number {
  enum {
    Int,
    Float
  }
  tag;
  union {
    int32_t int_value;
    float float_value;
  }
  value;
};
void read_num(struct Number * num) {
  switch (num -> tag) {
  case Int:
    printf("Integer %d", num -> value.int_value);
    break;
  case Float:
    printf("Float %f", num -> value.float_value);
    break;
  default:
    printf("data error");
    break;
  }
}
int main() {
  struct Number n = {
    tag: Int,
    value: {
      int_value: 10
    }
  };
  read_num( & n);
  return 0;
}
```

如果我們用C語言來類比，就需要程式設計師自己來保證讀寫的時候標記和資料類型是匹配的，編譯器無法自動檢查。當然，上面這個模擬只是為了通俗地解釋Rust的enum類型的基本工作原理，在實際中，enum的記憶體佈局未必是這個樣子，編譯器有許多優化，可以保證語義正確的同時減少記憶體使用，並加快執行速度。

可以手動指定每個變體自己的標記值如下。

```rust
fn main() {
    enum Animal {
        dog = 1,
        cat = 200,
        tiger,
    }
    let x = Animal::tiger as isize;
    println!("{}", x); // 201
}
```

### 標準庫內的Option\<T>

Rust的core與std庫中有一個極其常用的enum類型[Option\<T>](https://doc.rust-lang.org/core/option/index.html)，[variant.Some](https://doc.rust-lang.org/core/option/enum.Option.html#variant.Some), [Variant.None](https://doc.rust-lang.org/core/option/enum.Option.html#variant.None), 它的定義如下：

```rust
// T為泛型，由使用者指定
enum Option<T> {
    None,        // 沒有值
    Some(T),     // struct tuple, 型別T有值
}
```

由於它實在是太常用，標準庫將Option以及它的成員Some、None都加入到了Prelude中，使用者甚至不需要use語句聲明就可以直接使用。

它表示的含義是“<mark style="background-color:red;"><mark style="color:red;">類型T的值要麼存在、要麼不存在<mark style="color:red;"></mark><mark style="background-color:red;">”</mark>。比如Option\<i32>表達的意思就是“可以是一個i32類型的值，或者沒有任何值”。

從類型系統的角度來表達這個概念就意味著<mark style="color:blue;">編譯器需要檢查是否處理了所有應該處理的情況</mark>，這樣就可以避免在其他編程語言中非常常見的 bug。

<mark style="background-color:blue;">Rust 並沒有空值(null)，不過它確實擁有一個可以編碼存在或不存在概念的枚舉，即為Option\<T></mark>。

```rust
let some_number = Some(5);
let some_string = Some("a string");
let absent_number: Option<i32> = None;
```

## 類型遞迴定義

Rust裡面的複合資料類型是允許遞迴定義的。比如struct裡面嵌套同樣的struct類型，但是直接嵌套是不行的。

```rust
struct Recursive {
    data: i32,
    // 不可直接遞迴定義，會出現編譯錯誤
    // 因為無法算出Recursive的size
    rec: Recursive, 
}

/* 
error[E0072]: recursive type `Recursive` has infinite size
--> test.rs:2:1
|
2 | struct Recursive {
| ^^^^^^^^^^^^^^^^ recursive type has infinite size
3 | data: i32,
4 | rec: Recursive,
| -------------- recursive without indirection
|
= help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to make
`Recursive` representable
*/
```

。Rust是允許使用者手工控制記憶體佈局的語言。直接使用類型遞迴定義的問題在於，當編譯器計算Recursive這個類型大小的時候：`size_of::<Recursive>() == 4 + size_of::<Recursive>()` 這個方程在實數範圍內無解。

解決辦法很簡單，用指標間接引用就可以了，因為指標的大小是固定的(32-bit或64-bit)。

```rust
struct Recursive {
    data: i32,
    rec: Box<Recursive>,
}
```
