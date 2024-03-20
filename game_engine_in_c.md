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

Next up: Scripts!

## Day 2: Scripts
Ok, so we need a way to write scripts that will do stuff with game objects. We should be able to attach an arbitrary script to an arbitrary game object. If we were writing this in C++, we would maybe have each script inherit from a base `script` class and then use virtual dispatch.

But we aren't doing that. So, lets do a hacky thing with macros and a python script. This keeps us in C world, but also should be faster than virtual dispatch - we can do everything with switch statements.

My basic idea is that I want to be able to write a script (for example in `my_script.c`) where I can just fill in certain callback functions. For example, something like
```c
void onInit(...){
    // do stuff
}

void onUpdate(...){
    // do stuff
}
```
and then in some way attach this script to a gameobject and have the `onUpdate` method automatically called every frame etc..

So, lets do a hacky macro thing. In `fluxScript.h`, I have:
```c
#include "gameobject.h"
#include <assert.h>

#define fluxConcat_(X,Y) X ## _ ## Y
#define fluxConcat(X,Y) fluxConcat_(X,Y)

#ifdef SCRIPT

#define fluxCallback static inline void

#define onUpdate fluxConcat(SCRIPT,fluxCallback_onUpdate)
#define afterUpdate fluxConcat(SCRIPT,fluxCallback_afterUpdate)
#define onInit fluxConcat(SCRIPT,fluxCallback_onInit)
#define onDestroy fluxConcat(SCRIPT,fluxCallback_onDestroy)
#define onDraw fluxConcat(SCRIPT,fluxCallback_onDraw)
#define onDraw2D fluxConcat(SCRIPT,fluxCallback_onDraw2D)
#define script_data struct fluxConcat(SCRIPT,fluxData)

#endif
```
The idea is that when I want to write a script, say `my_script.c`, I first `#define SCRIPT my_script` and then `#include "fluxScript.h"`. Then, I can write functions like:
```c
fluxCallback onUpdate(...){
    ...
}
```
which will be mangled accordingly. So, for example:
```c
#define SCRIPT test
#include "fluxScript.h"

script_data{
    int x;
};

fluxCallback onInit(fluxGameObject obj, script_data* data){
    data->x = 0;
}

fluxCallback onUpdate(fluxGameObject obj, script_data* data){
    data->x++;
}
```
Here, `script_data` is data associated with each instance of a script. We want a way to manage all of this automatically. We can do this using a hacky python preprocessing script.

The basic idea is that we discover all scripts (by making the user have all of them in the same place), then gather them all into a single file and then write wrappers to let us do a kind of virtual dispatch on a generic `struct script` thing.

In the python script we first need to specify all the possible callbacks. I'm doing this in a global variable because I am lazy, but this will probably change in the future as we want to add more callbacks.

Starting off
```python
import sys
import os
import re

SCRIPT_CALLBACKS = [
    "onUpdate", "afterUpdate", "onInit", "onDestroy", "onDraw", "onDraw2D"
]
```

Lets at least try to be pythonic and put the all of the script processing stuff in a `ScriptProcessor` class. We want to be able to specify all the paths etc rather than hardcoding so
```python
# processes all Flux scripts and creates `struct script`
class ScriptProcessor:
    def __init__(self, project_path : str = "project", scripts_folder : str = "scripts", engine_path : str = "engine", output_file : str = "GENERATED_SCRIPTS.h"):
    # path of the project (where project_path/scripts is where all the scripts are)
    self.project_path : str = project_path
    # name of the scripts folder
    self.scripts_folder : str = scripts_folder
    # path of the scripts folder
    self.scripts_path : str = os.path.join(self.project_path,self.scripts_folder)
    # path of the engine sources
    self.engine_path : str = engine_path
    # name of the output scripts file
    self.output_file : str = output_file
    # path of the output scripts file
    self.output_path : str = os.path.join(self.engine_path,output_file)
```

Now we need a way to find the scripts. We can do this with a disgusting one liner:
```python
# finds all `.c` files in `scripts_path`
def find_scripts(self) -> list[str]:
    return [os.path.join(self.scripts_path,i) for i in os.listdir(self.scripts_path) if i.split(".")[-1].strip() == "c"]
```

