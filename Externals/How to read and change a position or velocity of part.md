So what do we know about the position of a part? 
The position of a part in Roblox is a `Vector3` which is just a Vector of a 3-dimensional space which holds the `x`, `y` and `z` axis as float.

So how would we create a `Vector3` struct in something like rust?. Well it's simple it's just this

```rust
#[derive(Copy, Clone, Debug)]
#[allow(non_snake_case)]
pub struct Vector3 {  
    pub x: f32,  
    pub y: f32,  
    pub z: f32,  
}
```

`#[derive(Copy, Clone, Debug)]` is only included because in my memory lib ([`memory_utils`](https://crates.io/crates/memory_utils)) it requires the `Copy` trait.

But now that we know what the struct looks like we need to know some more stuff. 
The parts address does NOT represent the actual part so directly adding the position / velocity offset to the part won't work. You first have to make it a primitive by using the primitive offset and adding it to the parts address.

Then when you have the primitive you can actually start adding the position / velocity offset to retrieve the primitive to read the actual position / velocity. This should look something like this:

```rust
let part = rbx::FindFirstChild(self.character_addr, part.to_string());  
if part == 0 { return }  
  
let primitive = process.read_memory::<usize>(part + offsets.Primitive).unwrap()  
let position = process.read_memory::<Vector3>(primitive + offsets.Position).unwrap();
```

So our full position reading should just look something like this:

```rust
fn get_position(&self, part: &str) -> Vector3 {  
    if self.character == 0 { return Vector3::zero() }  
    let part = rbx::FindFirstChild(self.character, part.to_string());  
    if part == 0 { return Vector3::zero() }  
  
    let primitive = process.read_memory::<usize>(part + offsets.Primitive).unwrap();  
    process.read_memory::<Vector3>(primitive + offsets.Position).unwrap()  
}
```


---

Changing the position / velocity is also really simple since it's just instead of reading from the address it's just writing.

```rust
fn set_position(&self, part: &str, position: Vector3) {  
    if self.character == 0 { return }  
    let part = rbx::FindFirstChild(self.character, part.to_string());  
    if part == 0 { return }  
  
	let primitive = process.read_memory::<usize>(part + offsets.Primitive).unwrap();  
    process.write_memory::<Vector3>(primitive + offsets.Position, &position).unwrap()  
}
```

