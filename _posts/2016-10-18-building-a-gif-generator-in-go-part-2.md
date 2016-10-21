---
layout: post
title:  Building a GIF Generator in Go, Part II
date:   2016-10-18
tags: [go]
category: tutorials
excerpt: In this second post we're going to update our program to accept user input.
---

<section class="callout secondary">
  <p>This is the second part of an introduction to programming with Go. To get up and running quickly, start with <a href="/2016/building-a-gif-generator-in-go-part-1">Part I</a>.</p>
</section>

In [Part I](/2016/building-a-gif-generator-in-go-part-1) we installed Go and put together the basic shell of our program. In this second post we're going to update our program to accept user input. To do that, we first need to learn about variables.

## Variables as containers
A variable in programming is a lot like a variable in algebra–it's a placeholder for some value. More accurately, a variable is a container, a box that can be empty or might hold something. Variables in programming also have "type"–that is, if variable `x` holds an integer, we say its type is "integer". Each variable in Go is given a single type, and only ever allowed to hold values of that type, so before we go further we need to know what types are available to us.

### Basic (scalar) types
The most basic types are called "scalar" types, which means it's only one of whatever it is. If our box holds only one item, it's a scalar box. Go has <a href="https://tour.golang.org/basics/11" target="_blank">several scalar types</a>, including:
* `int`, containing an integer
* `float32` and `float64`, containing floating point decimal numbers
* `bool`, containing either `true` or `false`
* `string`, containing characters "strung" together, i.e., text.

In part 3 we'll use variables that contain multiple values, but scalar types are enough for now.

### Declaration & Assignment
So how do you set the type of a variable, and how to you put a value into the box? Those two operations are known as "declaration" and "assignment". In Go, you declare a variable with the `var` keyword followed by a name and type:

```go
var someVariable int
```

You then assign a value (i.e., put something in the box) using the `=` operator:

```go
someVariable = 5
fmt.Println(someVariable)
// output: 5
```

Declaration and assignment may also be done at the same time:

```go
var someVariable int = 5
```

Now we can pass our `someVariable` to functions like `fmt.Println()`, assign new values at will, and even assign its value to other variables.

<section class="callout secondary">
  <p><strong>Important:</strong> Variables are declared only once, but may be assigned any number of times. Variables MUST be declared before use.</p>
</section>

The same syntax is used for other types as well, but for `string` you'll need to enclose the value in quotes:

```go
var someString string = "This is some text"
```

### Pointers
To build on our analogy, the "box" of a variable is really a small piece of your computer's RAM. The variable's type decides how big the box is (how much RAM to use) and what's allowed in it.

One special type of variable is called a "pointer". A pointer is also a small piece of RAM, but rather than having its own value it holds the memory address of another variable. In Go, pointers are assigned by putting an ampersand before the name of the variable to which you want to point. Compare the following declarations and assignments.

```go
var aCopyOfSomeVariable int = someVariable
var pointsToSomeVariable *int = &someVariable
```

The first assigns the value of `someVariable` to `aCopyOfSomeVariable`. It's duplicating the value, and changes to one variable won't affect the other. The second creates a pointer to `someVariable`. Using `pointsToSomeVariable`, you can actually modify the contents of `someVariable`.

<section class="callout secondary">
  <p>Notice the type of our pointer: <code>*int</code> means it points to an <code>int</code>. It's saying, "I'm not an int myself, but I can tell you where to find one."</p>
</section>

The subject of pointers is a complex one and I won't trouble you with more details now. What you need to know is that you can let a function modify one of your variables by giving it a pointer, which we'll do in just a bit.

## Accepting arguments

Our program needs to know three things:
1. The folder containing the frames for our GIF,
1. How much delay to put between each frame,
1. What to name the output file. 

Above our `main()` function, let's declare three variables named "path", "delay", and "output".

```go
var path, output string
var delay int

func main() {
// ...
```

As you can see, variable of the same type–in this case `string`–can be declared together so you don't have to write the type multiple times.

Now how do we get values into these variables? We could just hard-code values in `main()`, like so:

```go
func main() {
    path = "./some_folder/"
    output = "output.gif"
    delay = 5
}
```

But that's no good because we'll often need to change them. We need a way to get input from the user.

### The flag package
Let's use command-line flags to accept arguments from the user. To start, at the top of your file import the `flag` package from the standard library.

```go
import (
    "flag"
    "fmt"
)
```

