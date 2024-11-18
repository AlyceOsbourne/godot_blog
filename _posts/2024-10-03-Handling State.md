---
date created: Thursday, September 19th 2024, 7:15:49 am
date modified: Thursday, October 3rd 2024, 5:21:36 pm
share: true
layout: post
---

State machines are a fundamental concept in game development, enabling characters, enemies, and game systems to transition smoothly between different behaviours. In Godot, there are multiple ways to implement state machines, each with its own set of advantages and trade-offs. This post explores various state machine implementations in Godot, from simple `enum` and `match` statements to more complex delegate-based systems and class-based approaches. Each method has its own strengths and weaknesses, depending on your project’s requirements.

## Why Use State Machines?

In game development, objects often need to behave differently under various conditions. A player character might be idle, running, jumping, or attacking. An enemy might patrol, chase, or attack. Managing these behaviors can become complex, especially as the number of states and transitions grows.

State machines provide a structured way to manage these behaviors by:

- **Defining distinct states**: Clearly separating different behaviors.
- **Handling transitions**: Managing how and when to move from one state to another.
- **Improving maintainability**: Making the code easier to read, debug, and extend.

## Simple State Machines with Enums and Match Statements

The most straightforward way to implement a state machine in Godot is by using an `enum` to define states and a `match` statement to handle state transitions.

### Implementation

```gdscript
extends KinematicBody2D

enum State {
    IDLE,
    RUNNING,
    JUMPING,
    ATTACKING
}

var current_state: State = State.IDLE

func _physics_process(delta: float) -> void:
    match current_state:
        State.IDLE:
            handle_idle_state()
        State.RUNNING:
            handle_running_state()
        State.JUMPING:
            handle_jumping_state()
        State.ATTACKING:
            handle_attacking_state()

func handle_idle_state() -> void:
    if Input.is_action_pressed("ui_right"):
        current_state = State.RUNNING

func handle_running_state() -> void:
    if Input.is_action_just_pressed("ui_up"):
        current_state = State.JUMPING
    elif Input.is_action_just_pressed("attack"):
        current_state = State.ATTACKING
```

### Advantages

1. **Simplicity**: Easy to understand and implement, especially for small projects or when starting out.
2. **Clarity**: All state logic is centralized, making it straightforward to see how states transition.
3. **Performance**: Minimal overhead, as `match` statements are efficient.

### Disadvantages

1. **Scalability Issues**: As the number of states grows, the `match` statement can become unwieldy.
2. **Maintenance Difficulty**: Adding or modifying states requires changes in multiple places.
3. **Tight Coupling**: State logic is tightly coupled with the state machine, making it harder to reuse code.

---

## Delegate-Based State Machines

For more complex projects, a delegate-based approach using function references (or "delegates") can offer greater flexibility and modularity.

### Implementation

```gdscript
extends KinematicBody2D

var state_functions: Dictionary = {
    "idle": handle_idle_state,
    "running": handle_running_state,
    "jumping": handle_jumping_state,
    "attacking": handle_attacking_state
}

var current_state: String = "idle"
var state_function: FuncRef

func _ready() -> void:
    state_function = state_functions[current_state]

func _physics_process(delta: float) -> void:
    state_function.call_func(delta)

func handle_idle_state(delta: float) -> void:
    if Input.is_action_pressed("ui_right"):
        change_state("running")

func handle_running_state(delta: float) -> void:
    if Input.is_action_just_pressed("ui_up"):
        change_state("jumping")
    elif Input.is_action_just_pressed("attack"):
        change_state("attacking")

func change_state(new_state: String) -> void:
    current_state = new_state
    state_function = state_functions[current_state]
```

### Advantages

1. **Modularity**: Each state is a separate function, making the code more organized.
2. **Ease of Extension**: Adding new states doesn't require modifying a central `match` statement.
3. **Decoupling**: State logic is decoupled from the main update loop, improving readability.
4. **Dynamic Behavior**: States can be changed or added at runtime if needed.

### Disadvantages

