---
title: "Inside UnLua: How Lua Grafts Itself onto Unreal's Reflection System"
date: 2026-08-04
tags: ["Unreal Engine", "Lua", "UnLua", "C++", "Scripting"]
ShowToc: true
TocOpen: false
summary: "A source-level teardown of Tencent's UnLua: how the override mechanism smuggles Lua functions into UFunctions, how metatables double as the reflection cache, why value semantics are the sharpest edge, how the two garbage collectors are stitched together, and what the project's real maintenance status is in 2026."
---

Embedding a scripting language in an Unreal project is rarely about C++ being slow to write. It's usually about three concrete things: **mobile hot-updates** (App Store rules permit patching interpreted script, but not shipping a new binary), **an iteration loop that doesn't require a compile** for designers and gameplay programmers, and the fact that **Blueprint maintenance costs blow up at scale** — node graphs can't be diffed, can't be reviewed, and merge conflicts are effectively unresolvable.

Lua is the most battle-tested answer to that problem, and [UnLua](https://github.com/Tencent/UnLua) is Tencent's open-source take on it. Its pitch is "zero glue code": no bindings to write, full access to every `UCLASS` / `UPROPERTY` / `UFUNCTION` from Lua, and the ability to override Blueprint events outright.

This article isn't a usage guide — the official docs cover that. It's a teardown: **what that "zero glue" is actually bought with, where the costs land, and which parts bite.**

The analysis is based on `master` at `3f112e8`, functionally equivalent to **v2.3.6** (released 2023-11-07, the last tag). Every `file:line` reference points at that revision. The maintenance status as of 2026 gets its own section at the end — it's more nuanced than "abandoned."

## 1. Three roads, and it took the most expensive one

There are only three engineering routes for exposing C++ objects to a script VM:

| Approach | Examples | Cost | Benefit |
|----------|----------|------|---------|
| Hand-written bindings | Classic Lua projects | Glue for every class; silently rots when interfaces change | Lowest call overhead, fully controlled behaviour |
| Code generation | sluaunreal, puerts | Must run a generator, artifacts land in VCS, longer builds | Static type info, near hand-written performance |
| Runtime reflection | **UnLua** | Every call pays for the reflection layer; type errors only surface at runtime | Zero maintenance — engine upgrades and Blueprint field changes are picked up automatically |

UnLua took the third road. Unreal's reflection system (`UClass` / `UFunction` / `FProperty`) already is a complete runtime metadata layer built for the Blueprint VM and serialization, so UnLua effectively **bolts a second consumer onto it**. That single decision explains nearly everything downstream: adding a `UPROPERTY` requires no script-layer work at all, but every boundary crossing pays the reflection tax — and a lot of the "gotchas" are just Unreal's reflection semantics leaking straight through.

The whole plugin is 148 h/cpp files excluding third-party Lua — 118 of them in the runtime `UnLua` module, plus an editor module and a UHT plugin:

```
Plugins/UnLua/Source/
├── UnLua/                        # runtime (118 files), LoadingPhase = PreDefault
│   ├── Private/ReflectionUtils/  #   FClassDesc / FFunctionDesc / FPropertyDesc
│   ├── Private/Registries/       #   7 registries (Class/Object/Function/Property/Enum/Container/Delegate)
│   ├── Private/BaseLib/          #   hand-exported: TArray/TMap/TSet/Delegate/Object/Class...
│   ├── Private/MathLib/          #   hand-exported: FVector/FRotator/FTransform...
│   └── Private/LuaFunction.cpp   #   the heart of the override mechanism
├── UnLuaEditor/                  # bind button, template generation, IntelliSense generation
├── UnLuaDefaultParamCollector/   # UHT plugin: harvests C++ default parameter values
└── ThirdParty/Lua/               # lua-5.4.3 (default) and lua-5.4.4
```

## 2. How an Actor becomes a Lua table

The entry point is an empty interface, `UUnLuaInterface`, with a single method: `GetModuleName`. Implement it in a Blueprint, return `"Player.BP_PlayerCharacter_C"`, and the binding exists.

The real work happens in `FLuaEnv::TryBind` (`LuaEnv.cpp:336`), which hooks `GUObjectArray`'s create/delete listener and filters every new object:

```cpp
static UClass* InterfaceClass = UUnLuaInterface::StaticClass();
const bool bImplUnluaInterface = Class->ImplementsInterface(InterfaceClass);
...
if (Class->GetName().Contains(TEXT("SKEL_")))   // skip skeleton classes
    return false;
const auto ModuleName = ModuleLocator->Locate(Object);
```

Three details that are easy to miss:

- **The module name is read off the CDO** (`ULuaModuleLocator::Locate`, `LuaModuleLocator.cpp:18`), so it is a *class-level* property — every instance of a class binds to the same Lua module. Per-instance variation requires dynamic binding (passing a module name at `SpawnActor` / `NewObject` time).
- There's also `ULuaModuleLocator_ByPackage`, which derives the module path from the package path so you don't implement the interface at all. Worth switching to on a large project — it saves clicking the bind button on hundreds of Blueprints.
- Objects created on the async loading thread can't be bound immediately (Lua isn't thread-safe). They queue into `Candidates` and get processed when `OnAsyncLoadingFlushUpdate` returns to the game thread.

