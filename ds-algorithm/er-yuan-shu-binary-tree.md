# 二元樹(binary tree)

## 簡介

二元樹是每個節點最多有兩個子樹的樹結構。通常子樹被稱作「左子樹」（left subtree）和「右子樹」（right subtree）。二元樹常被用於實現二元查詢樹和二元堆積。

二元查詢樹的子節點與父節點的鍵一般滿足一定的順序關係，習慣上，左節點的鍵少於父親節點的鍵，右節點的鍵大於父親節點的鍵。

二元堆積是一種特殊的堆積，二元堆積是是完全二元樹（二元樹）或者是近似完全二元樹（二元樹）。可分為最大堆積和最小積。

* 最大堆積：父結點的鍵總是大於或等於任何一個子節點的鍵；
* 最小堆積：父結點的鍵總是小於或等於任何一個子節點的鍵。

二元樹的每個結點至多隻有二棵子樹(不存在度大於2的結點)，二元樹的子樹有左右之分，次序不能顛倒。二元樹的第$$i$$層至多有$$2^{i-1}$$個結點；深度為$$k$$的二元樹至多有$$2^{k-1}$$個結點；對任何一棵二元樹$$T$$，如果其終端結點數為$$n_0$$，分支度為2的結點數為$$n_2$$，則$$n_0=n_2+1$$。

一棵深度為$$k$$，且有$$2^{k-1}$$個節點稱之為滿二元樹；深度為$$k$$，有$$n$$個節點的二元樹，當且僅當其每一個節點都與深度為$$k$$的滿二元樹中，序號為$$1$$至$$n$$的節點對應時，稱之為完全二元樹。

## 二元樹與樹的區別&#x20;

二元樹不是樹的一種特殊情形，儘管其與樹有許多相似之處，但樹和二元樹有兩個主要差別：

* 樹中結點的最大分支度數沒有限制，而二元樹結點的最大度數為2。
* 樹的結點無左、右之分，而二元樹的結點有左、右之分。

## 二元樹結構

```rust
type TreeNode<K, V> = Option<Box<Node<K, V>>>;
#[derive(Debug)]
struct Node<K, V: std::fmt::Display> {
    left: TreeNode<K, V>,
    right: TreeNode<K, V>,
    key: K,
    value: V,
}

// 二元查詢樹要求鍵可排序，我們要求K實現PartialOrd
trait BinaryTree<K, V> {
    // 前序、中序、後序走訪
    fn pre_order(&self);
    fn in_order(&self);
    fn pos_order(&self);
}
trait BinarySearchTree<K: PartialOrd, V>: BinaryTree<K, V> {
    fn insert(&mut self, key: K, value: V);
}
impl<K, V: std::fmt::Display> Node<K, V> {
    fn new(key: K, value: V) -> Self {
        Node {
            left: None,
            right: None,
            value: value,
            key: key,
        }
    }
}
impl<K: PartialOrd, V: std::fmt::Display> BinarySearchTree<K, V> for Node<K, V> {
    fn insert(&mut self, key: K, value: V) {
        if self.key < key {
            if let Some(ref mut right) = self.right {
                right.insert(key, value);
            } else {
                self.right = Some(Box::new(Node::new(key, value)));
            }
        } else {
            if let Some(ref mut left) = self.left {
                left.insert(key, value);
            } else {
                self.left = Some(Box::new(Node::new(key, value)));
            }
        }
    }
}

// 二元樹的走訪
impl<K, V: std::fmt::Display> BinaryTree<K, V> for Node<K, V> {
    fn pre_order(&self) {
        println!("{}", self.value);
        if let Some(ref left) = self.left {
            left.pre_order();
        }
        if let Some(ref right) = self.right {
            right.pre_order();
        }
    }
    fn in_order(&self) {
        if let Some(ref left) = self.left {
            left.in_order();
        }
        println!("{}", self.value);
        if let Some(ref right) = self.right {
            right.in_order();
        }
    }
    fn pos_order(&self) {
        if let Some(ref left) = self.left {
            left.pos_order();
        }
        if let Some(ref right) = self.right {
            right.pos_order();
        }
        println!("{}", self.value);
    }
}

type BST<K, V> = Node<K, V>;
fn test_insert() {
    let mut root = BST::<i32, i32>::new(3, 4);
    root.insert(2, 3);
    root.insert(4, 6);
    root.insert(5, 5);
    root.insert(6, 6);
    root.insert(1, 8);
    if let Some(ref left) = root.left {
        assert_eq!(left.value, 3);
    }
    if let Some(ref right) = root.right {
        assert_eq!(right.value, 6);
        if let Some(ref right) = right.right {
            assert_eq!(right.value, 5);
        }
    }
    println!("Pre Order traversal");
    root.pre_order();
    println!("In Order traversal");
    root.in_order();
    println!("Pos Order traversal");
    root.pos_order();
}
fn main() {
    test_insert();
}

```
