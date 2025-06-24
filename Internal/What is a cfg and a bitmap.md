CFG stands for control flow guard. The control flow guard validates pointers and crashes if they are invalid / not verified. The bitmap just basically contains all of the valid pointers that can be used  / that are verified. 

So how do bypass the control flow guard?

First, we need to get our image (DLL) size: To do this, we get a pointer to the `IMAGE_NT_HEADERS` structure by using the `e_lfanew` offset from the DOS header. Then we can access the `SizeOfImage` property in `OptionalHeader`, which gives us the total size of the image (DLL) in memory, including all headers and sections which tells us how much memory range to iterate over.

Then we dereference the CFG bitmap pointer to get the base address of the CFG bitmap which tracks what addresses are considered valid. 

Now that we have the CFG bitmap we can iterate over each memory page in the module (in 64 KB steps). Each page is aligned to 64 KB boundaries.

In that loop we calculate the address of the CFG bitmap entry for the current memory page. 
`currentAddress >> 0x13` divides by `2^19` (i.e., 512KB), as each byte in the bitmap covers `8 * 64KB = 512KB`. 

Then we mark the entire byte (8 possible 64KB regions) as valid call targets by setting all 8 bits to 1.
This makes CFG think the current 64KB region is verified, allowing indirect calls/jmps/etc to anywhere inside it.

```cpp
const auto ntHeaders = reinterpret_cast<IMAGE_NT_HEADERS *>(  
    dllBase + reinterpret_cast<IMAGE_DOS_HEADER *>(dllBase)->e_lfanew);  
const auto imageSize = ntHeaders->OptionalHeader.SizeOfImage;  
const auto cfgBitmap = *reinterpret_cast<uintptr_t *>(CfgBitmapPointer);  
  
for (auto currentAddress = dllBase; currentAddress < dllBase + imageSize; currentAddress += 0x10000) {  
    const auto pageBitAddress = reinterpret_cast<int *>(cfgBitmap + (currentAddress >> 0x13));  
    *pageBitAddress = 0xFF;  
}
```


Source credits: [Pixeluted](https://github.com/Pixeluted), [Chocosploit](https://github.com/Pixeluted/Chocosploit/blob/master/Entry.cpp#L30)
Help: [felixtaken](https://discord.com/users/964160170273939516)
