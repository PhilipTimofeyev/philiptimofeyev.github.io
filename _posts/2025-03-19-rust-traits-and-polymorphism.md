---
title: Rust Traits and Polymorphism
author: philip
date: 2025-04-11 14:10:00 +0800
categories: [Blogging, Tutorial]
tags: [rust, traits, polymorphism, ruby, structs, beginner, inheritance, classes]
render_with_liquid: false
---
### Rust
Traits are an example of polymorphism in Rust, they allow different types to respond to the same message. This is one of the fundamental concepts of Object Orientated Programming and what allows us to write flexible and DRY code.

```rust
fn main() {

    struct Dog {
        bark: String
    }

    struct Cat {
        meow: String
    }

    trait Speak {
        fn speak(&self);
    }

    impl Speak for Dog {
        fn speak(&self) {
            println!("{}", self.bark)
        }
    }

    impl Speak for Cat {
        fn speak(&self) {
            println!("{}", self.meow)
        }
    }

    let dog  = Dog {bark: String::from("Woof!")};
    let cat  = Cat {meow: String::from("Meow!")};

    dog.speak();
    cat.speak();
}
```

Here we define two structs, each containing a single field name with a type of `String`.
  
We then define the trait `Speak` with a `speak(&self)` function that passes a reference of itself as an argument.  
  
After defining the`Speak` trait, we need to create an implementation of the trait for our two structs. We set up the implementation for both the `Dog` and the `Cat` structs using an `impl` block. This associates everything in the block, in our case the `speak` function, with the respective `Dog` and `Cat` struct. Then we define the `speak(&self)` function which passes a reference of itself in as an argument, and prints the `bark` or `meow` field value.

On `lines 27-28`, we initialize `Dog` and `Cat` structs, assigning an owned string of `"Woof!"` to the `bark` field and `"Meow!"` to the `meow` field.
When we call the `speak()` function on the structs, they print the value of their respective `bark` or `meow` fields.

### Ruby
Coming from a Ruby background, I like to draw parallels between Rust and Ruby to help solidify understanding of these fundamental programming concepts.

Although Rust uses composition and Ruby uses inheritance, the polymorphic thread can still be interwoven between their designs:

```ruby
class Animal
    def initialize(sound)
        @sound = sound
    end

    def speak
        puts @sound
    end
end


class Dog < Animal
    def initialize
        super("woof")
    end
end

class Cat < Animal
    def initialize
        super("meow")
    end
end


dog = Dog.new
cat = Cat.new

dog.speak
cat.speak
```

We create a base class, `Animal`, which has a constructor method that takes a single argument, `sound`. When a new `Animal` instance is initialized, it assigns the `sound` argument to the `@sound` instance variable. On lines 64-66 we define a `speak` method similarly to how we defined the `speak` trait in Rust. This method simply prints the `@sound` instance variable.

 We create a `Dog` and `Cat` class that inherits from `Animal`. This allows the classes to use the `speak` method without needing to define it in the respective class. We use the `super()` keyword to pass a `String` into the inherited `initialize` method, assigning it to the inherited `@sound` instance variable.

Now, we just need to call the `speak` method on the newly initialized `dog` and `cat` instances, allowing us to use the same `speak` method on different classes. Just like we used the same `speak` function on different structs in Rust!