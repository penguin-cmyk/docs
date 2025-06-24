### What even is the task scheduler?
--------------------
The `Roblox Task Scheduler` is a core component of the Roblox engine responsible for managing and executing tasks (or "jobs") in the game. It manages various game related processes, such as rendering, physics, scripting, and network updates. 

The `Scheduler` operates in a loop processing tasks during each frame to keep the game running smoothly.
### What is a step hook?
-----
A `step hook` refers to intercepting or "hooking" the scheduler's step function, which is called each frame to process tasks. By hooking this function, you can inject custom logic into the scheduler's execution loop, such as running scripts.
### Why do we need to yield tasks?
----
Yielding in the context of Roblox scripting refers to pausing the execution of a script or task and resuming it later. In Lua, yielding is achieved using coroutines, which allow scripts to "yield" control back to the scheduler, enabling other tasks to run before resuming. 

In our case we need yielding to handle asynchronous callbacks that need to be executed at a later time.

