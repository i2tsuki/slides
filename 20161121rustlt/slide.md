class: center, middle
name: inverse
layout: true
class: center, middle
---
## 結局 Rc とは何なのか？
.center[[@kizkoh](https://twitter.com/kizkoh)]

.center[2016-11-21]
.center[[RustのLT会！ Rust入門者の集い](https://rust.connpass.com/event/41467/)]

---
layout: false
## 一応自己紹介
.center[<img src="./images/twicon_b.png" width="240">]

.center[@kizkoh]
.center[KLab inc. Roppongi]

.center[Infra Engineer]

.center[インフラおじさん達に囲まれてパケットに溺れて働いている]

.center[libpnet, ring とかのリポジトリを眺めてる]
.center[開発環境は gentoo に rust-nightly 入ってるのでそれで!]

---
class: center, middle
## 初心者チックに Rc の話します!
(知っておかないとツライやつかつ知っておくと楽しいやつ)

---
layout: false
class: center, middle
## Rust 初心者が嵌るワナ
(borrow checker とのコンパイルバトル)
---
## Borrow checker compile error
```rust
#[derive(Debug)]
struct AStruct {
    value: i32,
}

fn main() {
    let double = move |x: &mut AStruct| {
        x.value *= 2;
    };

    let mut a_struct = AStruct {
        value: 1,
    };

    double(&mut a_struct);

    let mut b_struct = a_struct;
    
    double(&mut b_struct);
    double(&mut a_struct);
    println!("a_struct: {:?}", a_struct);
    println!("b_struct: {:?}", b_struct);
}
```
---
layout: false
## Borrow checker compile error

```sh
$ cargo build
   Compiling study v0.1.0
error[E0382]: use of moved value: `a_struct`
  --> src/main.rs:21:17
   |
18 |     let mut b_struct = a_struct;
   |         ------------ value moved here
...
21 |     double(&mut a_struct);
   |                 ^^^^^^^^ value used here after move
```
<br />
.center[定番ですね^^]
---
layout: false
class: center, middle

## .center[所有権が借用されたまま返却されない<br />⇩<br />borrow checker に怒られる]
---
## 所有権の共有

`std::rc::Rc` を使う
---
## 所有権の共有
```rust
use std::rc::Rc;
use std::cell::RefCell;

#[derive(Debug)]
struct AStruct {
    value: i32,
}

fn main() {
    let double = move |x: &mut AStruct| {
        x.value *= 2;        
    };

    let a_struct = Rc::new(RefCell::new(AStruct {
        value: 1,
    }));

    {
        let mut x = a_struct.borrow_mut();
        double(&mut x);
    }

    let b_struct = a_struct.clone();
    {
        let mut x = b_struct.borrow_mut();
        double(&mut x);
    }
	...
```
---
## 所有権の共有
```sh
% cargo run
a_struct: RefCell { value: AStruct { value: 8 } }
b_struct: RefCell { value: AStruct { value: 8 } }
```
---
## std::rc::Rc
- 所有権の共有に使う

- 参照カウント型ポインタ(スマートポインタ)

- 生ポインタへのアクセスをラップする

- pure rust

---
## 参照カウント
```rust
...
240 struct RcBox<T: ?Sized> {
241     strong: Cell<usize>,
242     weak: Cell<usize>,
243     value: T,
244 }
...
257 pub struct Rc<T: ?Sized> {
258     ptr: Shared<RcBox<T>>,
259 }
```

- strong  
  - `Rc<T>.clone()` すると増える
  - `drop(Rc<T>)` すると減る
  - `Weak::upgrade(&Weak<T>)` すると増える  
	
- weak  
  - `drop(weak<T>)` すると減る<br />
  - `Rc::downgrade(&Rc<T>)` すると増える<br />  

---
## 参照カウント
- 例えば `Rc::get_mut(Rc<T>)` するときに参照される

```rust
...
476    pub fn get_mut(this: &mut Self) -> Option<&mut T> {
477 ->      if Rc::is_unique(this) {
478             let inner = unsafe { &mut **this.ptr };
479             Some(&mut inner.value)
480         } else {
481             None
482         }
483     }
...
```
- `Rc::is_unique(&Rc<T>)`  
  - strong = 1, weak = 0 のとき `&mut` を返す

- `Rc<T>.clone()` すると strong += 1 される  
  - 所有権を共有したまま変更するためには RefCell を使う (`Rc<RefCell<T>>`)
---
## 参照カウント
- weak
  - `Rc::downgrade(&)` すると増える
 
``` rust
376     pub fn downgrade(this: &Self) -> Weak<T> {
377 ->      this.inc_weak();
378         Weak { ptr: this.ptr }
379     }
```
- strong
  - `Weak::upgrade(&Weak(T))` すると増える

``` rust
949     pub fn upgrade(&self) -> Option<Rc<T>> {
950         if self.strong() == 0 {
951             None
952         } else {
953 ->          self.inc_strong();
954             Some(Rc { ptr: self.ptr })
955         }
```
---

## 参照カウント
- strong
  - `drop(Rc<T>)` されると減る
``` rust 
609     fn drop(&mut self) {
610         unsafe {
611             let ptr = *self.ptr;
612 
613             self.dec_strong();
614             if self.strong() == 0 {
615                 // destroy the contained object
616                 ptr::drop_in_place(&mut (*ptr).value);
617 
618                 // remove the implicit "strong weak" pointer now that we've
619                 // destroyed the contents.
620                 self.dec_weak();
621 
622                 if self.weak() == 0 {
623                     deallocate(ptr as *mut u8, size_of_val(&*ptr), align_of_val(&*ptr))
624                 }
625             }
626         }
```

- drop で move するのでアクセスできない
- strong = 0 で Weak::upgrade(&Weak(T)) できないのでアクセスできない
---
## まとめ
#### - 参照カウントポインタの中身を知っていないと使いどころが分からずつらい!
- 参照回数を見てよしなに処理をしてくれるかしこいやつだよ  
(ドキュメントだけじゃ使い勝手あんまりわからないのでソース読もう!)  

#### - Rust の安全性を実現する処理なので知ると楽しい!

## 補足

- Cell とか RefCell とか Deref 飛ばしたので少々つらかったかも

  - [e016: RefCells and code smells](https://overcast.fm/+FNQyH35Q0)
  
  - [\`Deref\` coercions](https://doc.rust-lang.org/stable/book/deref-coercions.html)
---
layout: false
class: center, middle
.center[## Rust やっていくぞ!]
