Now retrieving the bytecode is way simpler than modifying the bytecode. But again we need to get the `embed_offset`, `embed_ptr`, `bytecode_ptr` and `bytecode_size`:

```rust
let embed_offset = match class(address).as_str() {  
    "ModuleScript" => MODULE_SCRIPT_EMBEDDED,  
    "LocalScript" => LOCAL_SCRIPT_EMBEDDED,  
    other => panic!("Unkown class: {}", other)  
};  
  
let embed_ptr = PROCESS.read_memory::<usize>(address + embed_offset).unwrap_or(0);  
let bytecode_ptr = PROCESS.read_memory::<usize>(embed_ptr + BYTECODE).unwrap();  
let bytecode_size = PROCESS.read_memory::<usize>(embed_ptr + BYTECODE_SIZE).unwrap();
```

Now we create a buffer  or the bytecode. After we defined that we will use [`ReadProcessMemory`](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-ReadProcessMemory) to retrieve the information and read it into the `bytecode_buffer`.

```rust
let mut bytecode_buffer = vec![0; bytecode_size];  
unsafe {  
    ReadProcessMemory(  
        PROCESS.handle,  
        bytecode_ptr as LPCVOID,  
        bytecode_buffer.as_mut_ptr() as LPVOID,  
        bytecode_size,  
        null_mut()  
    );  
}
```

Once the `bytecode_buffer` contains the bytecode we can just decompress the `bytecode_buffer` and return the read decompressed bytes. So that the user can decide what to do with it themself instead of directly turning it into a string. So the full code should look like this:

```rust
pub fn bytecode(address: usize) -> Result<Vec<u8>, String> {  
    let embed_offset = match class(address).as_str() {  
        "ModuleScript" => MODULE_SCRIPT_EMBEDDED,  
        "LocalScript" => LOCAL_SCRIPT_EMBEDDED,  
        other => panic!("Unkown class: {}", other)  
    };  
  
    let embed_ptr = PROCESS.read_memory::<usize>(address + embed_offset).unwrap_or(0);  
    let bytecode_ptr = PROCESS.read_memory::<usize>(embed_ptr + BYTECODE).unwrap();  
    let bytecode_size = PROCESS.read_memory::<usize>(embed_ptr + BYTECODE_SIZE).unwrap();  
  
    let mut bytecode_buffer = vec![0; bytecode_size];  
    unsafe {  
        ReadProcessMemory(  
            PROCESS.handle,  
            bytecode_ptr as LPCVOID,  
            bytecode_buffer.as_mut_ptr() as LPVOID,  
            bytecode_size,  
            null_mut()  
        );  
    }  
  
    match decompress(&bytecode_buffer) {  
        Ok(bytes) => Ok(bytes),  
        Err(e) => Err(format!("Failed to decompress bytecode: {}", e))  
    }  
}
```

