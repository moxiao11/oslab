> git checkout master 

解释一下lab5的代码原理

实验五的代码其实相比于实验四好一点，依旧先找要写的文件路径/kernel/src/task/process.rs.

## sys_waitid

要写的函数在文档里面写的很清楚流程，这就是根据伪代码来改编一下。

```text
A. 判定是否有匹配子进程
   无 -> return -1
B. 判定匹配子进程是否有 Zombie
   无 -> return -2
C. 回收 Zombie 并写回退出码
   remove(child) -> *translated_refmut(...) = exit_code -> return child_pid
```

### 判断是否有子进程

首先有一个叫current_task()的函数，能返回当前正在运行的任务进程,看一眼他的定义，是调用了进程PORCESSOR的各种操作
```rust
pub fn current_task() -> Option<Arc<TaskControlBlock>> {
    PROCESSOR.exclusive_access().current()
}
```

再往上翻会发现processor的定义
```rust
pub struct Processor {
    ///The task currently executing on the current processor
    current: Option<Arc<TaskControlBlock>>,

    ///The basic control flow of each core, helping to select and switch process
    idle_task_cx: TaskContext,
}
```
* current：存放当前进程的TCB 
* idle_task_cx: 存放进程上下文
 
这里看一眼上下文定义,即使一堆寄存器和栈指针。(TCB在后面)
```rust
pub struct TaskContext {
    /// Ret position after task switching
    ra: usize,
    /// Stack pointer
    sp: usize,
    /// s0-11 register, callee saved
    s: [usize; 12],
}
```
看了一圈发现就那样，current_task()就会返回当前进程，也不在意里面怎么实现了。
```rust
let current = current_task().unwrap();
```

接下来呢就得找内部子进程了，这东西可让我一顿好找啊。首先我们得到是一个TCB，里面和前面实验一样，都会有Inner的一个小类封装一手。这里的代码很多,我删掉了一部分，把用到的留下
```rust

pub struct TaskControlBlock {
    inner: UPSafeCell<TaskControlBlockInner>,
}

impl TaskControlBlock {
    //进入内部信息
    pub fn inner_exclusive_access(&self) -> RefMut<'_, TaskControlBlockInner> {
        self.inner.exclusive_access()
    }
}



pub struct TaskControlBlockInner {
    //子进程
    pub children: Vec<Arc<TaskControlBlock>>,
    // 退出码
    pub exit_code: i32,
}

impl TaskControlBlockInner {
    //判断是否是僵尸进程
    pub fn is_zombie(&self) -> bool {
        self.get_status() == TaskStatus::Zombie
}

```



我们会发现进入inner调用inner_exclusive_access()即可，可以访问TCB内部的一些信息，TCB内部呢，就会有一个children的数组，存着自己的子进程。那接下来一顿操作，从头到尾遍历会吧，当pid == -1(任意进程都能等待也就是任意进程都是子进程)或者pid相同的时候(子进程也是一个tcb有getpid操作)

```rust
let mut inner = current.inner_exclusive_access();
let mut flag = false;
    for child in inner.children.iter() {
        if pid == -1 || child.getpid() == pid as usize {
            flag = true;
            break;
        }
    }
    if !flag {
        return -1;
    }
```

### 寻找僵尸进程

看到这里，依旧遍历大法，从头到尾遍历，然后用那个is_zombie()函数

```rust
let mut idx = 0;
let mut found = false;
for (i, child) in inner.children.iter().enumerate() {
    if (pid == -1 || child.getpid() == pid as usize)
        && child.inner_exclusive_access().is_zombie()
    {
        idx = i;
        found = true;
        break;
    }
}
```
如果找到了呢？就返回僵尸进程的pid,也就是我记录的那个idx的位置的进程,然后退出码要返回。
```rust
if found {
    let tmp = inner.children.remove(idx);
    let code = tmp.inner_exclusive_access().exit_code;
    *translated_refmut(token, exit_code_ptr) = code;
    return tmp.getpid() as isize;
}
```

这里唯一要解释的就是

`*translated_refmut(token, exit_code_ptr) = code;`

其实就是根据父进程的token找到页表，然后就能看到退出码的物理地址进行修改。

ai给的解释: 
```text
根据当前进程的页表 token，把一个“用户态虚拟地址” exit_code_ptr 翻译成内核可以访问的可变引用。
```
叽里咕噜说啥呢,直接看函数定义，接收一个token和一个可变指针，然后PageTable通过token返回来一个页表,剩下的好像就看不懂了

```rust
pub fn translated_refmut<T>(token: usize, ptr: *mut T) -> &'static mut T {
    let page_table = PageTable::from_token(token);
    let va = ptr as usize;
    page_table
        .translate_va(VirtAddr::from(va))
        .unwrap()
        .get_mut()
}
```

其实是这样一个流程，比如父进程先这样
```rust
int exit_code = -1;//初始随机一个退出码
int pid = fork();

if (pid == 0) {
    exit(7);
} else {
    int ret = waitpid(pid, &exit_code);
}
```

父进程调用sys_fork会建立父子关系，能存下子进程的各种信息

当子进程退出的时候
```rust
inner.task_status = TaskStatus::Zombie;
inner.exit_code = 7;
```

父进程，比如-1的地址是0x80401000
```rust
int ret = waitpid(101, &exit_code);
sys_waitpid(101, 0x80401000 as *mut i32)
```

父进程从子进程中取出退出码
```rust
let code = tmp.inner_exclusive_access().exit_code;
```
然后需要修改这个exit_code，但是吧不能直接
```rust
*exit_code_ptr = code; // 错误
```
这里需要根据token得到物理地址，然后再写,也就是
```rust
let token = current_user_token();
*translated_refmut(token, exit_code_ptr) = code;
```