We might not want to implement all possible callbacks, so our script should figure out which callbacks we have implemented and auto generate the others. So
```python
# finds not implemented callbacks in a loaded script
# this just looks for instances of the callback name,
# SO, might get things wrong - be careful!
# should probably change this in the future, so
# TODO: fix me...
def find_not_implemented(self, raw : str) -> list[str]:
    return [i for i in SCRIPT_CALLBACKS if not i in raw]

# given a callback name, return an empty `implementation`
# i.e., a function that doesn't do anything
def get_empty_implementation(self,callback : str) -> str:
    return "fluxCallback {0}(fluxGameObject obj, script_data* data){{}}".format(callback)

# gets implementations for all not implemented callbacks
def get_extra_implementations(self, raw : str) -> str:
    return "\n".join([self.get_empty_implementation(i) for i in self.find_not_implemented(raw)])
```

We also want to be able to parse the name of this script (this is how we will identify the script in code in the engine etc.).
```python
# parses the script name from the raw text
# this is what SCRIPT is defined to at the start of the script,
# so we can do two simple `splits` and a strip to get it
def parse_script_name(self, raw : str) -> str:
    return raw.split("#define SCRIPT")[1].split("\n")[0].strip()
```

We then have enough to write a simple script processor
```python
# processes a single script file
def process_file(self, path : str) -> None:
    with open(path,"r") as f:
        raw = f.read()
    raw += "\n\n" + self.get_extra_implementations(raw) + "\n\n"
    self.output += raw
    self.script_names.append(self.parse_script_name(raw))

# process all scripts
def process_scripts(self) -> None:
    for i in self.scripts:
        self.process_file(i)
```

The init function then becomes
```python
# processes all Flux scripts and creates `struct script`
class ScriptProcessor:
    def __init__(self, project_path : str = "project", scripts_folder : str = "scripts", engine_path : str = "engine", output_file : str = "GENERATED_SCRIPTS.h"):
        # path of the project (where project_path/scripts is where all the scripts are)
        self.project_path : str = project_path
        # name of the scripts folder
        self.scripts_folder : str = scripts_folder
        # path of the scripts folder
        self.scripts_path : str = os.path.join(self.project_path,self.scripts_folder)
        # path of the engine sources
        self.engine_path : str = engine_path
        # name of the output scripts file
        self.output_file : str = output_file
        # path of the output scripts file
        self.output_path : str = os.path.join(self.engine_path,output_file)
        # list of all script files
        self.scripts : list[str] = self.find_scripts()
        # empty list of script names
        # when we process a script, we add the name of it to this list
        self.script_names : list[str] = []
        # the output generated file (initially empty)
        self.output : str = '#include "gameobject.h"\n#include "sceneallocator.h"\n' + "#ifdef FLUX_SCRIPTS_IMPLEMENTATION\n"
        # now we can process all the scripts
        self.process_scripts()
```

We want an enum to identify the scripts, so
```python
# generates `enum fluxScriptID`
def generate_enum_script_id(self):
    return "\nenum fluxScriptID{" + ",".join([get_script_enum_name(i) for i in self.script_names]) + "};\n"
```
`get_script_enum_name` is a function that mangles the script names into an enum. I am putting this in a separate file `pputils.py` because it will probably be useful in other preprocessing python scripts.

Ok, I'm getting bored. There are a bunch of other similar preprocessing functions that essentially do this:
* create a `fluxScript` data structure that is kind of a virtual class that contains all scripts.
* we create a function to allocate a `fluxScript` given a `enum fluxScriptID`.
* we then create functions for all the callbacks that operate on `fluxScript`s (so are ''generic'').

