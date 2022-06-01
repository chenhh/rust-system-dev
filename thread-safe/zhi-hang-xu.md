# 執行緒

## 建立執行緒

```rust
use std::thread;
use std::time::Duration;
// main and child threads會交互出現
fn main() {
    // child spawn thread
    thread::spawn(|| {
        for i in 1..10 {
            println!("{i} from the spawned thread!");
            // thread::sleep 調用強制線程停止執行一小段時間，這會允許其他不同的線程運行
            thread::sleep(Duration::from_millis(2));
        }
    });

    // main thread
    for i in 1..5 {
        println!("{i} from the main thread!");
        thread::sleep(Duration::from_millis(1));
    }
    // 有可能在main thread結束前，child thread還沒執行結束
    println!("all threads complete");
}
```

```rust
use std::thread;

static NTHREADS: i32 = 10;

// 这是主（`main`）執行緒
fn main() {
    // 提供vector 存放所建立的子執行緒（children）。
    let mut children = vec![];

    for i in 0..NTHREADS {
        // 啟動另一個執行緒（spin up）
        children.push(thread::spawn(move || {
            println!("this is thread number {i}")
        }));
    }

    for child in children {
        // 等待執行緒。
        let _ = child.join();
    }
}u
```

## 使用 join 等待所有執行緒結束

可以通過將 thread::spawn 的返回值儲存在變量中來修復新建線程部分沒有執行或者完全沒有執行的問題。thread::spawn 的返回值類型是 JoinHandle。JoinHandle 是一個擁有所有權的值，當對其調用 join 方法時，它會等待其線程結束。

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {i} from the spawned thread!");
            thread::sleep(Duration::from_millis(2));
        }
    });

    for i in 1..5 {
        println!("hi number {i} from the main thread!");
        thread::sleep(Duration::from_millis(1));
    }

    // 確保在main結束前，child thread都結束了
    handle.join().unwrap();
    println!("all threads complete");
}
```

## 應用：map-reduce

```rust
use std::thread;

// 這是 `main` 執行緒
fn main() {
    // 這是我們要處理的資料。
    // 我們會通過執行緒實現 map-reduce 演算法，從而計算每一位的和
    // 每個用空白符隔開的塊都會分配給單獨的執行緒來處理
    //
    // 試一試：插入空格，看看輸出會怎樣變化！
    let data = "86967897737416471853297327050364959
11861322575564723963297542624962850
70856234701860851907960690014725639
38397966707106094172783238747669219
52380795257888236525459303330302837
58495327135744041048897885734297812
69920216438980873548808413720956532
16278424637452589860345374828574668";

    // 創建一個向量，用於儲存將要創建的子執行緒
    let mut children = vec![];

    /*************************************************************************
     * "Map" 階段
     * 把資料分段，並進行初始化處理
     ************************************************************************/

    // 把資料分段，每段將會單獨計算
    // 每段都是完整資料的一個引用（&str）
    let chunked_data = data.split_whitespace();

    // 對分段的資料進行迭代。
    // .enumerate() 會把當前的迭代計數與被迭代的元素以元組 (index, element)
    // 的形式返回。接著立即使用 “解構賦值” 將該元組解構成兩個變數，
    // `i` 和 `data_segment`。
    for (i, data_segment) in chunked_data.enumerate() {
        println!("data segment {} is \"{}\"", i, data_segment);

        // 用單獨的執行緒每一段資料
        //
        // spawn() 返回新執行緒的控制碼（handle），我們必須擁有控制碼，
        // 才能獲取執行緒的返回值。
        //
        // 'move || -> u32' 語法表示該閉包：
        // * 沒有參數（'||'）
        // * 會獲取所捕獲變數的所有權（'move'）
        // * 返回無符號 32 位元整數（'-> u32'）
        //
        // Rust 可以根據閉包的內容推斷出 '-> u32'，所以我們可以不寫它。
        //
        // 試一試：刪除 'move'，看看會發生什麼
        children.push(thread::spawn(move || -> u32 {
            // 計算該段的每一位的和：
            let result = data_segment
                // 對該段中的字元進行迭代..
                .chars()
                // ..把字元轉成數位..
                .map(|c| c.to_digit(10).expect("should be a digit"))
                // ..對返回的數位類型的迭代器求和
                .sum();

            // println! 會鎖住標準輸出，這樣各執行緒列印的內容不會交錯在一起
            println!("processed segment {}, result={}", i, result);

            // 不需要 “return”，因為 Rust 是一種 “運算式語言”，每個程式碼塊中
            // 最後求值的運算式就是程式碼塊的值。
            result
        }));
    }

    /*************************************************************************
     * "Reduce" 階段
     * 收集中間結果，得出最終結果
     ************************************************************************/

    // 把每個執行緒產生的中間結果收入一個新的向量中
    let mut intermediate_sums = vec![];
    for child in children {
        // 收集每個子執行緒的返回值
        let intermediate_sum = child.join().unwrap();
        intermediate_sums.push(intermediate_sum);
    }

    // 把所有中間結果加起來，得到最終結果
    //
    // 我們用 “渦輪魚” 寫法 ::<> 來為 sum() 提供類型提示。
    //
    // 試一試：不使用渦輪魚寫法，而是顯式地指定 intermediate_sums 的類型
    let final_result = intermediate_sums.iter().sum::<u32>();

    println!("Final sum result: {}", final_result);
}
```
