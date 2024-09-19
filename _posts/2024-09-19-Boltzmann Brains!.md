---
share: true
layout: post
---

Today, I was scrolling through YouTube when a video about a type of Recurrent Neural Network called Boltzmann Machines (BM) caught my eye. They're a relatively simple kind of neural network often used for recommendations, like what video to watch next or what product to buy based on your preferences. As I listened to the explanation of how they work, I realized it sounded a lot like something I’d been tinkering with recently: Utility AI!

## What the Heck is a Boltzmann Machine?

A Boltzmann Machine operates with two layers. It starts by taking in binary inputs and processes them to predict the most fitting output. It does this using weights and biases to fine-tune its predictions. Essentially, it’s a model designed to learn from data and improve its recommendations over time, which might remind you of Utility AI in various applications.

The inputs in a BM are series of observations, much like utility values, and the output defines the recommendation, which, in our case, could be an action.

But, there are two big problems:
- The input is binary, meaning we need Boolean values.
- The input and output sizes are the same.

## Problem Solving

This got me wondering: can we keep the simplicity of the BM while making it more flexible for our needs?

I found that by tweaking the algorithm a little, we could use floating point values instead of just binary. This simple change lets us capture way more nuance in the input information, freeing us from making binary decisions. Essentially, it was as easy as swapping data types and using activation methods that worked better with floating point values.

The next challenge was figuring out how to have an output layer so the input and output could be different sizes. I wanted to consume some amount of information about the entity and game world, then make a decision about what action to take.

So, I looked into some relatives of the BM and discovered the Deep Restricted Boltzmann Machine. Unlike the version I had been fiddling with, this one allows for multiple layers of varying sizes—perfect! This not only enables us to have the input and output sizes we want, but it also gives us control over the size of the hidden layer, independent of the inputs and output. For this experiment, I chose a size of 4 for the input.

## The Plan

Armed with this new knowledge, I started piecing together my mad plan to create a Utility AI that can evolve during gameplay. My idea was to use some utility functions to gather data and map the output probability to an array index containing some action function. This would let us build the AI pipeline and give us some starting values to work with, like our input and output sizes.

We could then use the default utility actions as training data. Over time, we could sample data from the player and use that as additional training data as the game progressed.

## More Problems?

Now, I love Godot—really, I do—but it’s not without its problems. One issue is Arrays losing their types, which happens for a few reasons:
- **Nested Arrays**: Godot doesn't support typing of nested arrays, which made building training data painful and resulted in creating arrays just to use the `assign` function to type them correctly.
- **Array Methods**: Another caveat is the lack of generics in Godot. In practice, this means array methods suffer from type erasure. When you call `map` or `filter`, the original type information is lost, causing type errors in subsequent function calls.

Another issue I noticed was a higher error rate compared to my Python experiments, which concerned me. Python has the advantage of arbitrary precision (for our purposes anyway), but Godot is more limited in that regard.

## Silver Linings

Despite the problems, my initial results were encouraging. Although the error was still higher than I’d like, when inputting my utilities, I was getting approximately the right action. This meant that maybe, just maybe, this idea wasn’t just the ravings of a code-obsessed Cthulhu.

Initially, I still treated most of the values as binary choices:
- Are you hungry? - Eat Food
- Are you tired? - Go to Sleep
- Are you neither? - Idle

I also wondered if it would infer the importance of one value over another. So, I added one more encoding to state that if both hungry and tired, it should still choose to eat.

```gdscript
var data = [
    [0, 0, 0.001], # no utility, idle
    [1, 0, 0.001], # hungry
    [0, 1, 0.001], # tired
    [1, 1, 0.001], # hungry and tired
]

var target_output = [
    [0, 0, 1], # idle
    [1, 0, 0], # eat
    [0, 1, 0], # sleep
    [1, 0, 0], # hungry and tired, prioritize eating
]
```

## Testing My Assumptions

Assumptions can often speed up development, but they can also trip us up, especially if we invest a lot of time based on an assumption only to find out later we were painfully, critically wrong. Seeing that I could spend countless hours tinkering with this idea, I decided to run my test data and see if the BM learned what I hoped it would. Fingers crossed, I hoped it would understand the nuance in decisions that weren’t purely binary.

I watched as the numbers ticked down in the console, the error decreasing rapidly as hundreds of epochs passed in a flash. This was it—time to see if this idea was worth pursuing or if it would join the ever-growing graveyard of abandoned projects. I ran the training data through and watched as it gave the answers with near perfection. Unsure if this was just a lucky run, or if the BM could consistently do what I hoped, I reset the training and ran it a few more times. Each time, it gave me exactly what I was hoping for: decisions with *just* enough randomness to keep things interesting, all trained off an array of mostly 1s and 0s.

## Making New Observations
I played around with the training data to see how the BM would react, and noticed an interesting pattern. When the inputs were mostly 1s and 0s, the error rate dropped much faster and was consistently lower. However, when I used floating point values between 0 and 1, there was a noticeable loss in precision.

I think what's happening is that using 1s and 0s simplifies the system because of how multiplication works. When you multiply by 0, the result is 0, and when you multiply by 1, the value stays the same. This allows the BM to focus more on the relationships between the values rather than getting bogged down by noise in the data. And as we observed earlier, we can still map the relationship between two values, as we did when both hungry and tired.

## Next Experiments
Given the results of these initial test, I am much more hopeful that this can be used as part of an game AI, there are a few other experiments I plan to do, such as test the effect of various activation functions, testing how it reacted to values outside of the range 0 - 1, and most importantly, how does it scales with many more inputs and outputs, how does scaling the hidden layer alter the behaviour, and what would be the optimum size based on the given dataset. 

## Final Thoughts
I have thoroughly enjoyed exploring Boltzmann Machines, and I hope you have enjoyed this blog post, I plan to share more posts that focus more on thoughts and processes amongst the usual content. As there are many experiments to be done, and observations to be made, I will follow this post up with more findings soon.
