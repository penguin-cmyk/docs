Getting the real Datamodel is not the hardest task it's honestly really simple.
You just need one pointer and one offset. The pointer which is required is the `FakeDataModelPointer` which just points to the fake datamodel. 

First you need to rebase it tho by adding the `base address` of the Roblox Process to the pointer so you get the actual address. A example would look like this:

```rust
pub static BASE_ADDRESS: Lazy<usize> = Lazy::new(|| PROCESS.get_base_address().unwrap() as usize );  

fn rebase(address: usize) -> usize { BASE_ADDRESS.clone() + address }  
  
pub static FAKE_DATAMODEL_POINTER: Lazy<usize> = Lazy::new(|| rebase(0x6726238));
```

After we're done with that we can use the Pointer to retrieve the fake Datamodel by just reading it in memory:

```rust
let fake_dm = PROCESS.read_memory::<usize>(*FAKE_DATAMODEL_POINTER).unwrap_or(0);
```

And now we just need to use the `FakeDataModelToRealDataModel` offset which we will add to the fake Datamodel which we just read:

```rust
let real_dm = PROCESS.read_memory::<usize>( fake_dm + FAKE_DATAMODEL_TO_DATAMODEL ).unwrap_or(0);
```

