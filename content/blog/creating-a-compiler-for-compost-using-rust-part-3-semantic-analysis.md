---
title: 'Creating a Compiler for Compost using Rust, Part 3: Semantic Analysis'
date: 2022-11-27
tags: ["programming", "compost", "rust", "compiler"]
---

In the [previous installment](/blog/creating-a-compiler-for-compost-using-rust-part-2-syntactic-analysis/#expressions) of this series, I described how I turned the tokens into an abstract syntax tree (AST).
The AST contains all statements, expressions and types of the program, but doesn't link the together.
All the names of variables, modules, traits and functions are simply Rust `String`s without any meaning beyond that.

To give meaning to these names we can use semantic analysis, which resolves the names into references to the right piece of information in our program.
It can also give some more elaborate error messages if anything in the code doesn't link up correctly.

# Semantic Analysis

Semantic analysis, also known as context sensitive analysis, does many things. In the Compost compiler, this has been the most difficult phase to work out so far.

Firstly, it generates symbol tables from the code, which store 'symbols', the names of things, against real information about the program.
There are tables for traits, lets, modules, local variables, etc. Each part of the code that can be represented by a name needs to have a symbol table.

Secondly, it resolves each place where such a symbol is used to the correct information. This can be context dependent.
You can, for example, reference something differently from inside a module by leaving out the module prefix. Inside module `Op`, you can use `Add` instead of the full `Op\Add`.

Furthermore, it checks that all types match up. It needs to figure out the output type of each expression, and check if it matches the input type of wherever the expression is being used.

Finally, it transforms all `Expression` statement enums into what I called `Evaluation` enums, which have more complete information.
They contain references to the actual functions, constants, traits, structs and classes being referenced, rather than just their names in the form of a `String`.
In the `Evaluation` form they are almost ready to be evaluated into a result.

# Symbol Tables

I created the `Table` struct to contain all the logic for declaring and resolving symbols.
It splits symbols up by the `\` character into a vector of smaller strings, called a path.

When an item is declared, it is added to the `items` vector in the form of a tuple with the path on one side and a `Rc` (reference counted pointer) to the item on the right side.
Rust's `Rc` pointer was a good choice here, because items can be retrieved from the table in many places in the code which can then also pass them on to other places.
Using a regular `&` reference here would have caused Rust hell.

{{< highlight rust >}}
pub struct Table<T> {
    // The name of the table, just used for error messages.
    name: &'static str,
    // The items inside the table, with the path on the left.
    items: Vec<(Vec<String>, Rc<T>)>,
    // The longest path length in the table. Used for the algorithm that resolves symbols.
    longest_path: usize,
}
{{< /highlight >}}

The function that resolves a symbol loops through the items to find a match.
First, it tries the shortest possible path in the table that matches the given path.
If that fails, it will try longer and longer paths.

The function also takes into account the scope from where the table was accessed.
First it will try to search within the scope to find a resolution.
If that fails, it will try outside of the scope.

{{< highlight rust >}}
/// Resolves the best match of the given path.
/// When "Add" and "Op\Add" are available, "Add" should resolve to the former.
/// In the "Op" scope, "Add" should resolve to the latter.
/// When only "Op\Add" is available, "Add" should resolve to that.
/// Only when the name is not available inside the scope, global search should be tried.
pub fn resolve(&self, name: &str, scope: &str) -> CResult<Rc<T>> {
    // Form the path by combining the scope and the symbol name.
    // "Add" in scope "Op" will become "Op/Add".
    let path = [Self::path(scope), Self::path(name)].concat();

    // Check shortest paths that match first, then longer ones.
    for match_len in path.len()..=self.longest_path {
        for (item_path, item) in self.items.iter() {
            if item_path.len() == match_len {
                // Check if the shortened end of the path matches the given path.
                let start = item_path.len() - path.len();
                let shortened_item_path = &item_path[start..];

                // Return a Rc to the item on a match.
                if shortened_item_path == path {
                    return Ok(Rc::clone(item));
                }
            }
        }
    }

    if !scope.is_empty() {
        // If nothing was found, and the scope is not empty,
        // try searching everywhere by omitting the scope.
        self.resolve(name, "")
    } else {
        // If nothing was found even globally, return an error.
        error(ErrorMessage::NoResolution(self.name, name.into()))
    }
}
{{< /highlight >}}

The full code for `Table` can be found [here](https://github.com/sytzez/compost/blob/master/src/sem/table.rs).

# Analysing the AST

The `analyse_ast` function analyses the whole AST and returns a `SemanticContext`, which contains all accessible elements of the code within symbol tables.

{{< highlight rust >}}
pub struct SemanticContext {
    pub traits: Table<RefCell<Trait>>,
    pub lets: Table<RefCell<Let>>,
    pub interfaces: Table<RefCell<Interface>>,
}
{{< /highlight >}}

While the `SemanticContext` looks fairly simple. The process to get that result incorporates many steps of analysis over the AST.
All items are wrapped into Rust's `RefCell`, to allow them to be changed at different steps of analysis.

## Step 1: Populating Trait and Interface Identifiers.

The first step is simply populating all traits and interface identifiers that exist in the program.
An interface is a type that's automatically defined for each module, it's simply a combination of all the traits in that module.
Populating the traits and interfaces simply adds records to the right tables with dummy items, to be replaced later:

{{< highlight rust >}}
// Loop through all modules in the AST.
for module in ast.mods.iter() {
    // Add the identifier for this module's interface.
    let dummy_interface = context.interfaces.declare(&module.name, RefCell::new(vec![]))?;

    // Loop through all trait statements in this module.
    for trait_statement in module.traits.iter() {
        let name = format!("{}\\{}", module.name, trait_statement.name);

        // Add the identifier for this trait. The trait is connected to the interface of the module, which is needed for the next step.
        context
            .traits
            .declare(&name, RefCell::new(Trait::dummy(&name, &dummy_interface)))?;
    }

    // Each module has an eponymous trait. This trait can be used to convert other instances into something of that module's type.
    // For example, the String module creates a String trait, which can be defined on other modules to turn those into a String.
    context
        .traits
        .declare(&module.name, RefCell::new(Trait::dummy(&module.name, &dummy_interface)))?;
}
{{< /highlight >}}

Afterwards, there is a slightly complex process to expand the traits contained in each interface.
Because Compost has a feature calles 'automatic definitions', some traits are automatically defined on modules which they don't appear on in the code.
This process adds those traits to the interface types of those modules.
Sometimes, the process needs to be repeated a few times before everything is included.

{{< highlight rust >}}
// Fill module interfaces, made up of the module's own traits and def traits from other modules.
// By this point, all trait identifiers have been populated.
for module in ast.mods.iter() {
    let mut interface = vec![];

    // The module's own traits.
    for trait_statement in module.traits.iter() {
        let trayt = context
            .traits
            .resolve(&trait_statement.name, &module.name)?;

        interface.push(trayt);
    }

    // Traits added on from other modules through defs.
    for def in module.defs.iter() {
        let trayt = context.traits.resolve(&def.name, &module.name)?;

        interface.push(trayt);
    }

    let output = interface_type(&interface);

    context.interfaces.resolve(&module.name, "")?.replace(interface);

    let eponymous_trait = Trait {
        full_name: module.name.clone(),
        interface: context.interfaces.resolve(&module.name, "")?,
        inputs: vec![],
        output,
        default_definition: None,
    };

    context
        .traits
        .resolve(&module.name, "")?
        .replace(eponymous_trait);
}

// Add automatic definitions to interfaces. Repeat until stable.
loop {
    let mut added_num_of_traits: usize = 0;

    for module in ast.mods.iter() {
        let own_interface = context.interfaces.resolve(&module.name, "")?;

        let mut related_interfaces = vec![];

        for def in module.defs.iter() {
            let trayt = context.traits.resolve(&def.name, &module.name)?;

            related_interfaces.push(Rc::clone(&trayt.borrow().interface));
        }

        for related_interface in related_interfaces.iter() {
            for trayt in related_interface.borrow().iter() {
                if own_interface.borrow().iter().any(|t| t == trayt) {
                    continue;
                }

                own_interface.replace_with(|old| {
                    old.push(Rc::clone(trayt));
                    old.clone()
                });

                added_num_of_traits += 1
            }
        }
    }

    if added_num_of_traits == 0 {
        break
    }
}
{{< /highlight >}}

I must admit this code is not very clear. It can probably be refactored.

## Step 2: Analysing Input and Output Types

By this point, all traits and interfaces have been populated.
Since Compost types are composed of traits and interfaces, it's now possible to resolve all types in the program.
Types occur in the inputs and outputs of traits, lets and defs. 
Lets include the constructors of structs and classes.

{{< highlight rust >}}
// Loop over all global lets, and analyse their types.
for let_statement in ast.lets.iter() {
    let lett = Let::analyse_just_types(let_statement, &context, "")?;

    context
        .lets
        .declare(&let_statement.name, RefCell::new(lett))?;
}

// Loop over all modules.
for module in ast.mods.iter() {
    // Analyse all trait input and output types, and analyse their default definitions.
    for trait_statement in module.traits.iter() {
        let trayt = Trait::analyse(trait_statement, module, &context, false)?;

        context
            .traits
            .resolve(&trait_statement.name, &module.name)?
            .replace(trayt);
    }

    // Populate module let identifiers, analyse their types.
    for let_statement in module.lets.iter() {
        let name = format!("{}\\{}", module.name, let_statement.name);

        let lett = Let::analyse_just_types(let_statement, &context, &module.name)?;

        context.lets.declare(&name, RefCell::new(lett))?;
    }

    // Populate struct and class constructors and definition identifiers, and analyse their types.
    if let Some(struct_statement) = &module.strukt {
        // Just the inputs and output of the constructor.
        let constructor = Let {
            inputs: Struct::constructor_inputs(struct_statement),
            output: interface_type(context.interfaces.resolve(&module.name, "")?.borrow().as_ref()),
            evaluation: Evaluation::Zelf,
        };

        context
            .lets
            .declare(&module.name, RefCell::new(constructor))?;
    } else if module.class.is_some() {
        // Just the inputs and output of the constructor.
        let constructor = Let {
            inputs: Class::constructor_inputs(module, &context)?,
            output: interface_type(context.interfaces.resolve(&module.name, "")?.borrow().as_ref()),
            evaluation: Evaluation::Zelf,
        };

        context
            .lets
            .declare(&module.name, RefCell::new(constructor))?;
    }
}
{{< /highlight >}}

## Step 3: Analyse Expressions

Now that we have all types, as well as identifiers for all lets, traits and defs, including their input and output types,
we can finally analyse the expressions occurring throughout the program.
Expressions are part of let and def statements.

{{< highlight rust >}}
// Analyse global let expressions.
for let_statement in ast.lets.iter() {
    let lett = Let::analyse(let_statement, &context, "")?;

    context.lets.resolve(&let_statement.name, "")?.replace(lett);
}

for module in ast.mods.iter() {
    // Analyse module let expressions.
    for let_statement in module.lets.iter() {
        let lett = Let::analyse(let_statement, &context, &module.name)?;

        context
            .lets
            .resolve(&let_statement.name, &module.name)?
            .replace(lett);
    }

    // Re-analyse traits with default definitions.
    for trait_statement in module.traits.iter() {
        let trayt = Trait::analyse(trait_statement, module, &context, true)?;

        context
            .traits
            .resolve(&trait_statement.name, &module.name)?
            .replace(trayt);
    }

    // Analyse struct and class constructor and def expressions.
    if module.strukt.is_some() {
        let strukt = Struct::analyse(module, &context)?;

        context
            .lets
            .resolve(&module.name, "")?
            .replace(strukt.constructor());
    } else if module.class.is_some() {
        let class = Class::analyse(module, &context)?;

        context
            .lets
            .resolve(&module.name, "")?
            .replace(class.constructor());
    }
}
{{< /highlight >}}

Find the full code for `analyse_ast` [here](https://github.com/sytzez/compost/blob/master/src/sem/semantic_analyser.rs).

# Analysing Evaluations from Expressions

All the `*::analyse` methods that are used in the `analyse_ast` function are defined in different places. They analyse Lets, Traits, Modules, Defs, Types and Expressions in detail.

The core of these analyses it the `Evaluation::analyse` method, which take an `Expression` and analyses everything inside it within the right context, resulting in an `Evaluation`.

During this analysis, it resolves all the referenced traits and lets, checks that the called traits are callable on the expression they're called on, and verifies that all input and output types are matching1.

Some parts of this analysis, especially type checking, are still a work of process as of writing this blog. But the whole thing works well enough to make most programs run!

Find the source code of it [here](https://github.com/sytzez/compost/blob/master/src/sem/evaluation.rs).

# Conclusion

The most daunting part of the compiler so far has been semantic analysis.
It requires many traversions over the AST to resolve and analysis everything in the right order.
First we need to know what types (traits and interfaces) are available, then we need to know what lets we have, and what the input and output types of everything is.
Then we can analyse the expressions, and check that all types match up.

The resulting `SemanticContext` struct and its contents are so detailed that they can easily be resolved into a result to be displayed in the console, which is what happens in the `runtime` module.
However, ideally I'd like to write some more modules that turn this `SemanticContext` into intermediate code, which can then be compiled into an actual binary by LLVM.

Either way, even without compiling to a binary, I feel pretty satisfied with this compiler, at the moment more of an interpreter. that can parse Compost code and execute it!
If you are interested in the Compost programming language please check out the [GitHub repository](https://github.com/sytzez/compost/), which contains more information about the language in the README.
And also check out the [Compost Playground](http://compost-playground.sytzez.com/) in which you can run Compost code from your browser.
