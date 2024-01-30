# 核心代码讲解

## 调试工具实现

TODO: 这一节所列出的代码有**少量**变动，应当更新

### 常用API、GDB命令

#### TreeView命令注册

命令注册以后，用户可以直接点击界面上的按钮向插件进程发送消息

```ts
const setKernelInOutBreakpointsCmd = vscode.commands.registerCommand(
    "code-debug.setKernelInOutBreakpoints",
    () => {
        vscode.debug.activeDebugSession?.customRequest("setKernelInBreakpoints");
        vscode.window.showInformationMessage("Kernel In Breakpoints Set");
    }
);
```

弹出消息窗口

```ts
vscode.window.showInformationMessage("message"):
```

详见`src/frontend/extension.ts`

#### 插件进程 <==> Debug Adapter

1. 插件进程 --> Debug Adapter

   ```ts
   vscode.debug.activeDebugSession?.customRequest("requestName");
   ```

2. Debug Adapter解析customRequest

   ```ts
   protected customRequest(command: string, response: DebugProtocol.Response, args: any): void {
       switch (command) {
           case "requestName":
           this.sendEvent({ event: "eventName", body: ["test"] } as DebugProtocol.Event);
           this.sendResponse(response);
           break;
   ```

3. 插件进程监听Events和Responses

   ```ts
   	let disposable = vscode.debug.registerDebugAdapterTrackerFactory("*", {
   	createDebugAdapterTracker() {
   		return {
   			//监听VSCode即将发送给Debug Adapter的消息
   			onWillReceiveMessage:(message)=>{
   			    //...   	
   			},
   			onWillStartSession: () => { console.log("session started") },
   			//监听Debug Adapter发送给VSCode的消息
   			onDidSendMessage: (message) => {
                   //...
   				if (message.type === "event") {
   					//...
   					}//处理自定义事件
   					else if (message.event === "eventTest") {
   						//Do Something
   					}
   ```

详见`src/frontend/extension.ts`、`src/mibase.ts`

#### Debug Adapter <===> Backend

以setBreakPointsRequest为例：

```ts
    // src/mibase.ts
	//设置某一个文件的所有断点
	protected setBreakPointsRequest(response: DebugProtocol.SetBreakpointsResponse, args: DebugProtocol.SetBreakpointsArguments): void {
        //clearBreakPoints()、addBreakPoint() 实现见src/backend/mi2/mi2.ts
		this.miDebugger.clearBreakPoints(args.source.path).then(() => { //清空该文件的断点
            //......
			const all = args.breakpoints.map(brk => {
				return this.miDebugger.addBreakPoint({ file: path, line: brk.line, condition: brk.condition, countCondition: brk.hitCondition });
			});
            // ......
			
```

详见src/mibase.ts

#### GDB命令

- `add-symbol-file`
- `break`