At the end, the output from the python script looks something like:
```c
struct test_fluxData;
struct test2_fluxData;
#include "gameobject.h"
#include "sceneallocator.h"
#ifdef FLUX_SCRIPTS_IMPLEMENTATION
#define SCRIPT test
#include "fluxScript.h"

script_data{
    int x;
};

fluxCallback onInit(fluxGameObject obj, script_data* data){
    data->x = 0;
}

fluxCallback onUpdate(fluxGameObject obj, script_data* data){
    data->x++;
}

fluxCallback afterUpdate(fluxGameObject obj, script_data* data){}
fluxCallback onDestroy(fluxGameObject obj, script_data* data){}
fluxCallback onDraw(fluxGameObject obj, script_data* data){}
fluxCallback onDraw2D(fluxGameObject obj, script_data* data){}

#define SCRIPT test2
#include "fluxScript.h"

script_data{
    float y;
};

fluxCallback onInit(fluxGameObject obj, script_data* data){
    data->y = 0;
}

fluxCallback onUpdate(fluxGameObject obj, script_data* data){}
fluxCallback afterUpdate(fluxGameObject obj, script_data* data){}
fluxCallback onDestroy(fluxGameObject obj, script_data* data){}
fluxCallback onDraw(fluxGameObject obj, script_data* data){}
fluxCallback onDraw2D(fluxGameObject obj, script_data* data){}
#endif

enum fluxScriptID{fluxScript_test,fluxScript_test2};

struct fluxScriptStruct;
typedef struct fluxScriptStruct* fluxScript;
#ifdef FLUX_SCRIPTS_IMPLEMENTATION
struct fluxScriptStruct{
    enum fluxScriptID id;
    union {
        void* raw;
        struct test_fluxData* test_fluxData;
        struct test2_fluxData* test2_fluxData;
    };
};
#endif

void fluxCallback_onUpdate(fluxGameObject obj, fluxScript script)
#ifdef FLUX_SCRIPTS_IMPLEMENTATION
{
    switch(script->id){
        case fluxScript_test:
            test_fluxCallback_onUpdate(obj,script->test_fluxData);
            break;
        case fluxScript_test2:
            test2_fluxCallback_onUpdate(obj,script->test2_fluxData);
            break;
        default:
            assert((1 == 0) && "something terrible happened at compile time!");
            break;
    }
}
#else
;
#endif

void fluxCallback_afterUpdate(fluxGameObject obj, fluxScript script)
#ifdef FLUX_SCRIPTS_IMPLEMENTATION
{
    switch(script->id){
        case fluxScript_test:
            test_fluxCallback_afterUpdate(obj,script->test_fluxData);
            break;
        case fluxScript_test2:
            test2_fluxCallback_afterUpdate(obj,script->test2_fluxData);
            break;
        default:
            assert((1 == 0) && "something terrible happened at compile time!");
            break;
    }
}
#else
;
#endif

void fluxCallback_onInit(fluxGameObject obj, fluxScript script)
#ifdef FLUX_SCRIPTS_IMPLEMENTATION
{
    switch(script->id){
        case fluxScript_test:
            test_fluxCallback_onInit(obj,script->test_fluxData);
            break;
        case fluxScript_test2:
            test2_fluxCallback_onInit(obj,script->test2_fluxData);
            break;
        default:
            assert((1 == 0) && "something terrible happened at compile time!");
            break;
    }
}
#else
;
#endif

void fluxCallback_onDestroy(fluxGameObject obj, fluxScript script)
#ifdef FLUX_SCRIPTS_IMPLEMENTATION
{
    switch(script->id){
        case fluxScript_test:
            test_fluxCallback_onDestroy(obj,script->test_fluxData);
            break;
        case fluxScript_test2:
            test2_fluxCallback_onDestroy(obj,script->test2_fluxData);
            break;
        default:
            assert((1 == 0) && "something terrible happened at compile time!");
            break;
    }
}
#else
;
#endif

void fluxCallback_onDraw(fluxGameObject obj, fluxScript script)
#ifdef FLUX_SCRIPTS_IMPLEMENTATION
{
    switch(script->id){
        case fluxScript_test:
            test_fluxCallback_onDraw(obj,script->test_fluxData);
            break;
        case fluxScript_test2:
            test2_fluxCallback_onDraw(obj,script->test2_fluxData);
            break;
        default:
            assert((1 == 0) && "something terrible happened at compile time!");
            break;
    }
}
#else
;
#endif

void fluxCallback_onDraw2D(fluxGameObject obj, fluxScript script)
#ifdef FLUX_SCRIPTS_IMPLEMENTATION
{
    switch(script->id){
        case fluxScript_test:
            test_fluxCallback_onDraw2D(obj,script->test_fluxData);
            break;
        case fluxScript_test2:
            test2_fluxCallback_onDraw2D(obj,script->test2_fluxData);
            break;
        default:
            assert((1 == 0) && "something terrible happened at compile time!");
            break;
    }
}
#else
;
#endif

fluxScript fluxAllocateScript(enum fluxScriptID id)
#ifdef FLUX_SCRIPTS_IMPLEMENTATION
{
    fluxScript out = (fluxScript)flux_scene_alloc(sizeof(struct fluxScriptStruct));
    out->id = id;
    size_t sz = 0;
    switch(id){
        case fluxScript_test:
            sz = sizeof(struct test_fluxData);
            break;
        case fluxScript_test2:
            sz = sizeof(struct test2_fluxData);
            break;
        default:
            assert((1 == 0) && "something terrible happened at build time!");
            break;
    }
    out->raw = flux_scene_alloc(sz);
    return out;
}
#else
;
#endif
```

which we then implement like
```c
#define FLUX_SCRIPTS_IMPLEMENTATION
#include "GENERATED_SCRIPTS.h"
```
in a separate compilation unit.