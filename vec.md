# vec

[Struct std::vec::Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) ([中文](https://rustwiki.org/zh-CN/std/vec/index.html))

具有堆積已分配內容的連續可增長陣列類型，寫為 Vec。Vectors 有 O(1) 索引、amortized O(1) push、和 O(1) pop。Vectors 確保它們分配的位元組數永遠不會超過 isize::MAX 字節。

## 建立向量

```rust
// 用new()建立空的vec
let v: Vec<i32> = Vec::new();

// 或者用vec!巨集直接建立並插入元素
let v: Vec<i32> = vec![];
let v = vec![1, 2, 3, 4, 5];
let v = vec![0; 10]; // 十個零

// 由From trait新建
let vec2 = Vec::from([1, 2, 3, 4]);

// array to vec
let s = [10, 40, 30];
let x = s.to_vec();

// 建立長度為5，全為0的向量
let vec = vec![0; 5];
assert_eq!(vec, [0, 0, 0, 0, 0]);
```

## 插入、彈出、索引

```rust
let mut v = vec![1, 2];
// 插入元素到尾部
v.push(3);

// 彈出尾部元素並賦值
let mut v = vec![1, 2];
let two = v.pop();

// 可直接用整數索引元素(從0開始)，不支援負數索引
let mut v = vec![1, 2, 3];
let three = v[2];
```

## 切片(slicing)

Vec 可以是可變的。另一方面，切片是只讀對象。 要獲得 slice，請使用 `&`。

```rust
fn read_slice(slice: &[usize]) {
    println!("{:?}", slice);
}

fn main() {
    let v = vec![0, 1];
    read_slice(&v);

    // 您也可以這樣：
    let u: &[usize] = &v;
    println!("{:?}", u);
    // 或像這樣：
    let u: &[_] = &v;
    println!("{:?}", u);
}
```

## 容量和重新分配

vector 的容量(capacity)是為將新增到 vector 上的任何 future 元素分配的空間量。請勿將其與 vector 的長度(length)混淆，後者指定 vector 中的實際元素數量。

&#x20;<mark style="color:red;">如果 vector 的長度超過其容量，則其容量將自動增加，但必須重新分配其元素</mark>。

例如，容量為 10 且長度為 0 的 vector 將是一個空的 vector，具有 10 個以上元素的空間。將 10 個或更少的元素壓入 vector 不會改變其容量或引起重新分配。 但是，如果 vector 的長度增加到 11，則必須重新分配，這可能會很慢。因此，建議盡可能使用 `Vec::with_capacity` 來預先指定 vector 希望達到的大小。
