---
layout: post
title:  Building a GIF Generator in Go, Part I
date:   2016-10-13
tags: [go]
category: tutorials
excerpt: If you want to start programming and haven't yet, try learning in Go.
---

Want to learn how to program? I started in my mid-teens by borrowing Java books from the library, and I still find that the thoroughness of a book brings a huge advantage over learning online. That said, I never went past the basics with Java. Teenager me got tired of printing to a terminal and could see no other use of programming in the next 100 or so pages. I eventually switched to JavaScript and PHP so that I could take a break from the theoretical and start doing something useful and fun.

I want to help you avoid that scenario by throwing you right in. Rather than teach you all the theory of programming and make you wait a month or two to apply it, let's start by building a GIF generator!

* In Part I you'll learn the most basic syntax and constructs of Go, and build the most simple command-line utility.
* In Part II you'll learn about variables and Go's type system, and update your utility to accept user input.
* In Part III you'll take that user input to turn a folder of images into an animated GIF, along the way learning the rest of Go's fundamental types and behavior.

I assume you know your way around the computer, including use a terminal, but haven't programmed before. If you find something unclear or have a question, please bring it up in the comments.

Use this to get yourself motivated and off the ground, and then go read a more thorough book or two. Learn the theory and its application, and don't forget that programming is fun!

## Installing Go
The first step is to install Go. You'll get a command line tool that lets you run things like `go install` to compile your code, and the whole standard library of built-in packages.

Start by downloading the appropriate Go installer from <a href="https://golang.org/dl/" target="_blank">golang.org/dl</a>. Or if you're on a Mac and already have <a href="http://brew.sh/index.html" target="_blank">Homebrew</a> installed, you can run `brew install go` instead. Follow the instructions and open a command line when you're done. Run `go version` to make sure it's installed correctly. You should see something like this:
```
$ go version
go version go1.7 darwin/amd64
```

Next you'll need to setup a "workspace"–a folder to hold your code, code you've downloaded, and compiled binaries all in one place.

### Mac workspace
Create a folder in your home directory. Here I'll call it "go", but use any name you want.
```
$ mkdir ~/go
```
Open or create the file ".bash_profile" in your home directory, and add the following lines at the end (changing "go" to whatever name you used above):
```
export GOPATH=~/go
export PATH=$GOPATH/bin:$PATH
```
The first line exports an environment variable telling Go to use that folder as your workspace. The second line updates your `$PATH` environment variable so your shell knows to look for binaries in your Go workspace, letting you more easily run the code you compile.

### Windows (10) workspace
In Windows you need to do the same thing as on a Mac, just in a different way. Create a folder in your home directory. Here I'll call it "go", but use any name you want.
```
C:\Users\IEUser> mkdir go
```

Next open up your control panel and click on "System and Security", then "System". In the left menu you should see a link to "Advanced System Settings". Click it.
<img src="{{ site.baseurl }}/img/2016/windows-system-menu.png" alt="Windows system menu" class="float-right" />

At the bottom of the next screen will be an "Environment Variables" button. Click that, and you should see a list of variables. Add a new one called `GOPATH`, and set its value to `%USERPROFILE%\go` (or whatever folder you used for your workspace). Update the `PATH` variable by prepending `%USERPROFILE%\go\bin;`. Save your changes.

<section class="callout secondary">
<p><strong>Warning: Don't use Notepad for programming.</strong> It doesn't understand line endings from other operating systems, and will generally be difficult to work with. Use something like <a href="https://notepad-plus-plus.org/download/v7.html" target="_blank">Notepad++</a> or <a href="https://atom.io/" target="_blank">Atom</a>.</p>
</section>

As I'm using a Mac myself, all the following command line examples will be for the Mac. However, you should be able to easily adjust them to work on Windows. The `go` tool itself will work the same on either operating system.

### Testing it out
The environment variables you've added above will apply to the next shell you open, **but haven't affected your current session**. Close and reopen your terminal.

In your new terminal, run `go env`. In the output, you should see a "GOPATH" variable pointing to the directory you created:
```
...
GOOS="darwin"
GOPATH="/Users/joshuachamberlain/go"
GORACE=""
...
```
If you don't see it, double-check the steps above. If you still don't see it, leave a comment below.


