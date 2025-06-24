Writing strings compared to reading strings is actually really easy. 

So what do we know about roblox strings in general?
-  First we know that when add 0x10 to the address of the string it always returns the length
-  If the length if over 16 the address contains another address where "string" is located in

So now we can apply these by first turning our given in string to bytes and retrieving the length of the bytes. Because then we write to the given address + 0x10 the length. This should look something like this:

```rust
let bytes = value.as_bytes(); 
let length = bytes.len(); 
PROCESS.write_memory(address + 0x10, &(length as usize)).unwrap();
```

And since we also know that if the length is over (or equal) to 16 the address contains another address that contains the string. So what we do know is check for the length of the bytes and if the length is over or equal to 16 we allocate a piece of memory with the specific length and write the specific bytes to that address. Once we have the allocated address containing the bytes we can write that address in the address so that there is a new string in there. If the length is under 16 we just write the bytes directly.

```rust
if length >= 16 {
    let allocated_address = PROCESS.allocate(length).unwrap();
    PROCESS.write_bytes(allocated_address, bytes).unwrap();
    PROCESS.write_memory(address, &allocated_address).unwrap();
} else {
    PROCESS.write_bytes(address, bytes).unwrap();
}
```

So our full `write_string` function should look something like this:

```rust
pub fn write_string(address: usize, value: &str) {
    let bytes = value.as_bytes();
    let length = bytes.len();
    
    PROCESS.write_memory(address + 0x10, &(length as usize)).unwrap();
    
    if length >= 16 {
       let allocated_address = PROCESS.allocate(length).unwrap();
       PROCESS.write_bytes(allocated_address, bytes).unwrap();
       PROCESS.write_memory(address, &allocated_address).unwrap();
    } else {
      PROCESS.write_bytes(address, bytes).unwrap();
    }
}
```
