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
    // Immutable
    /// Process identifier
    pub pid: PidHandle,
    /// Mutable
    inner: UPSafeCell<TaskControlBlockInner>,
}

impl TaskControlBlock {
    /// Get the mutable reference of the inner TCB
    pub fn inner_exclusive_access(&self) -> RefMut<'_, TaskControlBlockInner> {
        self.inner.exclusive_access()
    }
    /// Get the address of app's page table
    pub fn get_user_token(&self) -> usize {
        let inner = self.inner_exclusive_access();
        inner.memory_set.token()
    }
}



pub struct TaskControlBlockInner {
    /// A vector containing TCBs of all child processes of the current process
    pub children: Vec<Arc<TaskControlBlock>>,

    pub exit_code: i32,
    /// Priority of the process
    pub priority: usize,

    /// stride of the process
    pub stride: Stride,
}

impl TaskControlBlockInner {
    /// get the user token
    pub fn get_user_token(&self) -> usize {
        self.memory_set.token()
    }
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

