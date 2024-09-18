---
share: true
---

You've likely heard the phrase "composition is better than inheritance," but how do you apply this principle effectively in Godot? More importantly, how do you implement it without creating a tangled web of dependencies, often referred to as spaghetti code?

## The General Approach

In many tutorials, you might see a pattern where a node contains specific behaviour that is then called from a parent or related nodes. While this can increase flexibility, it also leads to a higher degree of coupling, making nodes tightly dependent on each other.

```gd
class_name Player
extends Node2D

@onready var health: Node = $Health

func apply_damage(amount: int) -> void:
    health.decrease_health(amount)

func heal(amount: int) -> void:
    health.increase_health(amount)

func get_health() -> int:
    return health.get_health()
```

```gd
class_name Health
extends Node

@export var max_health: int = 100
var current_health: int

func _ready():
    current_health = max_health

func decrease_health(amount: int) -> void:
    current_health -= amount
    current_health = clamp(current_health, 0, max_health)
    if current_health <= 0:
        emit_signal("health_depleted")

func increase_health(amount: int) -> void:
    current_health += amount
    current_health = clamp(current_health, 0, max_health)

func get_health() -> int:
    return current_health

signal health_depleted
```

While this approach separates behaviour, it requires the parent to be aware of the component, which almost negates the benefits of abstraction. In such cases, you might wonder whether it's simpler to just include this logic directly in the `Player` class and avoid the additional complexity. However, doing so would bring us back to the original problem of tightly coupled code. But what if we approached this differently?

## Inverting the Relationship

Instead of making the parent aware of the components, I propose that components should be self-contained and should expose ways to interact with them using signals. This way, the parent doesn't need to be aware of the capabilities provided by the components.

```gd
extends Node

# Target node, which could be the parent, owner, root, current scene, etc. In this case, we'll use the parent for the health component.

@export var target: Node :
    get:
        if target == null:
            target = get_parent()
        return target
        
# Properties to simplify reading and writing metadata to the target.

var max_health: int :
    get:
        return target.get_meta("max_health", 1)
    set(v):
        target.set_meta("max_health", v)        

var health: int :
    get:
        return target.get_meta("health", max_health)
    set(v):
        target.set_meta("health", clamp(v, 0, max_health))     

# Behaviour methods for the node. There can be multiple methods.

func modify_health(amount: int):
    health += amount
    if health == 0:
        target.emit_signal("health_depleted")

# Register custom signals to the target, binding them to our component's methods.

func _enter_tree() -> void:
    target.add_user_signal("modify_health", [{
        name = "amount",
        type = TYPE_INT
    }])
    target.add_user_signal("health_depleted")
    
    target.connect("modify_health", modify_health)
    
func _exit_tree() -> void:
    target.remove_user_signal("modify_health")
    target.remove_user_signal("health_depleted")
```

At first glance, this might seem more complex, but it fully leverages signals, eliminates sources of coupling, and avoids spaghetti code.

### The Target

Notice that we're not using `@onready` or `$NodePath`. Instead, we decouple the component from the scene tree structure, making it less sensitive to changes. This approach allows us to specify the target directly in the editor. If the scene tree structure changes, the target generally updates without requiring manual intervention. It also enables us to jump directly to the connected node from the inspector. The target acts as a mediator, allowing components and other game elements to interact without hard references.

> Note: While this approach usually works well, bugs and quirks in Godot may occasionally cause these references to decouple. This is rare, but it can happen.

### Entering and Exiting the Tree

This pattern's core lies in using the `_enter_tree` and `_exit_tree` methods to register everything before the `_ready` phase. This ensures that any dependencies can be set up in the `_ready` phase. Additionally, the component cleans itself up when exiting the tree, meaning the behaviour only exists when the node is attached, avoiding memory leaks and other issues.

### Registering the Signals

We use `add_user_signal` to bind custom signals to the target. These signals allow us to use the behaviour via the parent and let external components listen for events. Generally, you want to bind signals to callables on the component itself, as we do with `modify_health` in our health component. Sometimes, however, you might expose signals like `health_depleted` for other components to react to.

### Storing Component Data