详细的输出及返回数据的格式可参考[官方文档](https://sourceware.org/gdb/onlinedocs/gdb/GDB_002fMI.html#GDB_002fMI)


### 关键的寄存器和内存的数据获取

VSCode 其实提供了几个重要的原生 request 接口，如 variablesRequest，其功能是展示 debugger 页中，左边VARIABLES 中变量的名字与值。每当 VSCode 的代码调试发生了暂停，VSCode 都会自动发送一个variablesRequest 向 DA 请求变量数据，那么我们只需要实现自定义的 variablesRequest，就可以做到自定义数据，如下我们可以在 TreeView 里展示寄存器

![vscode-scope](./docs/imgs/vscode-scope.png)



### 断点检测与切换

1. 当增删断点或`stopped`事件发生时，向Debug Adapter请求当前所有的断点信息（以及哪些断点被设置，哪些被缓存）

```ts
    //extension.ts
    onDidSendMessage: (message) => {
        if (message.command === "setBreakpoints"){//如果Debug Adapter设置了一个断点
            
            vscode.debug.activeDebugSession?.customRequest("update");
        }
        if (message.type === "event") {
            //...
            //如果（因为断点等）停下
            if (message.event === "stopped") {
                //更新寄存器信息
				//更新断点信息
                vscode.debug.activeDebugSession?.customRequest("update");   
            }
    //...
```

2. 当用户设置新断点时，判断这个断点能否在当下就设置，若否,则保存（VSCode编辑器和DA的断点是分离的，Debug Adapter不能控制编辑器的断点，故采用这种设计。见[此](https://stackoverflow.com/questions/55364690/is-it-possible-to-programmatically-set-breakpoints-with-a-visual-studio-code-ext)）

```ts
//src/mibase.ts-MI2DebugSession-setBreakPointsRequest
    protected setBreakPointsRequest(
		response: DebugProtocol.SetBreakpointsResponse,
		args: DebugProtocol.SetBreakpointsArguments
	): void {
		this.miDebugger.clearBreakPoints(args.source.path).then(
			() => {
				//清空该文件的断点
				const path = args.source.path;
				const spaceName = this.addressSpaces.pathToSpaceName(path);
				//保存断点信息，如果这个断点不是当前空间的（比如还在内核态时就设置用户态的
				//断点），暂时不通知GDB设置断点。
				//如果这个断点是当前地址空间，或者是内核入口断点，那么就通知GDB立即设置断点
				if ((spaceName === this.addressSpaces.getCurrentSpaceName())
				|| (path==="src/trap/mod.rs" && args.breakpoints[0].line===30)
				) {
					// TODO rules can be set by user
					this.addressSpaces.saveBreakpointsToSpace(args, spaceName);
					
				} 
				else {
					this.sendEvent({
						event: "showInformationMessage",
						body: "Breakpoints Not in Current Address Space. Saved",
					} as DebugProtocol.Event);
					this.addressSpaces.saveBreakpointsToSpace(args, spaceName);
					return;
				}
                //令GDB设置断点
				const all = args.breakpoints.map((brk) => {
					return this.miDebugger.addBreakPoint({
						file: path,
						line: brk.line,
						condition: brk.condition,
						countCondition: brk.hitCondition,
					});
				});
			//...
        //更新断点信息
		this.customRequest("update",{} as DebugAdapter.Response,{});
	}
```

3. 当断点组切换（比如从内核态进到用户态），令GDB移除旧断点（断点信息仍然保存在`MIDebugSession.AddressSpaces.spaces`中），设置新断点。见`src/mibase.ts-AddressSpaces-updateCurrentSpace`。

### 到达内核边界时的处理

#### 启动之后第一次进入内核

触发断点时，检测这个断点是否是内核边界的断点。

```typescript
    //src/mibase.ts 
    protected handleBreakpoint(info: MINode) {
        //...
        if (this.addressSpaces.pathToSpaceName(info.outOfBandRecord[0].output[3][1][4][1])==='kernel'){//如果是内核即将trap入用户态处的断点
            this.addressSpaces.updateCurrentSpace('kernel');
            this.sendEvent({ event: "inKernel" } as DebugProtocol.Event);
            if (info.outOfBandRecord[0].output[3][1][3][1] === "src/trap/mod.rs" && info.outOfBandRecord[0].output[3][1][5][1] === '135') {
                this.sendEvent({ event: "kernelToUserBorder" } as DebugProtocol.Event);//发送event
            }
        }
        //...
    }
```

若是，添加符号表文件，移除当前所有断点，加载用户态程序的断点。

```    ts
//extension.ts    
else if (message.event === "kernelToUserBorder") {
    //到达内核态->用户态的边界
    // removeAllCliBreakpoints();
    vscode.window.showInformationMessage("will switched to " + userDebugFile + " breakpoints");
    vscode.debug.activeDebugSession?.customRequest("addDebugFile", {
        debugFilepath:
            os.homedir() +
            "/rCore-Tutorial-v3/user/target/riscv64gc-unknown-none-elf/release/" +
            userDebugFile,
    });
    vscode.debug.activeDebugSession?.customRequest(
        "updateCurrentSpace",
        "src/bin/" + userDebugFile + ".rs"
    );

```

#### 进入用户态以后，想要再次进入内核

点击gotokernel按钮，判断当前要设置的断点是不是内核入口断点，如果是直接通知GDB添加断点。

```    ts
//src/mibase.ts 	
    case "goToKernel":
			this.setBreakPointsRequest(
				response as DebugProtocol.SetBreakpointsResponse,
				{
					source: { path: "src/trap/mod.rs" } as DebugProtocol.Source,
					breakpoints: [{ line: 30 }] as DebugProtocol.SourceBreakpoint[],
				} as DebugProtocol.SetBreakpointsArguments
			);
			this.sendEvent({ event: "trap_handle" } as DebugProtocol.Event);				
			break;

//src/mibase.ts
	protected setBreakPointsRequest(
		response: DebugProtocol.SetBreakpointsResponse,
		args: DebugProtocol.SetBreakpointsArguments
	): void {
		this.miDebugger.clearBreakPoints(args.source.path).then(
			() => {
				//清空该文件的断点
				const path = args.source.path;
				const spaceName = this.addressSpaces.pathToSpaceName(path);
				//保存断点信息，如果这个断点不是当前空间的（比如还在内核态时就设置用户态的断点），暂时不通知GDB设置断点
				//如果这个断点是当前地址空间，或者是内核入口断点，那么就通知GDB立即设置断点
				if ((spaceName === this.addressSpaces.getCurrentSpaceName()) || (path==="src/trap/mod.rs" && args.breakpoints[0].line===30)
				) {
					// TODO rules can be set by user
					this.addressSpaces.saveBreakpointsToSpace(args, spaceName);					
				} 
				else {
					this.sendEvent({
						event: "showInformationMessage",
						body: "Breakpoints Not in Current Address Space. Saved",
					} as DebugProtocol.Event);
					this.addressSpaces.saveBreakpointsToSpace(args, spaceName);
					return;
				}
				//令GDB设置断点
				const all = args.breakpoints.map((brk) => {
					return this.miDebugger.addBreakPoint({
						file: path,
						line: brk.line,
						condition: brk.condition,
						countCondition: brk.hitCondition,
					});
				});        
```

更新当前地址空间，更新符号表，

```    ts
//extension.ts
	else if (message.event === "trap_handle") {							
					//vscode.window.showInformationMessage("switched to trap_handle");
					vscode.debug.activeDebugSession?.customRequest("addDebugFile", {
						debugFilepath:
							os.homedir() +
							"/rCore-Tutorial-v3/os/target/riscv64gc-unknown-none-elf/release/os",
					});
					vscode.debug.activeDebugSession?.customRequest(
						"updateCurrentSpace",
						"src/trap/mod.rs"
					);
```