1. **Complexity**: Slightly more complex to set up and understand, especially for beginners.
2. **Overhead**: Indirect function calls can introduce minimal performance overhead.
3. **Debugging Difficulty**: Tracing the flow of execution can be harder due to indirection.

---

## Class-Based State Machines

For even more flexibility and organization, you can use a class-based approach. This approach involves creating a base `State` class that each state inherits from, and a `StateMachine` class that manages these states. In this expanded example, we also pass a reference to the object being controlled and the state machine, allowing the state to manipulate the object directly and manage state transitions internally.

### Implementation

```gdscript
# State.gd
extends Resource
class_name State

var owner: Node
var state_machine: Node

func _init(owner: Node, state_machine: Node) -> void:
    self.owner = owner
    self.state_machine = state_machine

func on_process(delta: float) -> void:
    pass

func on_enter() -> void:
    pass

func on_exit() -> void:
    pass
```

```gdscript
# IdleState.gd
extends State

func on_process(delta: float) -> void:
    if Input.is_action_pressed("ui_right"):
        state_machine.change_state("running")

func on_enter() -> void:
    print("Entering Idle State")
```

```gdscript
# RunningState.gd
extends State

func on_process(delta: float) -> void:
    if Input.is_action_just_pressed("ui_up"):
        state_machine.change_state("jumping")

func on_enter() -> void:
    print("Entering Running State")
```

```gdscript
# StateMachine.gd
extends Node
class_name StateMachine

var current_state: State = null
var states: Dictionary = {}

func _ready() -> void:
    states["idle"] = IdleState.new(self, self)
    states["running"] = RunningState.new(self, self)
    change_state("idle")

func _process(delta: float) -> void:
    if current_state:
        current_state.on_process(delta)

func change_state(new_state_name: String) -> void:
    if current_state:
        current_state.on_exit()

    current_state = states.get(new_state_name)
    current_state.on_enter()
```

### Advantages

1. **Encapsulation**: Each state is encapsulated into its own class, improving organization and separation of concerns.
2. **Extensibility**: Adding new states becomes as simple as creating a new class inheriting from `State`.
3. **State-specific Logic**: Each state can have its own `on_enter`, `on_exit`, and `on_process` methods, making it easy to manage transitions and state-specific behavior.
4. **Object Manipulation**: By passing references to the owner object and state machine, states can directly manipulate the object or trigger state transitions without hardcoding logic into the state machine itself.
5. **Maintainability**: The system is highly maintainable in larger projects where each state can be treated as a modular unit.

### Disadvantages

1. **Complexity**: This approach is more complex and might be overkill for smaller projects.
2. **Overhead**: Managing multiple state objects can introduce slight overhead, especially if many states are active simultaneously.

---

## Choosing the Right Approach

### When to Use Enums and Match Statements

- **Small Projects**: Ideal for simple games with a limited number of states.
- **Learning Phase**: Great for beginners to grasp the basics of state management.
- **Quick Prototyping**: Useful when you need to implement something quickly without worrying about scalability.

### When to Use Delegate-Based Systems

- **Large Projects**: Better suited for games with many states and complex behaviors.
- **Collaboration**: Helps keep code organized when working in a team.
- **Reusability**: Easier to reuse state logic across different objects or projects.

### When to Use Class-Based Systems

- **Complex Projects**: Ideal for projects where states need to be highly modular and independent.
- **Extensible Designs**: Perfect when you need to extend your state machine easily by adding new states without touching the existing code.
- **Clean Separation of Concerns**: When you want each state to encapsulate its own behavior fully, making debugging and updates easier.

---

## Conclusion

State machines are essential tools in a game developer's arsenal, and choosing the right implementation method can significantly impact your project's maintainability and scalability. Simple `enum` and `match` statements offer an easy entry point but can become cumbersome as complexity grows. Delegate-based systems provide greater flexibility and modularity, while class-based systems offer the highest level of organization and separation of concerns, at the cost of added complexity.

Ultimately, the best approach depends on your project's specific needs and your familiarity with Godot's scripting capabilities. By understanding the advantages and disadvantages of each method, you can make an informed decision that balances simplicity, performance, and scalability.
