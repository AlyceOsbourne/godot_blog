---
share: true
layout: post
---

In my last post I mentioned how virtual methods could be used to simplify a save system, well, I wanted to further demonstrate the power of virtual method, and how they can be used to create a quick debug screen, much like what you find in games like Minecraft.
## Understanding Virtual Methods

A virtual method is a way of defining an abstract method, for systems to interface. One benefit to using virtual methods is that it doesn't rely on inheritance. Another is that its an opt in system, only classes that define the method need be included in the system.

## Creating the Debug Screen: A Centralized Information Hub

Our goal is to create a `DebugScreen` class that serves as an heads up display, collecting and displaying debug information from various nodes in the scene tree. Each node that wants to share its data can implement its own version of a `_debug` method. The `DebugScreen` will gather this information and display it in a structured format.

## The Base `DebugScreen` Class

Let’s start by creating a base class for our debug screen. This class will collect data from any node that implements the `_debug` method and display it in a formatted table.

```gdscript
class_name DebugScreen
extends RichTextLabel

@export_group("Debug Options")
@export var max_lines_per_column = 10
@export var refresh_rate: float = .2

var __refresh := 0.0

static func __format(value, depth: int = 2) -> String:
    var indent = "\t" * depth
    var type_color = ""

    if value is Array:
        type_color = "color=#3498db"  # Light blue for Arrays
        return indent + "[%s]Array[/color]\n" % type_color + "\n".join(value.map(func(x): return __format(x, depth + 1)))
    elif value is Dictionary:
        type_color = "color=#e67e22"  # Orange for Dictionaries
        return indent + "[%s]Dictionary[/color]\n" % type_color + "\n".join(value.keys().map(
            func(k):
                var v = value[k]
                return "%s%s: %s" % [indent, k, __format(v, depth + 1)]
        ))
    elif value is String:
        type_color = "color=#2ecc71"  # Green for Strings
        return indent + "[%s]%s[/color]" % [type_color, value]
    elif value is int:
        type_color = "color=#9b59b6"  # Purple for Integers
        return indent + "[%s]%s[/color]" % [type_color, str(value)]
    elif value is float:
        type_color = "color=#f1c40f"  # Yellow for Floats
        return indent + "[%s]%s[/color]" % [type_color, str(value)]
    elif value is bool:
        type_color = "color=#e74c3c"  # Red for Booleans
        return indent + "[%s]%s[/color]" % [type_color, str(value)]
    elif value == null:
        type_color = "color=#7f8c8d"  # Grey for null
        return indent + "[%s]null[/color]"
    else:
        type_color = "color=#95a5a6"  # Light grey for Other types
        return indent + "[%s]%s[/color]" % [type_color, str(value)]

func _input(event: InputEvent) -> void:
    if event.is_action_released("toggle_debug", true):
        visible = !visible

func _process(delta: float) -> void:
    __refresh += delta
    if __refresh > refresh_rate:
        __refresh -= refresh_rate
    else:
        return

    var data_target = {}
    get_tree().root.propagate_call("_debug", [data_target])
    var lines = data_target.keys().map(
        func(k: String):
            var v = data_target.get(k)
            return "%s: %s" % [k, __format(v)]
    )
    var total_lines = lines.size()
    var num_columns = ceil(float(total_lines) / max_lines_per_column)
    var total_cells = num_columns * max_lines_per_column

    while lines.size() < total_cells:
        lines.append("")

    var bbcode = "[table=%d]" % num_columns
    for row in range(max_lines_per_column):
        for col in range(num_columns):
            var index = row + col * max_lines_per_column
            var text = lines[index]
            bbcode += "[cell]%s[/cell]" % text
    bbcode += "[/table]"

    self.parse_bbcode.call_deferred(bbcode)
```

**What’s Happening Here?**

1. **Data Collection**: The `_process` function periodically collects data from all nodes that have implemented the `_debug` method. This is done using `propagate_call`, which calls `_debug` on each node and gathers their data into the `data_target` dictionary.

2. **Data Formatting**: The `__format` function prettifies the collected data, using colours and indentation to make it easier to read.

3. **Display**: The data is then displayed in a table format using the `RichTextLabel`’s BBCode feature.

## Implementing the `_debug` Method in Nodes

To contribute data to the debug screen, a node simply needs to implement its own version of the `_debug` method. Here’s an example:

```gdscript
extends Node

func _debug(data: Dictionary) -> void:
    data[name] = {
        "game_name": ProjectSettings.get_setting("application/config/name"),
        "version": ProjectSettings.get_setting("application/config/version"),
        "engine": Engine.get_version_info()["string"],
        "architecture": Engine.get_architecture_name(),
        "framerate": Engine.get_frames_per_second(),
        "window_size": get_viewport_rect().size,
    }
```

This method adds the node’s specific debug information to the `data` dictionary. The `DebugScreen` then gathers this data from all nodes and displays it.

![Pasted image 20240921202000](../Assets/Pasted%20image%2020240921202000.png)
## Why Use Virtual Methods?

Virtual methods make it easy to extend the functionality of your debug screen without modifying the `DebugScreen` class itself. This keeps your code clean and maintainable:

- **Modular Design**: Each node decides what data to provide, without the debug screen needing to know the details.
- **Scalability**: New nodes can easily be added without altering existing code.
- **Flexibility**: The base class provides a consistent structure, but the specifics are up to the nodes.
## Conclusion

Using virtual methods to create a custom debug screen in Godot gives you a powerful, flexible way to monitor your game’s state. By defining a `_debug` method that each node can implement, you create a modular and maintainable system that can grow with your project. This approach keeps everything organized, adaptable, and easy to manage as your game evolves.
