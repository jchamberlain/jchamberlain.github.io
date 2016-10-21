---
layout: post
title:  Building a GIF Generator in Go, Part III
date:   2016-10-21
tags: [go]
category: tutorials
excerpt: With just a few slices and structs, we can at last generate GIFs.
---

<section class="callout secondary">
  <p>This is the third installment of an introduction to programming with Go. To get up and running quickly, start with <a href="/2016/building-a-gif-generator-in-go-part-1">Part I</a>.</p>
</section>

In [Part II](/2016/building-a-gif-generator-in-go-part-2) we learned how to use variables and updated our program to parse command-line flags. This third post is where we finally get to the fun part: generating a GIF! Let's begin by discussing some more advanced (non-scalar) variables types.

## Non-scalar variables

### Arrays
An array is a variable holding multiple values of the same type. It has a fixed size and each element in the array is given an index starting at zero. For example, the following code declares an array of 5 integers:
```go
var things [5]int
// indexes will be 0,1,2,3,4
```

Go automatically initializes each element to its "zero value", so `things` is a list of 5 zeroes. Each element is accessible via its index both for getting and setting:
```go
fmt.Println(things[0]) // prints "0", the value of the first element
things[0] = 20 // assigns the value 20 to the first element
fmt.Println(things[0]) // prints "20"
```

To set all the values at once, write the type followed by comma-separated values inside curly braces:
```go
things = [5]int{2,4,6,8,10}
```

Go also comes with a built-in function `len()` for getting the length of an array (or slice):
```go
fmt.Println(len(things)) // prints 5
```

<section class="callout secondary">
  <p><strong>Problem:</strong> What if you need to add a 6th integer to our <code>things</code> array? Sorry, the size you give an array at declaration time is the size it's stuck with forever. Your only option for increasing the size of your array is to create a new, bigger array and then copy the values from the old array into it.</p>
</section>

### Slices
Arrays are foundational for programming in any language, but in Go you rarely interact with them directly. You're more likely to use the handy-dandy "slice". A slice in Go is basically a wrapper around an array to make working with lists of data more convenient.

As noted above, arrays never change size. Slices, however, do, and it's a neat little trick. Slices show you only part of the underlying array (hence the name "slice"), so that it *appears* to be a shorter array. Each time you add an element to a slice, Go checks whether there's enough space in the slice's underlying array. If there is, your new value is put in the array's next unused slot; if there isn't, a bigger array is created to replace the first one and your value put into it. Either way, the slice is lengthened so that you "see" more of the array. Voilà.

If you don't quite get that, don't worry. Slices will make more sense as you use them.

Slices are declared with square brackets like an array, but without a size.
```go
var someSlice []int
```
<section class="callout secondary">
  <p><strong>Warning:</strong> Unlike arrays, slices aren't ready to use at declaration time. A variable of type slice doesn't automatically contain an actual slice. You have to assign one to it.</p>
</section>

There are two ways to create the actual slice that goes into a slice variable: using curly braces (as with array) or `make()`.
```go
var someSlice []int
// Implicitly create a slice of length 3 with the given values.
someSlice = []int{1,2,3}
// Explicitly create a slice of length 3 with an underlying array of length 5.
// The last two elements in the underlying array will be hidden until needed.
someSlice = make([]int, 3, 5)
```

As with an array, you can access a slice's elements by index. The cool thing about slices is that you can add additional elements using the `append()` function. `append()` takes the slice as the first argument and the value you want added as the second argument.
```go
var slice []int = []int{1,2,3}
fmt.Println(len(slice)) // prints "3"
slice = append(slice, 25)
fmt.Println(len(slice)) // prints "4"
fmt.Println(slice[3]) // prints "25"
```
Notice how we wrote `slice = append(slice, 25)`. The `append()` function doesn't modify the slice but instead returns a new one, so always remember to reassign the result.

### Maps
Arrays and slices are great for some types of lists, but not others. Let's say you want to record your friends' birthdays. You could just write an array of their birthdays, but then you wouldn't know which birthday belonged to which friend. For these scenarios, Go gives us `map`.

