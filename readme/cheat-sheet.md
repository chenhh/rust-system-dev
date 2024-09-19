---
description: cheatsheet
---

# 小抄



## 基本資料類型

佔用空間可用`std::mem::size_of::<type_name>()`印出。

| 資料類型                | 佔用空間             | 備註                                             |
| ------------------- | ---------------- | ---------------------------------------------- |
| bool                | 1 byte           | 只有true/false兩個值。                               |
| char                | 4 bytes          | 它可以描述任何一個符合unicode標準的字元值。在程式碼中，單個的字元字面量用單引號包圍。 |
| i8/i16/i32/i64/i128 | 1/2/4/8/16 bytes | 有號整數                                           |
| u8/u16/u32/u64/u128 | 1/2/4/8/16 bytes | 無號整數                                           |
| isize/usize         | 指標的長度            | 有號/無號整數                                        |
| f32/f64             | 4/8 bytes        | 浮點數                                            |

## 複合類型

tuple自已沒有名字，且每個元素都沒有名字。&#x20;

struct自已有名字，且每個元素都有自己的名字。&#x20;

tuple struct自已有名字，但每個元素都沒有名字。

```rust
// 元組中包含兩個元素,第一個是i32類型,第二個是bool類型
let a = (1i32, false);

// 函數回傳值為tuple
fn myfunc(a: i32, b: i32) -> (i32, i32, i32) {
    (a, b, a + b)
}

// unit type, 不佔記憶體空間
let empty : () = ();

// struct宣告
// 每個元素之間採用逗號分開，最後一個逗號可以省略不寫。
// 類型在冒號後面，但是不能使用自動類型推導功能，必須顯式指定。
struct Point {
    x: i32,
    y: i32,
}

// struct類型的初始化
let p = Point { x: 0, y: 0 };
// 簡略寫法
let p = Point { x, y };

//struct內部可以沒有成員
struct Foo1;
struct Foo2();
struct Foo3{}

// tuple struct不需為成員命名
struct Color(i32, i32, i32);

// tuple struct可用tuple方式初始化
let v1 = Color(0, 1, 2);
// tuple struct也可用struct方式初始化
let v2 = Point{0: 10, 1: 12, 2: 13};
```
