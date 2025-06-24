
To start compressing we need to define three constants that will be used later:

```rust
const BYTECODE_SIGNATURE: [u8; 4] = *b"RSB1";
const BYTECODE_HASH_MULTIPLIER: u8 = 41;
const BYTECODE_HASH_SEED: u32 = 42;
```

-  `BYTECODE_SIGNATURE`: This is a 4-byte prefix (`RSB1`) that marks the buffer as Roblox  
   compressed bytecode. It's always the the first 4 bytes of the resulting buffer.
-  `BYTECODE_HASH_SEED`: Is used to compute an XXH32 hash of the final buffer.
-  `BYTECODE_HASH_MULTIPLIER`: Is a multiplier applied per-byte in the XOR loop. And used to  
   generate the key.

So now that we have our constants defined, we prepare a buffer to hold the compressed data.
It should look like this:

```rust
let data_size = bytecode.len();
let max_size = zstd::zstd_safe::compress_bound(data_size);
let mut buffer = vec![0u8; max_size + 8];
```

-  `data_size` is the number of bytes in the original uncompressed bytecode.
-  `max_size` is the worst-case size of the compressed data using the ZSTD algorithm (i.e., how  
   much space we need)
   
After we defined both of these we will create a buffer with size of `max_size + 8` which is because we need to make room for the 4-byte signature and the 4-byte length header.

Next we write the Roblox signature and the original bytecode size to the beginning of the buffer:

```rust
buffer[..4].copy_from_slice(&BYTECODE_SIGNATURE);
buffer[4..8].copy_from_slice(&(data_size as u32).to_le_bytes());
```

- Bytes `0..4` = `"RSB1"` signature.
- Bytes `4..8` = original size of the bytecode (32-bit LE).
- After this, `buffer[8..]` is where we'll write the compressed data.

Now we compress the original bytecode into the buffer at index 8:

```rust
match zstd::bulk::compress_to_buffer(
    bytecode,
    &mut buffer[8..],
    zstd::zstd_safe::max_c_level()
)
```

- `bytecode` is the original uncompressed data.
- `&mut buffer[8..]` is where the compressed result will be written.
- `max_c_level()` gives us the best compression level ZSTD can offer.

If successful, the function returns `compressed_size`.

Next we truncate the buffer because after compression we compute the final size (header + compressed data) and then truncate the buffer to that size to remove any unused space at the end.

```rust
let total_size = compressed_size + 8;
buffer.truncate(total_size);
```


So at this point the bytecode should look like this:

```css
[ "RSB1" ][ original_size ][ compressed_bytecode... ]
```


But Roblox adds one more layer of XOR-based obfuscation where we will use the xxh32 hash of the buffer and the multiplier we defined at the start. Where we now compute the 32-bit hash of the buffer (header + compressed data) and then this hash will act as a base XOR key for obfuscation:

```rust
let key = xxh32(&buffer, BYTECODE_HASH_SEED);
let key_bytes = key.to_le_bytes();
```


Now we run a loop to XOR each byte of the buffer using the hash key and the byte index:

```rust
for i in 0..total_size {
    buffer[i] ^= key_bytes[i % 4]
        .wrapping_add((i as u8).wrapping_mul(BYTECODE_HASH_MULTIPLIER))
        & 0xFF;
}
```

For each byte:
- We use the corresponding byte in the `key_bytes`, based on `i % 4` (since it's 4 bytes).
- We add `i * BYTECODE_HASH_MULTIPLIER` and wrap it to fit in a byte.
- Finally, we apply a bitwise `& 0xFF` to make sure it stays within 8-bit range.
- We XOR the current byte with the result — thus scrambling it.

If that all succeeded we will return the buffer else we will return a error. So the final code should look similar to this:

```rust
pub fn compress(bytecode: &[u8]) -> Result<Vec<u8>, String> {  
    const BYTECODE_SIGNATURE: [u8; 4] = *b"RSB1";  
    const BYTECODE_HASH_MULTIPLIER: u8 = 41;  
    const BYTECODE_HASH_SEED: u32 = 42;  
  
    let data_size = bytecode.len();  
    let max_size = zstd::zstd_safe::compress_bound(data_size);  
    let mut buffer = vec![0u8; max_size + 8];  
  
    buffer[..4].copy_from_slice(&BYTECODE_SIGNATURE);  
    buffer[4..8].copy_from_slice(&(data_size as u32).to_le_bytes());  
  
    match zstd::bulk::compress_to_buffer(  
        bytecode,  
        &mut buffer[8..],  
        zstd::zstd_safe::max_c_level()  
    ) {  
        Ok(compressed_size) => {  
            let total_size = compressed_size + 8;  
            buffer.truncate(total_size);  
  
            let key = xxh32(&buffer, BYTECODE_HASH_SEED);  
            let key_bytes = key.to_le_bytes();  
  
            for i in 0..total_size {  
                buffer[i] ^= key_bytes[i % 4]  
                    .wrapping_add((i as u8).wrapping_mul(BYTECODE_HASH_MULTIPLIER))  
                    & 0xFF;  
            }  
  
            Ok(buffer)  
        },  
        Err(e) => Err(format!("ZSTD compression error: {}", e))  
    }  
}
```