Step two is `UUnLuaManager::BindClass`, which does something rarely mentioned — **the module table gets shallow-copied**:

```cpp
if (!Class->IsChildOf<UBlueprintFunctionLibrary>())
{
    // one Lua module may be bound to a UClass and its subclasses, so copy one
    // out to serve as the metatable for their instances
    lua_newtable(L);
    lua_pushnil(L);
    while (lua_next(L, -3) != 0) { ... }
}
```
> `UnLuaManager.cpp:283`

The table `require` returned is *not* the one instances actually use. **Mutating the original module table at runtime is invisible to already-bound instances** — which is also the root reason hot reload needs extra machinery.

Step three, `FObjectRegistry::Bind` (`ObjectRegistry.cpp:113`), wires up three layers:

```
INSTANCE (one empty table per object)
  ├─ .Object     = RAW_UOBJECT (userdata holding the UObject*)
  └─ metatable → MODULE (copy of the module table, __index = the Lua Index function)
                   ├─ .Super      = parent module table (set by UnLua.Class(super))
                   ├─ .Overridden = CLASS_METATABLE
                   └─ metatable → CLASS_METATABLE (one per UStruct, __index = Class_Index C function)
```

So `self.Foo` naturally resolves in the order: the instance's own fields → the Lua module chain → Unreal reflection. Each layer owns one concern; the semantics are clean.

Finally `Bind` calls Lua's `Initialize`. At that point the object still carries `RF_NeedInitialization`, and `FFunctionDesc::CheckObject` (`FunctionDesc.cpp:550`) will reject any UFunction call:

```cpp
if (Object->HasAnyFlags(RF_NeedInitialization))
{
    Error = FString::Printf(TEXT("attempt to call UFunction '%s' in lua Initialize function on object '%s'."), ...);
```

`Initialize` is for pure Lua state only. Don't touch the engine there.

## 3. The core: how overriding actually works

This is the most interesting part of the plugin — and the part where the official documentation is **out of date**.

### 3.1 A UFunction has two call paths

`UObject::ProcessEvent` eventually reaches `UFunction::Invoke`. If the function has `FUNC_Native`, it calls the thunk function pointer (`FNativeFuncPtr`) directly; otherwise it goes through `ProcessInternal`, where the Blueprint VM interprets the bytecode in `Script`.

`Docs/CN/How_To_Implement_Overriding.md` describes two schemes: replacing the thunk, and **registering a new opcode to inject into the bytecode**. The latter really existed in 1.x. But grep the 2.3.6 source and there's no custom opcode registration anywhere — 2.x replaced it with something slicker.

### 3.2 Smuggling a pointer inside the bytecode

These two lines at the top of `LuaFunction.cpp` are the cleverest thing in the plugin:

```cpp
static constexpr uint8 ScriptMagicHeader[] = {EX_StringConst, 'L', 'U', 'A', '\0', EX_UInt64Const};
static constexpr size_t ScriptMagicHeaderSize = sizeof ScriptMagicHeader;
```
> `LuaFunction.cpp:23`

When overriding a Blueprint function the class already implements (`ULuaFunction::SetActive`, `LuaFunction.cpp:226`):

```cpp
Script = Function->Script;          // original bytecode moves onto the ULuaFunction
Children = Function->Children;      // parameter property chain is shared, not copied
...
Function->FunctionFlags |= FUNC_Native;
Function->SetNativeFunc(&execScriptCallLua);
Function->Script.Empty();
Function->Script.AddUninitialized(ScriptMagicHeaderSize + sizeof(ULuaFunction*));
const auto Data = Function->Script.GetData();
FPlatformMemory::Memcpy(Data, ScriptMagicHeader, ScriptMagicHeaderSize);
FPlatformMemory::WriteUnaligned<ULuaFunction*>(Data + ScriptMagicHeaderSize, this);
```

The original function's bytecode array is emptied and replaced by a **6-byte magic header plus a raw `ULuaFunction*`**. The header is built from real opcodes (`EX_StringConst` "LUA\0" followed by `EX_UInt64Const`), so the buffer still *looks* like "push a string constant, push a 64-bit constant" — internally consistent rather than garbage.

Retrieval (`ULuaFunction::Get`, `LuaFunction.cpp:52`) is a magic comparison plus a pointer read:

