VM Shuffles randomize the order of struct members in Roblox's Luau virtual machine (VM) structures (e.g., `lua_State`). They are computed at compile time and are a layer of protection to make the VM harder to reverse engineer.

You need them because they hide fields like `lua_State::stack` or `Proto::code` which are crucial for the executor to work. Accessing those fields without having the structure's layout randomized may result in undefined behavior.
