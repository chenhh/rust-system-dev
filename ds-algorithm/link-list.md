# 鏈結串列(link list)

## 簡介



鏈結串列是一種物理儲存單元上非連續、非順序的儲存結構，資料元素的邏輯順序是通過鏈表中的指標連結次序實現的。鏈串列由一系列結點（鏈表中每一個元素稱為結點）組成，結點可以在執行階段動態生成。每個結點包括兩個部分：一個是儲存資料元素的資料域，另一個是儲存下一個結點位址的指標域。

由於不必須按順序儲存，鏈表在給定位置插入的時候可以達到O(1)的複雜度，比另一種線性表順序表快得多，但是在有序資料中查詢一個節點或者訪問特定下標的節點則需要O(n)的時間，而線性表相應的時間複雜度分別是O(logn)和O(1)。

使用串列可以克服陣列需要預先知道資料大小的缺點，串列可以充分利用電腦記憶體空間，實現靈活的記憶體動態管理。但是串列失去了陣列隨機讀取的優點，同時串列由於增加了結點的指標域，空間開銷比較大。

串列最明顯的好處就是，常規陣列排列關聯項目的方式可能不同於這些資料項目在記憶體或磁碟上的順序，資料的存取往往要在不同的排列順序中轉換。串列允許插入和移除表上任意位置上的節點，但是不允許隨機存取。串列有很多種不同的類型：單向串列，雙向串列以及循環串列。

## 簡單的List結構

```rust
use crate::List::{Cons, Nil};

enum List {
    // Cons: 包含一個元素和一個指向下一個節點的指針的元組結構
    // 因為使用Box<List>，只能有一個所有者(只能指向一個節點)
    Cons(u32, Box<List>),
    // Nil: 表示一個串列節點的末端
    Nil,
}

impl List {
    // 建立一個空串列
    fn new() -> List {
        Nil
    }
    // 在前面加一個元素節點，並且連結舊的串列和返回新的串列
    fn prepend(self, elem: u32) -> List {
        // `Cons` 也是 List 類型的
        Cons(elem, Box::new(self))
    }
    // 返回串列的長度
    fn len(&self) -> u32 {
        // `self` 的類型是 `&List`, `*self` 的類型是 `List`,
        // 匹配一個類型 `T` 好過匹配一個引用 `&T`
        match *self {
            // 因為`self`是借用的，所以不能轉移 tail 的所有權
            // 因此使用 tail 的引用
            Cons(_, ref tail) => 1 + tail.len(),
            // 空的串列長度都是0
            Nil => 0,
        }
    }
    // 返回連串列的字串表達形式
    fn stringify(&self) -> String {
        match *self {
            Cons(head, ref tail) => {
                // `format!` 和 `print!` 很像
                // 但是返回一個堆上的字串去替代列印到控制台
                format!("{}, {}", head, tail.stringify())
            }
            Nil => {
                format!("Nil")
            }
        }
    }
}

fn main() {
    let mut list = List::new();
    list = list.prepend(1);
    list = list.prepend(2);
    list = list.prepend(3);
    // linked list has length: 3
    println!("linked list has length: {}", list.len());
    // 3, 2, 1, Nil
    println!("{}", list.stringify());
}
```

## 參考資料

鏈表在safe rust裡確實十分複雜，其概念跟借用規則背道而馳，要完全safe，且實現很優雅的的鏈表相當不容易。以下為大量的範本。

* \[github] [https://github.com/rust-unofficial/too-many-lists](https://github.com/rust-unofficial/too-many-lists)。
* \[文件] [https://rust-unofficial.github.io/too-many-lists/](https://rust-unofficial.github.io/too-many-lists/)。