```cpp
if (FPlatformMemory::Memcmp(Data, ScriptMagicHeader, ScriptMagicHeaderSize) != 0)
    return nullptr;
return FPlatformMemory::ReadUnaligned<ULuaFunction*>(Data + ScriptMagicHeaderSize);
```

Why go to that trouble? Because **the association has to live on the UFunction object itself**. An external map would be invalidated by Blueprint recompilation, `FuncMap` rebuilds, or class GC; a pointer hidden in `Script` lives and dies with the UFunction.

### 3.3 One override, three UFunctions

After an override there are three same-named things in memory:

| Object | Where | Contents |
|--------|-------|----------|
| Original `UFunction` | still in the owning class's `FuncMap` | `FUNC_Native` + thunk = `execScriptCallLua`; `Script` holds magic + pointer |
| `ULuaFunction` | `ULuaOverridesClass` (transient package) | copy of the original bytecode, shared param chain, `FFunctionDesc` |
| `<Name>__Overridden` | same | full duplicate of the original implementation; this is what `self.Overridden` reaches |

`ULuaOverridesClass` (`LuaOverridesClass.cpp:19`) is a shadow `UClass` created in the transient package, deliberately flagged `CLASS_NewerVersionExists` to dodge `FBlueprintActionDatabase::RefreshClassActions` — otherwise ghost classes would show up in the editor's node palette. It splices itself into the target class's `Children` list so GC and `TFieldIterator` can see the `ULuaFunction`s inside.

**Why a shadow class instead of patching in place?** An override needs a container that can own UFunctions without belonging to the game class: a `ULuaFunction` must be a field of *some* `UClass` to be referenced and collected properly, yet must not actually join the game class's field list and pollute serialization.

The other branch is overriding an **inherited** function — say `AActor::ReceiveBeginPlay` from a Blueprint class. Here `Function->GetOuter() != Class`, so the `bAddNew` path runs: the parent's UFunction is left untouched, and a new `ULuaFunction` whose thunk is `execCallLua` is simply added to the subclass's `FuncMap`. This is what the vast majority of real usage hits.

### 3.4 What can be overridden

```cpp
bool ULuaFunction::IsOverridable(const UFunction* Function)
{
    static constexpr uint32 FlagMask = FUNC_Native | FUNC_Event | FUNC_Net;
    static constexpr uint32 FlagResult = FUNC_Native | FUNC_Event;
    return Function->HasAnyFunctionFlags(FUNC_BlueprintEvent) || (Function->FunctionFlags & FlagMask) == FlagResult;
}
```
> `LuaFunction.cpp:74`

In plain terms: only **Blueprint events** (`BlueprintImplementableEvent`, `BlueprintNativeEvent`, and events/functions defined in a Blueprint graph) plus **non-networked native events**. Add the `ClassReps` scan in `GetOverridableFunctions` (i.e. `RepNotify` functions) and you have the complete overridable set.

This explains two FAQs:

- **Why you write `ReceiveBeginPlay`, not `BeginPlay`** — the Blueprint editor shows the `DisplayName`; the real `UFUNCTION` is `ReceiveBeginPlay`.
- **Why an ordinary `BlueprintCallable` C++ function can't be overridden** — it has no `FUNC_Event`. To replace a chunk of Blueprint logic, the official answer is to Collapse To Function first, then override that function.

The override list itself is an **intersection of two sets** (`UnLuaManager.cpp:309`):

```cpp
// replace the corresponding UFunctions on the class with every function in the Lua table
for (const auto& LuaFuncName : BindInfo.LuaFunctions)
{
    UFunction** Func = BindInfo.UEFunctions.Find(LuaFuncName);
    if (Func) ULuaFunction::Override(Function, Class, LuaFuncName);
}
```

Function names in the Lua table ∩ overridable UFunction names. A name that matches nothing is just an ordinary Lua method, and **nothing is reported** — which is exactly why a typo'd function name costs so much debugging time.

Input events and `AnimNotify` take a different route: they have no corresponding UFunction, so UnLua duplicates a template function off `UUnLuaManager` (`InputAction` / `InputAxis` / `TriggerAnimNotify`…), renames it to the target name, attaches it to the game class, and binds the `FInputActionBinding` delegate to that name (`UnLuaManager.cpp:367` onwards). So writing `function M:Fire_Pressed()` in Lua works by **conjuring a UFunction out of thin air**.

### 3.5 Finding the Lua function at runtime

Once the thunk fires, control reaches `FFunctionRegistry::Invoke` (`FunctionRegistry.cpp:22`). On the first call it walks the `Super` chain looking for a Lua function of the same name, then caches the result as a registry ref:

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

If nothing is found it doesn't crash — it **falls back to the original implementation**:

```cpp
// the lua module may have failed to load, so forward to the original function
const auto Overridden = Function->GetOverridden();
if (Overridden && Stack.Code)
    Overridden->Invoke(Context, Stack, RESULT_PARAM);
```

