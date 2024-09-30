# const與static

## 簡介

在軟體開發過程中，如果一個變數總是保持不變，我們可以宣告為常量，如果一個變數全域性唯一，可以使用靜態變數。Rust語言中使用const, static來實現這2個場景，但與其他語言稍有不同，<mark style="color:red;">rust中的</mark><mark style="color:red;">`const`</mark> <mark style="color:red;"></mark><mark style="color:red;">和</mark><mark style="color:red;">`static`</mark><mark style="color:red;">不能一起使用</mark>。

#### const有以下特點

* 每個使用const常數的地方，相當於拷貝了一份const常數，所以如果有 3 個地方用到常數，則會呼叫3次該型別的drop函式。
* 如果是定義常數引用，則不會有drop函式呼叫，因為使用他的地方是拷貝引用，沒有生成物件。
* 常數自定義結構體裸指標，不能實現Drop trait,否則編譯不過。

#### static有以下特點

* static變數全域性唯一，生命週期在整個應用中，所有權不能轉移，drop函式不會被呼叫。
* 不可變static變數必須實現Sync trait，以實現跨執行緒存取。
* 可變static 變數可以不實現 sync特質，對可變static的存取需要unsafe塊或unsafe函式中使用。
* 該型別變數在多執行緒存取時，必現確保安全性，比如加鎖, 可變static 必現在unsafe中存取。

## const變數

如果變數值在程式中為常數，可宣告為const，通常會以<mark style="color:red;">大寫字母命名變數，而且一定要指定變數類型</mark>。

在編譯時，會將變數值直接以常數取代(不佔用記憶體)。

```rust
fn main(){
    const MY_VAR:i32 = 100;
    println!("my var is {MY_VAR}");
}
```

## static變數

static為全局變數(生命週期)，而let定義的是區域變數，離開區域即銷毀。

```rust
static PI: f32 = 3.14159;
static mut X: i32 = 10;

fn main() {
    println!("PI: {PI}");
    // 修改static變數必須在unsafe區塊中
    unsafe {
        X = 100;
        println!("X: {X}");
    }
}

```

## const fn

const fn \[rust 2018] 是在編譯時就能預先執行並給出執行結果的函式，這些函式能使用在需要使用常數值的地方，並提供直接寫上常數值一樣的效果。這就像是 C++ 中的 constexpr 一樣。

```rust
fn main() {
    // const fn，所以它可以在編譯時就被求值，並且當成常數來使用。
    const fn ret_5() -> usize {
        5
    }

    static FOO: [i32; ret_5()] = [0, 1, 2, 3, 4];

    assert_eq!(FOO.len(), 5);
}
```

## 參考資料

[Rust const, static使用面面觀](https://www.gushiciku.cn/pl/gm1Y/zh-tw)

[針對常數泛型參數的分類實現](https://zjp-cn.github.io/rust-note/forum/impl-const-param.html)
