> git checkout lab3 

解读一下lab3需要用到的代码

我们已经知道当调用一些函数，read(),write(),fork() and so on，会从用户态经过syscall陷入到内核态中,这个实验主要是追溯trace的过程，通过id追到sys_trace这个在lab2已经写过了，就不多说这个原理了
```rust
/kernel/src/syscall/mod.rs
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        SYSCALL_EXIT => sys_exit(args[0] as i32),
        SYSCALL_YIELD => sys_yield(),
        SYSCALL_GET_TIME => sys_get_time(args[0] as *mut TimeVal, args[1]),
        SYSCALL_TRACE => sys_trace(args[0], args[1], args[2]),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}
```

然后就转到了一大堆代码的部分,request就是要说明你要追溯的东西，给每个要追溯的东西定个号,然后match匹配一下,所以我们现在要补的就是两个`trace_get_task_`某某函数里面缺失的
，然后ctrl + 左键继续找呗。

```rust
pub fn sys_trace(request: usize, syscall_id: usize, data: usize) -> isize {
    trace!("kernel: sys_trace request={}, syscall_id={:#x}", request, syscall_id);    
    
    const TRACE_GET_SYSCALL_COUNT: usize = 0x00;
    const TRACE_GET_TASK_INFO: usize = 0x01;
    const TRACE_GET_TOTAL_SYSCALLS: usize = 0x02;
    // 匹配
    match request {
        TRACE_GET_SYSCALL_COUNT => {
            let syscall_id = syscall_id;
            get_syscall_times(syscall_id) as isize
        },
        TRACE_GET_TASK_INFO => {
            if data == 0 {
                return -1;
            }
            let info_ptr = data as *mut TaskInfo;
            match get_current_task_info() {
                Some((status, syscall_times)) => {
                    unsafe {
                        (*info_ptr).status = status;
                        (*info_ptr).syscall_times = syscall_times;
                    }
                    0
                },
                None => -1,
            }
        },
        TRACE_GET_TOTAL_SYSCALLS => {
            get_total_syscall_count() as isize
        },
        _ => {
            trace!("Invalid trace request: {}", request);
            -1
        },
    }
}

```
转到某个地方会发现是`TASK_MANAGER`下面有这个函数，说明我们现在要补全了这个函数了

```rust
pub fn get_syscall_times(syscall_id: usize) -> usize {
    TASK_MANAGER.get_syscall_times(syscall_id)
}

/// Get current task info
pub fn get_current_task_info() -> Option<(usize, usize)> {
    TASK_MANAGER.get_current_task_info()
}

/// Get total syscall count for current task
pub fn get_total_syscall_count() -> usize {
    TASK_MANAGER.get_total_syscall_count()
}
```

首先开场定义了一个任务管理的东西,定义了总任务数和一个内部任务信息：
```rust
pub struct TaskManager {
    /// total number of tasks
    num_app: usize,
    /// use inner value to get mutable access
    inner: UPSafeCell<TaskManagerInner>,
}
```

然后看看这个TaskManagerInner 

```rust
pub struct TaskManagerInner {
    tasks: [TaskControlBlock; MAX_APP_NUM],
    current_task: usize,
    syscall_count: [SyscallTrace; MAX_APP_NUM],
}
```
`tasks`：所有任务控制块, 就是有个max_app_num,相当于

```C++  
TaskControlBlock tasks[MAX_APP_NUM] ; 
```
顺便看一下TaskControlBlock是什么东西
```rust
pub struct TaskControlBlock {
    /// The task status in it's lifecycle
    pub task_status: TaskStatus,
    /// The task context
    pub task_cx: TaskContext,
}
pub enum TaskStatus {
    /// uninitialized
    UnInit,
    /// ready to run
    Ready,
    /// running
    Running,
    /// exited
    Exited,
}
```
`current_task`：当前正在运行的任务编号

`syscall_count`：每个任务的系统调用统计,至于这个SyscallTrace,你去追溯一下源代码就发现
```rust
pub struct SyscallTrace {
    pub syscall_count: BTreeMap<usize, usize>,
}
```

用C++来看就是记录了一个哈希表，进程id映射到系统调用次数的。

这里可能会有点乱，我们整理一下
```text
*TaskManager*
├── num_app // Task总数
└── inner  // 内部信息
    └── *TaskManagerInner*
        ├── tasks // 任务
        │   └── *TaskControlBlock*
        │       ├── task_cx //task上下文
        │       └── task_status // task状态
        ├── current_task // 当前task
        └── syscall_count 
            └── *SyscallTrace* 
                └── syscall_count // 系统调用数量
                    └── BTreeMap<usize, usize>
```

## mark_current_suspended

然后就是补全代码了,这个函数没写全所以也需要补，挂起的时候把当前任务的状态标位ready
```rust
/// Change the status of current `Running` task into `Ready`.
fn mark_current_suspended(&self) {
    let mut inner = self.inner.exclusive_access();
    let current = inner.current_task;
    //when suspend current task, and you want to call it again, you need to modify its TaskStatus
}
```
所以根据上面那个结构就可以得到:
```rust
inner.tasks[current].task_status = TaskStatus::Ready;
```

## get_total_syscall_count

得到系统调用次数，那就是和上面前两行先一样，exclusive_access()似乎是更安全，没进去细致的看
```rust
let inner = self.inner.exclusive_access() ; 
let current_task_no = inner.current_task ;
```

然后把所有系统调用的数量加起来就行,如果会用c++的哈希表，应该这个问题不大，values()就是获得值，然后sum求和一下
```rust
return inner.syscall_count[current_task_no].syscall_count.values().sum() ; 
```


## get_current_task_info

这个函数要返回状态，得到总系统调用数量其实和上面一样,因为返回值是一个usize，说明这里需要将各种状态映射到一个整数，这里就需要match表达式出场了，如果不太会用，可以看sys_trace。

```rust
let status = match inner.tasks[current_task_no].task_status {
            TaskStatus::UnInit => 0 , 
            TaskStatus::Ready => 1 , 
            TaskStatus::Running => 2 , 
            TaskStatus::Exited => 3 , 
        };
```

然后得到系统调用次数
```rust
let times = inner.syscall_count[current_task_no].syscall_count.values().sum() ;
return Some((status , times)) ; 
```
返回值是option类型，一定要注意返回一个Some.


