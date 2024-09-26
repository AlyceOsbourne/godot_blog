---
share: true
layout: post
date created: Friday, August 23rd 2024, 2:29:45 am
date modified: Thursday, September 26th 2024, 5:19:33 pm
dg-publish: true
---

Resources in Godot are an incredibly useful feature that can simplify your project’s data management. However, they can seem a bit mysterious at first. This guide will clarify what Resources are, why they’re important, and how you can use them effectively. I’ll also share a handy technique for loading Resources from a CSV file, which can make managing large amounts of data much easier.

## What is a Resource?

A Resource in Godot is essentially a container for data that can be reused throughout your project. This data could be anything—textures, scripts, audio files, or even custom data structures. Resources are designed to be saved to disk and shared across various parts of your project, making them invaluable when you need consistent data access.

Resources differ from Nodes, which are more about behaviour and scene management. While Nodes manage the logic and interaction within your game, Resources focus on storing the data that these Nodes might need. Think of Nodes as the active elements in your project, and Resources as the data they rely on.

### Why Use Resources?

1. **Reusability**: Resources allow you to reuse data across multiple scenes and nodes. For example, if you create a resource that defines a character's stats, you can easily apply it to multiple characters without duplicating the data.

2. **Efficiency**: By referencing Resources rather than duplicating data, you save memory and ensure consistency. Changes made to a Resource will automatically apply wherever that Resource is used.

3. **Persistence**: Resources can be saved and loaded from disk, making them ideal for data that needs to be retained between game sessions, like player stats or settings.

4. **Flexibility**: Resources can be extended with custom scripts, allowing you to create complex and reusable data structures that can be easily managed across your project.

### When Should You Use Resources?

You should consider using Resources when you need to:

- When you wish to manipulate the variables in the Inspector.
- Share data consistently across multiple scenes or nodes.
- Maintain data that should persist, such as game settings or player progress.
- Reuse data in a way that avoids duplication, ensuring consistency across your project.
- Manage large datasets with similar structures, like levels, items, or characters.

Resources are perfect for any data that needs to be consistent and accessible throughout your project.

### Creating and Using Resources

Creating a Resource in Godot is straightforward. You define a Resource by extending the `Resource` class. For example, here’s how you might create a `CharacterStats` resource to hold a character’s attributes:

```gdscript
class_name CharacterStats
extends Resource

@export var health: int
@export var mana: int
@export var strength: int
@export var agility: int
```

With this `CharacterStats` resource, you can easily apply and modify stats across different characters in your project, ensuring consistency without duplicating data.

### Saving Resources

Resources can be saved, both in the inspector, and in code, much like how scenes can be saved.

This enables you to to reuse them between nodes and scenes. A perfect example of when its desirable to save a resource is when using nodes like the `TileMapLayer`, and wish to reuse the `TileSet` between scenes.

Saving in code can be done in a few ways. Firstly, you have the `ResourceSaver`, which is a singleton designed to write `Resource` objects to file.

```
ResourceSaver.save(obj, "path://to/save.res")
load("path://to/save.res")
```

You also can use `Marshalls` to convert to `base64`.

```
Marshalls.variant_to_base64(obj)
Marshalls.base64_to_variant(b64_string)
```

And you can use JSON.

```
JSON.stringify(inst_to_dict(obj))
dict_to_inst(JSON.parse_string(json_str))
```

### Loading Resources from CSV

Managing large sets of data manually can be tedious. Fortunately, you can load Resources directly from a CSV file, making the process much more efficient.

Here’s a script that demonstrates how to do this:

```gdscript
class_name ResourceManager
extends Resource

@export var types: Array[GDScript] = []

var loaded: Dictionary:
    get:
        if not loaded:
            loaded = load_resources()
        return loaded

func load_resource(res):
    var resources = []
    var file = FileAccess.open(res.resource_path.replace(".gd", ".csv"), FileAccess.READ_WRITE)
    var keys = file.get_csv_line()
    var col_len = len(keys)
    while true:
        var row = file.get_csv_line()
        if len(row) < col_len:
            break
        resources.append(res.new.callv(row))
    return resources

func load_resources():
    var r = {}
    for type in types:
        r[type.resource_path.rsplit(r"/", false, 1)[1].rstrip(".gd")] = load_resource(type)
    return r
```

### How This Works

- **`types` Array**: This array holds the different types of Resources you want to load. Each type corresponds to a GDScript file that defines a Resource class.

- **Loading Data**: The `load_resources` function iterates through each type, finds the corresponding CSV file, and loads the data. The CSV file should have the same name as the script file, with each row representing a new instance of the Resource.

- **Creating Instances**: For each row in the CSV, a new instance of the Resource type is created and filled with the data from the CSV columns. This is particularly useful for projects with a lot of data that follows a similar structure.

### Example Usage

Imagine you have a `WeaponStats.gd` script that defines attributes like damage, range, and cooldown. By creating a `WeaponStats.csv` file, you can load an entire arsenal of weapons into your game without needing to manually create each one in the editor. This method makes it easy to update your data by simply editing the CSV file, usually using some table editing software, such as Excel.

> Note, if you choose to use this method, do note, they are singleton objects, and as such they shouldn't be edited. It is possible to use the Prototype pattern here though, and duplicate them where needed. This is quite useful when you have say, Weapons, and you want the player to be able to enchant or modify them.

## Final Thoughts

Resources in Godot are a powerful tool for keeping your project organised and efficient. By understanding how and when to use them, you can ensure that your data is consistent and easily accessible across your project. And with tricks like loading Resources from a CSV file, you can streamline your workflow, making it easier to manage large datasets.
