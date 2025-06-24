To obtain the Lua state from a Roblox instance, you'll need two key functions:

1: `GetGlobalStateForInstance`: 
This function retrieves an encrypted Lua state pointer from a script context.
   -  **First**:
     The address of the script context plus the `GlobalState` offset (required).
   -  **Second**: 
    A pointer to store the identity 
   -  **Third**:
     A pointer to store the script reference

2: `DecryptLuaState`
This function decrypts the encrypted Lua state pointer obtained from `GetGlobalStateForInstance`. The only argument requires is the encrypted state pointer plus the `DecryptState` offset.

A example usage would look like this:
```cpp
namespace RBX { 
	using GetGlobalStateForInstance_t = uintptr_t(__fastcall*)(uintptr_t, uintptr_t*, uintptr_t*); 
	const GetGlobalStateForInstance_t GetGlobalStateForInstance = (GetGlobalStateForInstance_t)REBASE(0xDC7E20); 
	
	using DecryptLuaState_t = lua_State*(__fastcall*)(uintptr_t); 
	const DecryptLuaState_t DecryptLuaState = (DecryptLuaState_t)REBASE(0xB4D320);
}

uintptr_t identity = 0; 
uintptr_t script = 0;

auto encrypted_state = RBX::GetGlobalStateForInstance(
	script_context + offsets::GlobalState, 
	&identity, 
	&script
);

auto state = RBX::DecryptLuaState(encrytped_state + offsets::DecryptState);
```


Help & Source credits: [felixtaken](https://discord.com/users/964160170273939516)