A pragmatic design: if the script is broken, the game degrades to its original Blueprint behaviour instead of exploding.

### 3.6 The cost: `FuncMap` is process-global

An override mutates `UClass::FuncMap`, and there is exactly one `UClass` per process. Which means:

- an override applies to **all instances, all Worlds, all PIE sessions** simultaneously;
- bind once in the editor and the state survives into the next PIE run unless it's restored;
- recompiling a Blueprint wipes `FuncMap` and silently kills the override.

The fix for the last one is refreshingly blunt — stuff a sentinel function into `FuncMap` and check whether it's still there next time (`UnLuaManager.cpp:262`):

```cpp
#if WITH_EDITOR
    // handle the case where a Blueprint recompile cleared FuncMap
    if (Class->FindFunctionByName("__UClassBindSucceeded", EIncludeSuperFlag::Type::ExcludeSuper))
        return true;
    ULuaFunction::RestoreOverrides(Class);
#endif
```

`FLuaOverrides` also exposes `Suspend`/`Resume` and registers a `GUObjectArray` delete listener so overrides are restored automatically when a class is destroyed. All of it is tax on mutating global state.

## 4. Metatables are the cache

`self.Health` looks like one table lookup. It's worth counting the actual steps.

The Lua-side `Index` function (embedded as a C string chunk at `UnLuaLib.cpp:163`):

```lua
local function Index(t, k)
    local mt = getmetatable(t)
    local super = mt
    while super do                              -- 1. walk the Lua module chain first
        local v = rawget(super, k)
        if v ~= nil and not rawequal(v, NotExist) then
            rawset(t, k, v)                     --    found: cache into the instance table
            return v
        end
        super = rawget(super, "Super")
    end

    local p = mt[k]                             -- 2. fall through to CLASS_METATABLE.__index
    if p ~= nil then
        if type(p) == "userdata" then
            return GetUProperty(t, p)           -- 3a. property: must be read every time
        elseif type(p) == "function" then
            rawset(t, k, p)                     -- 3b. function: cache the closure
        elseif rawequal(p, NotExist) then
            return nil
        end
    else
        rawset(mt, k, NotExist)                 -- 4. even "doesn't exist" is cached
    end
    return p
end
```

On the C side, `Class_Index` → `GetField` (`LuaCore.cpp:1061`):

```cpp
lua_getmetatable(L, 1);
lua_pushvalue(L, 2);
int32 Type = lua_rawget(L, -2);
if (Type == LUA_TNIL)
    GetFieldInternal(L);        // only the first access goes through reflection
```

After resolving an `FFieldDesc`, `GetFieldInternal` **writes the result back into the metatable** (`LuaCore.cpp:1030`): properties as a userdata wrapping a `TSharedPtr<ITypeOps>`, functions as a `Class_CallUFunction` closure. Inherited fields get cached in both the super and the derived metatable.

So the real cost breaks down as:

| Access | First time | Afterwards |
|--------|-----------|------------|
| Lua method `self:Foo()` | module-chain rawget | instance-table hit, no metamethod |
| UFunction `self:K2_GetActorLocation()` | module-chain miss → C call → reflection lookup → build `FFunctionDesc` | instance-table hit on the closure, straight into `CallUE` |
| Property `self.Health` | as above | **every time**: N module-chain misses → `Class_Index` (C call) → metatable rawget → returns descriptor → `GetUProperty` (second C call) → actual read |
| Nonexistent field | one reflection lookup | `NotExist` sentinel, pure table hit |

Property access is the one category that can't be cached away — it **must** run every time, and it crosses the C boundary twice. Type checking, on by default (`ENABLE_TYPE_CHECK=1`), adds an `IsA` walk to every property access (`LowLevel.cpp:135`):

```cpp
UClass* OwnerClass = Property->GetOwnerClass();
if (Object->IsA(OwnerClass)) return true;
luaL_error(L, ... "Access property from invalid owner. %s should be a %s.");
```

The practical takeaway is blunt: **don't write `self.A.B.C` on a hot path** — hoist properties and functions into locals. Worth noting too that the bundled benchmark (`Content/Script/Tests/Benchmark/UnLuaBenchmarkProxy.lua`) measures the raw-userdata path after `local RawObject = Proxy.Object`, skipping the Lua `Index` and the module chain. It reports a **lower bound**, not what your gameplay code will see.

## 5. Calls in both directions

### Lua → UE

The skeleton of `FFunctionDesc::CallUE` (`FunctionDesc.cpp:171`):

1. `Buffer->Get()` for the parameter frame;
2. `PreCall`: `InitializeValue` per property, `WriteValue_InContainer` from the Lua stack, missing arguments filled from the UHT-collected defaults;
3. `Object->UObject::ProcessEvent(FinalFunction, Params)` — note the **non-virtual** call, bypassing any subclass override of `ProcessEvent`;
4. `PostCall`: return value and out-params pushed back to Lua, then `DestroyValue`;
5. `Buffer->Pop(Params)`.

