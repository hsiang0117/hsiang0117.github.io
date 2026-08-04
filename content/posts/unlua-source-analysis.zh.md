---
title: "UnLua 源码剖析：Lua 是怎么长进 UE 反射系统里的"
date: 2026-08-04
tags: ["Unreal Engine", "Lua", "UnLua", "C++", "Scripting"]
ShowToc: true
TocOpen: false
summary: "从源码层面拆解腾讯的 UE Lua 插件 UnLua：覆写机制如何把 Lua 函数塞进 UFunction、元表如何充当反射缓存、值语义为什么是最容易炸的地方、两套 GC 的接缝怎么缝，以及 2026 年它的真实维护状态。"
---

在 UE 项目里嵌一门脚本语言，动机通常不是"C++ 写得慢"，而是三件更具体的事：**移动端要能热更**（iOS 审核规则允许解释执行的脚本走补丁通道，重编二进制不行）、**策划和玩法程序需要一个不用等编译的迭代循环**、**蓝图在逻辑规模上去之后维护成本失控**（连线图不能 diff、不能 review、合并冲突基本无解）。

Lua 是这个位置上被验证过最多次的答案，而 [UnLua](https://github.com/Tencent/UnLua) 是腾讯开源的那一份。它的卖点是"零胶水代码"——不写任何 binding，Lua 里就能访问全部 `UCLASS` / `UPROPERTY` / `UFUNCTION`，还能直接覆写蓝图事件。

这篇文章不讲怎么用（官方文档够了），而是把它拆开看：**这套"零胶水"是靠什么换来的，代价落在哪里，哪些地方会咬人**。

分析基于 `master` 分支的 `3f112e8`，功能上等同于 **v2.3.6**（2023-11-07 发布，最后一个 tag）。文中所有 `文件:行号` 都指向这个版本。文末会单独说 2026 年的维护现状——它比"停更了"要复杂一点。

## 一、三条路里选了最贵也最省事的那条

把 C++ 对象暴露给脚本，工程上只有三条路：

| 方案 | 代表 | 代价 | 收益 |
|------|------|------|------|
| 手写绑定 | 传统 Lua 项目 | 每个类都要写胶水，接口一改就漏 | 调用开销最低，行为完全可控 |
| 代码生成 | sluaunreal、puerts | 需要跑生成器，产物进版本库，编译时间上涨 | 静态类型信息、接近手写的性能 |
| 运行时反射 | **UnLua** | 每次调用都要过反射层，类型错误只能运行时发现 | 零维护——引擎升级、蓝图改字段，脚本侧自动跟上 |

UnLua 选了第三条。UE 的反射系统（`UClass` / `UFunction` / `FProperty`）本来就是给蓝图 VM 和序列化用的完整运行时元数据，UnLua 相当于**在蓝图 VM 旁边挂了第二个消费者**。这个决定解释了它后面几乎所有的设计：好处是你新加一个 `UPROPERTY` 不需要动任何脚本层代码，坏处是每一次跨界访问都要付反射的钱，而且很多"坑"本质上是 UE 反射语义直接透出来的结果。

整个插件（不含第三方 Lua）只有 148 个 h/cpp，其中运行时模块 `UnLua` 占 118 个，另外两个是编辑器模块和一个 UHT 插件：

```
Plugins/UnLua/Source/
├── UnLua/                        # 运行时（118 文件）：LoadingPhase = PreDefault
│   ├── Private/ReflectionUtils/  #   FClassDesc / FFunctionDesc / FPropertyDesc
│   ├── Private/Registries/       #   7 个 registry（Class/Object/Function/Property/Enum/Container/Delegate）
│   ├── Private/BaseLib/          #   手写导出：TArray/TMap/TSet/Delegate/Object/Class...
│   ├── Private/MathLib/          #   手写导出：FVector/FRotator/FTransform...
│   └── Private/LuaFunction.cpp   #   覆写机制的核心
├── UnLuaEditor/                  # 绑定按钮、模板生成、IntelliSense 生成
├── UnLuaDefaultParamCollector/   # UHT 插件：把 C++ 默认参数值收集成表
└── ThirdParty/Lua/               # lua-5.4.3（默认）与 lua-5.4.4
```

## 二、一个 Actor 是怎么变成一张 Lua 表的

入口是一个空接口 `UUnLuaInterface`，只有一个方法 `GetModuleName`。蓝图里实现它、返回 `"Player.BP_PlayerCharacter_C"`，绑定就成立了。

真正干活的是 `FLuaEnv::TryBind`（`LuaEnv.cpp:336`）。它挂在 `GUObjectArray` 的对象创建/删除监听上，每个新对象都过一遍：

```cpp
static UClass* InterfaceClass = UUnLuaInterface::StaticClass();
const bool bImplUnluaInterface = Class->ImplementsInterface(InterfaceClass);
...
if (Class->GetName().Contains(TEXT("SKEL_")))   // 跳过骨架类
    return false;
const auto ModuleName = ModuleLocator->Locate(Object);
```

几个容易忽略的细节：

- **模块名是从 CDO 上取的**（`ULuaModuleLocator::Locate`，`LuaModuleLocator.cpp:18`），所以它是**类级别**的，同一个类的所有实例只能绑同一个 Lua 模块。要按实例区分，得走动态绑定（`SpawnActor` / `NewObject` 时传模块名）。
- 官方还提供了 `ULuaModuleLocator_ByPackage`：直接把包路径转成模块路径，连接口都不用实现。大型项目值得换成这个，省掉几百个蓝图逐个点绑定按钮。
- 异步加载线程里创建的对象不能立刻绑（Lua 不是线程安全的），会先进 `Candidates` 队列，等 `OnAsyncLoadingFlushUpdate` 回到主线程再处理。

绑定的第二步是 `UUnLuaManager::BindClass`，这里有一个很少被提到的行为——**模块表会被浅拷贝一份**：

```cpp
if (!Class->IsChildOf<UBlueprintFunctionLibrary>())
{
    // 一个LuaModule可能会被绑定到一个UClass和它的子类，复制一个出来作为它们的实例的元表
    lua_newtable(L);
    lua_pushnil(L);
    while (lua_next(L, -3) != 0) { ... }
}
```
> `UnLuaManager.cpp:283`

也就是说 `require` 回来的那张表不是实例真正用的那张。**运行时往原模块表上塞东西，已绑定的实例看不到**——这条同时也是热重载必须做额外工作的根因。

第三步 `FObjectRegistry::Bind`（`ObjectRegistry.cpp:113`）把三层串起来：

```
INSTANCE（每个对象一张空表）
  ├─ .Object     = RAW_UOBJECT（userdata，里面存着 UObject*）
  └─ metatable → MODULE（模块表的拷贝，__index = Lua 的 Index 函数）
                   ├─ .Super      = 父模块表（UnLua.Class(super) 设的）
                   ├─ .Overridden = CLASS_METATABLE
                   └─ metatable → CLASS_METATABLE（每个 UStruct 一张，__index = Class_Index C 函数）
```

于是 `self.Foo` 的查找顺序天然就是：实例自己的字段 → Lua 模块链 → UE 反射。三层各管一段，语义很干净。

最后 `Bind` 会调用 Lua 侧的 `Initialize`。注意这时对象还带着 `RF_NeedInitialization`，而 `FFunctionDesc::CheckObject`（`FunctionDesc.cpp:550`）会拦住任何 UFunction 调用：

```cpp
if (Object->HasAnyFlags(RF_NeedInitialization))
{
    Error = FString::Printf(TEXT("attempt to call UFunction '%s' in lua Initialize function on object '%s'."), ...);
```

`Initialize` 里只能初始化纯 Lua 状态，别碰引擎。

## 三、核心：覆写是怎么做到的

这是 UnLua 最有意思的部分，也是官方文档**已经过时**的部分。

### 3.1 先看 UFunction 有两种调用路径

`UObject::ProcessEvent` 最终会走到 `UFunction::Invoke`。如果函数带 `FUNC_Native`，就直接调它的 thunk 函数指针（`FNativeFuncPtr`）；否则进 `ProcessInternal`，由蓝图 VM 解释 `Script` 里的字节码。

`Docs/CN/How_To_Implement_Overriding.md` 讲的是两套方案：替换 thunk 函数，以及**注册一个新的 opcode 注入字节码**。后者在 1.x 里确实存在。但在 2.3.6 的代码里 grep 不到任何自定义 opcode 注册——2.x 换了个更巧的做法。

### 3.2 把指针藏在字节码里

`LuaFunction.cpp` 开头这两行是整个插件最精妙的地方：

```cpp
static constexpr uint8 ScriptMagicHeader[] = {EX_StringConst, 'L', 'U', 'A', '\0', EX_UInt64Const};
static constexpr size_t ScriptMagicHeaderSize = sizeof ScriptMagicHeader;
```
> `LuaFunction.cpp:23`

覆写一个"类自己身上已有实现"的蓝图函数时（`ULuaFunction::SetActive`，`LuaFunction.cpp:226`）：

```cpp
Script = Function->Script;          // 原字节码搬到 ULuaFunction 上保存
Children = Function->Children;      // 参数属性链直接共享（不是拷贝）
...
Function->FunctionFlags |= FUNC_Native;
Function->SetNativeFunc(&execScriptCallLua);
Function->Script.Empty();
Function->Script.AddUninitialized(ScriptMagicHeaderSize + sizeof(ULuaFunction*));
const auto Data = Function->Script.GetData();
FPlatformMemory::Memcpy(Data, ScriptMagicHeader, ScriptMagicHeaderSize);
FPlatformMemory::WriteUnaligned<ULuaFunction*>(Data + ScriptMagicHeaderSize, this);
```

原函数的字节码数组被清空，换成 **6 字节魔数头 + 一个裸的 `ULuaFunction*` 指针**。魔数头本身是用真实 opcode 拼的（`EX_StringConst` "LUA\0" + `EX_UInt64Const`），所以这段 buffer 看上去像"压一个字符串常量再压一个 64 位常量"，格式上是自洽的。

拿回来的时候（`ULuaFunction::Get`，`LuaFunction.cpp:52`）就是比对魔数 + 读指针：

```cpp
if (FPlatformMemory::Memcmp(Data, ScriptMagicHeader, ScriptMagicHeaderSize) != 0)
    return nullptr;
return FPlatformMemory::ReadUnaligned<ULuaFunction*>(Data + ScriptMagicHeaderSize);
```

为什么要这么绕？因为**关联关系需要长在 UFunction 对象自己身上**。用外部 map 记录的话，蓝图重编译、`FuncMap` 重建、类被 GC，任何一次都会让映射失效；藏在 `Script` 里则跟着 UFunction 一起生死。

### 3.3 一次覆写产生三个 UFunction

覆写完成后，内存里同名的东西有三份：

| 对象 | 位置 | 内容 |
|------|------|------|
| 原 `UFunction` | 还在原类的 `FuncMap` 里 | `FUNC_Native` + thunk = `execScriptCallLua`，`Script` 里是魔数+指针 |
| `ULuaFunction` | `ULuaOverridesClass`（transient 包） | 原字节码副本、共享的参数链、`FFunctionDesc` |
| `<Name>__Overridden` | 同上 | 原实现的完整复制品，`self.Overridden` 走到这里 |

`ULuaOverridesClass`（`LuaOverridesClass.cpp:19`）是个影子 `UClass`，建在 transient 包里，还特意打了 `CLASS_NewerVersionExists` 标记来躲开 `FBlueprintActionDatabase::RefreshClassActions`——不然编辑器的蓝图节点列表里会冒出一堆幽灵类。它把自己挂进目标类的 `Children` 链表，好让 GC 和 `TFieldIterator` 能看见里面的 `ULuaFunction`。

**为什么要搞影子类而不是原地改？** 因为覆写需要一个"能挂 UFunction 且不属于业务类"的容器：`ULuaFunction` 必须是某个 `UClass` 的 field 才能被正常引用和回收，同时又不能真的加进业务类的字段列表污染序列化。

另一条分支是覆写**继承来的**函数（比如在 BP 类里覆写 `AActor::ReceiveBeginPlay`）。此时 `Function->GetOuter() != Class`，走 `bAddNew` 路径：不动父类的 UFunction，只是在子类 `FuncMap` 里加一条新的 `ULuaFunction`，thunk 直接就是 `execCallLua`。这也是绝大多数实际用法。

### 3.4 谁能被覆写

```cpp
bool ULuaFunction::IsOverridable(const UFunction* Function)
{
    static constexpr uint32 FlagMask = FUNC_Native | FUNC_Event | FUNC_Net;
    static constexpr uint32 FlagResult = FUNC_Native | FUNC_Event;
    return Function->HasAnyFunctionFlags(FUNC_BlueprintEvent) || (Function->FunctionFlags & FlagMask) == FlagResult;
}
```
> `LuaFunction.cpp:74`

翻译过来：只有**蓝图事件**（`BlueprintImplementableEvent` / `BlueprintNativeEvent` / 蓝图里自定义的 Event 和 Function）以及**非网络的 native event** 能被覆写。加上 `GetOverridableFunctions` 里额外扫的 `ClassReps`（`RepNotify` 函数），构成完整的可覆写集合。

这解释了两个高频问题：

- **为什么要写 `ReceiveBeginPlay` 而不是 `BeginPlay`**：蓝图里看到的是 `DisplayName`，真实的 `UFUNCTION` 名字是 `ReceiveBeginPlay`。
- **为什么普通的 `BlueprintCallable` C++ 函数覆写不了**：它没有 `FUNC_Event`。想改一段蓝图逻辑，官方给的办法是先在蓝图里 Collapse To Function，再覆写这个函数。

覆写清单本身是**两个集合求交**（`UnLuaManager.cpp:309`）：

```cpp
// 用LuaTable里所有的函数来替换Class上对应的UFunction
for (const auto& LuaFuncName : BindInfo.LuaFunctions)
{
    UFunction** Func = BindInfo.UEFunctions.Find(LuaFuncName);
    if (Func) ULuaFunction::Override(Function, Class, LuaFuncName);
}
```

Lua 表里的函数名 ∩ 可覆写 UFunction 名。名字对不上就只是个普通 Lua 方法，**不会有任何报错**——拼错函数名导致"覆写没生效"的调试成本，这里就是根源。

输入事件、`AnimNotify` 走的是另一套：它们没有对应的 UFunction，UnLua 就拿 `UUnLuaManager` 上预先写好的模板函数（`InputAction` / `InputAxis` / `TriggerAnimNotify`…）复制一份、改成目标名字挂到业务类上，再把 `FInputActionBinding` 的委托绑到这个名字（`UnLuaManager.cpp:367` 起）。所以 Lua 里写 `function M:Fire_Pressed()` 就能收到输入，本质是"凭空造一个 UFunction 出来"。

### 3.5 运行时怎么找到 Lua 函数

thunk 被调用后进 `FFunctionRegistry::Invoke`（`FunctionRegistry.cpp:22`）。第一次调用时它沿着 `Super` 链在元表里找同名 Lua 函数，然后把结果 `luaL_ref` 进注册表缓存起来：

```cpp
do {
    lua_pushstring(L, FuncDesc->GetLuaFunctionName());
    lua_rawget(L, -2);
    if (lua_isfunction(L, -1)) { ...; FuncRef = luaL_ref(L, LUA_REGISTRYINDEX); break; }
    lua_pop(L, 1);
    lua_pushstring(L, "Super");
    lua_rawget(L, -2);
    lua_remove(L, -2);
} while (lua_istable(L, -1));
```

找不到时不会崩，而是**回落到原实现**：

```cpp
// 可能因为Lua模块加载失败导致找不到对应的function，转发给原函数
const auto Overridden = Function->GetOverridden();
if (Overridden && Stack.Code)
    Overridden->Invoke(Context, Stack, RESULT_PARAM);
```

这个设计很务实：脚本挂了，游戏退化成原来的蓝图行为，而不是当场炸。

### 3.6 代价：`FuncMap` 是进程级的

覆写改的是 `UClass::FuncMap`，而 `UClass` 在整个进程里只有一份。这意味着：

- 一个类的覆写对**所有实例、所有 World、所有 PIE 会话**同时生效；
- 编辑器里绑定过一次，状态会留到下一次 PIE，除非 `Restore`；
- 蓝图重编译会清空 `FuncMap`，覆写静默失效。

第三点的处理办法相当朴素——往 `FuncMap` 里塞一个哨兵函数，下次绑定时看它还在不在（`UnLuaManager.cpp:262`）：

```cpp
#if WITH_EDITOR
    // 兼容蓝图Recompile导致FuncMap被清空的情况
    if (Class->FindFunctionByName("__UClassBindSucceeded", EIncludeSuperFlag::Type::ExcludeSuper))
        return true;
    ULuaFunction::RestoreOverrides(Class);
#endif
```

`FLuaOverrides` 还提供 `Suspend`/`Resume`，并且注册了 `GUObjectArray` 的删除监听，在类被销毁时自动 `Restore`。这些都是"改全局状态"必须付的税。

## 四、访问路径：元表就是缓存

`self.Health` 看起来是一次表查找，实际走的步数值得算一下。

Lua 侧的 `Index` 函数（用 C 字符串内嵌在 `UnLuaLib.cpp:163` 的 chunk 里）：

```lua
local function Index(t, k)
    local mt = getmetatable(t)
    local super = mt
    while super do                              -- 1. 先走 Lua 模块链
        local v = rawget(super, k)
        if v ~= nil and not rawequal(v, NotExist) then
            rawset(t, k, v)                     --    找到就缓存进实例表
            return v
        end
        super = rawget(super, "Super")
    end

    local p = mt[k]                             -- 2. 落到 CLASS_METATABLE 的 __index
    if p ~= nil then
        if type(p) == "userdata" then
            return GetUProperty(t, p)           -- 3a. 属性：每次都要真读
        elseif type(p) == "function" then
            rawset(t, k, p)                     -- 3b. 函数：闭包缓存进实例表
        elseif rawequal(p, NotExist) then
            return nil
        end
    else
        rawset(mt, k, NotExist)                 -- 4. 连"不存在"也缓存
    end
    return p
end
```

C 侧的 `Class_Index` → `GetField`（`LuaCore.cpp:1061`）逻辑是：

```cpp
lua_getmetatable(L, 1);
lua_pushvalue(L, 2);
int32 Type = lua_rawget(L, -2);
if (Type == LUA_TNIL)
    GetFieldInternal(L);        // 只有第一次会走反射解析
```

`GetFieldInternal` 解析出 `FFieldDesc` 后，**把结果写回元表**（`LuaCore.cpp:1030`）：属性存成一个装着 `TSharedPtr<ITypeOps>` 的 userdata，函数则直接压一个 `Class_CallUFunction` 闭包。继承来的字段会同时写进父类元表和子类元表两份缓存。

所以真实成本是：

| 访问 | 首次 | 之后 |
|------|------|------|
| Lua 方法 `self:Foo()` | 模块链 rawget | 实例表命中，无 metamethod |
| UFunction `self:K2_GetActorLocation()` | 模块链 miss → C 调用 → 反射解析 → 建 `FFunctionDesc` | 实例表命中闭包，直接进 `CallUE` |
| 属性 `self.Health` | 同上 | **每次**：模块链 miss ×N → `Class_Index`（C 调用）→ 元表 rawget → 返回描述符 → `GetUProperty`（第二次 C 调用）→ 真读 |
| 不存在的字段 | 一次反射查找 | `NotExist` 哨兵，纯表命中 |

属性读写是唯一没法被缓存掉的那一类——**必须每次执行**，而且要过两次 C 边界。更贵的是默认开着的类型检查（`ENABLE_TYPE_CHECK=1`），每次属性访问都会做一次 `IsA`（`LowLevel.cpp:135`）：

```cpp
UClass* OwnerClass = Property->GetOwnerClass();
if (Object->IsA(OwnerClass)) return true;
luaL_error(L, ... "Access property from invalid owner. %s should be a %s.");
```

实践结论很直接：**热路径上别写 `self.A.B.C`**，把属性和函数都拉到 local 里。顺便说一句，仓库自带的 benchmark（`Content/Script/Tests/Benchmark/UnLuaBenchmarkProxy.lua`）测的是 `local RawObject = Proxy.Object` 之后的裸 userdata 路径，跳过了 Lua `Index` 和模块链——它给的是**下限**，不是你在业务代码里会得到的数字。

## 五、双向调用与参数

### Lua → UE

`FFunctionDesc::CallUE`（`FunctionDesc.cpp:171`）的骨架：

1. `Buffer->Get()` 拿参数帧；
2. `PreCall`：逐个 `InitializeValue` + 从 Lua 栈 `WriteValue_InContainer`，缺的参数用 UHT 收集来的默认值补；
3. `Object->UObject::ProcessEvent(FinalFunction, Params)`——注意是**非虚调用**，绕过任何子类对 `ProcessEvent` 的重写；
4. `PostCall`：返回值和出参压回 Lua 栈，然后 `DestroyValue`；
5. `Buffer->Pop(Params)`。

参数帧的分配策略由 `ENABLE_PERSISTENT_PARAM_BUFFER`（默认开）决定。持久模式下每个 UFunction 维护一个 buffer 栈，靠计数器支持递归（`ParamBufferAllocator.cpp:38`，这个计数器正是 2.3.5 修 #563 递归覆盖 bug 加的）；关掉则退化成每次 `FMemory::Malloc` + `Memzero` + `Free`。**默认配置下，Lua 调 UE 函数在预热后不产生堆分配**，代价是这些 buffer 只增不减。

有个容易踩的语义：如果 Lua 调的这个 UFunction **自己就被 Lua 覆写过**，会被重定向到原实现（`FunctionDesc.cpp:213`）：

```cpp
#if ENABLE_CALL_OVERRIDDEN_FUNCTION
    const auto LuaFunction = ULuaFunction::Get(Function.Get());
    if (LuaFunction && LuaFunction->GetOverridden())
        FinalFunction = LuaFunction->GetOverridden();
#endif
```

这就是 `self.Overridden.ReceiveBeginPlay(self)` 能调到原蓝图逻辑的原因——`Overridden` 是 `CLASS_METATABLE`，从它拿到的是反射闭包，闭包进 `CallUE` 后被重定向。也因此**只能写 `.` 不能写 `:`**（`Content/Script/Tutorials/02_OverrideBlueprintEvents.lua` 里专门写了这条注意）：`self.Overridden:SayHi(name)` 会把元表当成 self 传进去。

### UE → Lua

`FFunctionDesc::CallLua`（`FunctionDesc.cpp:78`）要处理一个麻烦：调用可能来自蓝图字节码，参数还躺在指令流里。于是有 `bUnpackParams` 分支，手动 `Stack.Step` 把每个参数解到 buffer 里，顺便重建一条 `FOutParmRec` 链；否则直接用 `Stack.Locals`。

之后是 `lua_pcall`，然后按顺序把出参写回。这里有一段很诚实的注释（`FunctionDesc.cpp:466`）：

```cpp
// out value
// suppose out param is also pushed on stack? this is assumed done by user... so we can not trust it
```

Lua 函数不返回值时怎么办？`ENABLE_TYPE_CHECK` 开着就报错，关着就用默认值（2.3.5 改的行为）。返回值和出参的顺序由 `UNLUA_LEGACY_RETURN_ORDER` 控制。

### 默认参数与协程

C++ 函数签名里的默认值在反射数据里是**不存在**的。UnLua 为此专门做了个 UHT 插件 `UnLuaDefaultParamCollector`，编译期扫所有 `BlueprintCallable`/`Exec` 函数，把默认值生成成 `GDefaultParamCollection`，运行时 `PreCall` 里补上。为了让 Lua 少写几个参数，代价是一个 build 期插件——这个取舍挺能说明 UnLua 的风格。

Latent 函数（`Delay`、`MoveTo` 这类）走协程：`PreCall` 检测到名为 `LatentInfo` 的参数时，合成一个 `FLatentActionInfo`，回调目标写成 `UUnLuaManager::OnLatentActionCompleted`，`LinkID` 就是协程的 registry ref（`FunctionDesc.cpp:300`）：

```cpp
FLatentActionInfo LatentActionInfo(ThreadRef, GetTypeHash(FGuid::NewGuid()), TEXT("OnLatentActionCompleted"), (Env.GetManager()));
```

引擎那边 latent action 完成后回调 → `Env->ResumeThread(LinkID)` → `lua_resume`。所以 Lua 里可以直接写 `UE.UKismetSystemLibrary.Delay(self, 1.0)` 然后往下写，前提是这段代码跑在协程里。

## 六、值语义：最容易炸的地方

这一节是我认为 UnLua 最需要提前知道的部分。

`FPropertyDesc::GetValueInternal` 有个 `bCreateCopy` 参数。以结构体为例（`PropertyDesc.cpp:1219`）：

```cpp
if (bCreateCopy)
{
    void *Userdata = NewUserdataWithPadding(L, StructSize, StructName.Get(), UserdataPadding);
    StructProperty->InitializeValue(Userdata);
    StructProperty->CopySingleValue(Userdata, ValuePtr);     // 真拷贝
}
else
{
    UnLua::PushPointer(L, (void*)ValuePtr, StructName.Get(), bFirstPropOfScriptStruct);  // 借用指针
}
```

而 `Class_Index` 读属性时传的是 **`false`**（`LuaCore.cpp:1218`）：

```cpp
(*Property)->ReadValue_InContainer(L, Self, false);
```

也就是说 `local t = self.SomeTransform` 拿到的**不是拷贝，是一个指向 UObject 内存的视图**。改 `t` 就是改对象本身（很多人靠这个特性写 in-place 修改），但反过来：对象销毁、数组扩容搬家、参数帧复用之后，这个视图就是悬垂指针。

更隐蔽的是覆写函数的参数。`CallLuaInternal`（`FunctionDesc.cpp:449`）：

```cpp
Property->ReadValue_InContainer(L, InParams, !UNLUA_LEGACY_ARGS_PASSING);
```

`UNLUA_LEGACY_ARGS_PASSING` 默认是 **1**，取反就是 `bCreateCopy = false`——**Lua 覆写函数收到的结构体参数，是指向调用方参数帧的指针**。函数返回后那块 buffer 会被复用。把参数存进成员变量留到下一帧用，是标准的踩雷姿势。2.3.3 加的"默认传参"设置就是让你把它改成拷贝模式：安全，但每次调用多一次结构体拷贝。

容器同理：`TArray` 的 `Get` 返回元素拷贝，`GetRef` 返回引用——示例代码里 `InterpFloats:GetRef(1)` 用的就是后者，因为它接着要改这个元素。

UnLua 自己也知道这个设计有风险，配了两道防线：

**1. 悬垂检查（默认关）**。`FDanglingCheck`（`LuaDanglingCheck.cpp`）在跨界调用外面套一个 guard，guard 析构时把这次调用中借出去的 struct userdata 指针置空、container userdata 打上 released 标记：

```cpp
void* Userdata = GetUserdataFast(L, -1, &TwoLevelPtr);
check(TwoLevelPtr)
*(void**)Userdata = nullptr;
```

于是你跨帧访问会得到一条 Lua 报错而不是随机崩溃。开发期建议开着。

**2. `ReleasedPtr` 哨兵**。UObject 被销毁时，`FObjectRegistry::Unbind`（`ObjectRegistry.cpp:200`）不是简单地扔掉映射，而是把 userdata 里存的指针改写成一个特殊值：

```cpp
*((void**)Userdata) = (void*)LowLevel::ReleasedPtr;
```

之后 Lua 侧任何访问都会命中 `IsReleasedPtr` 检查，报 `attempt to read property 'X' on released object`。这比裸指针访问友好得多。

## 七、两套 GC 的接缝

Lua 有自己的 GC，UE 有自己的 GC，两边都想管对象生死，接缝处就是 bug 的温床。UnLua 的处理是：**Lua 侧全部用弱表，UE 侧用显式 root 集合**。

`FLuaEnv` 构造时建了一批弱值表（`LuaEnv.cpp:121`、`ObjectRegistry.cpp:52`）：`UnLua_ObjectMap`、`StructMap`、`ArrayMap`、`ScriptContainerMap`、`UnLua_ManualRefProxyMap`。它们只是"同一个 C++ 地址复用同一个 Lua 对象"的缓存，不阻止回收。

UE 侧有两个 `FObjectReferencer`（`LuaEnv.cpp:118`）：

- `AutoObjectReference`：Lua 侧持有期间自动 root，Lua GC 掉对应 userdata 时通过 `NotifyUObjectLuaGC` 移除；
- `ManualObjectReference`：`UnLua.Ref` / `UnLua.Unref` 手动控制，配一个带 `__gc` 的 `FManualRefProxy` 兜底。

绑定过的对象另有一层：它的 `INSTANCE` 表被 `luaL_ref` 强引用在注册表里（`ObjectRegistry.cpp:155`），直到 `Unbind` 才释放。所以"Lua 表会不会比 UObject 活得久"这个问题的答案是：**会，但只在 `Unbind` 之前，而 `Unbind` 由 `NotifyUObjectDeleted` 驱动**。

`FLuaEnv::NotifyUObjectDeleted`（`LuaEnv.cpp:265`）按固定顺序通知全部 registry——顺序本身就是踩过坑的产物：

```cpp
PropertyRegistry->NotifyUObjectDeleted(Object);
FunctionRegistry->NotifyUObjectDeleted(Object);
if (Manager) Manager->NotifyUObjectDeleted(Object);
ObjectRegistry->NotifyUObjectDeleted(Object);
ClassRegistry->NotifyUObjectDeleted(Object);
EnumRegistry->NotifyUObjectDeleted(Object);
```

Lua GC 参数也值得一提（`LuaEnv.cpp:135`）：

```cpp
#if 504 == LUA_VERSION_NUM
    lua_gc(L, LUA_GCGEN, 0, 0);          // 5.4 默认开分代 GC
#else
    lua_gc(L, LUA_GCSETPAUSE, 100);      // 5.3 走激进的增量参数
    lua_gc(L, LUA_GCSETSTEPMUL, 5000);
#endif
```

而且 lua_State 用的是 UE 的分配器（`FLuaEnv::DefaultLuaAllocator`），所以 Lua 堆会出现在 UE 的内存统计里——这在排查移动端内存时很重要。

顺带一句，从 CHANGELOG 看，"lua 在 GC 时偶现崩溃"这类问题从 2.2.0 的 GC 机制调整一直修到 2026 年 4 月（`develop` 上最新的提交就是"UObject在lua侧gc时偶现崩溃问题修复"）。两套 GC 的接缝确实是这个插件长期的痛点。

实践规则：

- 需要在 Lua 里长期持有的对象，别只靠一个 Lua 变量——要么它本来就有 UE 侧引用，要么 `UnLua.Ref`；
- 结构体和容器引用**当成局部变量用完就扔**，不要跨帧保存；
- 开发期打开悬垂检查，让问题在报错处暴露而不是在崩溃处；
- 访问可能已销毁的对象前先 `IsValid`。

## 八、性能：能从代码里推出来的部分

先说结论：**Lua → UE 的反射调用，成本量级上和"蓝图节点调 C++ 函数"是同一档**，因为两者最终都走 `ProcessEvent`。UnLua 真正的胜负手在于**逻辑本身用 Lua 跑还是用蓝图 VM 跑**——密集的分支、循环、表操作，Lua 明显更快；而跨界频次高的代码，两边都不便宜。

各操作的固定开销来源：

| 操作 | 每次都要付的成本 |
|------|------------------|
| 属性读 | 模块链 rawget ×N → C 调用 `Class_Index` → 元表 rawget → `GetCppInstance` → `IsA`（类型检查开时）→ 第二次 C 调用 `GetUProperty` → `ReadValue` |
| 属性写 | 同上，末端换成 `WriteValue` |
| UFunction 调用 | 闭包命中（已缓存）→ `PreCall`（逐参数 `InitializeValue` + 写入）→ `GetFunctionCallspace` → `ProcessEvent` → `PostCall` + `DestroyValue` |
| 被覆写函数被调用 | thunk → `IUnLuaModule::GetEnv` → registry 查 `ULuaFunction` → 可能的 `Stack.Step` 解参 → 逐参数压栈 → `lua_pcall` → 出参写回 |
| 静态导出函数调用 | 直接 C 函数，无 `FFunctionDesc`、无参数帧、无 `ProcessEvent` |
| `FString` / `FName` 进出 | 每次都有编码转换和分配（`TCHAR_TO_UTF8` / `FString` 构造） |
| 数学类型运算 | `FVector` 等是手写导出的，但每次运算结果是一块新 userdata |

可调的开关（`UnLua.Build.cs:91`）：

| 宏 | 默认 | 影响 |
|----|------|------|
| `ENABLE_TYPE_CHECK` | 1 | 关掉可省下每次属性访问的 `IsA` 和每个参数的类型校验；代价是类型错误变成未定义行为 |
| `ENABLE_PERSISTENT_PARAM_BUFFER` | 1 | 参数帧复用，消除每次调用的 malloc/free |
| `UNLUA_LEGACY_ARGS_PASSING` | 1 | 1 = 结构体参数借指针（快、危险），0 = 拷贝（慢、安全） |
| `ENABLE_CALL_OVERRIDDEN_FUNCTION` | 1 | 提供 `self.Overridden`，每次 `CallUE` 多一次 `ULuaFunction::Get` |
| `AUTO_UNLUA_STARTUP` | 1 | 引擎启动即建 lua_State |
| `UNLUA_ENABLE_DEBUG` | 0 | 大量日志，只在排查时开 |
| `ENABLE_UNREAL_INSIGHTS` | 0 | `lua_sethook` 把 Lua 调用打进 Insights，与死循环检测互斥 |

代码层面能确定的优化手段就那么几条，但都实在：把 UFunction 和常用属性缓存到 local；用静态导出接管真正的热点；把 per-frame 的 Tick 逻辑留在 C++/蓝图里，只在事件驱动的地方进 Lua；数学运算尽量 in-place 而不是链式产生临时对象。

内存侧要记住三笔账：每个绑定对象一张 Lua 表（还会随着方法调用把闭包 `rawset` 进去，越用越大）、每个 `UStruct` 一张元表（同时是字段解析缓存）、每个被 Lua 调过的 UFunction 一个只增不减的参数 buffer 池。

## 九、热更新与工程化

热更新能力其实藏在模块加载器里。`FLuaEnv` 往 `package.searchers` 插了三个 searcher（`LuaEnv.cpp:98`），其中文件系统那个的查找顺序是（`LuaEnv.cpp:622`）：

```cpp
// 优先加载下载目录下的单文件
FPaths::Combine(FPaths::ProjectPersistentDownloadDir(), Pattern)
// 其次是打包目录下的文件
FPaths::Combine(FPaths::ProjectDir(), Pattern)
```

**下载目录优先于包内目录**——这就是整条热更通道：把新脚本下到 `PersistentDownloadDir`，`require` 自然优先命中。默认搜索路径是 `Content/Script/?.lua;Plugins/UnLua/Content/Script/?.lua`，改 `package.path` 无效（UE 有自己的文件系统），要改 `UnLua.PackagePath`。想做加密或自定义打包格式，走 `FLuaEnv::AddLoader` 注册自定义 loader。

开发期的 `UnLua.HotReload()` 是另一回事：`HotReload.lua` 明确写着参考云风的方案，尽量保留 upvalue 和运行时 table。但结合前面两个事实——模块表在绑定时被**拷贝**过、Lua 方法会被 `rawset` 缓存进**实例表**——就能推出它的能力边界：新逻辑对新实例总是生效，对老实例取决于缓存有没有被正确替换。文档自己的说法是"为开发期设计，尽量替换。最差的结果就是重新启动"，这个定位很诚实。

工程化的其他部分：

- **IntelliSense**：`UnLuaEditor` 能遍历所有 `UClass`/`UStruct`/`UEnum` 生成 EmmyLua 注解存根，配 `---@type BP_PlayerCharacter_C` 就有补全，还带一个 commandlet 方便进 CI。缺点是生成物需要在类型变化后重新生成。
- **打包**：Lua 文件不是 asset，要在 Packaging 设置的 "Additional Non-Asset Directories to Package" 里加 Script 目录，这是官方 FAQ 第二条。
- **静态导出**：`UNLUA_EXPORT_CLASS` / `ADD_FUNCTION` / `ADD_PROPERTY` / `EXPORT_FUNCTION` 一套宏，静态初始化期注册，和反射数据在同一张元表里合并。插件自己的 `FVector`、`TArray` 就是这么导出的——**它自己的热点也不走反射**，这点很说明问题。
- **测试**：`UnLuaTestSuite` 里按 GitHub issue 号组织回归用例（`Content/Script/Tests/Regression/Issue###/`），几十个，甚至包括中文蓝图名（Issue322）这种 case。对一个开源插件来说算相当规范。

## 十、2026 年的现状：不是停更，是搬家了

这部分容易被误判，所以说清楚（数据取自 GitHub API，2026-08）：

- 最后一个 tag 是 **v2.3.6，2023-11-07**，之后近三年没有发布过版本；
- `master` 分支自 2024 年起只有 **1 个提交**（2025-07-07 改 LICENSE 的公司主体）；
- 但 **`develop` 分支还活着**：UE5.4 支持在 2024-05（PR #700），UE5.6 编译修复在 2025-09 ~ 2025-12（PR #758），最新提交是 **2026-04-02** 的"UObject 在 lua 侧 gc 时偶现崩溃问题修复"（PR #765）；
- 推动这些提交的主要是社区贡献者（jozhn、crazytuzi 等）而非官方投入；
- UE5.7 目前**不支持**，相关 issue 还开着；
- 2755 stars / 718 forks / 185 个 open issues+PRs。

结论应该是："**官方发布线冻结在 UE5.3，真正可用的新版本在 `develop` 上，且靠社区维护**"。如果你现在要接 UE5.4+，实际做法是从 `develop` 拉代码，并接受"没有 tag、没有 release note、issue 响应看志愿者心情"。

同一时期的邻居们：

| 项目 | 状态（2026-08） | 绑定方式 |
|------|------------------|----------|
| **UnLua** | 2755★，release 冻结在 2023-11，develop 活跃到 2026-04 | 运行时反射 |
| **sluaunreal**（腾讯） | 1974★，最后 push 2025-10，tag 2.1.4（2024-06） | 反射 + 代码生成 |
| **puerts**（腾讯，TS/JS） | 6161★，最后 push 2026-07-31，三者中最活跃 | 代码生成 + 反射 |
| **UnrealEnginePython** | 最后 push 2022-06，事实上已废弃 | 反射 |
| **Angelscript**（Hazelight） | 不在 GitHub 开源，走 angelscript.hazelight.se | 编译期静态绑定 |

关于 UnLua 的采用规模，官方 FAQ 里有一句可以直接引用的原话：

> 腾讯内部已知的有四十款左右项目在使用UnLua，外部项目暂时无法统计。

另外顺手说一个常被问的问题：**能不能换 LuaJIT**。仓库里确实有 `feature/luajit` 和 `feature/lua51` 分支（都停在 2023-02 的"增加：luajit的构建脚本"），2.3.5 也加了"自定义 Lua 版本"设置。但代码层面有硬约束——`LuaCore.cpp` 直接 `#include "lstate.h" / "lobject.h"`，自己重写了 `lua_index2value`，还在 userdata 尾部塞了一个魔数结构体来打标记：

```cpp
#define USERDATA_MAGIC  0x1688
#define BIT_TWOLEVEL_PTR        (1 << 5)
struct FUserdataDesc { uint16 magic; uint8 tag; uint8 padding; };
```

这套东西依赖的是 Lua 5.4 的 `TValue`/`Udata` 内存布局。换 LuaJIT 不是改个宏的事，得重做这一层。**把它当成一个绑定在 Lua 5.4 上的插件来评估比较现实。**

## 什么时候该用它

适合：已经在用蓝图组织玩法、需要移动端热更、团队里有 Lua 积累、能接受"运行时才发现类型错误"的项目。它最大的价值是**边际成本极低**——加一个 `UPROPERTY` 不需要动任何绑定代码，这个好处在项目生命周期长、迭代频繁时会不断兑现。

不适合：追求跨界调用极致性能（那应该静态绑定或干脆写 C++）、需要强类型和大规模重构支持（考虑 puerts 的 TS 路线或 Angelscript）、要跟最新引擎版本（`develop` 拉代码的运维成本要算进来）、团队没人愿意在出问题时读这套覆写机制的源码。

最后一条其实是关键。UnLua 的代码质量不错，但它做的事情——在字节码里藏指针、给 UClass 挂影子类、改进程级 `FuncMap`、在两套 GC 之间借指针——注定不是黑盒。**用它就要有人能读懂它**。这篇文章希望能帮你把这个"有人"变得容易一点。
