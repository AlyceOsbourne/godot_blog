---
share: true
layout: post
date created: Saturday, September 21st 2024, 1:24:39 pm
date modified: Thursday, September 26th 2024, 5:20:05 pm
dg-publish: true
---

When it comes to saving data in Godot, there are many opinions on the best ways to handle save data. Some developers swear by autoloads, others lean on signals, and even Godot’s official documentation offers its own ideas. In this guide, I’ll introduce a flexible method that ensures each object takes responsibility for its own data. This modular approach keeps your project organized and easy to maintain as it grows.

## Serialization and Deserialization Basics

Every save system revolves around two fundamental processes:

- **Serialization**: Converting game data—variables, objects, states—into a format like JSON or binary that can be stored on disk.
- **Deserialization**: Reversing that process to restore the game’s state from the saved data.

The key challenge is managing how to gather and distribute the data you want to save, and Godot offers several ways to handle this.

## Common Approaches: Autoloads, Signals, and Groups

### Autoloads

Autoloads are popular because they offer a single point of control over your data.

**Pros:**
- **Centralized Management**: All save/load logic in one place.
- **Global Access**: Any script can access the data without specific references.

**Cons:**
- **Cluttered Code**: As your game grows, the autoload script can become overloaded.
- **Scalability Issues**: Managing multiple objects with unique save requirements can be cumbersome.
- **Tight Coupling**: Objects become dependent on a central save system, reducing flexibility.

### Signals

Using signals makes the system more dynamic by letting objects respond to save and load events.

**How It Works:**
A save manager emits signals when it needs data, and connected nodes respond with their own save or load logic.

```gdscript
signal save_data_requested(data: Dictionary)
signal load_data_requested(data: Dictionary)

func request_save_data() -> Dictionary:
    var data = {}
    emit_signal("save_data_requested", data)
    return data

func request_load_data(data: Dictionary) -> void:
    emit_signal("load_data_requested", data)
```

Nodes connect to these signals and handle their own data.

```gdscript
extends Node

@export var player_data := PlayerData.new()
@export var world_data := {}

func _ready():
    SaveManager.connect("save_data_requested", _on_save_data_requested)
    SaveManager.connect("load_data_requested", _on_load_data_requested)

func _on_save_data_requested(data: Dictionary) -> void:
    data[name] = { "player_data": inst_to_dict(player_data), "world_data": world_data }

func _on_load_data_requested(data: Dictionary) -> void:
    var node_data = data[name]
    player_data = dict_to_inst(node_data["player_data"])
    world_data = node_data["world_data"]
```

**Pros:**
- **Loose Coupling**: Objects don’t need direct references to each other, reducing dependencies.
- **Dynamic Participation**: Any object can join the save/load process without central registration.
- **Flexible Events**: Easily adjust save/load behavior by connecting or disconnecting signals.

**Cons:**
- **Manual Connections**: Each object must manually connect to signals, which can be tedious in large projects.
- **Order Sensitivity**: The sequence in which objects connect to signals can impact the process, causing potential bugs.
- **Signal Overload**: Too many signals can create overhead, especially with many nodes responding simultaneously.

### Groups

Groups offer a way to categorize nodes that need to save or load data.

**How It Works:**
You assign nodes to a group, like `saveable`, and have your save manager iterate through the group to handle saving and loading.

```gdscript
extends Node

var player_data: PlayerData = PlayerData.new()
var world_data: Dictionary = {}

func _ready():
    add_to_group("saveable")

func _save(data: Dictionary) -> void:
    data[name] = {
        "player_data": inst_to_dict(player_data),
        "world_data": world_data
    }

func _load(data: Dictionary) -> void:
    var node_data = data[name]
    player_data = dict_to_inst(node_data["player_data"])
    world_data = node_data["world_data"]
```

The save manager then iterates through the group:

```gdscript
extends Node

func save_all_data() -> Dictionary:
    var data = {}
    for node in get_tree().get_nodes_in_group("saveable"):
        var node_data = {}
        assert(node.has_method("_save"), "%s is missing _save method" % name)
        node.call("_save", node_data)
        data[node.name] = node_data
    return data

func load_all_data(data: Dictionary) -> void:
    for node in get_tree().get_nodes_in_group("saveable"):
        if node.name in data:
            assert(node.has_method("_load"), "%s is missing _load method" % name)
            node.call("_load", data[node.name])
```

**Pros:**
- **Centralized Control**: Call functions on all group members with a single command.
- **Dynamic Membership**: Nodes can be added or removed from groups on the fly.

**Cons:**
- **Manual Management**: You have to ensure each node is correctly added or removed from the group, and check to see if the appropriate methods have been implemented.
- **Scalability**: Large groups can be challenging to manage and may impact performance.

## Enter Virtual Methods

If you’ve written any GDScript, you’re probably familiar with methods like `_process`, `_to_string`, and `_get_property_list`. These are functions you can override to add specific behavior to an object. You can use a similar approach for custom save/load logic.

### Implementing Virtual Methods

By implementing custom virtual methods, you can create a decentralized save/load system that’s both flexible and scalable.

```gdscript
class_name SaveHandler

static func write(data: Dictionary, path: String) -> void:
    var file = FileAccess.open(path, FileAccess.ModeFlags.WRITE_READ)
    file.store_string(JSON.stringify(data))
    file.close()

static func read(path: String) -> Dictionary:
    return JSON.parse_string(FileAccess.get_file_as_string(path))

static func save_data(path: String) -> void:
    var data = {}
    Engine.get_main_loop().root.propagate_call("_save", [data])
    write(data, path)

static func load_data(path: String) -> void:
    Engine.get_main_loop().root.propagate_call("_load", [read(path)])
```

Using `propagate_call`, you trigger a function call on all nodes in the scene tree that have implemented the `_save` or `_load` methods. Nodes that don’t implement these methods are simply ignored.

**Pros:**
- **No Manual Setup**: Nodes don’t need to be added to groups or connected to signals. They just implement `_save` and `_load` to opt-in.
- **Encapsulation**: Each node is responsible for its own save/load logic, keeping your code neat and maintainable.
- **Flexible Logic**: You can customize save/load behaviour for each node, supporting diverse data handling needs.

Here’s an example of a node implementing these methods:

```gdscript
extends Node

var player_data: PlayerData = PlayerData.new()
var world_data: Dictionary = {}

func _save(data: Dictionary) -> void:
    data[name] = {
        "player_data": inst_to_dict(player_data),
        "world_data": world_data
    }

func _load(data: Dictionary) -> void:
    var node_data = data[name]
    player_data = dict_to_inst(node_data["player_data"])
    world_data = node_data["world_data"]

```

As you can see, this approach is almost identical to the groups method, sans `_ready` method.

### Why I Love This Method
- **Modularity**: Each node handles its own save/load logic, making changes to one node easy without affecting others.
- **Scalability**: As your game grows, each node scales independently, accommodating unique save needs without bottlenecks.
- **Encapsulation**: Nodes manage their own data, reducing errors from a shared global state.
- **Flexibility**: Nodes can use different serialization methods or formats, allowing for tailored solutions.

With this approach, you get a highly modular, flexible save system that adapts as your game evolves.

## Wrapping up

There is no one right way to approach how you manage save data, with each technique having its own place. What is right for you will depend on the structure and style of your project, I hope that by showing you various methods you find a way that works with your own machinations.
