Reading strings from the roblox process is fairly simple. So what do we need?

We need the address of the string of course and the length of the string (which is just address + 0x10)

## Basic string reader 
---------------------------------
So first we create a character pointer which points to the current character in that memory region. Then we define the `offset` which will be increased by 1 every time in the loop and is added to the address. After that we we will define our output which we'll use (I'll be using a vector). 

A example would look like this:

```rust
let mut character_ptr = 0;  
let mut offset = 0;  
  
let mut result = Vec::new();
```

Now that we have the basics defined we can start with the actual logic by having a `while loop` which checks if the length is bigger than the offset and if yes it will break. In that loop we'll redefine the character pointer by reading the address + the offset as a u8. If the address is 0 (aka invalid) we'll break out of the loop. 

A example would look like this:

```rust
while offset < length {  
    character_ptr = PROCESS.read_memory::<u8>(address + offset).unwrap_or(0);  
    if character_ptr == 0 { break }
}    
```

Ok now that we got all the basics done we can finally start actually reading the string where I will be using the `String::from_utf8_lossy` function in rust and also increasing the the offset by one. So that is the full loop which would look like this:

```rust
while offset < length {  
    character_ptr = PROCESS.read_memory::<u8>(address + offset).unwrap_or(0);  
    if character_ptr == 0 { break }  
  
    offset += 1;  
    result.push(String::from_utf8_lossy(&[character_ptr]).to_string());  
};
```

Then at the end we will just join all the results into one string by doing `result.join("")`. Now we have a complete read string basic function which we can use. It would look like this:

```rust
fn string(address: usize, length: usize) -> String {  
    let mut character_ptr = 0;  
    let mut offset = 0;  
  
    let mut result = Vec::new();  
  
    while offset < length {  
        character_ptr = PROCESS.read_memory::<u8>(address + offset).unwrap_or(0);  
        if character_ptr == 0 { break }  
  
        offset += 1;  
        result.push(String::from_utf8_lossy(&[character_ptr]).to_string());  
    };  
  
    result.join("")  
}
```


## Using the string reader
----------------------------------
But this is just our base function since we'll now concentrate on Roblox strings specifically. As I said earlier we need to get the length of the string which is just address  + 0x10 and should look like this:

```rust
let length = PROCESS.read_memory::<usize>(address + 0x10).unwrap_or(0);
```

And what we'll also be adding to the function is a class option where if you know that it's a class we add 0x8 to the address so that we retrieve the actual class string and not something else. It would look like this:

```rust
if class { address += 0x8 }
```

Now that we have that out of the way we can now actually use the length where we check if the length is over or equal to 16 and if yes we read the address again. If not we'll just directly pass in the address to the `string` function.

So a complete string reader would look somewhat like this:

```rust
pub fn read_string(mut address: usize, class: bool) -> String {  
    let length = PROCESS.read_memory::<usize>(address + 0x10).unwrap_or(0);  
  
    if class { address += 0x8 }  
  
    if length >= 16 { string(PROCESS.read_memory::<usize>(address).unwrap_or(0), length) }  
    else { string(address, length) }  
}
```