Maps are like arrays/slices in that they hold multiple values, but instead of indexing by ordered numeric keys you get to choose whatever keys you want. For our example, you could use your friends' names as keys.

To declare a map, write "map", the key type in square brackets, then the value type. As with slices, you must also create and assign the actual map to the variable before use. Here's our example:
```go
// Declare and assign a map with string keys and string values.
// As with slices, we could use make() instead of {}.
var birthdays map[string]string = map[string]string{}
// Add some birthdays to our map.
birthdays["joe"] = "9/30/1993"
birthdays["michele"] = "4/1/1998"
```

### Structs
The most flexible variable type in Go is `struct`. In fact, structs are most often used when *defining your own types*–that's how flexible they are. You've probably noticed by now that every variable we've defined, even slices and maps, can hold only a single type of variable. Even if it can hold lots of values, and even if you can name them what you want, they still all have to be of the same type. A struct is a special type able to hold whatever values you want with whatever names you want.

We won't dwell long on structs here because we only interact with them a little in our program. But glance over this example of their use:
```go
type Person struct {
    Name string
    Age int
    Height float32
}

var somebody Person = Person{Name: "Bob", Age: 55, Height: 5.9}
fmt.Println(somebody.Name) // Prints "Bob"
somebody.Name = "Alyssa"
fmt.Println(somebody.Name) // Prints "Alyssa"
```

Notice how we access properties on our struct variable `somebody`: variable name, dot, property name.

## Continuing our program

Now that you know all the main types in Go, we can continue with our GIF generator.

### Getting the files
If you recall, our GIF generator knows the path from which to fetch images for our GIF, so the next thing to do is get those images.

Import the `io/ioutil` and `os` packages. The first contains <a href="https://golang.org/pkg/io/ioutil/" target="_blank">a bunch of handy helpers</a> for input/output; the second gives us <a href="https://golang.org/pkg/os/" target="_blank">some platform-independent operating system tools</a>.
```go
import (
    "flag"
    "fmt"
    "io/ioutil"
    "os"
)
```

`ioutil` comes with a function `ReadDir()` which–you guessed it–reads the contents of a directory. At the bottom of our `main()` function, let's look in `path` for the images we're going to load:
```go
func main() {
    // ...

    // fmt.Println("This will be a GIF generator!")
    var files []os.FileInfo
    var err error
    files, err = ioutil.ReadDir(path)
    if err != nil {
        fmt.Println(err)
        return
    }
}
```

First we declare our variables, most importantly `files`, which is a slice (not array) of <a href="https://golang.org/pkg/os/#FileInfo" target="_blank">`os.FileInfo`</a> structs. Then we assign the results of `ioutil.ReadDir()` to our variables, and encounter two new concepts: multiple returns and error handling.

**Multiple returns:** Notice how we got both `files` and `err` from `ioutil.ReadDir()`? Functions in Go may return multiple values separated by commas, i.e., `return thing1, thing2`. When a function does this, the caller *must* receive multiple values. If you really want to ignore a returned value, put `_` (underscore) instead of a variable name in the correct place. For example, we could have ignored the error by writing `files, _ = ioutil.ReadDir(path)`. That's bad. Don't ignore errors.

**Error handling:** It's common practice to make a function's last return value be a variable of type `error`. If no error occurred, that variable will be `nil`, which basically means it doesn't hold anything. We've done the most simple form of error handling: if it's not `nil`, print the error and then exit the function.


### Refactoring and short variable declaration
Look back at the last example. Notice how it's getting a little crowded? As your program gets bigger, you should regularly take time to clean it up before the mess gets out of hand. Such cleanup is commonly called "refactoring". Let's do our first refactoring by making our variable declarations shorter.

Up till now we've been explicitly declaring our variables with their types and separately initializing them with values, even if that means writing out the type twice. For example:
```go
var someSlice []int = []int{1,2,3}
```

