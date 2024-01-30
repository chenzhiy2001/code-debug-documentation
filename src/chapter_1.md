# 安装code-debug

## 安装与使用

#### 获取插件

在[Release](https://github.com/chenzhiy2001/code-debug/releases)中下载vsix格式的安装包，回到您的VSCode，按`Ctrl-Shift-P`输入vsix，点击`Extensions: Install from VSIX`

#### 用自动配置脚本配置环境

1. 下载[test.sh](https://github.com/chenzhiy2001/code-debug/blob/master/test.sh)，尽量在home目录下运行
2. 执行chmod +x test.sh命令，为文件添加权限
3. 执行./test.sh，开始执行，请保证网络畅通，可能要很长时间
4. 执行完毕后配置环境变量：
```plain
vim ~/.bashrc
在.bashrc最后面添加以下语句
export PATH=$PATH:/home/zly/qemu-system-riscv64/build
export PATH=$PATH:/opt/riscv/bin
退出
source ~/.bashrc
```
5. 使用命令检查是否安装成功：
    1. rustc --version   (rustc 1.74.0-nightly (59a829484 2023-08-30))
    2. npm -v  (版本在9以上)
    3. node -v (版本在18以上)
    4. qemu-system-riscv64 --version  （QEMU emulator version 7.0.0）
    5. /opt/riscv/bin/riscv64-unknown-elf-gdb  （出现（gdb命令行，输入以下命令，有输出的话，表示有python支持））
```plain
(gdb) python
print("114514")
end 
```
6. 如果还是有问题请查看test.sh文件，里面用回车符隔开了下载各个工具的命令，可以把它单独复制出来到新的文件重新运行

### 待调试OS的配置

1. 在仓库目录下(.../code-debug)运行 npm install 命令

1. 在vscode中打开本项目，按F5执行，会弹出一个新的窗口

1. 在新窗口中打开rCore-Tutorial-v3项目，在 .vscode 文件夹中添加 launch.json文件，并输入以下内容，然后保存并再编译一遍rCore，接着在新窗口内按F5就可以启动gdb并调试。

   如果GDB并没有正常启动，可以尝试把下面的gdbpath改成绝对路径(如“/home/username/riscv64-unknown-elf-toolchain-10.2.0-2020.12.8-x86_64-linux-ubuntu14/bin”)。

   ```
   //launch.json
   {
       "version": "0.2.0",
       "configurations": [
           {
               "type": "gdb",
               "request": "launch",
               "name": "Attach to Qemu",
               "executable": "${userHome}/rCore-Tutorial-v3/os/target/riscv64gc-unknown-none-elf/release/os",
               "target": ":1234",//不能和Qemu开放的tcp端口重叠
               "remote": true,
               "cwd": "${workspaceRoot}",
               "valuesFormatting": "parseText",
               "gdbpath": "riscv64-unknown-elf-gdb",
               "showDevDebugOutput":true,
               "internalConsoleOptions": "openOnSessionStart",
               "printCalls": true,
               "stopAtConnect": true,
               "qemuPath": "qemu-system-riscv64",
               "qemuArgs": [
                   "-M",
                   "128m",
                   "-machine",
                   "virt",
                   "-bios",
                   "${userHome}/rCore-Tutorial-v3/bootloader/rustsbi-qemu.bin",
                   "-display",
                   "none",
                   "-device",
                   "loader,file=${userHome}/rCore-Tutorial-v3/os/target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000",
                   "-drive",
                   "file=${userHome}/rCore-Tutorial-v3/user/target/riscv64gc-unknown-none-elf/release/fs.img,if=none,format=raw,id=x0",
                   "-device",
                   "virtio-blk-device,drive=x0",
                   "-device",
                   "virtio-gpu-device",
                   "-device",
                   "virtio-keyboard-device",
                   "-device",
                   "virtio-mouse-device",
                   "-serial",
                   "stdio",
                   "-serial",
                   "pty",
                   "-s",
                   "-S"
               ],
            "userSpaceDebuggeeFolder":"${userHome}/rCore-Tutorial-v3/user/target/riscv64gc-unknown-none-elf/release/",
            "KERNEL_IN_BREAKPOINTS_LINE":65, // src/trap/mod.rs中内核入口行号。可能要修改
            "KERNEL_OUT_BREAKPOINTS_LINE":124, // src/trap/mod.rs中内核出口行号。可能要修改
            "GO_TO_KERNEL_LINE":30, // src/trap/mod.rs中，用于从用户态返回内核的断点行号。在rCore-Tutorial-v3中，这是set_user_trap_entry函数中的stvec::write(TRAMPOLINE as usize, TrapMode::Direct);语句。
            "KERNEL_IN_BREAKPOINTS_FILENAME":"src/trap/mod.rs",
            "KERNEL_OUT_BREAKPOINTS_FILENAME":"src/trap/mod.rs",
            "GO_TO_KERNEL_FILENAME":"src/trap/mod.rs"
           },
       ]
   }
   ```
    - 这里解释一下`KERNEL_IN_BREAKPOINTS_LINE`和`GO_TO_KERNEL_LINE`的区别。以rCore-Tutorial-v3为例，`KERNEL_IN_BREAKPOINTS_LINE`对应`trap_return`函数的断点，而`GO_TO_KERNEL_LINE`对应`trap_return`函数调用的`set_user_trap_entry`子函数的断点。而`set_user_trap_entry`子函数实际上只有一行语句：`stvec::write(TRAMPOLINE as usize, TrapMode::Direct);`。也就是说，`KERNEL_IN_BREAKPOINTS_LINE`指向中断处理例程，而`GO_TO_KERNEL_LINE`精确地指向中断处理例程中的`stvec::write(TRAMPOLINE as usize, TrapMode::Direct);`语句。
