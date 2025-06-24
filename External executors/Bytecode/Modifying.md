Modifying the bytecode of a `LocalScript` or `ModuleScript` is now really easy. First we need to turn the code that is inputted into the function to bytes and after that we need to compile the code using `luau_compile` ( I will be using [`mlua`](https://github.com/mlua-rs/mlua) for this).

```rust
let byte_code = code.as_bytes();  
let compiled = compile(byte_code, lua_CompileOptions::default())?;
```

After that we need to retrieve the embed offset of the specified script which is done by checking the class and returning a different offset for the given in class. After that we read the embed pointer which will just be `address + embed_offset`.

```rust
let embed_offset = match class(address).as_str() {  
    "ModuleScript" => MODULE_SCRIPT_EMBEDDED,  
    "LocalScript" => LOCAL_SCRIPT_EMBEDDED,  
    other => panic!("Unkown class: {}", other)  
};  
  
let embed_ptr = PROCESS.read_memory::<usize>(address + embed_offset).unwrap_or(0);
```

Now we will use [`VirtualAllocEx`](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocex) to allocate a address in the Roblox process with these allocation types: `MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE`

Then we check if the allocated address is null. If yes then the function will return the error `Failed to allocate memory`

```rust
let mut old_protection: DWORD = 0;  
let allocated_address = VirtualAllocEx(PROCESS.handle, null_mut(), compiled.len(), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);  
  
if allocated_address.is_null() {  
    return Err("Failed to allocate memory".to_string());  
}
```

In that allocated address we write the compiled data as bytes so that our code is now in the memory. Once we wrote that address we will use [`VirtualProtectEx`](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-VirtualProtectEx) to protect that region of memory with the `PAGE_EXECUTE_READ` permissions.

```rust
let write_status = PROCESS.write_bytes(allocated_address as usize, &compiled);  
if write_status.is_err() {  
    return Err("Failed to write to memory".to_string());  
}  
  
let success = VirtualProtectEx(PROCESS.handle, allocated_address, compiled.len(), PAGE_EXECUTE_READ, &mut old_protection) != 0;  
  
if !success {  
    return Err("Failed to protect memory".to_string());  
}
```

And now that we have the address in the roblox process with our memory in it we can write the allocated addressed into the `embed_ptr + BYTECODE` and write the bytecode size to `embed_ptr + BYTECODE_SIZE`. 

But what if we want to revert to the old bytecode?. For that we will just read the bytecode address and the bytecode size and then after waiting some time we will write to these address the saved / old bytecode and size.

```rust
if old_bytecode {  
    let original_bytecode_ptr = PROCESS.read_memory::<usize>(embed_ptr + BYTECODE).unwrap();  
    let original_bytecode_size = PROCESS.read_memory::<usize>(embed_ptr + BYTECODE_SIZE).unwrap();  
  
    std::thread::spawn(move || {  
        std::thread::sleep(std::time::Duration::from_millis(850));  
  
        PROCESS.write_memory::<usize>(embed_ptr + BYTECODE, &original_bytecode_ptr).unwrap();  
        PROCESS.write_memory::<usize>(embed_ptr + BYTECODE_SIZE, &original_bytecode_size).unwrap();  
    });  
}
```

So this is how the full code should look like:

```rust
pub fn set_bytecode(address: usize, code: &str, old_bytecode: bool) -> Result<bool, String>  {  
    let byte_code = code.as_bytes();  
    let compiled = compile(byte_code, lua_CompileOptions::default())?;  
  
    let embed_offset = match class(address).as_str() {  
        "ModuleScript" => MODULE_SCRIPT_EMBEDDED,  
        "LocalScript" => LOCAL_SCRIPT_EMBEDDED,  
        other => panic!("Unkown class: {}", other)  
    };  
  
    let embed_ptr = PROCESS.read_memory::<usize>(address + embed_offset).unwrap_or(0);  
  
    if old_bytecode {  
        let original_bytecode_ptr = PROCESS.read_memory::<usize>(embed_ptr + BYTECODE).unwrap();  
        let original_bytecode_size = PROCESS.read_memory::<usize>(embed_ptr + BYTECODE_SIZE).unwrap();  
  
        std::thread::spawn(move || {  
            std::thread::sleep(std::time::Duration::from_millis(850));  
  
            PROCESS.write_memory::<usize>(embed_ptr + BYTECODE, &original_bytecode_ptr).unwrap();  
            PROCESS.write_memory::<usize>(embed_ptr + BYTECODE_SIZE, &original_bytecode_size).unwrap();  
        });  
    }  
  
  
    unsafe {  
        let mut old_protection: DWORD = 0;  
        let allocated_address = VirtualAllocEx(PROCESS.handle, null_mut(), compiled.len(), MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);  
  
        if allocated_address.is_null() {  
            return Err("Failed to allocate memory".to_string());  
        }  
  
        let write_status = PROCESS.write_bytes(allocated_address as usize, &compiled);  
        if write_status.is_err() {  
            return Err("Failed to write to memory".to_string());  
        }  
  
        let success = VirtualProtectEx(PROCESS.handle, allocated_address, compiled.len(), PAGE_EXECUTE_READ, &mut old_protection) != 0;  
  
        if !success {  
            return Err("Failed to protect memory".to_string());  
        }  
  
        PROCESS.write_memory::<usize>(embed_ptr + BYTECODE, &(allocated_address as usize))  
            .map_err(|e| format!("Failed to write bytecode pointer: {}", e))?;  
  
        PROCESS.write_memory::<usize>(embed_ptr + BYTECODE_SIZE, &compiled.len())  
            .map_err(|e| format!("Failed to write bytecode size: {}", e))?;  
  
        Ok(true)  
    }  
}
```