That's tedious. Go gives us a special operator `:=` that implicitly declares the variable using the type of the value you provide. The previous example becomes:
```go
someSlice := []int{1,2,3}
```

It's still clear that we're making a slice of ints, but we only had to write it once. That's a win.

<section class="callout secondary">
  <p>The short declaration operator <code>:=</code> is still declaring the variable, and thus just like <code>var</code> can only be used once for a given variable name. Using <code>:=</code> on a variable that already exists will throw an error.</p>
</section>

If we use `:=`, we won't have to explicitly declare `files` and `err`. Instead, they'll be declared to use whatever types `ioutil.ReadDir()` returns, like so:
```go
func main() {
    // ...

    // fmt.Println("This will be a GIF generator!")
    files, err := ioutil.ReadDir(path)
    if err != nil {
        fmt.Println(err)
        return
    }
}
```

Our first refactor has removed two lines of unnecessary code and made our program that much cleaner.

### The for loop
`ioutil.ReadDir()` has given us a slice of files. We need to write a block of code that runs repeatedly, once for every file. We call this a "loop". Go has only one type of loop, `for`, but it can be used in several ways. I will teach you only `for` combined with `range`, which can be thought of as, "for each in this range of things, do this".

To loop over our files, printing the name of each, we write:
```go
for _, info := range files {
    fmt.Println(info.Name())
}
```

Used on an array or slice, `range` returns the index and value of each element. Since we don't need the index, we put an underscore instead of a variable name. We've called the value `info` because it's an `os.FileInfo` object/struct.

The code inside the curly braces is run one for every element in `files`, each time getting a new value in `info`.

### Creating the GIF
By now you're dying to finish, so let's grab some image processing tools, put all the pieces together, and build a GIF.

Import packages `image`, `image/color/palette`, `image/draw`, `image/gif`, and `image/jpeg` (but see explanation):
```go
import (
    "flag"
    "fmt"
    "image"
    "image/color/palette"
    "image/draw"
    "image/gif"
    "io/ioutil"
    "os"

    _ "image/jpeg"
)
```

That last one should stand out. We don't need `image/jpeg` directly, but importing it means `image.Decode()` will be able to decode JPEGs. Putting the underscore in front of a package lets us import it for side effects only, even though our code doesn't use it.

Next we need to create a new `gif.GIF` struct. This will hold all the info needed to encode our final GIF. Put this just above your `for` loop:
```go
anim := gif.GIF{}
```

We're using the short declaration `:=` again for brevity, and instantiating the struct without putting anything into it–hence the empty curly braces.

Inside your `for` loop, we need to open each file:
```go
for _, info := range files {
    f, err := os.Open(path + "/" + info.Name())
    if err != nil {
        fmt.Printf("Could not open file %s. Error: %s\n", info.Name(), err)
        return
    }
    defer f.Close()
}
```
Going step by step, we:
1. Try opening the file using <a href="https://golang.org/pkg/os/#Open" target="_blank">`os.Open()`</a> `os.Open()` returns a pointer to an <a href="https://golang.org/pkg/os/#File" target="_blank">`os.File`</a> object and an error, if any.
2. Check for an error before using the opened file. In case of an error, we print a custom message using <a href="https://golang.org/pkg/fmt/#Printf" target="_blank">`fmt.Printf()`</a>, which takes a format string with placeholders in it, followed by the values with which to replace the placeholders. `%s` means "treat the value to go here as a string". The first `%s` will be replaced with the result of `info.Name()`; the second `%s` will be replaced by the value of `err`.
3. Make sure the file gets closed when we're done with it. Keeping a file open ties up resources, so we need to close it. But the code below this point may take several different paths, which may mean having to close it in multiple places. Go gives us the `defer` statement for this situation. `defer` takes a function call to be run as the very last thing before a function exits, allowing us to close the file at the end of `main()` no matter what else happens.

