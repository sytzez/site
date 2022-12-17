---
title: "Sketch for a new programming language: Part 2"
date: 2022-10-16
tags: ["programming", "compost"]
---

## A solution to inheritance

I've had some more thought about my programming language "Compost".
It might have its own solution to the old problems of object oriented programming, mainly class inheritance.

One of the problems class inheritance tries to solve is code reuse between classes.
In OOP, you'd solve this by creating a superclass which contains the methods which are shared between classes.
The classes that use those method then need to inheritd from that super class.
Using class inheritance comes with many problems, and is not seen as an ideal solution by many people today.

I want to show an example of the classic "Shapes" library that is often used to explaing object oriented programming,
but implemented in Compost.
Read the comments as you go through the code comments to see what Compost offers.

{{< highlight ruby >}}
mod Vec2
    class
        x: Float
        y: Float
    traits
        X: Float
        Y: Float
        Multiply: (factor: Float) -> Vec2
        Add: (other: Vec2) -> Vec2
    defs
        X: x
        Y: y
        Multiply: Vec2(X * factor, Y * factor)
        Add: Vec2(X + other.X, Y + other.Y)
        # Converts it to a Point. The Point trait is auto-declared by the Point module
        .Point: Point(X, Y) 

# For 'public' fields I might short-hand this:
mod Vec2
    class
        x: Float
    traits
        X: Float
    defs
        X: x
# To this: (the uppercase first character would auto-define all of the above)
mod Vec2
    class
        X: Float

# A type of Vec2 with the same x and y. By defining the X and Y traits of Vec2,
# it auto-defines all other Vec2 traits. You can use Vec2 and DiagonalVec2 instances
# completely interchangeably because they have the same interfaces of traits.
mod DiagonalVec2 
    class
        value: Float
    defs
        Vec2
            X: value
            Y: value

mod Point
    class
        x: Float
        y: Float
    traits
        X: Float
        Y: Float
        Translate: (offset: Vec2) -> Point
    defs
        X: x
        Y: y
        Translate: Point(x + offset.X, y + offset.Y)
        .Vec2: Vec2(x, y)

# A function to create a 'diagonal' point. This would work the same as
# creating a whole DiagonalPoint class, since a class is just used a function.
let DiagonalPoint(value: Float): DiagonalVec2(value).Point

# Shape declares some traits but doesn't define them. It works like an interface.
mod Shape 
    traits
        Center: Point
        Area: Float
        Perimeter: Float

# Rectangle declares some traits but doesn't define all of them.
# It defines some of Shape's traits using those declarations.
mod Rectangle 
    traits
        TopLeft: Point
        BottomRight: Point
        Size: Vec2
    defs
        # Just to make clear we purposely haven't defined this: ? means undefined.
        TopLeft: ?
        BottomRight: TopLeft.Translate(Size)
        Size: ?
        # We define Shape's traits here. So if any class defines Rectangle's traits,
        # Shape's traits will also be defined for that class.
        Shape 
            Center: Rectangle.TopLeft.Translate(Rectangle.Size.Multiply(0.5))
            Area: Rectangle.Size.X * Rectangle.Size.Y
            Perimeter: (Rectangle.Size.X + Rectangle.Size.Y) * 2

# Declares but doesn't define a Square.Size trait. Square is another interface.
# It automatically defines the Rectangle.Size trait if the Square.Size trait is defined
mod Square 
    traits
        Size: Float
    defs
        Size: ?
        Rectangle:
            Size: Vec2(Square.Size, Square.Size)

# One type of Square class, created by giving a top_left corner and a size. 
# This will define all of Square's, all of Rectangle's and all of Shape's traits.
mod TopLeftSquare 
    class
        top_left: Point
        size: Float
    defs
        Square
            Size: size
        Rectangle
            TopLeft: top_left

mod CenterSquare # Another type of Square class
    class
        center: Point
        size: Float
    defs
        Square
            Size: size
        Rectangle
            Center: center
            TopLeft: center.Translate(Rectangle.Size.Multiply(-0.5))

# A constant definition.
let Pi: 3.14159265359

mod Circle
    class
        center: Point
        radius: Float
    defs
        Shape
            Center: center
            Area: Pi * radius * radius
            Perimeter: 2 * Pi * radius

# A function definition that takes any shape.
let AreaPlusPerimeter(shape: Shape.Area & Shape.Perimeter)
    shape.Area + shape.Perimeter

# The main function
let Main
    # Creates a square by specifying the top left corner and the size.
    # Then returns the Shape.Center trait of it.
    TopLeftSquare
        top_left: Point(x: 10, y: 10)
        size: 100
    Center

{{< /highlight >}}

## Defining functions or variables

A thing I wasn't sure about before is how to define functions and "variables".
I came to the conclusion that those two are really the same, since "variables" won't change. They are really constants due to the fact that we're a functional language.
When "changing" a "variable" we're just defining a new constant.

It was difficult to find a common keyword to define a function or a constant. I've come to the realisation that we don't even need a keyword.
Functions or constants will be defined like this:

{{< highlight ruby >}}
    Pi: 3.14

    AddOne(x: Int): x + 1

    LongerFunction(instance: MyInterface)
        instance
        Method1
        Method2
        Method1
{{< /highlight >}}

Functions can be defined using a semicolon or by adding one or more indented lines below the name.

In the last function I showcase how method chaining is done. We simply put them on the next line.

So how do we define constants inside functions? Like this:

{{< highlight ruby >}}
    AnotherFunction(instance: MyInterface)
        ConstantOne: instance.Method1
        ConstantTwo: 3
        ConstantThree
            instance
            Method3(ConstantTwo)
            Method1
        ConstantOne.Method4(ConstantThree)
{{< /highlight >}}

To explain: it creates `ConstantOne`, which will contain the result of `instance.Method1`.
`ConstantTwo` is simply `3`.
`ConstantThree` contains `Method3(3)` called on our instance, and then `Method1` called on that.
The function returns `ConstantOne` with `Method4(ConstantThree)` called on it.

I just realized it might be difficult the notice the difference between chaining methods and defining functions/constants.
I'll have to find a solution for that. Maybe I should just use `let` to define anything. That would look like this:

{{< highlight ruby >}}
    let AnotherFunction(instance: MyInterface)
        let ConstantOne: instance.Method1
        let ConstantTwo: 3
        let ConstantThree
            instance
            Method3(ConstantTwo)
            Method1
        ConstantOne.Method4(ConstantThree)
{{< /highlight >}}
