Now that we know how Roblox compresses bytecode, the natural next step is understanding how to decompress bytecode. Since Roblox just doesn't just use standard compression (like [ZSTD](https://en.wikipedia.org/wiki/Zstd)); it wraps the compressed data with a custom 8-byte header (`RSB1` + original size) and adds a layer of XOR-based obfuscation using a derived hash key.

Like in compression we need to define the same 3 constants again which are:

```rust
const BYTECODE_SIGNATURE: [u8; 4] = *b"RSB1";
const BYTECODE_HASH_MULTIPLIER: u8 = 41;
const BYTECODE_HASH_SEED: u32 = 42;
```

-  `BYTECODE_SIGNATURE`: This is a 4-byte prefix (`RSB1`) that marks the buffer as Roblox  
   compressed bytecode. It's always the the first 4 bytes of the resulting buffer.
-  `BYTECODE_HASH_SEED`: Is used to compute an XXH32 hash of the final buffer.
-  `BYTECODE_HASH_MULTIPLIER`: Is a multiplier applied per-byte in the XOR loop. 

So what do we need to do first? First we need to validate that the given in compressed data is valid so we do a check to see if 8 is bigger than given in compressed bytecode. This is to validate if there could even possibly be the 4-byte signature and the original uncompressed code size in there.

```rust
if compressed.len() < 8 {
    return Err("Compressed data too short".to_string());
}
```

So now that we have a basic validation we can make a mutable copy of the data so we can XOR and mutate it in-place.

```rust
let mut compressed_data = compressed.to_vec();
```

As we know already we need to retrieve the hash key. This is the clever part since the first 4 bytes of the buffer (after XOR obfuscation) originally looked like this during compression:

```rust
let key = xxh32(&buffer, BYTECODE_HASH_SEED);
```

But Roblox makes the 4-byte hash (`u32`) in a custom way so we must reverse the masking process byte-by-byte.

Now how would something like that work? Each byte of the hash was XOR'd with a per-index calculation during compression so we undo it by XOR'ing with the signature byte (`RSB1`) and subtracting that by `i * multiplier`. This gives us the original hash key as raw bytes (stored in `header_buffer`).

```rust
let mut header_buffer = vec![0u8; 4];

for i in 0..4 {
    header_buffer[i] = compressed_data[i] ^ BYTECODE_SIGNATURE[i];
    header_buffer[i] = header_buffer[i]
        .wrapping_sub((i as u8).wrapping_mul(BYTECODE_HASH_MULTIPLIER))
        & 0xFF;
}
```

Now that we retrieved the original hash key we need to deobfuscate the full buffer by cycling trough the `header_buffer` using `i % 4` and adding the position scaled multiplier. Then we XOR each byte and use the `0xFF` mask.

```rust
for i in 0..compressed_data.len() {
    compressed_data[i] ^= (header_buffer[i % 4]
        .wrapping_add((i as u8).wrapping_mul(BYTECODE_HASH_MULTIPLIER)))
        & 0xFF;
}

```

What we need to do now is to verify the integrity of the hash. We check if the reconstructed `hash_value` matches the `xxh32` hash of the deobfuscated buffer. If it doesn't match, something went wrong.

```rust
let hash_value = u32::from_le_bytes(header_buffer.try_into().unwrap());
let rehash = xxh32(&compressed_data, BYTECODE_HASH_SEED);

if rehash != hash_value {
    return Err("Hash mismatch during decompression".to_string());
}
```

After we fully verified the integrity we can no extract the uncompressed size. Since we know that the first 4 bytes is the signature and the other 4 bytes is the size we just need to slice the compressed data.

```rust
let decompressed_size = u32::from_le_bytes(compressed_data[4..8].try_into().unwrap());
```

And at the end we just decompress using the Zstandard where we skip the first 8 bytes (`signature + original size`) and then use `decode_all` from the [`zstd`](https://crates.io/crates/zstd) crate to decompress it.
Finally we check if the output matches original size.

```rust
let compressed_data = &compressed_data[8..]; // strip off the 8-byte header

match decode_all(compressed_data) {
    Ok(decompressed) => {
        if decompressed.len() != decompressed_size as usize {
            return Err("Decompressed size mismatch".to_string());
        }
        Ok(decompressed)
    }
    Err(e) => Err(format!("ZSTD decompression error: {}", e))
}
```


------------------

So the full code should look like this:
```rust
pub fn decompress(compressed: &[u8]) -> Result<Vec<u8>, String> {  
    const BYTECODE_SIGNATURE: [u8; 4] = *b"RSB1";  
    const BYTECODE_HASH_MULTIPLIER: u8 = 41;  
    const BYTECODE_HASH_SEED: u32 = 42;  
  
    if compressed.len() < 8 {  
        return Err("Compressed data too short".to_string());  
    }  
  
    let mut compressed_data = compressed.to_vec();  
    let mut header_buffer = vec![0u8; 4];  
  
    for i in 0..4 {  
        header_buffer[i] = compressed_data[i] ^ BYTECODE_SIGNATURE[i];  
        header_buffer[i] = header_buffer[i]  
            .wrapping_sub((i as u8).wrapping_mul(BYTECODE_HASH_MULTIPLIER))  
            & 0xFF;  
    }  
  
    for i in 0..compressed_data.len() {  
        compressed_data[i] ^= (header_buffer[i % 4]  
            .wrapping_add((i as u8).wrapping_mul(BYTECODE_HASH_MULTIPLIER)))  
            & 0xFF;  
    }  
  
    let hash_value = u32::from_le_bytes(header_buffer.try_into().unwrap());  
  
    let rehash = xxh32(&compressed_data, BYTECODE_HASH_SEED);  
    if rehash != hash_value {  
        return Err("Hash mismatch during decompression".to_string());  
    }  
  
    let decompressed_size = u32::from_le_bytes(  
        compressed_data[4..8].try_into().unwrap()  
    );  
  
    let compressed_data = &compressed_data[8..];  
    match decode_all(compressed_data) {  
        Ok(decompressed) => {  
            if decompressed.len() != decompressed_size as usize {  
                return Err("Decompressed size mismatch".to_string());  
            }  
            Ok(decompressed)  
        }  
        Err(e) => Err(format!("ZSTD decompression error: {}", e))  
    }  
}
```