<section class="callout secondary">
  <p><strong>Note:</strong> We've put a new line character <code>\n</code> at the end of our message in <code>fmt.Printf()</code>. As the name implies, <code>fmt.Println()</code> prints your string followed by a newline, so that it won't get jumbled together with whatever comes next. <code>fmt.Printf()</code> doesn't, so we add one ourselves</p>
</section>

Next we need to convert the images we've opened to use a palette, and then add them to our `gif.GIF` object. After `defer` and inside your `for` loop, skip a line line and add the following:
```go
palleted := image.NewPaletted(img.Bounds(), palette.Plan9)
draw.FloydSteinberg.Draw(palleted, img.Bounds(), img, image.ZP)

anim.Image = append(anim.Image, palleted)
anim.Delay = append(anim.Delay, delay*100)
```
In the first line we create a new, empty image the same size as `img` and call it `paletted`. Next we draw the contents of `img` on top of `paletted`, effectively copying the picture from one to the other.

The third line adds `paletted` to the `anim.Image` slice, which is the list of frames to put in the GIF. Last, we add the delay for this frame to the `admin.Delay` slice. (`anim.Delay` is in hundreds of a second, so we multiple `delay` by 100.)

And FINALLY, we'll output a GIF with these three lines (place them at the end of `main()`, outside the loop):
```go
f, _ := os.Create(output)
defer f.Close()
gif.EncodeAll(f, &anim)
```
Remembering that `output` contains the output filename as given by the user, this should all make sense. `gif.EncodeAll()` takes the contents of `anim`, encodes it as a GIF, and puts the result in `f` which points to our file.

## And we're done!
Here's what your finished program should look like:
```go
package main

import (
    "flag"
    "fmt"
    "image"
    "image/color/palette"
    "image/draw"
    "image/gif"
    "io/ioutil"
    "os"

    _ "image/jpeg"
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

    // fmt.Println("This will be a GIF generator!")
    files, err := ioutil.ReadDir(path)
    if err != nil {
        fmt.Println(err)
        return
    }

    anim := gif.GIF{}
    for _, info := range files {
        f, err := os.Open(path + "/" + info.Name())
        if err != nil {
            fmt.Printf("Could not open file %s. Error: %s\n", info.Name(), err)
            return
        }
        defer f.Close()
        img, _, _ := image.Decode(f)

        // Image has to be palleted before going into the GIF
        paletted := image.NewPaletted(img.Bounds(), palette.Plan9)
        draw.FloydSteinberg.Draw(paletted, img.Bounds(), img, image.ZP)

        anim.Image = append(anim.Image, paletted)
        anim.Delay = append(anim.Delay, delay*100)
    }

    f, _ := os.Create(output)
    defer f.Close()
    gif.EncodeAll(f, &anim)
}
```

Run `go install`, put some images in a folder, and make a GIF.

### Next steps & Further Reading
<a href="https://gobyexample.com/" target="_blank">Go by Example</a> covers a lot more ground than this tutorial in easy-to-follow examples with simple explanations, while the official <a href="https://tour.golang.org/welcome/1" target="_blank">Tour of Go</a> takes you piece-by-piece through all of Go. <a href="https://golang.org/doc/effective_go.html" target="_blank">Effective Go</a> is geared more toward experienced developers, but is still a must-read. Get what you can from it now, and keep going back as you mature in your programming.

In the meantime, exercise your skills by improving what you just wrote:
* The call to <a href="https://golang.org/pkg/image/#Decode" target="_blank">`image.Decode()`</a> doesn't check the returns, so anything using `img` might break. The same goes for our use of `os.Create()` when saving our output. Can you make the code more robust by handling any possible errors?
* We can read JPEG's but not PNGs. Can you import `image/png` to fix that?
* Our loop over `files` tries to open all files, even non-images. Can you check the file's extension in an `if` clause before continuing? (Hint: the <a href="https://golang.org/pkg/strings/" target="_blank">strings</a> package has a `HasSuffix()` function that might help.)

As always, feel free to ask for help in the comments. Happy coding!