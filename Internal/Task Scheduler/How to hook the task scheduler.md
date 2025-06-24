
So how do we initialize a task scheduler hook?

First we need to get the task scheduler by rebasing the `RawScheduler` which we will be calling `TaskSchedulerSingleton`. Once we have rebased that we need to look trough all of the jobs which we get through adding the `JobStart` offset the `TaskScheduler` and as well adding the `JobEnd` offset to the `TaskScheduler`. 

Which what we now do is loop trough all of these jobs and search for `WaitingHybridScriptsJob` which we will call `wshj`. That would look like this:


```cpp
uintptr_t base = (uintptr_t)GetModuleHandleA(0);
    uintptr_t taskscheduler = reinterpret_cast<uintptr_t>(base + rbx::TaskSchedulerSingleton);

    uintptr_t wshj = 0;
    uintptr_t jobstart = *(uintptr_t*)(taskscheduler + rbx:job_start);
    uintptr_t jobend = *(uintptr_t*)(taskscheduler + rbx:job_end);

    while (jobstart != jobend) {
        std::string job_name = *(std::string*)(*(uintptr_t*)(jobstart) + rbx:job_name);

        if (*(std::string*)(*(uintptr_t*)(jobstart) + rbx:job_name) == "WaitingHybridScriptsJob") {

            uintptr_t job = *(uintptr_t*)jobstart;
            uintptr_t script_context_ptr = *(uintptr_t*)(job + rbx::script_context);
            uintptr_t datamodel = *(uintptr_t*)(script_context_ptr + rbx::parent);
            uintptr_t datamodel_name = *(uintptr_t*)(datamodel + rbx::name);
            std::string datamodel_name_t = *(std::string*)(datamodel_name);

            if (datamodel_name_t == "Ugc") {
                wshj = *reinterpret_cast<uintptr_t*>(jobstart);
                break;
            }
        }
        jobstart += 0x10;
    }
```


Now that we retrieved the  `WaitingHybridScriptsJob` we can start the actual hooking.  We will be using `vtable hooking` where we first create a new vtable, but first we define the `HOOK_INDEX` and the `VFTABLE_SIZE`. `HOOK_INDEX` is just 6 and `VFTABLE_SIZE` is gonna be 55. We'll also be needing to define a `vt_function` type for the `original_step` so that we can return the old result. 

```cpp
#define HOOK_INDEX 6
#define VFTABLE_SIZE 55

using vt_function_t = void*(__fastcall*)(void*, void*, void*);  
vt_function_t original_step = nullptr;
```

Once we have all of that done we can start the actual hooking by first creating a new `vtable` with the `VFTABLE_SIZE` + 1 and in there we'll copy the original `vtable` of the `WaitingHybridScriptsJob` and we'll also get the original step and we'll replace it with our and then changing the `vtable` of the `WaitingHybridScriptsJob`: 

```cpp
void** vtable = new void*[VFTABLE_SIZE + 1]; 
memcpy(vtable, *(void**) whsj, sizeof(uintptr_t) * VFTABLE_SIZE); 
original_step = (vt_function_t)vtable[HOOK_INDEX]; 
vtable[6] = scheduler_hook; 
*(void**)whsj = vtable;
```

The `sechduler_hook` function could look something like this

```cpp
void* scheduler_hook(void *a1, void *a2, void *a3) {
    hook(); // calling our actual hook
    return original_step(a1, a2, a3); // returning old result
}
```

Now we just need to define a `hook` function is called every step. Now what should this function have? It should have general `execution_requests` for direct execution and `yielding_requests` to defer callbacks.

A example could look like this:

```cpp
int hook() { 
	if (!execution_requests.empty()) {  
		const std::string top = execution_requests.front(); 
		execution_requests.pop(); 
		rbx::execute(rbx::current_state, top); 
	} 
	if (!yielding_requests.empty()) { 
		std::function yielding_request = yielding_requests.front();
		yielding_requests.pop(); 
		yielding_request(); 
	} 
	return 0; 
}
```

--- 
**Note**:
This may be needed to get it fully working:

```cpp
*(BYTE*)(*(uintptr_t*)(whsj + rbx::ScriptContext) + rbx::RequireBypass) = 1;
```
---

Source & Help Credits: 
   -  Retrieving `WaitingHybridScriptsJob`, `original_step`, `hook`: [Visual](https://discord.gg/9ZbdreSP8r)
   -  Hooking the `vtable`: [felixtaken](https://discord.com/users/964160170273939516)