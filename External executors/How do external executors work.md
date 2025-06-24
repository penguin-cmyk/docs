
So first of all we get all the core modules by searching for `CoreGui/RobloxGui/Modules` in the Datamodel. 

If that is found we look for the `PlayerListManager` inside of the modules (`PlayerList/PlayerListManager`). As we'll need to search for our module which we are gonna hijack which should also be located in `CoreGui/RobloxGui/Modules`.

After we got all of that we need to find the `VRNavigation` ModuleScript, which is located here
`StarterPlayer/StarterPlayerScripts/PlayerModule/ControlModule/VRNavigation`.

Once we have the base ModuleScript's found we can work from there. First we unlock the `VRNavigation` and then write the `VRNavigation` address into the address from the `PlayerListManager` + the offset of itself which is 0x8. So it will look similar to this:

```rust
PROCESS.write_memory::<usize>(player_list_manager + 0x8, &vrnavigation).unwrap();
```

Once we have done that the vr navigation modulescript is now loaded with the `PlayerListManager` and execution of the `PlayerListManager` is redirected to the `VRNavigation`

Once we have redirected the execution we can now start modifying the bytecode of the `VRNavigation` and the ModuleScript which you want to hijack. A example would look like this

```rust
rbx::set_bytecode(vrnavigation, init_script, true).unwrap(); // Revert to old bytecode
rbx::set_bytecode(module_to_hijack, init_script, false).unwrap();
```

After that we send `VK_ESCAPE` key event when the roblox client is open and that opens and closes escape. This will refresh the `PlayerList` within the escape menu which reloads `PlayerListManager` and with our execution redirected it will use the `VRNavigation` instead of the original `PlayerListManager` module.

Once we've loaded everything we can revert back to the "old roblox" by writing the original `PlayerListManager` address to the playerlist self. It would look something like this:

```rust
PROCESS.write_memory::<usize>(playerlist + 0x8, &playerlist).unwrap();
```

