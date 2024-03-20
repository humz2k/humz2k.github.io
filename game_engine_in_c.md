# Writing a Game Engine in C

This is a kind of unstructured log of the process of writing a game engine in C, using [raylib](https://www.raylib.com/) as the backend. As a disclaimer, I have no idea what I am doing or what best practices are.

## Day 1: Allocator
The first thing we need to do is figure out how the game engine is going to be structured.

My initial thoughts are:
* We divide the game into 'scenes', which are loaded and unloaded according to some logic (like when a user presses the 'level 0' button, we load the level 0 scene).
* Each scene contains a number of 'prefabs'. When we load a scene, we 'instantiate' these prefabs into game objects, which are the entities in the scene. This is also nice because then we can instantiate more prefabs while the scene is going.
* Each prefab contains some information - the initial transform of the entity, whether it has a model, what the path to the model is etc.. We use this information to construct an independent GameObject.
* Each prefab (and thus GameObject) has some 'scripts' associated with it. These scripts function like callbacks (? terminology confusing idk if I am using that right), in the sense that each script can implement, for example, an `onUpdate` function that is called every frame, or an `onInit` function that is called on GameObject creation.

This is a nice rough idea of how we want things to be structured. One thing that would be nice though is to be able to allocate memory/load models etc. that are automatically freed/unloaded when the current scene is closed. We are going to need to load models/allocate memory for GameObjects whenever a scene is loaded, and we only want to free/unload them when the scene is closed.

So the first bit of code we can write is a simple allocator that keeps track of memory allocations/loaded models.

This is the interface I went with (in `sceneallocator.h`):
```c
#ifndef _FLUX_SCENEALLOCATOR_H_
#define _FLUX_SCENEALLOCATOR_H_

#include <stdlib.h>
#include "raylib.h"

// initializes the scene allocator for the current scene (so should be called every scene load)
void flux_init_scene_allocator(void);

// closes the scene allocator for the current scene (so should be called every scene close)
void flux_close_scene_allocator(void);

// allocates some heap space that will be cleared on scene close
void* flux_scene_alloc(size_t sz);

// loads a model that will be cleared on scene close
Model flux_scene_load_model(const char* path);

#endif
```

So, we would call `flux_init_scene_allocator` whenever we load a scene, and `flux_close_scene_allocator` whenever we close a scene. Then, we can use `flux_scene_alloc` and `flux_scene_load_model` to allocate memory/load models without having to worry about unloading/freeing (because this is done in `flux_close_scene_allocator`).

The implementation is fairly simple. We store allocations and models in contiguous arrays. So, in `sceneallocator.c`:
```c
#include <stdlib.h>
#include <stdio.h>
#include <assert.h>
#include "raylib.h"
#include "raymath.h"
#include "gameobject.h"
#include "sceneallocator.h"

// stores all the allocations of the current scene in resizable array
static void** allocations = NULL;
// number of active allocations
static int n_allocations = 0;
// size of the allocations array (n_allocations may be less than allocations_size)
static int allocations_size = 0;

// similarly for models
static Model* models = NULL;
static int n_models = 0;
static int models_size = 0;
```
This lets us write hacky hardcoded grow-able arrays for allocations and models. It would probably be smart to write a general resizable array at some point, because we will want to be able to load shaders/textures etc in the same way.

First we implement `flux_init_scene_allocator`. I will probably want to clean this up in the future, but this works for now:
```c
// initializes the scene allocator for the current scene (so should be called every scene load)
void flux_init_scene_allocator(void){
    // allocations and models must be NULL
    // if not, then something bad happened
    assert((NULL == allocations) && (NULL == models) && "was fluxCloseSceneAllocator called on scene close?");
    // set number of allocations and models to 0
    n_allocations = 0;
    n_models = 0;
    // and the initial size of the allocations and models arrays to be 10
    allocations_size = 10;
    models_size = 10;
    // malloc accordingly
    assert(allocations = (void**)malloc(sizeof(void*) * allocations_size));
    assert(models = (void**)malloc(sizeof(Model) * models_size));
}
```

For `flux_close_scene_allocator` we want to free all the allocations and models by looping through the active allocations/models:
```c
// closes the scene allocator for the current scene (so should be called every scene close)
void flux_close_scene_allocator(void){
    assert((n_allocations >= 0) && "n_allocations was less than 0???");
    assert((n_allocations < allocations_size) && "n_allocations is less than allocations_size!");
    assert((n_models >= 0) && "n_models was less than 0???");
    assert((n_models < models_size) && "n_models is less than models_size!");
    // loop through active allocations and free the memory
    for (int i = 0; i < n_allocations; i++){
        free(allocations[i]);
    }
    // finally free the allocations array and set to NULL
    free(allocations); allocations = NULL;
    // loop through active models and free the Model
    for (int i = 0; i < n_models; i++){
        UnloadModel(models[i]);
    }
    free(models); models = NULL;
}
```

Finally, for the allocation and model loading methods, we do the hacky hardcoded grow-able array stuff:
```c
// allocates some heap space that will be cleared on scene close
void* flux_scene_alloc(size_t sz){
    assert((n_allocations >= 0) && "n_allocations was less than 0???");
    // if we are out of space, we must grow the array
    if (n_allocations >= allocations_size){
        assert((allocations_size > 0) && "allocations_size should never be less than 1!");
        // double the size of the allocations array
        allocations_size *= 2;
        // realloc accordingly
        assert(allocations = realloc(allocations,sizeof(void*) * allocations_size));
        assert((n_allocations < allocations_size) && "n_allocations was still bigger than allocations_size after resize!");
    }
    // get the pointer to be returned
    void* out; assert(out = (void*)malloc(sz));
    // store this pointer in the correct place in allocations
    allocations[n_allocations] = out;
    // and then increment n_allocations
    n_allocations++;
    // finally return the malloced data
    return out;
}

Model flux_scene_load_model(const char* path){
    assert((n_models >= 0) && "n_models was less than 0???");
    // if we are out of space, we must grow the array
    if (n_models >= models_size){
        assert((models_size > 0) && "models_size should never be less than 1!");
        // double the size of the models array
        models_size *= 2;
        // realloc accordingly
        assert(models = realloc(models,sizeof(Model) * models_size));
        assert((n_models < models_size) && "n_models was still bigger than models_size after resize!");
    }
    // get the model to be returned
    Model out = LoadModel(path);
    // store this model in the correct place in models
    models[n_models] = out;
    // and then increment n_models
    n_models++;
    // finally return the loaded model
    return out;
}
```

Next up: Prefabs and GameObjects!