It's generally a good idea to store the component's data on the target using metadata. This way, you don't need to define a script on the target just for variable storage, and other components can read the data without reaching into the target node's children. Getters and setters make this cleaner and simpler. Otherwise, you might end up with cumbersome patterns like:

```gd
target.set_meta("health", target.get_meta("health", target.get_meta("max_health", 1)) + amount)
```

This is wordy, hard to read, and becomes tedious when used multiple times in your script.

## Is This More Work?

Initially, yes. However, since we have common patterns here, we can create an abstraction and make strategic use of inheritance to enhance our composition. Let's make a few assumptions about our behaviour:

We likely want:
- A way to create custom signals.
- A way to bind callables to these signals.
- A way to add the target to groups.

And we want the reverse when exiting the tree.

Let's create a class that handles all of this, allowing us to focus on behaviour instead.

```gd
class_name Behaviour
extends Node

@export var target: Node :
    get:
        if target == null:
            target = get_parent()
        return target

func _enter_tree() -> void:
    if has_method("signals_to_create"):
        var sigs: Dictionary = call("signals_to_create")
        for key in sigs.keys():
            target.add_user_signal(key, sigs[key])

    if has_method("signals_to_bind"):
        var sigs: Dictionary = call("signals_to_bind")
        for k in sigs:
            target.connect(k, self[sigs[k]])
    
    if has_method("groups_to_add"):
        for group: StringName in call("groups_to_add"):
            target.add_to_group(group)
    
func _exit_tree() -> void:
    if has_method("groups_to_add"):
        for group: StringName in call("groups_to_add"):
            target.remove_from_group(group)
            
    if has_method("signals_to_bind"):
        var sigs: Dictionary = call("signals_to_bind")
        for k in sigs:
            target.disconnect(k, self[sigs[k]])
            
    if has_method("signals_to_create"):
        var sigs: Dictionary = call("signals_to_create")
        for key in sigs.keys():
            target.remove_user_signal(key)
```

This class handles the creation and binding of signals and adding the target to groups, allowing subclasses to optionally create methods that define the signals and bindings.

Creating new behaviours becomes trivial. Our health component would now look something like this:

```gd
extends Behaviour

# Accessing our metadata

var max_health: int :
    get:
        return target.get_meta("max_health", 1)
    set(v):
        target.set_meta("max_health", v)        

var health: int :
    get:
        return target.get_meta("health", max_health)
    set(v):
        target.set_meta("health", clamp(v, 0, max_health))   

# Optional methods to create and bind signals

func signals_to_create():
    return {
        modify_health = [
            {name = "amount", type = TYPE_INT}
        ],
        health_depleted = []
    }
    
func signals_to_bind():
    return {
        modify_health = "mod_health"
    }

# Behaviour implementation

func mod_health(amount):
    health += amount
    if health == 0:
        target.emit_signal("health_depleted")  
```

Now, it's as simple as calling `emit_signal("modify_health", -1)` on the target from anywhere in your code to activate the behaviour.

> Note
> To avoid errors, you should check to see if the signal exists when using this pattern, using `node.has_signal("modify_health")`.

### Final Thoughts

In this post, we've explored an alternative approach to applying the principle of composition over inheritance in Godot. By inverting the relationship between components and their parent nodes, we can achieve greater flexibility and decoupling in our code. Instead of relying on the parent to manage its components directly, we allow components to manage themselves, exposing their behaviour by leveraging Godot's signal system.

This method not only reduces the risk of creating tightly coupled, brittle systems but also allows for more modular and reusable code. By decoupling the components from the scene tree structure and using metadata to store data, we create a more robust system that is less sensitive to changes and easier to maintain. The added abstraction layer might seem like more work initially, but the benefits become apparent as your project grows in complexity. With this approach, you gain a more scalable, maintainable, and adaptable codebase.

Ultimately, this pattern of composition fosters a cleaner, more organised architecture, which can significantly reduce the likelihood of running into the dreaded spaghetti code. While it may deviate from more traditional practices, the long-term advantages make it a compelling choice for Godot developers seeking to build flexible and maintainable systems.