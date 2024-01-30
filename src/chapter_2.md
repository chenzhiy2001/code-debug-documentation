# 使用code-debug

1. 在code-debug文件夹下`git pull`更新软件仓库，确保代码是最新的，然后按F5运行插件，这时会打开一个新的VSCode窗口。 **后续操作步骤均在新窗口内完成！**
1. 在新窗口内，打开rCore-Tutorial-v3项目，按照上面的提示配置`launch.json`并保存。
1. 按F5键，即可开始使用本插件。
1. 清除所有断点（removeAllCliBreakpoints按钮）
1. 设置内核入口（setKernelInBreakpoints按钮）、出口断点（setKernelOutBreakpoints按钮）
1. 设置内核代码和用户程序代码的断点（这里有个bug尚未解决：必须得在initproc.rs的println!语句或`fn main()`处设置断点）
1. 按continue按钮开始运行rCore-Tutorial
1. 当运行到位于内核出口的断点时，插件会自动切换到用户态的断点
1. 在用户态程序中如果想观察内核内的执行流，可以点击gotokernel按钮，然后点击继续按钮，程序会停在内核的入口断点，这时，可以先把内核出口断点设置好（点击setKernelOutBreakpoints按钮），接下来，可以在内核态设置断点，点击继续，运行到内核的出口断点之后，会回到用户态。

[视频演示](https://www.bilibili.com/video/BV1gW4y1x7ys/?spm_id_from=333.999.0.0&vd_source=ab418999e896fd33cc8eefbeab063d7f)

### 常见问题和解决办法
#### 只能打内核态断点，不能打用户态断点
很可能是因为边界断点失效。要解决这个问题，需要更改边界断点的位置。
以rCore-Tutorial-v3为例，我们来看它的trap_return函数，这个函数用于从内核态进到用户态。
```rust
122 #[no_mangle]
123 pub fn trap_return() -> ! {
124     disable_supervisor_interrupt();
125     set_user_trap_entry();
126     let trap_cx_user_va = current_trap_cx_user_va();
127     let user_satp = current_user_token();
128     extern "C" {
129         fn __alltraps();
130         fn __restore();
131     }
132     let restore_va = __restore as usize - __alltraps as usize + TRAMPOLINE;
133     //println!("before return");
134     unsafe {
135         asm!(
136             "fence.i",
137             "jr {restore_va}",
138             restore_va = in(reg) restore_va,
139             in("a0") trap_cx_user_va,
140             in("a1") user_satp,
141             options(noreturn)
142         );
143     }
144 }
```

我们发现，第170行之后中断被关闭了，中断关闭后这个执行流不会被打断（多核呢？）所以124，125，126，127行都可以作为边界断点，可以逐一尝试。128-134行经过测试发现会卡在132行不动。
如果边界断点是可行的（表现为，虽然用户态断点不能触发，但是右下角有提示进行了断点组切换）那就是gdb在设置用户态断点的时候发生了错误，导致 gdb 跟踪失败。这可能跟编译器有关系，目前有一个临时的解决办法，就是经过我们测试，第17行是可以正常工作的，其他行则不行：
```rust
1 #![no_std]
2 #![no_main]
3 
4 extern crate user_lib;
5 
6 use user_lib::{println, getpid};
7 
8 use user_lib::{exec, fork, wait, yield_};
9 
10 #[no_mangle]
11 fn main() -> i32 {
12 
13 
14     println!("aaaaaaaaaaaaaa");
15     let a = getpid();
16     println!("pid is {}",a);
17    
18 
19 
20     if fork() == 0 {
21         exec("user_shell\0", &[core::ptr::null::<u8>()]);
22     } else {
    //...
```