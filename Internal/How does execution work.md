The execution process begins by compiling the provided code into bytecode. A new `Luau thread` (state) is created and sandboxed to make standard libraries read-only. The thread's identity is set to 8, and its capabilities are assigned to match the specified capabilities.

Next, the `luau::vm_load` function is called to load the bytecode into the new thread. The chunk is given a name (which can be arbitrary), and the environment index is set to 0, indicating the default global environment. If the loading process fails, the function returns false. Otherwise, the closure is retrieved from the stack, its prototype capabilities are set, and the script's execution is scheduled using task_defer on the thread.

--- 
What do these functions look like?

`vm_load`:

```cpp
using luavm_load_t = int32_t(__fastcall*)(
	lua_State* state, 
	std::string* bytecode, 
	const char* chunk_name, 
	int32_t env
); 
auto vm_load = (luavm_load_t)REBASE(0xCCCFB0);
```

`task_defer`:

```cpp
using task_defer_t = int32_t(__cdecl*)(lua_State *state);
auto task_defer = (task_defer_t)REBASE(0x1172FB0);
```
--- 

What does a example of execution look like?

```cpp
bool execution::execute(lua_State* state, const std::string& code) {
    std::string bytecode = this->compile(code);

    if (bytecode.at(0) == 0) {
        const char* error = bytecode.c_str() + 1;
        return false;
    }
    lua_State* L = lua_newthread(state);
    luaL_sandboxthread(L);
    lua_pop(state, 1);

    L->userdata->identity = 8;
    L->userdata->capabilities = capabilities;

    if (vm_load(L, &bytecode, "", 0) != LUA_OK) {
        lua_pop(L, 1);
        return false;
    }

    const Closure* cl = (Closure*)lua_topointer(L, -1);

    set_capabilities(cl->l.p, &capabilities);

    task_defer(L);
    return true;
}
```

--- 

Credits:
   -  `vm_load` & `task_defer` type: [Chocosploit](https://github.com/Pixeluted/Chocosploit/blob/f2a515f9b6fdc12e0b3676f4e0289e6d92e723b1/OffsetsAndFunctions.hpp#L220)
   -  `execute`: [Visual](https://discord.gg/9ZbdreSP8r)
