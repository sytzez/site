---
title: "Compost update December '22: Match, If and Booleans"
date: 2022-12-20
tags: ["compost"]
---

It's been due time to add control flow into Compost. We have it now!

## Match Expressions

Match expressions allow you to use different flows based on some element's type.
Here's an example:

```
lets
    StringOrNothing(thing: @String | ?): String
        match matchedThing: thing
            @String: matchedThing.String
            ?      : 'Nothing'
```

It takes a thing that implements the `String` trait or not.
If it implements `String`, it returns that string. Otherwise it returns the String 'nothing'.

## Booleans and If Expressions

We now have hte boolean literals `true` and `false`.
There are also the operators `=`, `<` and `>`, implemented on `Int`, that can return a boolean.
`=` is also implemented on `String`, and can be implemented on any module using `Op\Eq`.
Booleans themselves have the boolean operators `!`, `&` and `|`.

Booleans come in useful with the new `if then else` expressions, which need a condition resulting in a boolean type.
The else part is necessary, because every expression needs to return something.

Example:

```
lets
    PrintInOrder(a: @String & Op\Gt, b: @String & Op\Gt): String
        if a > b
        then b.String + ', ' + a.String
        else a.String + ', ' + b.String

    Main: String
        PrintInOrder(a: 2, b: 1)
```

## Linked List

Using these new toys, let's see what we can build.
First off, a linked list of numbers!

```
mod LinkedList
    class
         prev: Self | ?
         item: Int
    traits
         First: Int
         Last: Int
         Sum: Int
         Len: Int
         Push: (pushed: Int) -> Self
    defs
        Last: item
        First
            match prev: prev
                Self: prev.First
                ?   : item
        Len
            match prev: prev
                Self: prev.Len + 1
                ?   : 1
        Sum
            match prev: prev
                Self: prev.Sum + item
                ?   : item
        Push
            LinkedList
                prev: Self
                item: pushed
        String
            match prev: prev
                Self: prev.String + ', ' + item.String
                ?   : item.String
 
lets
    MyList: LinkedList
        LinkedList
            prev: ?
            item: 1
        .Push(pushed: 2)
        .Push(pushed: 3)
        .Push(pushed: 4)

    Main: String
        MyList.String
```

I store the list in reverse order to make pushing easier.

## Binary Tree

We can also make a binary tree structure that automatically sorts inserted numbers.

```
mod BinaryTreeItem
    using
        Op\Lt
        Op\Gt
        Op\Eq
        String

mod BinaryTree
    class
        item: BinaryTreeItem
        leftNode: Self | ?
        rightNode: Self | ?
    traits
        Insert: (insertedItem: BinaryTreeItem) -> Self
        Contains: (givenItem: BinaryTreeItem) -> Bool
        Max: BinaryTreeItem
        Min: BinaryTreeItem
        Size: Int
    defs
        Insert
            if insertedItem < item
            then
                match node: leftNode
                    Self
                        BinaryTree
                            item: item
                            leftNode: node.Insert(insertedItem: insertedItem)
                            rightNode: rightNode
                    ?
                        BinaryTree
                            item: item
                            leftNode: BinaryTree
                                item: insertedItem
                                leftNode: ?
                                rightNode: ?
                            rightNode: rightNode
            else if insertedItem > item
            then
                match node: rightNode
                    Self
                        BinaryTree
                            item: item
                            leftNode: leftNode
                            rightNode: node.Insert(insertedItem: insertedItem)
                    ?
                        BinaryTree
                            item: item
                            leftNode: leftNode
                            rightNode: BinaryTree
                                item: insertedItem
                                leftNode: ?
                                rightNode: ?
            else Self
        Contains
            item = (givenItem)
            | match node: leftNode
                Self: node.Contains(givenItem: givenItem)
                ?:    false
            | match node: rightNode
                Self: node.Contains(givenItem: givenItem)
                ?:    false
        Max
            match node: rightNode
                Self: node.Max
                ?:    item
        Min
            match node: leftNode
                Self: node.Min
                ?:    item
        Size
            match node: leftNode
                Self: node.Size
                ?:    Int(value: 0)
            + match node: rightNode
                Self: node.Size
                ?:    Int(value: 0)
            + 1
```

## What's to come

To make these data structures more usable, I will be working a templates feature.
I'm also working on an Iterator module with mapping, filtering, reducing and collecting functionality, which will also use type templating. That will make Compost a lot more useful!