The `flag` package has a lot of functions, and you can read the docs at <a href="https://golang.org/pkg/flag/" target="_blank">golang.org/pkg/flag/</a>. We're going to use `StringVar()` and `IntVar()`. These do all the work of parsing user input, converting it to the desired type, and putting it into the variables we specify. We'll start with setting `path`.

```go
func main() {
    flag.StringVar(&path, "p", "", "path to the folder containing images")
}
```

The arguments to `StringVar()` are:
1. A pointer to a string variable. Here we point to `path` by writing `&path`.
1. The command line flag. We've entered "p", which means the user will type `go-gif-generator -p /the/folder`.
1. The default value. We're going to require a path to be entered (more on that later), so we provide an empty string as the default.
1. A description of the argument. This is used to automatically generate help messages for the user.

Let's also set `output` and `delay`, defaulting to "output.gif" and 5 seconds. When we're done setting our arguments, we call `flag.Parse()` to parse the user input.

```go
func main() {
    flag.StringVar(&path, "p", "", "path to the folder containing images")
    flag.StringVar(&output, "o", "output.gif", "the name of the generated GIF")
    flag.IntVar(&delay, "d", 5, "delay between frames in seconds")
    flag.Parse()
}
```

That's it. Now the user can change our app's behavior on-the-fly. Put all the pieces together and try compiling.

### Validating user input
We mentioned that a path was required, but so far we've done nothing to enforce that. While we're at it, let's also make sure `delay` is something reasonable. We need to write a conditional statement: if `path` is empty, tell the user it's required and then exit.


Go has the usual bag of comparison operators: `==`, `!=`, `<`, `<=`, `>`, `>=`. These are just like what you use in math, but written with characters easily available on your keyboard. We'll use a simple `==` to check if `path` is equal to empty string, i.e., "".

<section class="callout secondary">
<p>Note that the comparison operator for equality is <code>==</code>, not <code>=</code>. The single equals sign is already used for assignment.</p>
</section>

A conditional in Go is written as `if` followed by one or more conditions. Inside curly braces you put whatever code you want to run when the condition evaluates to `true`. In our case we write:

```go
if path == "" {
   fmt.Println("A path is required")
   flag.PrintDefaults()
   return
}
```

Pretty simple: first, `flag.PrintDefaults()`–a function in the `flag` package–will print the arguments we defined above along with their descriptions. Second, since we can't continue without a path, we use `return` to stop our function and thereby exit our program.

<section class="callout secondary">
<h4>Functions &amp; Return</h4>
<p>The last thing a function does is "return" a value to whatever code is calling it. When the return happens, the function exits and any code after that line is not executed. For example, if I type <code>return 1</code>, my function stops right where it is and passes the number 1 back to the caller.</p>

<p>Some functions, like our function <code>main()</code>, don't return anything. In that case we're allowed to use a lone <code>return</code> to signal that the function execution should stop.</p>
</section>

Go gives us the boolean operators `&&` (and) and `||` (or) so that we can put multiple conditions in a single `if`. To check `delay`, we'll use `||`. We'll also not print the usage page this time:

```go
if delay < 1 || delay > 10 {
   fmt.Println("delay must be between 1 and 10 inclusively")
   return
}
```

In English this means, "if delay is less than 1 or greater than 10, complain to the user."
<section class="callout secondary">
<p>On American keyboards the | character is shift+\, found just above enter/return.</p>
</section>

## Putting it all together
Here's what our code should look like at this point:

```go
package main

import (
    "flag"
    "fmt"
)

var path, output string
var delay int

func main() {
    flag.StringVar(&path, "p", "", "path to the folder containing images")
    flag.StringVar(&output, "o", "output.gif", "the name of the generated GIF")
    flag.IntVar(&delay, "d", 5, "delay between frames in seconds")
    flag.Parse()

    if path == "" {
        fmt.Println("A path is required")
        flag.PrintDefaults()
        return
    }

    if delay < 1 || delay > 10 {
        fmt.Println("delay must be between 1 and 10 inclusively")
        return
    }

    fmt.Println("This will be a GIF generator!")
}
```

Compile it with `go install`, and then try out the different options. For practice, add a few more `fmt.Println()` calls to display the user input, or add a few more arguments to accept. In the next lesson we'll do something with all these settings!

<section class="callout secondary">
  <p>Ready to keep going? Finish your GIF generator in <a href="/2016/building-a-gif-generator-in-go-part-3/">Part III</a>.</p>
</section>