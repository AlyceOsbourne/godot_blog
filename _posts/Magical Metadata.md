---
share: true
---

In my last post about [Self Binding Components](Self Binding Components), I explored the idea of using metadata to store variables linked to your components. This approach allows other components to access the data they need without rummaging through child nodes. Today, I’m excited to delve deeper and show you some different ways to use and access metadata in Godot.

## What Is Metadata?

In Godot, any object that extends from `RefCounted` comes with its own metadata. Think of metadata as a built-in dictionary—a neat little storage space where you can tuck away all sorts of data types. This makes it an incredibly versatile tool for managing your objects' state and sharing data between components.

### Working with Metadata in the Inspector

You can easily manage metadata in the Inspector, which is perfect if you prefer a visual approach. Simply select an object and look for the `Add Metadata` button at the bottom of the panel.

![Pasted image 20240813174135](./Pasted%20image%2020240813174135.png)

Once you click the button, a dialogue will pop up, asking you to name your metadata and choose a type.

![Pasted image 20240813174353](./Pasted%20image%2020240813174353.png)

You’ll find all the usual data types here, just like those available in an exported dictionary. This feature is particularly handy for quickly checking what metadata is available and setting it up manually. But you’ll want to harness the full power of metadata by manipulating it through code. That’s where the real magic happens!

## Accessing Metadata in Code

Godot provides a few different methods for working with metadata in code, each with its own unique advantages:

- **`get_meta("key", default_value)`**: This is your go-to method for retrieving a value from metadata. It’s especially useful because you can provide a fallback value if the key doesn’t exist. This little safety net helps you avoid those annoying errors when a key isn’t found.

- **`set_meta("key", value)`**: As the name suggests, this method lets you set a value for a specific key in the metadata. And here’s a neat trick: if you want to delete a metadata entry, just set its value to `null`.

- **`has_meta("key")`**: Similar to `has_method` and `has_signal`, this method checks whether a particular key exists in the metadata. It’s incredibly useful when you use the presence of metadata as a flag or condition in your code.

Now, here’s a slightly lesser-known trick that can make your code look cleaner:

- **`self["metadata/key"]`**: This shorthand syntax allows you to access metadata in a more concise way. For instance, `self["metadata/health"]` fetches the `health` value stored in the metadata.

- **`self["metadata/key"] = value`**: You can also set metadata values using this syntax, like `self["metadata/health"] = 100`. And just like with `set_meta`, setting the value to `null` will remove the entry.

### Shorthand: Pros and Cons

The shorthand syntax is a fantastic way to keep your code tidy and easy on the eyes. However, it does come with a slight drawback: you can’t specify a default value when retrieving metadata. So, it’s best to use this approach when you’re confident that the metadata key exists, or when a default value isn’t strictly necessary.

## Why Use Metadata?

Now, you might be wondering why you’d want to use metadata in the first place. Here are a few reasons that might convince you:

- **Avoids unnecessary class attributes**: Metadata lets you store values without cluttering your classes with additional attributes. This keeps your codebase cleaner and more manageable.
  
- **Inspector readability**: Like exported variables, metadata is visible in the Inspector. This makes it easy to see and modify your object’s state directly in the editor.

- **Simple interface with error handling**: The methods for accessing and manipulating metadata are straightforward, and built-in safeguards like default values help you avoid common errors.

- **Promotes modular design**: Metadata can be accessed by other components without creating messy dependencies, reducing the likelihood of “spaghetti code” in your project.

## When Should You Use Metadata Over Variables?

This is where things can get a bit subjective, as it largely depends on your specific use case. However, I can share some general guidelines on how I approach the decision between using variables and metadata.

### Variables

I tend to think of un-exported variables as private, used internally within a script to manage state or perform operations. Exported variables, on the other hand, are more about configuring an object and shaping its behaviour. They allow you to define the specific attributes that will affect how an object functions, often acting as a way to compose the desired behaviour by swapping out implementations. Additionally, exported variables can be a handy way to reference parts of the scene tree without resorting to node paths.

### Metadata

In contrast, I view metadata as being tied to the state of the object—especially when the object’s state needs to be accessible or shared without being tightly coupled to a specific script. Metadata is available regardless of whether the object has a bound script, making it an excellent choice for storing values that represent the object’s current state. Think of it as a blackboard where you can jot down important details like health, inventory contents, dialogue choices, and so on. This can be particularly useful in complex projects where various components need to interact with an object’s state without directly modifying its variables.

## Final Thoughts
The decision to use metadata or variables should ultimately be guided by the needs of your project. Metadata offers a flexible and accessible way to manage state, especially when you want to avoid over-complicating your scripts. Variables, on the other hand, are ideal for defining an object’s behaviour and configuration. By understanding the strengths of each, you can make more informed choices that lead to cleaner, more maintainable code.
