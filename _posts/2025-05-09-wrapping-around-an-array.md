---
title: Quick Tip — Wrapping Around an Array Using Modulo
author: philip
date: 2025-05-09 13:32:00 +0800
categories: [Quick, Tip, Tutorial]
tags: [Ruby, array, grid, wrap, quick, tip]
render_with_liquid: false
---



Sometimes we deal with structures like boards or grids that contain rows and columns for keeping track of objects. As these objects move around the grid, they inevitably reach the edge or boundary and need a new place to go. In the classic snake game, when the snake reaches a wall, the game is over. But what if we wanted the snake to appear on the opposite side?

#### Set Up & Getting to the Boundary

I'm going to use Ruby to demonstrate the concept, but it translates to other languages as well.

```ruby
# Example row
row = ["_", "S", "_"]

# Index of where the 'snake head' is
current_idx = 1

# Moves snake 1 square to the right
row[current_idx] = "_"
next_idx = current_idx + 1
row[next_idx] = "S"

# row => ["_", "_", "S"]
```

Now that we've reached the wall, what would happen if we moved to the right again?

```ruby
row = ["_", "_", "S"]

# Snake index
current_idx = 2

# Moves snake 1 square to the right
row[current_idx] = "_"
next_idx = current_idx + 1
row[next_idx] = "S"

# row => ["_", "_", "_", "S"]
```

Well that's not correct. Ruby will create a space for the element in the array at the index we provide. Other languages might exhibit different behavior such as giving an out of bounds error.

#### Using Conditional

A clunky way to handle this would be to use a conditional statement:

````ruby
row = ["_", "_", "S"]

# Snake index
current_idx = 2

# Resets Snake to beginning of the array if attempts to go past the end
row[current_idx] = "_"
next_idx = current_idx + 1
next_idx = 0 if next_idx >= row.size
row[next_idx] = "S"

# row => ["S", "_", "_"]
````

Although this works when we move the snake a distance of `1`, it will fail if we need to move over two or more spaces since it will always place the snake at index `0`, instead of properly wrapping around the full length of the move value.

#### Using Modulo Operator

To elegently handle this, we can make use of the modulo operator:

```ruby
row = ["_", "_", "S"]
current_idx = 2

row[current_idx] = "_"
next_idx = (current_idx + 1) % row.size
row[next_idx] = "S"

# row => ["S", "_", "_"]
```

The line `(current_idx + 1) % row.size`  works by capitalizing on the modulo operator (`%`), which returns the remainder of a division. In this case, we get `3 % 3` which has a remainder of `0`, giving us an index of `0` which puts our Snake back at the beginning. 

To move by a larger distance such as `2`, we would get a `next_idx` value of `4`, then we use the modulo operator, `4 % 3`, giving us a remainder of `1` which would set the snake at index `1`: `["_", "S", "_"]`. This correctly wraps around the array and solves our problem in a nice and clean way.

If we wanted to go the other way and move our snake to the left, we simply subtract the distance we want to move instead of adding it: `(current_idx - 1) % row.size` and this will wrap the snake around the end of the array instead of the beginning.

