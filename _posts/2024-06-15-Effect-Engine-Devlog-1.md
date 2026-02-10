---
title: "Effect Engine Devlog #1"
layout: post
author: Xavier
excerpt: Welcome to the very first devlog of Effect Engine! Being the first devlog, this one will be a bit unusual. It will roughly cover how the game engine currently works as well as what I am currently working on. There won't be many demos at the moment or the forseeable future, as the game engine is still in a state where it cannot make a fully functional game. However, I hope to change that slightly later on with the introduction of working text and menus.
---

![image](https://www.dropbox.com/scl/fi/lor1e0i8tw7do1t7g2lgo/title.jpg?rlkey=84bf0g0vs23ylr6ekof7l92pz&st=foa4dwtm&raw=1)

Welcome to the very first devlog of Effect Engine! Being the first devlog, this one will be a bit unusual. It will roughly cover how the game engine currently works as well as what I am currently working on. There won't be many demos at the moment or the forseeable future, as the game engine is still in a state where it cannot make a fully functional game. However, I hope to change that slightly later on with the introduction of working text and menus.

## The engine as it stands

The Effect Engine is currently written in Rust and utilises wgpu. This may change in the future to also utilise Vulkan directly via [Ash](https://github.com/ash-rs/ash). However that is a future problem, as my immediate goals are to clean up the current codebase, then add text and functional menus. In its current state, the game engine is mostly a 2D renderer with some additional features such as audio and a built in camera.

The renderer currently is only capable of producing 2D graphics, however basic 3D with custom shader support is planned in the future. The architecture is fairly simple. The renderer divides everything up into "render layers". These render layers are then drawn in ascending order based on their LayerID.

{% highlight rust %}
render_pass.set_bind_group(1, self.camera.bind_group(), &[]);
    for (_, layer) in self.layers.iter() {
        render_pass.set_bind_group(0, layer.bind_group(), &[]);
        render_pass.set_vertex_buffer(0, layer.vertex_buffer());
        render_pass.set_vertex_buffer(1, layer.entity_buffer().unwrap());
        render_pass.draw_indexed(0..6 as u32, 0, 0..layer.entity_count() as u32);
    }
    drop(render_pass);
    self.queue.submit(std::iter::once(command_encoder.finish()));
    surface_texture.present();
    Ok(())
{% endhighlight %}

Each layer stores its related entity data in buffer, along with a texture atlas. The entity data just includes basic position and transformation data as well as its current layer, and how to extract its required texture from the texture atlas. What this means is, each texture is only stored once per layer, reducing duplication of the data. The benefit of the texture atlas and associated layer in this case is that all entities of a particular layer can be drawn in a single draw call, reducing CPU overhead.

In order to update each entity, each layer has to be updated individually by calling a function in Layer2DSystem, which takes the entities then updates the buffers in the layer based on the entity data. In the future, I plan to design this such that each layer can be updated in parallel.

Speaking of which, the current architectural goal, in terms of thread model, is to have a singular main thread, which uses thread pools to delegate work which can be done in parallel, as well as having any engine functionality that can be done in parallel, do so. This should make the problem of utilising the full CPU as well as synchronisation a lot easier. I still have a lot to learn on this topic, so this will likely change as my understanding of modern game engine architecture improves.

## So... what am I doing currently?

I noticed a few issues with the engine as it stands..namely, there is a lot of repetition and initialising various structures is very verbose.

For example, take this pipeline initialisation:

{% highlight rust %}
pub async fn new(window: winit::window::Window, v_sync: bool) -> Self {
        let instance = wgpu::Instance::new(wgpu::InstanceDescriptor {
            backends: wgpu::Backends::all(),
            ..Default::default()
        });
        let window = Arc::new(window);
        let surface = instance.create_surface(window.clone()).unwrap();
        let adapter = instance
            .request_adapter(&wgpu::RequestAdapterOptions {
                power_preference: wgpu::PowerPreference::default(),
                force_fallback_adapter: false,
                compatible_surface: Some(&surface),
            })
            .await
            .unwrap();
        let (device, queue) = adapter
            .request_device(
                &wgpu::DeviceDescriptor {
                    label: Some("Adapter"),
                    required_features: adapter.features(),
                    required_limits: wgpu::Limits::default(),
                },
                None,
            )
            .await
            .unwrap();
        let indices: [u16; 6] = [0, 1, 2, 0, 2, 3];
        let index_buffer = device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
            label: Some("Index Buffer"),
            contents: bytemuck::cast_slice(&indices),
            usage: wgpu::BufferUsages::INDEX,
        });
..
{% endhighlight %}

There are quite a number of steps here, and a lot of what is done here is repeated later on. Not to mention the user has very little control of these options, unless I create a huge list of parameters to pass. There is a solution for this, the builder pattern.

The builder pattern allows you to easily create any structure, using as minimal options as possible if you want, but also granting the ability to use more advanced features if you wish. Not only that, it is very easy to extend with new options and features without breaking existing code. It is essentially a cleaner version of [function overloading](https://en.wikipedia.org/wiki/Function_overloading). The builder pattern does add some verbosity for the structure it is "building", but in my opinion the trade off is worth it, as it makes future usability and maintainability much simpler.

For example, take this new `BufferAllocator` structure, which builds a `Buffer` structure:

{% highlight rust %}
pub struct BufferAllocator {
    usage: wgpu::BufferUsages,
    mapped_at_creation: bool,
    size: u64,
    label: Option<&'static str>,
    data: Option<Vec<u8>>,
}

impl Default for BufferAllocator {
    /// Default usage: Vertex stage visibility and COPY_DST.
    /// Not mapped at creation
    /// Size of 5096 bytes
    /// No label
    fn default() -> Self {
        let usage = wgpu::BufferUsages::VERTEX | wgpu::BufferUsages::COPY_DST;
        let mapped_at_creation = false;
        let size = 5096;
        let label = None;
        let data = None;
        Self {
            usage,
            mapped_at_creation,
            size,
            label,
            data,
        }
    }
}

impl BufferAllocator {
    pub fn usage(mut self, usage: wgpu::BufferUsages) -> Self {
        self.usage = usage;
        self
    }

    pub fn mapped_at_creation(mut self, mapped_at_creation: bool) -> Self {
        self.mapped_at_creation = mapped_at_creation;
        self
    }

    /// Set the size of the buffer in bytes
    pub fn size(mut self, size: u64) -> Self {
        self.size = size;
        self
    }

    pub fn label(mut self, label: &'static str) -> Self {
        self.label = Some(label);
        self
    }

    pub fn data(mut self, data: Vec<u8>) -> Self {
        self.data = Some(data);
        self
    }

    /// Create the buffer
    fn allocate_buffer(&self, device: &wgpu::Device) -> wgpu::Buffer {
        match self.data.as_ref() {
            Some(data) => device.create_buffer_init(&wgpu::util::BufferInitDescriptor {
                label: self.label,
                contents: data,
                usage: self.usage,
            }),
            None => device.create_buffer(&wgpu::BufferDescriptor {
                label: self.label,
                size: self.size,
                usage: self.usage,
                mapped_at_creation: self.mapped_at_creation,
            }),
        }
    }

    pub fn allocate(self, device: &wgpu::Device) -> Buffer {
        let buffer = self.allocate_buffer(device);
        let size = match self.data {
            Some(data) => data.len(),
            None => 0,
        };
        let capacity = self.size as usize;
        let usage = self.usage;
        Buffer::new(buffer, size, capacity, usage)
    }
}
{% endhighlight %}

As you can see, some good defaults are provided, while also allowing some choices to be optional, simplifying intialisation of the Buffer structure. You may notice that the member functions don't take a reference, but instead actually `move` the data of self. Then it returns itself. This allows one to continuously call member functions in a chain. An example of its usage is as follows:

{%highlight rust %}
pub fn write(&mut self, data: &[u8], device: &wgpu::Device, queue: &wgpu::Queue) {
    let size = std::mem::size_of_val(data);
    if size > self.capacity {
        let capacity = size * 2;
        let buffer = BufferAllocator::default()
            .size(capacity as u64)
            .usage(self.usage)
            .allocate_buffer(device);
        queue.write_buffer(&buffer, 0, data);
        self.capacity = capacity;
        self.size = size;
    } else {
        queue.write_buffer(&self.buffer, 0, data);
        self.size = size;
    }
}
{% endhighlight %}

There are of course two different allocate functions. The one used here actually returns a `wgpu::Buffer` because this is an internal write function of my own Buffer structure, but as you can see, it makes things simpler than manually building a structure or passing a huge amount of arguments. Another example is the actual inialisation of the engine itself,

{% highlight rust %}
let (mut app, event_loop) = EffectAppBuilder::default()
    .resizable_window(false)
    .fullscreen_mode(FullScreenMode::WINDOWED)
    .window_dimensions(840, 480)
    .monitor(0)
    .build()
    .get_wgpu_2d();
{% endhighlight %}

Most of these arguments are optional, and there will be even more arguments to come. Imagine if they were all function overloads or all had to be manually passed, it would be a nightmare! However with the builder pattern, the initialisation can be even simpler!:

{% highlight rust %}
let (mut app, event_loop) = EffectAppBuilder::default()
    .build()
    .get_wgpu_2d();
{% endhighlight %}

I think this adequately highlights the strength of this design pattern.

Another thing I have been working on is simplifying functionality that is often repeated, eg creating and writing to buffers.

Forgive the long block of code, but it is necessary to highlight how verbose it is:

{% highlight rust %}
impl WebLayer2DSystem {
    fn alloc_buffer(
        data: &[u8],
        size: u64,
        device: &wgpu::Device,
        queue: &wgpu::Queue,
        label: &str,
        index: bool,
    ) -> wgpu::Buffer {
        let usage;
        if index {
            usage = wgpu::BufferUsages::INDEX;
        } else {
            usage = wgpu::BufferUsages::VERTEX;
        }
        let buffer = device.create_buffer(&wgpu::BufferDescriptor {
            label: Some(label),
            size,
            usage: usage | wgpu::BufferUsages::COPY_DST,
            mapped_at_creation: false,
        });
        queue.write_buffer(&buffer, 0, data);
        buffer
    }

    // alloc buffer to 2x the size and set new max entity count
    fn create_entity_buffer(
        entities: &[&WebEntity2D],
        device: &wgpu::Device,
        queue: &wgpu::Queue,
    ) -> wgpu::Buffer {
        let ents = entities.iter().map(|e| e.to_raw()).collect::<Vec<_>>();
        let data: &[u8] = bytemuck::cast_slice(ents.as_slice());
        let size = std::mem::size_of_val(data) as u64;
        WebLayer2DSystem::alloc_buffer(data, size * 2, device, queue, "Entity Buffer", false)
    }

    fn _create_index_buffer(device: &wgpu::Device, queue: &wgpu::Queue) -> wgpu::Buffer {
        let mut indices: Vec<u16> = Vec::new();
        indices.extend_from_slice(&[0, 1, 2, 0, 2, 3]);
        let data: &[u8] = bytemuck::cast_slice(&indices);
        let size = (std::mem::size_of::<u16>() * 6) as u64;
        WebLayer2DSystem::alloc_buffer(data, size, device, queue, "Index Buffer", true)
    }

    fn create_vertex_buffer(
        _texture_size: PhysicalSize<u32>,
        tex_coord_size: PhysicalSize<f32>,
        device: &wgpu::Device,
        queue: &wgpu::Queue,
    ) -> wgpu::Buffer {
        let width = 0.5;
        let height = 0.5;
        let verts = vec![
            Vertex {
                position: [width, height, 0.0],
                tex_coords: [tex_coord_size.width, 0.0],
            },
            Vertex {
                position: [-width, height, 0.0],
                tex_coords: [0.0, 0.0],
            },
            Vertex {
                position: [-width, -height, 0.0],
                tex_coords: [0.0, tex_coord_size.height],
            },
            Vertex {
                position: [width, -height, 0.0],
                tex_coords: [tex_coord_size.width, tex_coord_size.height],
            },
        ];
        let data: &[u8] = bytemuck::cast_slice(verts.as_slice());
        let size = (std::mem::size_of::<Vertex>() * verts.len()) as u64;
        WebLayer2DSystem::alloc_buffer(data, size, device, queue, "Vertex Buffer", false)
    }

    /// Set the entity data for the particular layer.
    /// Ensure every entity has a texture from the specified layer otherwise you will run into problems.
    pub fn set_entities(
        layer: &mut WebLayer2D,
        entities: &[&WebEntity2D],
        device: &wgpu::Device,
        queue: &wgpu::Queue,
    ) {
        // allocating exactly amount needed each time may increase the number of allocations needed..
        // perhaps a strategy of allocatin 2X needed data would be better
        layer.entity_count = entities.len();

        if layer.entity_count() > layer.entity_maximum() || layer.entity_buffer().is_none() {
            // Allocate new buffers
            layer.entity_buffer = Some(WebLayer2DSystem::create_entity_buffer(
                &entities, device, queue,
            ));
            layer.entity_maximum = layer.entity_count * 2;
        } else {
            // Reuse buffers
            let ents = entities.iter().map(|e| e.to_raw()).collect::<Vec<_>>();
            let entity_data: &[u8] = bytemuck::cast_slice(ents.as_slice());
            queue.write_buffer(&layer.entity_buffer.as_ref().unwrap(), 0, entity_data);
        }
    }
}
{% endhighlight %}

This is the old code that essentially updated the layer buffers based on new entities passed to it. This is obviously far too complicated, leading to the potential for more mistakes that could lead to memory bugs or performance issues. As you can see there is a lot of repeated code, and extra parameters passed around that only really serve the purpose of a flag. There is a much better way to do this. Here are the relevant parts of the `Buffer` structure:

{% highlight rust %}
pub struct Buffer {
    buffer: wgpu::Buffer,
    size: usize,
    capacity: usize,
    usage: wgpu::BufferUsages,
}

impl Buffer {
    /// Overwrite the data in the buffer, data must be a slice of bytes
    pub fn write(&mut self, data: &[u8], device: &wgpu::Device, queue: &wgpu::Queue) {
        let size = std::mem::size_of_val(data);
        if size > self.capacity {
            let capacity = size * 2;
            let buffer = BufferAllocator::default()
                .size(capacity as u64)
                .usage(self.usage)
                .allocate_buffer(device);
            queue.write_buffer(&buffer, 0, data);
            self.capacity = capacity;
            self.size = size;
        } else {
            queue.write_buffer(&self.buffer, 0, data);
            self.size = size;
        }
    }
}
{% endhighlight %}

The usage is stored within the buffer, so a flag no longer needs to be passed around. The allocation and writing itself is then fully managed within one function. So if the amount of data passed exceeds the buffer size, this will be handled by the write function. Now the Layer structure no longer has to store the capacity itself or manage any of these details. This functionality can now also be easily reused for anything else that may need a buffer. Remember, the buffer is created using the `BufferAllocator` builder, and calling the `allocate()` function.

The managing of buffers related to the entities is now much simpler for `Layer2DSystem`!:

{% highlight rust %}
pub struct Layer2DSystem;

impl Layer2DSystem {
    pub fn set_entities(
        layer: &mut Layer2D,
        entities: &[&Entity2D],
        device: &wgpu::Device,
        queue: &wgpu::Queue,
    ) {
        let data: Vec<Entity2DRaw> = entities.iter().map(|e| e.to_raw()).collect();
        match layer.entity_buffer.as_mut() {
            Some(buf) => {
                buf.write(bytemuck::cast_slice(&data), device, queue);
            }
            None => {
                // Not using the data method as it is better for when the entities won't increase,
                // as reallocating a lot when creating new layers / extending the buffer could cause stutters.
                // the *2 means 2 times the needed buffer size will be allocated
                let mut buf = BufferAllocator::default()
                    .usage(wgpu::BufferUsages::VERTEX)
                    .size((std::mem::size_of::<Entity2DRaw>() * data.len()) as u64 * 2)
                    .allocate(device);
                buf.write(bytemuck::cast_slice(&data), device, queue);
                layer.entity_buffer = Some(buf);
            }
        }
        // Extend indices to match number of entities
        layer.entities = entities.len();
        let mut data = Vec::new();
        for i in 0..entities.len() {
            data.extend_from_slice(&INDICES);
        }
        layer
            .index_buffer
            .write(bytemuck::cast_slice(&data), device, queue);
    }
}
{% endhighlight %}

Yes, this is currently the entire implementation of `Layer2DSystem`. Much better! Once the buffer is initially created, it is then just a single function call! Okay yes I admit, some of the code such as the vertex calculation was just moved elsewhere, namely to the initialisation of the layer itself, since I will be switching to Texture Arrays over Texture Atlases. Since textures in a texture array must be uniform in this case, all vertex calculation can be done upon initialisation. The reason I chose to require the layers to have all the textures they need upon initialisation is because it was much simpler to code for a texture atlas. However it is not likely a huge deal for a texture array, so this is functionality I can probably provide later on.

## My plans for the future

A lot of the plans are written down in a README in the [repository](https://github.com/XavierCS-dev/Effect-Engine), but to give you an idea of my immediate goals after the current codebase cleanup..it is to release 0.3.0-alpha once the cleanup is done, then working on the GUI. Most games require some sort of text display and menus, so it is important I get this done as soon as possible, I plan to make a GUI library so common menu design patterns are easy to do, for example, a list of buttons. The main issue will be figuring out mouse and menu collisions, as well as mouse and non-menu collisions for point and click games, this may result in needing to work on the physics system before mouse menu control would be supported.

Now, talking about this site. I plan to write mostly dev logs for the engine, however I also want to write some articles about news in the tech world that I am interested in, mostly new hardware, technologies, leaks and speculation among other things. In terms of article frequency, there is no guarantee, multiple articles may be written in the same week, or it might be years between articles. Regarding the theme and appearance, I may change it in the future, however it would be a substantial amount of effort, so it would require significant readership to motivate me, something which I believe is unlikely due to the nature of this site.

I hope you enjoyed this first devlog! I actually plan on making the repository for this site public, so issues can be made to correct any errors. Oh and don't worry, once engine development is further along..there will be more images..and videos, rather than blocks of code.


