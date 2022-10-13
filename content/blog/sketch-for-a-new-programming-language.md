---
title: "Sketch for a new programming language"
date: 2022-10-13
tags: ["programming", "compost"]
---

I'm coming up with the spec of a new programming language. Working title: "Compost". Don't worry, it's just for fun. For now this is just a first sketch of the ideas I had today. They are subject to change.

The main idea of the language is ultimate composability and recomposability. It forces the programmer to think in terms of the minimum amount of dependencies required for each part of the code, and makes
each part as reusable as possible. It will also use 'value' semantics and it will be highly encapsulated. It's not possible to directly change or read any fields of an object except through traits.
In fact it will be impossible (and irrelevant) to know what class and object is, types will be defined solely on the basis of traits.

Classes will not be allowed to have any raw fields (such as ints, bools, floats), instead their dependencies are defined on the basis of traits, making them completely interchangeable.

Let's start off with some example code:


{{< highlight ruby >}}
mod Cat # Defines a module, an interface and a class, all called 'Cat'. Also defined a trait called 'Cat' with type () -> Cat, to transform other things into a Cat
  class # Dependencies of the Cat class
    name: String # String is an interface
    birthDate: Date # Date is also an interface
  traits # The traits making up the Cat interface
    Name: String # Trait Cat.Name has no arguments and returns a String interface
    Age: (now: Date) -> Timespan # Cat.Age takes a Date interface and returns a Timespan interface
  defs # The trait definitions for the Cat class
    Name # Defines the Cat.Name trait for Cat
      name # Simply return Cat.name
    Age # Defines the Cat.Age trait for Cat
      Date.Diff: birthDate, now # Call function Diff from module Date. Use Cat.birthDate and the argument from the Age trait
    String # Define the standard String trait for a Cat, meaning the Cat can be transformed into a String
      Name + " " + Age.String # Return the Cat.name and Cat.Age converted into a String
      # Because this String definition doesn't use any dependencies of Cat, it can be implemented automatically for other classes implementing the Cat interface
{{< /highlight >}}

I could shorten the defs to:

{{< highlight ruby >}}
defs
  Name: name
  Age: Date.Diff: name, now
  String: name + " " + Age.String
{{< /highlight >}}

A lot of stuff is implied from the context by the code. The full code would be:

{{< highlight ruby >}}
mod Cat
  traits # Some traits on module Cat
    Name: String
    Age: (now: Date) -> Timespan
  interface Cat # Interface Cat, consisting of traits
    Cat.Name
    Cat.Age
    String
  class Cat implements Cat # Class Cat implementing the interface. Lists dependencies.
    name: String
    birthDate: Date
  def Cat.Name for Cat
    Cat.name
  def Cat.Age for Cat
    Date.Diff: Cat.birthDate, Age.now
  def String for Cat
    Cat.Name + " " + Cat.Age.String
{{< /highlight >}}

This exports a *lot* of things that can be used outside of the mod:
- The `Cat.Name` and `Cat.Age` traits.
- The `Cat` interface, made out of the `Cat.Name`, `Cat.Age` and `String` traits.
- One function: `Cat`, which takes a `name` and a `birthDate` and returns an object implementing `Cat`. Ergo, the `Cat` constructor.
- The `Cat` trait, which can be defined for other classes that can be converted into a Cat or return a Cat. Type: `() -> Cat`.
- No other functions. We haven't declared any, we could.
- The `Cat` class, which can be used *only* to create subclasses inheriting its trait definitions. It can't be a type.

Let's define another class in this context:

{{< highlight ruby >}}
mod Human
  class
    name: String
    cat: Cat # This dependency needs to implement the Cat interface. It doesn't have to be the Cat class.
    birthDate: Date
  traits
    Name: String
    Age: (now: Date) -> Timespan
  defs
    Name: name
    Age: Date.Diff: birthDate, now
    Cat: cat # We define the trait Cat from the Cat module for Human
    String: Name + " " + Age.String + " owning cat: " + Cat.String # This calls the Cat trait on our Human, and calls the String trait on the resulting Cat.
{{< /highlight >}}

As you might have guessed, we now have some very similar traits:
- `Cat.Name` and `Human.Name`
- `Cat.Age` and `Human.Age`

This might be a good thing. If you think of a name as an id, it's good to keep different types of them. A Cat and Human might have the same name, but they refer to different entities.

However, you might also say a name is a name, and it shouldn't matter what it's naming. In that case, you can extract the `Name` trait from both:

{{< highlight ruby >}}
trait Name: String
{{< /highlight >}}

And then you don't have to declare the trait on the Human and the Cat, you only have to define its implementation. It will still be part of the Cat and Human interfaces if you define it within their modules.

There are two things I had imagined which we haven't covered yet, declaring functions and structs.

A function may take some arguments, and *MUST* return at least one value. It *CAN NOT* have side effects. So we're fully functional ;).
It can return the same type as one of its arguments though, in which case we can use the function to replace the original value.

{{< highlight ruby >}}
fn ConstantValue: () -> Int
  42
fn ConstantValue: Int # Shorthand
  42
fn AddOne: (x: Int) -> Int
  x + 1
fn AddOne: &Int # Shorthand for the above
  AddOne.0 + 1
fn AddOne: (x: Int) -> x # Signifies we could be replacing x
  x + 1
fn AddOne: (&x: Int) # Shorthand for the above
  x + 1
{{< /highlight >}}

The last method can be used like this:
{{< highlight ruby >}}
  let x: ConstantValue
  AddOne!: x # The exclamation mark lets us know we're changing x
  let y: AddOne(x) # Doesn't change x
{{< /highlight >}}


A trait can also have a `&` type, in which we can use it to "update" the class, i.e. override it, or override part of it. I still have to figure out how this will work exactly, but I'd like to be able to do `Cat.Rename!: "Bobo"`

Lastly, instead of classes, we can define structs. While class dependencies can only be things implementing traits, struct dependencies can only be core values.

{{< highlight ruby >}}
mod I64 implements Int
  struct
    x: i64
  defs
    Op.Add: I64(x + Op.Add.right.x) # Creates a new struct of itself. We can access the private memory of `right` because it's the same struct type.
    # etc.
{{< /highlight >}}

Like with a class module, we can define traits for the struct, and implement an interface.

`Op.Add` Is defined like this:

{{< highlight ruby >}}
mod Op
  trait Add: (right: Self) -> Self
{{< /highlight >}}

The Self type obviously refers to whatever type the trait is being defined on.

This is it for now, it's just a sketch. I might post more about it in the future.
