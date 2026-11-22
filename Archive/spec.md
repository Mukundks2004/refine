# Spec

Context:

Let's say we are interested in formalizing an algorithm- merge sort. In theory, this algorithm will run on a machine with a bit RAM model. We are interested in what the time complexity will be given this RAM model. A model is how the RAM works (e.g. one bit per register and a certain number of permitted operations). Changing the model changes the time complexity of an algorithm written in that model.

Question: is the time complexity the same regardless of RAM model? Does adding operations make algos e.g. merge sort easeir?

That means that your low level data structures and high level algorithms need to be written in terms of the bit RAM operations.

For us, the allowable bit ram operations are `+`, `-`, `*`, `<`, and `=`.

Operations that are not allowed are `sqrt` and `floor`.

We want to encode this in lean then write a translation layer from a small assembly to this lean so we can write an algo in this assembly and then prove it is a certain time complexity in our model.

And then we can write algos in C, compile them to this assembly, convert them into lean, and prove that they take O(n^2) time complexity.

Is this ok?
