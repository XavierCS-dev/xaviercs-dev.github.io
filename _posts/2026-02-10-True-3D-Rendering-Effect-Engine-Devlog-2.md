---
title: "True 3D Rendering - Effect Engine Devlog #2"
layout: post
author: Xavier
excerpt: Effect Engine has underdone significant changes from the last blog post, including a transition to Vulkan and Full 3D.
---
It has been a while since my last post, and much has changed since then. Firstly, I have ditched Rust and WGPU for C++ and Vulkan. I felt the Rust rendering ecosystem was not developing in the way I liked, and relying on Rust bindings to C/C++ libraries made updating dependencies difficult, hence the switch. Although Rust's memory safety will be sorely missed, development has become much smoother.

## What can the engine do?
Not much currently:
![Plane of grass cubes against a sky-blue background](https://www.dropbox.com/scl/fi/0k6031h9w9c8oqf7uuj2u/article_img.webp?rlkey=dd5h82cb1xpsunv0lwps2pgi2&st=wzmh2o5x&raw=1)

This isn't too surprising considering this is actually the 2nd iteration of the Vulkan + C++ version of the engine, and there was a long period where I didn't work on the engine.

The cubes above are drawn as 10,000 instances of a single mesh which was loaded from a glTF file. [Greedy meshing](https://0fps.net/2012/06/30/meshing-in-a-minecraft-game/){:target="_blank"} was not used here. The debug UI in the top left corner was created using Dear ImGui.

In terms of interactivity, there are simple input controls, I have both an FPS and Fly cameras implemented with keyboard movement along the xz plane. It is also possible to press a key-bind that reveals the cursor and allows you to interact with the debug interface.


### Vulkan
Creating the Vulkan renderer is where the majority of the time on this project has been spent so far. At the moment the architecture is rather simple, there are no render graphs and pipeline barriers are inserted manually.
#### Scene Data
The majority of the renderer's working data is stored in a "Scene Data" class, which includes the instance buffers, uniform buffers, vertex buffers and descriptor sets. Uniform and instance data is considered to be "per frame data", and are updated each frame. This means multiple copies, 2 total in this case, can be stored, so while the GPU is reading data from one set of buffers, you can simultaneously update the other set. I plan to create a separate structure for managing static meshes as this means data will be copied less often due to buffer resizing, while eliminating gaps in the buffer allocation.
#### Buffer Allocator
At some point, I will be using `DrawIndexedIndirect` to issue draws specified by the GPU, and in this case, having one large buffer for vertices. `DrawIndexedIndirect` can be issued just once, as I can pass all of the buffer data in one go. In my case, I pass pointers via push constants as I am using Buffer Device Address.

Having one giant buffer sounds fine, until you start thinking about the case where you need to manage it for longer lived data. Over time when data is added and removed,  you end up with lots of "holes" in the buffer, with no way to remove them, i.e. fragmentation. You can copy the data over to a new buffer, then find and update every single index into that buffer, but performing this operation may lead to spikes in frame time, depending on how much data you have, as you must stall the GPU to ensure the buffer is not being used.

The solution for me was to create a buffer allocator which attempts to fit new data into free memory ranges of an allocation, reducing the need for more thorough defragmentation operations. It is actually quite easy to produced a poor allocator. For example, if you don't merge adjacent free memory ranges, you can end up with many small free ranges that the majority of allocations can't fit into. It is also very easy to create complicated allocate and free functions with hidden bugs, for example my free function was initially faulty in that it didn't properly detect double frees of partially overlapping ranges.

Per frame data can get away with a much simpler "bump allocator", where the current offset to write data is increased by the data size, as the whole buffer will be reset the next time the frame data is accessed by setting the offset to 0.
#### Transfer Queue
I was having serious issues with stutters and poor frame rate when relying on blocking data transfers, i.e. transfers that block the CPU until they are fully complete. Currently I still have poor frame pacing when using any of the FIFO presentation modes (vsync), but the overall performance has improved significantly after creating a dedicated system for transferring data.

The transfer queue works on two types of transfers, direct writes to GPU or "device" memory, and transfers using a staging buffer. Direct writes to device memory occur using `vmaMapMemoryToAllocation`, which is a host blocking operation, therefore I have these types of transfers execute in a worker thread. The main thread adds jobs to a `dequeue` which is guarded by a `mutex`, and it signals the worker thread that work is available using a `condition_variable`.

The next kind of transfer is a staging transfer. I always allocate staging buffers that are host visible (the memory can be mapped into CPU address space), so the previous transfer method also applies to transferring data into a staging buffer. The following step is to transfer the data from the staging buffer to the final destination buffer. This is done with a transfer command that is submitted to the GPU, meaning it can be done on the main thread. Currently I have "frames in flight" for the transfer queue. This was so that I could submit draws to the GPU, and even start work on the next frame before the transfers finished. This feature is now obsolete as I recently found out, if you signal [semaphores](https://en.wikipedia.org/wiki/Semaphore_(programming)){:target="_blank"} on the host (relevant for direct transfers to host visible memory), you cannot use the semaphore in a submit chain that is eventually waited on by the presentation engine, as the presentation engine has no guarantees the signal will ever be received. What this means is, all transfers must be completed before the final submit that comes before the Swapchain present, and so the per frame in flight staging buffers are no longer needed.

This brings us to the main synchronisation between transfers. All transfers are ordered strictly in a FIFO fashion. That is, the first transfer to be submitted, will always completely finish before the next one starts. This synchronisation is performed using a timeline semaphore, a type of semaphore where its value is always incremented, and you wait on a specific value before you begin your operations. Each time a transfer is requested, a transfer is given a value to wait on, then it signals value + 1 once it is done. This enforces ordering in a way a binary semaphore (signalled or unsignalled) can't; a binary semaphore provides no way for each transfer to know whether it is its "turn" or not. This might seem unnecessary if you only consider the direct transfer to host visible memory, but the staging transfers which submit a transfer command to a queue, would otherwise be asynchronous to the direct transfers without the semaphore! That is a problem when one transfer relies on another to occur, the obvious one being a transfer from a staging buffer to another buffer relying on the transfer to the staging buffer.

There are problems with this method, for example, many transfers are independent, and in theory could execute in parallel, but determining such cases is difficult in practice.
#### Instances and Draw Calls
Currently instances are managed in "blocks":
```c++
struct RenderInstanceBlock {
    uint32_t first_instance;
    uint32_t count;
    EfeMeshID mesh;
};

```
These blocks are able to grow and shrink over time, and are written to a buffer each frame.
The set of draw calls for that frame is then updated based on the offsets in the render instance block. Each block has corresponding CPU side data which is what is written to the instance buffers.

A major problem with the current architecture is that the CPU instance data doesn't currently have any properly defined way to be stored. I just have a singular vector storing data for the one particular instance block shown in the image at the top of this article. There is also no way to manage visibility of instances or instance blocks, as all instance blocks must be updated every frame, otherwise they will have stale offsets.

I am not sure which heading to put this under, and it is too short to qualify for its own section, so I will leave it here. Currently I have two draw passes, one clears an image, then draws the cubes and debug interface to it. The second pass reads the image as an input attachment and copies it into the Swapchain. This second pass will be used to apply post-processing effects later down the line.
### Dynamic Libraries
In the future, I plan to add hot reload for any game created with this engine. I don't plan to have an editor, so having hot reload at the minimum will help maintain development speed. There are several ways to go about hot reloading, but I have chosen to do it with [dynamic libraries](https://en.wikipedia.org/wiki/Dynamic_library){:target="_blank"} and an [Entity Component System (ECS)](https://en.wikipedia.org/wiki/Entity_component_system){:target="_blank"}.

I currently have neither an ECS or hot reloading working, but Effect Engine can run code from dynamic libraries! For example I have the following start up code written in Zig which is compiled into a dynamic library loaded by the game engine:
```zig
// Export is needed so the function can be called
pub export fn init_app(app: *c_engine.EfeAppHandle, api: *c_engine.EfeAppAPI) void {
	// Zig strings have a null sentinel value already.
    api.set_app_name.?(app, "Ocean Game");
    api.check_app_api.?("0.0.1");
    api.set_resolution.?(app, 2560, 1440);
    api.enable_debug_ui.?(app, true);
    api.enable_vsync.?(app, false);
}

```
If you have made it this far, you probably already know what a dynamic library is, but as a refresher, a dynamic library is compiled code that you don't need to link with at build time. This means the library can be freely swapped out for other libraries at run time. This is useful for hot reload because you can edit and recompile the code in the library, and swap it out at run time. If the main application owns all of the memory the dynamic library uses, it becomes much easier to preserve state across library reloads, making iteration for game development much faster, as you don't need to repeatedly replicate previous state.

The ECS comes in because it makes it very easy for the engine to own all of the memory, the library simply needs to define the systems, components, and the initial state.

## Plans for the future
I eventually plan to open source the engine completely once it has more features and I am happy with the current state of the code. I still have much to learn and therefore many improvements to make.

I have already discussed my plans to improve instance management, adding an ECS, adding hot reloading and using GPU driven rendering. I plan to make a simple [voxel](https://voxel.wiki/wiki/introduction/){:target="_blank"} sandbox with my engine, but this requires voxel-specific rendering techniques for reasonable performance. Then overtime will come basic graphical features, such as a sky box, ambient occlusion, lighting and so on. I am aim to implement [Physically Based Rendering (PBR)](https://en.wikipedia.org/wiki/Physically_based_rendering){:target="_blank"} in later stages of development.

My plans are loose with a general idea of what I want to. Progress will likely be slow for a long while, as I can only work on this project in my spare time after work. Perhaps when the ECS is implemented, it could make a good article!

For now, I think I will get on with improving instance management.