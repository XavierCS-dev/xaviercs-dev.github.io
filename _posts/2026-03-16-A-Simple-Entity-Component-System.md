---
title: "A Simple Entity Component System - Effect Engine Devlog #3"
layout: post
author: Xavier
excerpt: A simple entity component system to manage player defined behaviour with reasonable performance.
---
Being able to render some 3D models to a window is all well and good, but this doesn't constitute a particularly useful game engine. What we need is some way for a developer to move "things" around within 3D space without having to recompile the game engine. 

## Why use ECS?
There are many arguments you can find online about why Object Oriented Programming (OOP) is "bad". Many of these arguments don't convince me as they consist of arguing against poorly written OOP code that abuses inheritance.

So, why an Entity Component System (ECS) then? There are two main reasons. Firstly, if you want to add some existing behaviour to an entity, you just add the necessary components to it and the relevant system will pick it up. Then your job is done, it really is that easy. There is also a more technical reason for my use of ECS.

One of the goals for my engine is to allow developers to write their "scripts" in any language that can interface with C and compile to a library with a C ABI. Actually achieving this in practice will likely be very difficult, especially when said user code must be dynamically linked against in development mode, and statically linked against in release mode. So for now, I will only officially support C based game projects. Also I just like C. Anyway, the point is, C doesn't have OOP objects, and so developers cannot make use of OOP without it being manually implemented.

How could user written logic interface with with actual entity data? Remember, the engine must own all of the memory to keep hot reload simple, and when dynamic linking, the engine can only know the size and alignment of the data types, unless some schema is used to transmit type information. One thing you could do, is provide an allocation interface, where the C code allocates objects, the engine then passes these objects to the C code in an update function. Deducing struct types is likely to be very difficult. You could have a function pointer at the start of the struct with a function that knows its type, but manually defining these for each object isn't great from a user perspective, and each time a hot reload happens, all relevant function pointers are now invalidated and you have to figure out which ones to change.

```c
typedef struct Entity {
    void (*update)(void* entity, double delta_time);
    // Other members below
    ...
} Entity;
```

You could potentially fix that by storing all of the similar structs together. Of course since we aren't using OOP that can't be done via polymorphism, and manually maintaining arrays of every type of struct is going to be a large maintenance burden. Instead these structs will be stored together based on what they are composed of. The way this is done is by storing a vector of arrays of components with an unordered map with their component ID as a key and corresponding vector index within an `std::unordered_map`. These sets of components... archetypes, if you will, can be given an ID to distinguish them from other combinations. 

```c++
struct ComponentArray {
    ComponentArray(void* data, uint32_t component_size)
        : data{data}, component_size{component_size} {}

    void* operator[](size_t index) {
        return reinterpret_cast<std::byte*>(data) + (index * component_size);
    }

    // Alignment is assumed to be the same as C
    void* data;
    size_t component_size;
};

class ComponentData {
  public:
  // structs such as EfeEcsCompInfo are a part of the C API, and C doesn't
  // have namespaces, so some manual name mangling is required here.
    ComponentData(std::span<EfeEcsCompInfo> infos);
    ComponentData(const ComponentData&) = delete;
    ComponentData& operator=(const ComponentData&) = delete;
    ComponentData(ComponentData&& other)
        : m_component_arrays{std::move(other.m_component_arrays)},
          m_component_index{std::move(other.m_component_index)},
          m_length{other.m_length}, m_capacity(other.m_capacity) {}
    ComponentData& operator=(ComponentData&& other) {

        std::swap(m_component_arrays, other.m_component_arrays);
        std::swap(m_component_index, other.m_component_index);
        std::swap(m_length, other.m_length);
        std::swap(m_capacity, other.m_capacity);

        return *this;
    }
    ~ComponentData();

    bool contains_component(EfeEcsCompID id);

    EfeEcsCompIter get_component(EfeEcsCompID id);
    void insert_entity(EfeEcsEntID id, std::span<EfeEcsComp> components);

    void reset();
    void clear();
    uint32_t length();

  private:
    void m_realloc(uint32_t count);
    // component arrays and ids share the same index
    std::vector<ComponentArray> m_component_arrays{};
    // Stores the indices of the component arrays for each component index. 
    // The reason I did it this way was because it gave the user an index into
    // the query to get the right component array for the component they wanted. 
    // It is now obsolete as I have since reworked how component arrays are
    // retrieved.
    std::unordered_map<EfeEcsCompID, uint32_t> m_component_index{};
    uint32_t m_length{};
    uint32_t m_capacity{};
};

using ArchetypeID = uint32_t;

// Could "ComponentData" just be renamed "Archetype" and given an ID? 
// Yes probably, but you are getting my code in its current, imperfect state,
// this is a devlog, not a tutorial after all :)
struct Archetype {
    ArchetypeID id;
    ComponentData component_data;
};
```

