---
share: true
---

Many of the best games have soared in popularity thanks to their extensive mod support. Players love being able to add their personal touch to a game’s world, whether that means introducing quality-of-life features or completely overhauling the gameplay. Today, we’re going to dive into how you can load mods into your games using Godot and, more importantly, how you can support modding from the ground up as you code.

## Loading Mod Files in Godot

Godot makes loading mod files relatively easy with the help of `DirAccess` and `FileAccess`. These allow you to traverse folders and read files. Below is a basic script that shows how to load mods in your Godot project.

```swift
@tool
class_name ModLoader
extends Object

static var root: Node:
    get: return Engine.get_main_loop().root

static func load_mods(mods_folder: String = "user://mods", mod_file_name: String = "mod.gd"):
    var mods = (
        (DirAccess.get_directories_at(mods_folder) as Array) # get the folders in the mods folder
        .map(func(mod_folder): return "%s/%s/%s" % [mods_folder, mod_folder, mod_file_name]) # get the path to the `mod.gd` file
        .filter(func(mod_file): return FileAccess.file_exists(mod_file)) # make sure the mod file exists
        .map(load) # load the resource
        .map(func(x): return x.new(root)) # Instantiate with a reference to the root
    )
    mods.sort_custom(  # we sort the mods based on their dependencies, to ensure the correct load order
        sort_mods
    )
    for mod: Plugin in mods:
        var m = (mod.get_script().resource_path as String).rsplit("/", true, 1) # get the mods name
        mod.name = m[0].rsplit("/", true, 1)[1] # Set the node name
        mod.folder = m[0] + "/" # set the folder name
        mod.owner = root # set the owner to be root
        mod.add_to_group("mods") # add to the mods group
        root.add_child.call_deferred(mod, true, Node.InternalMode.INTERNAL_MODE_FRONT) # and finally add to the root as a internal node

static func unload_mods():
    # free all of the mod nodes in the mods group
    Engine.get_main_loop().call_group("mods", "queue_free")

static func sort_mods(x, y) -> int:
    # Sorts mods based on their dependencies, does very basic detection of circular dependencies
    var x_requires = x.get("REQUIRES") if "REQUIRES" in x else []
    var y_requires = y.get("REQUIRES") if "REQUIRES" in y else []

    assert(not y.MODID in x_requires and x.MODID in y_requires, "Circular Dependency Detected")

    if y.MODID in x_requires:
        return 1

    if x.MODID in y_requires:
        return -1

    return 0

```

This code might look a little dense at first glance, but let’s break it down:

1. **Gathering Directories**: The script starts by fetching all the directories within the given mods folder that contain a `mod.gd` file.
2. **Loading the Mods**: It then loads the scripts, instantiates each mod as a plugin, and adds it to the root node as an internal node.
3. **Sorting Mods**: The mods are sorted based on their dependencies using the `sort_mods` function, ensuring the correct load order.
4. **Unloading Mods**: The `unload_mods` function allows you to free all mod nodes at once by calling `queue_free()` on each mod in the "mods" group.

## Creating a Plugin Class

To manage each mod, we create a `Plugin` class that extends `Node`. This class handles things like loading configuration files and managing the mod’s unique ID (MODID), which is crucial for sorting mods correctly.

```swift
class_name Plugin
extends Node

var folder: String
var config: ConfigFile = ConfigFile.new()
var root: Node

func load_config():
    if has_method("default_config"):
        var def = call("default_config")
        for sec in def:
            for key in def[sec]:
                config.set_value(sec, key, def[key])
    var err = config.load("".join([folder, "config.ini"]))
    if err != OK:
        return

func save_config():
    if len(config.get_sections()) == 0:
        return
    config.save("".join([folder, "config.ini"]))

func _to_string() -> String:
    return "Mod: %s" % get("MODID")

func _enter_tree() -> void:
    load_config()

func _exit_tree() -> void:
    save_config()

func _validate_property(property: Dictionary) -> void:
    if property.name == "MODID" and get("MODID") != null:
        property.usage |= PROPERTY_USAGE_READ_ONLY

func _init(root: Node) -> void:
    assert("MODID" in self, "File %s did not define a MODID, cannot finish loading.")
    self.root = root
```

This `Plugin` class is key for managing each mod’s lifecycle:

- **Config Management**: The `load_config()` and `save_config()` methods allow each mod to manage its own configuration file. The `default_config` method, if present, initializes the config with default values.
- **Lifecycle Hooks**: `_enter_tree()` and `_exit_tree()` handle loading and saving the mod’s configuration when it’s added to or removed from the scene tree.
- **MODID Validation**: The MODID is crucial as it identifies the mod and helps in sorting and dependency management. We make sure it’s set and read-only to avoid accidental changes.

## Supporting Modding in Your Code

To truly support modding in your game, a little foresight goes a long way. Here are some key strategies:

- **Registries**: Create registries for game objects that mods can easily add to. This allows mods to introduce new items, enemies, or other elements without directly modifying the base game code.
- **Signals and Groups**: Use signals for events and assign nodes to groups. This allows mods to easily interact with your game’s systems, such as adding custom behaviours when specific events occur.
- **Method Overloading**: Allow mods to overload methods by separating the logic of various classes and methods. This makes your codebase more flexible and allows mods to replace or extend functionality as needed.
- **Composition Over Inheritance**: Favour composition to improve the modularity of your code. This makes it easier for mods to mix and match components, replacing or enhancing parts of your game without breaking existing functionality.

## How Can Mods Modify Your Game?

In Godot, the sky’s the limit when it comes to modding. By allowing custom scripts to be loaded, mods can:

- **Add New Objects**: Mods can add new objects to your game’s registries or directly to the scene tree, introducing new gameplay elements.
- **Modify Existing Classes**: Mods can inject new signals, methods, or attributes into existing classes, altering how your game behaves.
- **Extend and Replace Classes**: Mods can create subclasses or replace existing classes entirely, introducing new mechanics or altering existing ones.

The extent of modding is really up to you. How much effort you want to invest in supporting mods will determine the level of customization players can achieve.

## Should You Support Mods?

While nearly every game can be modded to some degree, you should weigh the effort against the potential benefits. For sandbox games, supporting modding can significantly extend the game’s lifespan by allowing the community to add new gameplay mechanics and loops. On the other hand, narrative-driven games might not benefit as much from mod support, as mods could potentially disrupt the carefully crafted story experience.

## Final Thoughts

Supporting mods in your game can be a powerful way to engage your community and extend the life of your project. Godot’s flexible and open-ended design makes it relatively easy to implement modding support, allowing players to leave their own mark on your game world. Whether you’re creating a sandbox game or a more linear experience, consider the potential impact of modding and how it could enhance—or detract from—your vision. Ultimately, modding should be a feature that complements your game, not one that overshadows its core experience.