## Structure of a Go program
Now you're ready to start programming! Well, almost.

### Packages
Let's talk about packages and how they're stored. Go organizes code in "packages". What you're about to build can be considered a package, and thousands of others are available online.

Packages are stored in your workspace at `$GOPATH/src/`, and organized into folders named according to the URL at which each package can be downloaded. For instance, if your code is available at `https://github.com/jchamberlain/my-cool-thing`, it should be stored locally in `$GOPATH/src/github.com/jchamberlain/my-cool-thing/`. Packages can have sub-packages, so you might also have `my-cool-thing/thing1/`.

<section class="callout secondary">
<p>You don't have to put your code online, and you can name your source folders anything, so long as they exist in `$GOPATH/src/`. The final folder determines the default name of your program.</p>
</section>

Since you're about to build your own package, create a new folder for your project. Throughout this tutorial I'll be using the folder `$GOPATH/src/github.com/jchamberlain/go-gif-generator/`. This will result in a program called `go-gif-generator`.

### Imports and func main()
Now open up a text editor. At the very top of the file, tell Go that this is the main sub-package of your project:
```go
package main
```

The "main" package of any project is what Go turns into an executable program.
<section class="callout secondary">
<p>Note that there's no semicolon. Go discourages semicolons except where necessary to put multiple statements on a single line.</p>
</section>

Next we'll skip a line and import a package called `fmt` from the standard library to help us print text:
```go
import "fmt"
```

The `import` command can also import multiple packages at the same time:
```go
import (
    "fmt"
    "strings"
)
```

But where it all begins is a function called "main". The "main" package must have a "main" function for Go to call to start it all off. Skip a line and add one that prints a line of text:
```go
func main() {
    fmt.Println("This will be a GIF generator!")
}
```

As you see, functions in Go are defined with the `func` keyword followed by a name of your choosing and a pair of parentheses. The body of the function is a block of any valid Go code–even more function definitions–indented within a set of curl braces. The block of code is run when the function is "called". To call a function defined by `func doSomething() {}`, simple type `doSomething()`.

Go is calling `main()` for us, but we're also calling a function ourselves: `Println()` as provided by the `fmt` package.
<section class="callout secondary">
<p>To use a function, type, or variable published by an imported package, type the package name, a period, and then the function/type/variable.</p>
</section>

Let's look at it all together:
```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    fmt.Println("This will be a GIF generator!")
}
```
Save this file as `main.go` in your package's folder.

## Compiling and running Go code
Now at last we can compile our code. The Go compiler will take our code along with whatever libraries/packages we're using, make sure it's valid, and convert it into an executable binary. The binary will automatically be put into `$GOPATH/bin` which, if you recall, we added to our `$PATH` so we could run it just like any other program on the command line.

If you're not there already, go to your project's directory in a terminal. Then run `go install`.
```
$ cd $GOPATH/src/github.com/jchamberlain/go-gif-generator/;
$ go install;
```

Oops! If you've followed my instructions exactly, you'll be seeing a compiler error:
```
# github.com/jchamberlain/go-gif-generator
./main.go:5: imported and not used: "strings"
```

Go has refused to compile our code because we included the `strings` package unnecessarily. Go's compiler is picky in a really good way. Even though this error isn't a bug per se, it is the kind of mistake that will make our code harder to maintain in the long run. There are lots of other errors the compiler will tell you about, and you'll come to enjoy reading them and begin to see your own coding improve as you make the same mistakes less and less.

<section class="callout secondary">
<p>The "./main.go:5" means the error is on line 5 of "main.go" in our current directory.</p>
</section>

Let's remove "strings" from the import block. Our code should now look like this:
```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("This will be a GIF generator!")
}
```

Now we can run `go install` without any errors, and try running our program:
```
$ go install
$ go-gif-generator
This will be a GIF generator!
```

Congratulations! You've successfully entered the world of programming. We'll start putting meat on these bones in Part II.

If it didn't work, or if you have questions, leave a comment below.