"That's weird.", you may ask, "Why don't you store a big array of structures that each have all of the components?".  We will get to that don't worry.

Now we can store the function pointer in one place as it will be the same for all members of an archetype. We do have one issue here though. With OOP you get polymorphism. This means that say for example, you have a move function, any object with the `IMove` interface will work with the function! This is something that we also want. If we want some sort of system that is able to operate on any moveable object, it would be a good idea if we could somehow query all of the archetypes for ones that contain **at least** a `MoveComponent`, allowing us to implement a kind of pseudo polymorphism. Doing this every frame is likely to hurt performance a lot, especially when we have a large number of systems. So instead we can keep a vector of system objects, which contain the C function pointer to the user defined system, as well as a set of all the archetypes it can operate on. 

```c++
struct SystemData {
    EfeEcsSystem system{nullptr};
    std::unordered_set<EfeEcsCompID> components{};
    std::unordered_set<ArchetypeID> archetypes{};
};
```

Now we can discuss the reason we decided to take the unconventional approach of storing separate arrays of components. Since our systems match to archetypes that have *at least* the components it needs, there are likely to be extra bits of data we don't need. If we had arrays of structs containing components, we would be wasting the CPU cache by loading data that is not actually needed! I am yet to actually profile this in practice. Perhaps in the future I will write on article on setting up [tracy](https://github.com/wolfpld/tracy){:target="blank"} in my engine.

Now we need some functionality to delete a particular entity. Since the entity ID may be needed in the future, I have chosen add functionality which allows a user to get the entity ID based on the position of the component, then delete the components based on the entity ID. It would be more efficient to directly delete the components on a particular index but this is currently how the ECS works.

```c++
class ComponentData {
  public:
    ...
    void delete_entity(EfeEcsEntID id);
    EfeEcsEntID get_entity(uint32_t component_index);

    void reset();
    void clear();
    uint32_t length();

  private:
    ...
    // Corresponding entity index for a component index
    std::vector<EfeEcsCEntID> m_reverse_map{};
    std::unordered_map<EfeEcsEntID, uint32_t> m_entity_index{};
    ...
};
...
```

```c
// game.c
void bounce_system(EfeEcsWorld* world, EfeEcsQuery* query, EfeEcsRunAPI* api) {
    // I am currently doing a large rewrite of how user scripts are handled, 
    // so this code will look somewhat different.
    EfeEcsCompIter iter_pos = api->get_component(query, 2);
    EfeEcsCompIter iter_health = api->get_component(query, 3);
    Position* pos_data = iter_pos.data;
    Health* health_data = iter_health.data;
    for (auto i = 0; i < iter_pos.length; ++i) {
        printf("Y: %f, Health: %d\n", pos_data[i].y, health_data[i].health);
    }
    if (iter_pos.length > 0) {
        EfeEcsEntID entity_id = api->get_entity(query, 0);
        api->delete_entity(world, entity_id);
    }
}
```

Finally we now have a user friendly way for the C code to drive logic within a game. We have Entities with IDs defined by their constituent Components, and the Systems which operate on Archetypes of entities....hang on!

### Conclusion
Of course there are going to be other solutions to my original conundrum, but ECS is already a well established design pattern and personally I find it very intuitive for what I am trying to do. At the time of writing, I do have frame time spikes particularly when deleting entities in my ECS, and my ECS also currently lacks the ability for entities to switch their components at run time. Something that is particularly useful as it can act as a kind of built in event system. There is a lot about my implementation that could be improved, but I will get round to it after my hot reload system is in working condition and the big refactor is complete.

If you discover any problems with this article, please raise an issue [here](https://github.com/XavierCS-dev/xaviercs-dev.github.io/issues){:target="blank"}.