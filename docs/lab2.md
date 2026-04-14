> git checkout lab2 

解读一下lab2需要用到的代码

## 用户态操作

首先会发现lib.h里面有很多会声明的函数，lib.c没有，这些就是我们要实现的，可以仿照一下write，然后去写出剩下的（注意类型的转换），照猫画虎，应该是不难的。这些函数都是用户层面的，但会发生系统调用,然后陷入内核态。

```rust
isize read(int fd ,void *buf , size_t count ) 
{
    return syscall(SYSCALL_READ , (size_t)fd , (size_t)buf , count )  ;
}
isize open(const char *path , int flags) 
{
    return syscall(SYSCALL_OPEN , (size_t)path , (size_t)flags , 0) ;
}
isize write(int fd, const void *buf, size_t count) {
    return syscall(SYSCALL_WRITE, (size_t)fd, (size_t)buf, count);
}
isize close (int fd ) 
{
    return syscall(SYSCALL_CLOSE, (size_t)fd , 0, 0 ) ; 
}
void exit(int code) {
    syscall(SYSCALL_EXIT, (size_t)code, 0, 0);
    while(1);
}

```

这里可以介绍一下这些函数参数是啥含义

`fd` : 文件描述符,指向某个地方

| fd | 指向什么  |
| -- | ----- |
| 0  | 标准输入  |
| 1  | 标准输出  |
| 2  | 标准错误  |
| 3  | a.txt |
| 4  | b.txt |

read(3, buf, 10)就是指向a.txt。

`buf`：buf就是指要从哪儿读写，缓冲区.

`count`:是最多读多少内容，假设缓冲区很小，那你读一大堆，~~缓冲区不炸了吗~~.

`path`: 这应该知道吧打开哪个文件的路径

`flags` : 标志位，只读、只写

`code` : 退出码，看看是不是正常退出

如果会c语言我觉得会很了解，但如果没接触过就真的一头雾水，需要花时间琢磨。

## syscall

然后接下来就会发生syscall,去内核态找这个系统调用函数，这里syscall会接受一点参数，只要是匹配一下你要干什么操作，你给一个id，然后做对应的活，之后就是一大堆你传入的参数,把那些传进来(这里有个小偷懒的技巧，如果你不知道函数是什么类型，去找到那个函数的定义，然后 as 某某类型),这里也是照猫画虎

```rust
const SYSCALL_WRITE: usize = 64;
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}
```
补一下对应的码，但是呢，还需要补几个常量，这个常量存在用户态的lib.h里面一定要对应上。
```rust
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        SYSCALL_EXIT => sys_exit(args[0] as i32),
        SYSCALL_CLOSE => sys_close(args[0]) , 
        SYSCALL_READ => sys_read(args[0], args[1] as *mut u8, args[2]) , 
        SYSCALL_OPEN => sys_open(args[0] as *const u8, args[1] as u32 ),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}
```

## sys_read()

接下来其实才是最麻烦的，要补充内核态下的这些操作，对于读来说，你需要将内核数据msg复制到缓冲区，但是刚才说了，缓冲区是有大小限制的，你需要取缓冲区的len和msg的len最小值为直接传递的值(中文注释最后也说了copy_len)
```rust
let copy_len = core::cmp::min(_len , _msg.len()) ;
```
然后就是copy了，去查文档,把msg作为指针传入缓冲区，大小为len。
```rust
unsafe{ core::ptr::copy(_msg.as_ptr() , buf , copy_len)}
```

想了解这个函数的可以，看文字，不想了解的，直接照葫芦画瓢，看那个文档里面有example(讨厌看英文)
```rust
use std::ptr;

unsafe fn from_buf_raw<T>(ptr: *const T, elts: usize) -> Vec<T> {
    let mut dst = Vec::with_capacity(elts);

    unsafe { core::ptr::copy(ptr, dst.as_mut_ptr(), elts); }

    unsafe { dst.set_len(elts); }
    dst
}
```
最后范围copy_len，类型为isize

## sys_open()

sys_open看文档看的多,如果你看了文档就会发现，open文件的时候，需要先把路径的长度算出来,看中文注释会发现用*path.add(len)就可以,直到找到路径最后
```rust
let mut len : usize = 0 ; 
while *_path.add(len) != 0 {
    len += 1; 
}; 
```

然后将字符串切片,再from_utf8一下，我觉得只调用这几个函数很抽象,不如讲讲真正发生了什么
```rust
let t = core::slice::from_raw_parts(_path, len); 
s = core::str::from_utf8(t).unwrap() ; 
```
用户态会传进来一个open("test.txt" , 'r') ; 

然后需要我们会发现const char *path(存的首地址)经过syscall这时候我们就要用下面的操作，来算一下到底多长
```rust
while *path.add(len) != 0 :
    len++ ;
```
用from_rwa_part来记录一下这个东西到底是些什么字符
```rust
core::slice::from_raw_part(_path ,len); 
```


执行这句之后：
```rust
s = core::str::from_utf8(ascii).unwrap();
```
就会把这几个字符拼接起来变成'test.txt';

完整代码如下,与其直接写完，不如顺着流程来一遍，每步发生了什么。
```rust

pub fn sys_open(_path: *const u8, _flags: u32) -> isize {
    trace!("kernel: sys_open");
   let mut s: &str = "";
    unsafe{
        let mut len : usize = 0 ; 
        while *_path.add(len) != 0 {
            len += 1; 
        }; 
        let ascii = core::slice::from_raw_parts(_path, len); 
        s = core::str::from_utf8(ascii).unwrap() ; 
    }
    print!("sys_open: path = {}\n", s);
    
    FD_MOCK_FILE as isize
}
```