Frame allocation is governed by `ENABLE_PERSISTENT_PARAM_BUFFER` (on by default). In persistent mode each UFunction keeps a stack of buffers, with a counter to support recursion (`ParamBufferAllocator.cpp:38` — that counter is exactly what 2.3.5 added to fix the recursive-overwrite bug, #563). Turn it off and every call degrades to `FMemory::Malloc` + `Memzero` + `Free`. **In the default configuration, a warmed-up Lua→UE call performs no heap allocation** — at the price of buffers that only ever grow.

One subtle semantic: if the UFunction Lua is calling **is itself overridden by Lua**, the call is redirected to the original implementation (`FunctionDesc.cpp:213`):

```cpp
#if ENABLE_CALL_OVERRIDDEN_FUNCTION
    const auto LuaFunction = ULuaFunction::Get(Function.Get());
    if (LuaFunction && LuaFunction->GetOverridden())
        FinalFunction = LuaFunction->GetOverridden();
#endif
```

That's why `self.Overridden.ReceiveBeginPlay(self)` reaches the original Blueprint logic: `Overridden` *is* `CLASS_METATABLE`, so you get a reflection closure, and the closure gets redirected inside `CallUE`. It's also why you must use `.` and never `:` — `Content/Script/Tutorials/02_OverrideBlueprintEvents.lua` calls this out explicitly. `self.Overridden:SayHi(name)` would pass the metatable as `self`.

### UE → Lua

`FFunctionDesc::CallLua` (`FunctionDesc.cpp:78`) has a complication: the call may come from Blueprint bytecode, with arguments still sitting in the instruction stream. Hence the `bUnpackParams` branch, which manually `Stack.Step`s each argument into a buffer and rebuilds an `FOutParmRec` chain; otherwise it just uses `Stack.Locals`.

Then `lua_pcall`, then out-params written back in order. There's a refreshingly honest comment here (`FunctionDesc.cpp:466`):

```cpp
// out value
// suppose out param is also pushed on stack? this is assumed done by user... so we can not trust it
```

What if the Lua function returns nothing? With `ENABLE_TYPE_CHECK` on you get an error; with it off, default values are used (changed in 2.3.5). The ordering of return value vs out-params is controlled by `UNLUA_LEGACY_RETURN_ORDER`.

### Default arguments and coroutines

Default values in a C++ signature **do not exist** in reflection data. UnLua built a whole UHT plugin for this, `UnLuaDefaultParamCollector`, which scans every `BlueprintCallable`/`Exec` function at compile time, emits `GDefaultParamCollection`, and fills the gaps in `PreCall`. A build-time plugin so Lua can omit a few trailing arguments — that trade tells you a lot about the project's priorities.

Latent functions (`Delay`, `MoveTo`, …) run on coroutines. When `PreCall` sees a parameter named `LatentInfo`, it synthesizes an `FLatentActionInfo` whose callback target is `UUnLuaManager::OnLatentActionCompleted`, with the coroutine's registry ref as the `LinkID` (`FunctionDesc.cpp:300`):

```cpp
FLatentActionInfo LatentActionInfo(ThreadRef, GetTypeHash(FGuid::NewGuid()), TEXT("OnLatentActionCompleted"), (Env.GetManager()));
```

When the engine's latent action completes it calls back into `Env->ResumeThread(LinkID)` → `lua_resume`. So you can write `UE.UKismetSystemLibrary.Delay(self, 1.0)` and just continue — as long as the code is running inside a coroutine.

## 6. Value semantics: the sharpest edge

This is the part I'd most want to know before starting.

`FPropertyDesc::GetValueInternal` takes a `bCreateCopy` flag. For structs (`PropertyDesc.cpp:1219`):

```cpp
if (bCreateCopy)
{
    void *Userdata = NewUserdataWithPadding(L, StructSize, StructName.Get(), UserdataPadding);
    StructProperty->InitializeValue(Userdata);
    StructProperty->CopySingleValue(Userdata, ValuePtr);     // real copy
}
else
{
    UnLua::PushPointer(L, (void*)ValuePtr, StructName.Get(), bFirstPropOfScriptStruct);  // borrowed pointer
}
```

And when `Class_Index` reads a property it passes **`false`** (`LuaCore.cpp:1218`):

```cpp
(*Property)->ReadValue_InContainer(L, Self, false);
```

So `local t = self.SomeTransform` gives you **a view into the UObject's memory, not a copy**. Mutating `t` mutates the object (plenty of code relies on this for in-place edits) — but the flip side is that once the object is destroyed, or an array reallocates, or a parameter frame is reused, that view is a dangling pointer.

The subtler case is override parameters. From `CallLuaInternal` (`FunctionDesc.cpp:449`):

```cpp
Property->ReadValue_InContainer(L, InParams, !UNLUA_LEGACY_ARGS_PASSING);
```

`UNLUA_LEGACY_ARGS_PASSING` defaults to **1**, so the negation makes `bCreateCopy = false` — **struct arguments handed to an overriding Lua function point into the caller's parameter frame**, and that buffer is recycled after the call returns. Stashing such an argument in a member variable for use next frame is the canonical way to shoot yourself. The "argument passing" setting added in 2.3.3 exists to switch this to copy mode: safe, at the cost of a struct copy per call.

Containers behave the same way: `TArray`'s `Get` returns a copy of an element, `GetRef` returns a reference — the sample code's `InterpFloats:GetRef(1)` uses the latter precisely because it's about to modify the element.

UnLua knows this is risky and ships two mitigations:

**1. Dangling check (off by default).** `FDanglingCheck` (`LuaDanglingCheck.cpp`) wraps boundary calls in a guard; on destruction the guard nulls out every struct userdata pointer lent out during the call and flags container userdata as released:

```cpp
void* Userdata = GetUserdataFast(L, -1, &TwoLevelPtr);
check(TwoLevelPtr)
*(void**)Userdata = nullptr;
```

Cross-frame access then yields a Lua error instead of a random crash. Worth enabling during development.

**2. The `ReleasedPtr` sentinel.** When a UObject is destroyed, `FObjectRegistry::Unbind` (`ObjectRegistry.cpp:200`) doesn't just drop the mapping — it rewrites the pointer stored in the userdata:

```cpp
*((void**)Userdata) = (void*)LowLevel::ReleasedPtr;
```

Any later access from Lua hits the `IsReleasedPtr` check and reports `attempt to read property 'X' on released object`. Far friendlier than dereferencing freed memory.

## 7. Stitching two garbage collectors together

Lua has a GC. Unreal has a GC. Both want to own object lifetime, and the seam between them is where bugs breed. UnLua's answer: **weak tables on the Lua side, explicit root sets on the Unreal side.**

`FLuaEnv`'s constructor creates a batch of weak-valued tables (`LuaEnv.cpp:121`, `ObjectRegistry.cpp:52`): `UnLua_ObjectMap`, `StructMap`, `ArrayMap`, `ScriptContainerMap`, `UnLua_ManualRefProxyMap`. They're purely "same C++ address → same Lua object" caches and never prevent collection.

On the Unreal side there are two `FObjectReferencer`s (`LuaEnv.cpp:118`):

- `AutoObjectReference` — rooted automatically while Lua holds the object, removed via `NotifyUObjectLuaGC` when Lua collects the userdata;
- `ManualObjectReference` — driven by `UnLua.Ref` / `UnLua.Unref`, with an `FManualRefProxy` carrying a `__gc` as a backstop.

Bound objects get one more layer: their `INSTANCE` table is strongly referenced via `luaL_ref` in the registry (`ObjectRegistry.cpp:155`) until `Unbind`. So "can the Lua table outlive the UObject?" — yes, but only until `Unbind`, and `Unbind` is driven by `NotifyUObjectDeleted`.

`FLuaEnv::NotifyUObjectDeleted` (`LuaEnv.cpp:265`) notifies every registry in a fixed order, and that order is itself scar tissue:

```cpp
PropertyRegistry->NotifyUObjectDeleted(Object);
FunctionRegistry->NotifyUObjectDeleted(Object);
if (Manager) Manager->NotifyUObjectDeleted(Object);
ObjectRegistry->NotifyUObjectDeleted(Object);
ClassRegistry->NotifyUObjectDeleted(Object);
EnumRegistry->NotifyUObjectDeleted(Object);
```

The Lua GC configuration is worth a look too (`LuaEnv.cpp:135`):

```cpp
#if 504 == LUA_VERSION_NUM
    lua_gc(L, LUA_GCGEN, 0, 0);          // 5.4: generational GC by default
#else
    lua_gc(L, LUA_GCSETPAUSE, 100);      // 5.3: aggressive incremental settings
    lua_gc(L, LUA_GCSETSTEPMUL, 5000);
#endif
```

And the `lua_State` uses Unreal's allocator (`FLuaEnv::DefaultLuaAllocator`), so the Lua heap shows up in Unreal's memory stats — which matters a lot when profiling mobile memory.

Incidentally, the CHANGELOG shows "intermittent crash during Lua GC" being chased from the 2.2.0 GC rework all the way to April 2026 — the newest commit on `develop` is literally "fix intermittent crash when a UObject is GC'd on the Lua side." The seam between the two collectors has been this plugin's long-running sore spot.

Rules of thumb:

- If Lua needs to hold an object long-term, don't rely on a Lua variable alone — either the object already has an Unreal-side reference, or call `UnLua.Ref`;
- treat struct and container references as **locals you discard immediately**; never keep them across frames;
- enable the dangling check during development so problems surface where they're caused, not where they crash;
- check `IsValid` before touching an object that may have been destroyed.

## 8. Performance: what the code lets you derive

The headline: **a reflected Lua→UE call is in the same cost bracket as a Blueprint node calling a C++ function**, because both end up in `ProcessEvent`. UnLua's real win is **where the logic itself runs** — dense branching, loops and table work are clearly faster in Lua than in the Blueprint VM, while call-heavy boundary code isn't cheap on either side.

Fixed costs per operation:

| Operation | Paid every time |
|-----------|-----------------|
| Property read | N module-chain rawgets → C call `Class_Index` → metatable rawget → `GetCppInstance` → `IsA` (with type checking on) → second C call `GetUProperty` → `ReadValue` |
| Property write | same, ending in `WriteValue` |
| UFunction call | cached closure hit → `PreCall` (per-argument `InitializeValue` + write) → `GetFunctionCallspace` → `ProcessEvent` → `PostCall` + `DestroyValue` |
| Overridden function invoked | thunk → `IUnLuaModule::GetEnv` → registry lookup for the `ULuaFunction` → possible `Stack.Step` unpacking → push arguments → `lua_pcall` → write back out-params |
| Statically exported call | plain C function: no `FFunctionDesc`, no parameter frame, no `ProcessEvent` |
| `FString` / `FName` crossing | encoding conversion and allocation every time (`TCHAR_TO_UTF8` / `FString` construction) |
| Math type arithmetic | `FVector` and friends are hand-exported, but each operation allocates a fresh userdata for the result |

The knobs (`UnLua.Build.cs:91`):

| Macro | Default | Effect |
|-------|---------|--------|
| `ENABLE_TYPE_CHECK` | 1 | Off removes the per-property-access `IsA` and per-argument type validation; type errors become undefined behaviour |
| `ENABLE_PERSISTENT_PARAM_BUFFER` | 1 | Reuses parameter frames, eliminating per-call malloc/free |
| `UNLUA_LEGACY_ARGS_PASSING` | 1 | 1 = struct arguments borrow pointers (fast, dangerous); 0 = copies (slower, safe) |
| `ENABLE_CALL_OVERRIDDEN_FUNCTION` | 1 | Provides `self.Overridden`, at one extra `ULuaFunction::Get` per `CallUE` |
| `AUTO_UNLUA_STARTUP` | 1 | Creates the `lua_State` at engine startup |
| `UNLUA_ENABLE_DEBUG` | 0 | Heavy logging; diagnostics only |
| `ENABLE_UNREAL_INSIGHTS` | 0 | `lua_sethook` pipes Lua calls into Insights; mutually exclusive with the dead-loop check |

There are only a handful of optimizations the code actually supports, but they're all real: hoist UFunctions and hot properties into locals; take over genuine hotspots with static exports; keep per-frame Tick logic in C++/Blueprint and enter Lua only on events; prefer in-place math over chains that allocate temporaries.

On memory, keep three ledgers in mind: one Lua table per bound object (which grows as method closures get `rawset` into it), one metatable per `UStruct` (which doubles as the field-resolution cache), and one monotonically growing parameter-buffer pool per UFunction that Lua has ever called.

## 9. Hot update and tooling

The hot-update capability actually lives in the module loader. `FLuaEnv` inserts three searchers into `package.searchers` (`LuaEnv.cpp:98`), and the filesystem one searches in this order (`LuaEnv.cpp:622`):

```cpp
// prefer a standalone file in the download directory
FPaths::Combine(FPaths::ProjectPersistentDownloadDir(), Pattern)
// then the packaged directory
FPaths::Combine(FPaths::ProjectDir(), Pattern)
```

**The download directory wins over the packaged one** — that is the whole patch pipeline: drop new scripts into `PersistentDownloadDir` and `require` picks them up. The default search path is `Content/Script/?.lua;Plugins/UnLua/Content/Script/?.lua`; changing `package.path` does nothing (Unreal has its own filesystem), so you change `UnLua.PackagePath`. For encryption or a custom package format, register a loader via `FLuaEnv::AddLoader`.

`UnLua.HotReload()` during development is a different thing. `HotReload.lua` states outright that it follows 云风 (Cloud Wu)'s well-known approach, preserving upvalues and live tables where it can. But combine two earlier facts — the module table was **copied** at bind time, and Lua methods get `rawset` into **instance tables** — and its limits follow: new logic always applies to new instances, and applies to existing ones only insofar as the caches were correctly replaced. The file's own framing is honest: designed for development, replace what it can, worst case you restart.

The rest of the engineering surface:

- **IntelliSense** — `UnLuaEditor` walks every `UClass`/`UStruct`/`UEnum` and emits EmmyLua annotation stubs, so `---@type BP_PlayerCharacter_C` gives you completion; a commandlet exists for CI. The downside is that stubs must be regenerated when types change.
- **Packaging** — Lua files aren't assets, so the Script directory must be listed under "Additional Non-Asset Directories to Package." That's FAQ item #2 for a reason.
- **Static export** — `UNLUA_EXPORT_CLASS` / `ADD_FUNCTION` / `ADD_PROPERTY` / `EXPORT_FUNCTION`, registered during static initialization and merged into the same metatable as reflected data. The plugin's own `FVector` and `TArray` are exported this way: **its own hotspots don't go through reflection either**, which is telling.
- **Tests** — `UnLuaTestSuite` organizes regression cases by GitHub issue number (`Content/Script/Tests/Regression/Issue###/`), dozens of them, including cases like Chinese Blueprint names (Issue322). Quite disciplined for an open-source plugin.

## 10. Status in 2026: not abandoned, relocated

This is easy to get wrong, so precisely (figures from the GitHub API, August 2026):

- The last tag is **v2.3.6, 2023-11-07** — no release in nearly three years;
- `master` has had **1 commit** since 2024 (a LICENSE entity change on 2025-07-07);
- but **`develop` is alive**: UE5.4 support landed 2024-05 (PR #700), UE5.6 compile fixes 2025-09 through 2025-12 (PR #758), and the newest commit is **2026-04-02**, "fix intermittent crash when a UObject is GC'd on the Lua side" (PR #765);
- those commits come mainly from community contributors (jozhn, crazytuzi, and others) rather than dedicated corporate staffing;
- UE5.7 is **not supported**; the issues are still open;
- 2,755 stars / 718 forks / 185 open issues+PRs.

The accurate summary is: **the official release line is frozen at UE5.3; the usable newer code lives on `develop` and is community-maintained.** If you're targeting UE5.4+ today, that means pulling from `develop` and accepting no tags, no release notes, and issue response at volunteer pace.

The neighbours over the same period:

| Project | Status (2026-08) | Binding approach |
|---------|------------------|------------------|
| **UnLua** | 2,755★, releases frozen at 2023-11, develop active to 2026-04 | Runtime reflection |
| **sluaunreal** (Tencent) | 1,974★, last push 2025-10, tag 2.1.4 (2024-06) | Reflection + codegen |
| **puerts** (Tencent, TS/JS) | 6,161★, last push 2026-07-31, the most active of the three | Codegen + reflection |
| **UnrealEnginePython** | Last push 2022-06, effectively abandoned | Reflection |
| **Angelscript** (Hazelight) | Not open-sourced on GitHub; distributed via angelscript.hazelight.se | Compile-time static binding |

On adoption, the official (Chinese) FAQ has a line worth quoting, translated:

> Around forty projects inside Tencent are known to use UnLua; external projects can't be counted.

And the perennial question: **can you swap in LuaJIT?** There are `feature/luajit` and `feature/lua51` branches (both stuck at a February 2023 "add LuaJIT build script" commit), and 2.3.5 added a "custom Lua version" setting. But the code imposes a hard constraint: `LuaCore.cpp` directly `#include`s `lstate.h` / `lobject.h`, reimplements `lua_index2value` itself, and appends a magic struct to the tail of every userdata for tagging:

```cpp
#define USERDATA_MAGIC  0x1688
#define BIT_TWOLEVEL_PTR        (1 << 5)
struct FUserdataDesc { uint16 magic; uint8 tag; uint8 padding; };
```

All of that depends on Lua 5.4's `TValue`/`Udata` memory layout. Switching to LuaJIT is not a matter of flipping a macro — that layer has to be rebuilt. **Evaluate it as a plugin welded to Lua 5.4.**

## When to reach for it

Good fit: projects already organizing gameplay in Blueprint, needing mobile hot-updates, with existing Lua expertise on the team, and able to live with type errors that only surface at runtime. Its greatest strength is **marginal cost** — adding a `UPROPERTY` requires touching no binding code at all, and that dividend keeps paying out over a long, iteration-heavy project life.

Poor fit: chasing peak boundary-call performance (static binding, or just write C++); needing strong typing and large-scale refactoring support (consider puerts' TypeScript route, or Angelscript); tracking the newest engine versions (budget for the `develop`-branch maintenance); or a team where nobody wants to read the override machinery when something breaks.

That last one is the real filter. UnLua's code is good, but what it does — hiding pointers inside bytecode, hanging shadow classes off `UClass`, mutating a process-global `FuncMap`, lending raw pointers across two garbage collectors — is inherently not a black box. **Using it means someone has to be able to read it.** I hope this article makes that someone a little easier to find.
