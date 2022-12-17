---
title: 'Creating a Compiler for Compost using Rust, Part 1: Lexical Analysis'
date: 2022-11-27
tags: ["programming", "compost", "rust", "compiler"]
---

I've set out to write a compiler for Compost, the experimental programming language I came up with about a month ago.
The repository for it can be found [on GitHub](https://github.com/sytzez/compost) and there is also a [playground](http://compost-playground.sytzez.com) available where you can try out the language.
This is the first part of a series of blogs detailing my experience of writing my first compiler.

# Structure of a Compiler

Before embarking on this journey, I did some research on how compilers normally work. Almost all sources described these phases that a compiler goes through:

- **Lexical Analysis** or **Tokenization**
- **Syntactic Analysis** or **Parsing**
- **Semantic Analysis**
- **Intermediate Code Generation**
- **Code Optimization**
- **Binary Code Generation**

So far, I've implemented the first three steps in the compiler, and added a different fourth step: Running the Code and Displaying the Result.

Because the Compost language currently doesn't support any type of input, the result of a program will always be the same. In a sense, each program can be optimized into a single constant, which is then output to the console.
For this reason, it doesn't make a lot of sense to compile into a binary at this point. The compiler just outputs the constant result for now.

However, I am planning to add input methods to the language in the future, which will make it worth to implement the remaining three steps of compilation. For the last two steps I'm planning to use LLVM, which will hopefully make that part a lot less painful.

# Lexical Analysis

In this post I will focus on how I implemented the first phase of compilation in Rust. Lexical Analysis, or Tokenization, is the process of grouping characters of the code into a sequence of 'tokens', which are the smallest meaningful units code can be broken up into. It will also throw meaningful errors when there is an unexpected character.

Rust's enums are ideal for representing tokens. This is the `Token` enum of my compiler as of writing this article:

{{< highlight rust >}}
pub enum Token {
    Down(Level),
    Up(Level),
    Next(Next),
    Eof,
    Kw(Kw),
    Local(String),
    Global(String),
    Op(Op),
    Lit(Lit),
    Space,
}
{{< /highlight >}}

The types `Level`, `Next`, `Kw` (Keyword), `Op` (Operator) and `Lit` (Literal) are also Rust enums. Example:

{{< highlight rust >}}
pub enum Kw {
    Mod,
    Class,
    Struct,
    Traits,
    Defs,
    Lets,
    Zelf,
}
{{< /highlight >}}

To read tokens from a string, I've written the `next_token` functions. It returns the next token, and the amount of characters that token spans.

{{< highlight rust >}}
type SizedToken = (Option<Token>, usize);

pub fn next_token(code: &str) -> Result<SizedToken, ErrorMessage> {
    let char = match code.chars().next() {
        Some(c) => c,
        None => return Ok((Some(Token::Eof), 1)),
    };

    let token = match char {
        ' ' => (Some(Token::Space), 1),
        '#' => (None, comment_size(code)),
        '(' => (Some(Token::Down(Level::Paren)), 1),
        ')' => (Some(Token::Up(Level::Paren)), 1),
        ':' => (Some(Token::Down(Level::Colon)), 1),
        '\n' | '\r' => (Some(Token::Next(Next::Line)), 1),
        ',' => (Some(Token::Next(Next::Comma)), 1),
        '+' => (Some(Token::Op(Op::Add)), 1),
        // ...
        'a'..='z' => next_local_token(code),
        'A'..='Z' | '\\' => next_global_token(code),
        '0'..='9' => next_number_token(code),
        '\'' => next_string_token(code),
        _ => return Err(ErrorMessage::UnexpectedChar(char.to_string())),
    };

    Ok(token)
}
{{< /highlight >}}

Based on the first character, it either returns a simple token with size 1, or it uses other functions to continue to read more complex tokens.
`ErrorMessage` is another enum which is used for errors across the compiler.

All of the code above can be found [here](https://github.com/sytzez/compost/blob/master/src/lex/token.rs).

# Levels in Compost

`Down`, `Up` and `Next` are features of Compost's syntax. Compost uses levels to organise its code. Look at this Compost code:

{{< highlight ruby >}}
mod ModuleA
    class
        x
            Int
        y
            Int

mod ModuleB
    class
        x: Int
        y: Int

mod ModuleC
    class(x: Int, y: Int)
{{< /highlight >}}

All three modules are semantically the same. The only difference is the levels syntax they use.

The three ways of organising code into levels are:

- Indentation. The more indented something is, the deeper the level.
- Using colons (`:`). Code on the same line after the colon is on a deeper level. After a comma (`,`) the code is on the original level again.
- Using parentheses (`(` and `)`). Code within parentheses is on a deeper level.

During tokenization, each `Token` is assigned a level based on these three syntaxes. The type for a Token with a level is a simple tuple:

{{< highlight rust >}}
pub type LeveledToken = (Token, usize);
{{< /highlight >}}

To keep track of the level while tokenizing, I've created a utility struct called `LevelStack`.

{{< highlight rust >}}
/// The type of level syntax.
pub enum Level {
    Colon,
    Paren,
}

/// Used to separate levels.
pub enum Next {
    Comma,
    Line,
}

/// Utility to keep track of the depth level of our code.
struct LevelStack {
    // The current stack of Colon and Parentheses levels.
    levels: Vec<Level>,
    // The current indentation level.
    indentation: usize,
}

impl LevelStack {
    fn new() -> Self {
        LevelStack {
            levels: vec![],
            indentation: 0,
        }
    }

    /// Go deeper.
    fn push(&mut self, level: Level) {
        self.levels.push(level)
    }

    /// Go up to a specific type of Level.
    fn pop(&mut self, level: &Level) {
        if let Some(popped_level) = self.levels.pop() {
            match level {
                Level::Paren => {
                    // Keep popping until we're at an opening parenthesis.
                    if popped_level != Level::Paren {
                        self.pop(level)
                    }
                }
                Level::Colon => {
                    // If the last level wasn't a colon, push it back.
                    if popped_level != Level::Colon {
                        self.push(popped_level)
                    }
                }
            }
        }
    }

    /// Add one indentation level.
    fn indent(&mut self) {
        self.indentation += 1
    }

    /// Process a 'Next' token.
    fn next(&mut self, next: &Next) {
        match next {
            Next::Line => {
                // Clear all colon levels.
                self.levels.retain(|level| level != &Level::Colon);
                // Reset indentation.
                self.indentation = 0;
            }
            // Go back to the latest Colon level when encountering a comma.
            Next::Comma => self.pop(&Level::Colon),
        }
    }

    /// Gets the current level.
    fn level(&self) -> usize {
        self.levels.len() + self.indentation
    }
}
{{< /highlight >}}

The full code can be found [here](https://github.com/sytzez/compost/blob/master/src/lex/tokenizer.rs).

# The Tokenize Function

All of the functionality described above comes together in the `tokenize` function. It takes a string of raw code and returns the leveled tokens.

{{< highlight rust >}}
pub fn tokenize(code: &str) -> Tokens {
    let mut position: usize = 0;
    let mut level_stack = LevelStack::new();
    let mut leveled_tokens: Vec<LeveledToken> = vec![];
    let mut is_beginning_of_line = true;

    while position <= code.len() {
        // Use the next_token function to get the next token.
        let sized_token = match next_token(&code[position..]) {
            Ok(sized_token) => sized_token,
            Err(message) => // ... pass on the error
        };

        // Move forward the position we are at in the code.
        position += sized_token.1;

        if let Some(token) = sized_token.0 {
            // Check if we are still at the beginning of a line. 
            // Spaces at the beginning are not regarded as 'the line', but as markers for indentation.
            is_beginning_of_line = is_beginning_of_line && token == Token::Space;

            match token {
                // Add indentation if we have another space at the beginning of the line.
                Token::Space => if is_beginning_of_line { level_stack.indent() }
                // Push or pop levels according to the received tokens.
                Token::Down(level) => level_stack.push(level),
                Token::Up(level) => level_stack.pop(&level),
                Token::Next(next) => {
                    level_stack.next(&next);

                    if next == Next::Line { is_beginning_of_line = true }
                }
                Token::Eof => leveled_tokens.push((Token::Eof, 0)),
                // Any other tokens are regular tokens without anything to do with our levels.
                // We simply add them to our vector of tokens at the current level.
                _ => leveled_tokens.push((token, level_stack.level())),
            }
        }
    }

    leveled_tokens.into()
}
{{< /highlight >}}

This code is available [here](https://github.com/sytzez/compost/blob/master/src/lex/tokenizer.rs).

# The 'Tokens' Utility Struct

As you might have noticed, the output type of the `tokenize` function is `Tokens`. This is another utility struct which facilitates traversing the tokens in the next phase: Syntactic Analysis.

{{< highlight rust >}}
/// Provides utility functions that help traversing the tokens.
pub struct Tokens {
    tokens: Vec<LeveledToken>,
    position: usize,
}

impl Tokens {
    /// Advance to the next token.
    pub fn step(&mut self) {
        self.position += 1;
    }

    /// Whether there are more tokens left.
    pub fn still_more(&self) -> bool {
        self.position < self.tokens.len()
    }

    /// Whether the current token is deeper than the given level.
    pub fn deeper_than(&self, level: usize) -> bool {
        self.still_more() && self.level() > level
    }

    pub fn deeper_than_or_eq(&self, level: usize) -> bool {
        self.still_more() && self.level() >= level
    }

    /// The remaining tokens.
    pub fn remaining(&self) -> &[LeveledToken] {
        &self.tokens[self.position..]
    }

    /// The current token.
    pub fn token(&self) -> &Token {
        &self.tokens[self.position].0
    }

    /// The current level.
    pub fn level(&self) -> usize {
        self.tokens[self.position].1
    }
}
{{< /highlight >}}

The source of this struct is in [this file](https://github.com/sytzez/compost/blob/master/src/lex/tokens.rs).

# Conclusion

Tokenization is probably the most straightforward part of compilation. By linearly traversing the characters of the code we get a vector of tokens.
Rust's enum type is ideal for representing tokens. Using enums will come in handy at later phases because we can use `match` statements to decide what happens when a token is encountered.

The most complex part of this phase was adding 'levels', which is a unique feature of the Compost language that provides multiple syntaxes for organising the code.

If you are curious to see how I turn these tokens into an abstract syntax tree, read [Part 2: Syntactic Analysis](/blog/creating-a-compiler-for-compost-using-rust-part-2-syntactic-analysis/).
