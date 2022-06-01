# ref關鍵字

## 簡介

在通過 `let` 綁定來進行模式匹配或解構時，`ref` 關鍵字可用來創建結構體/元組的欄位的引用。

```rust
#[derive(Clone, Copy)]
struct Point { x: i32, y: i32 }

fn main() {
    let c = 'Q';

    // 設定陳述式中左邊的 `ref` 關鍵字等價於右邊的 `&` 符號。
    let ref ref_c1 = c;
    let ref_c2 = &c;

    // 解引用
    println!("ref_c1 equals ref_c2: {}", *ref_c1 == *ref_c2);

    let point = Point { x: 0, y: 0 };

    // 在解構一個結構體時 `ref` 同樣有效。部份解引用
    let _copy_of_x = {
        // `ref_to_x` 是一個指向 `point` 的 `x` 欄位的引用。
        let Point { x: ref ref_to_x, y: _ } = point;

        // 返回一個 `point` 的 `x` 欄位的拷貝。
        *ref_to_x
    };

    // `point` 的可變拷貝
    let mut mutable_point = point;

    {
        // `ref` 可以與 `mut` 結合以創建可變引用。
        let Point { x: _, y: ref mut mut_ref_to_y } = mutable_point;

        // 通過可變引用來改變 `mutable_point` 的欄位 `y`。
        *mut_ref_to_y = 1;
    }

    println!("point is ({}, {})", point.x, point.y);
    println!("mutable_point is ({}, {})", mutable_point.x, mutable_point.y);

    // 包含一個指標的可變元組
    let mut mutable_tuple = (Box::new(5u32), 3u32);
    
    {
        // 解構 `mutable_tuple` 來改變 `last` 的值。
        let (_, ref mut last) = mutable_tuple;
        *last = 2u32;
    }
    
    println!("tuple is {:?}", mutable_tuple);
}

```
