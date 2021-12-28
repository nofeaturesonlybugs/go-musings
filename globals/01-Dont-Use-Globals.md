# Don't Use Globals  
This article is written for the Go programmer who has learned how to write small, basic programs but has done so using global variables.  You've heard that global variables are bad -- although you don't fully understand why -- and you want to organize your code better to remove the globals.

Let's look at a small pseudocode program that uses some global variables:  

```go
package main

import ...

var (
    // DB is our database handle.
    DB *sql.DB
    // Ctx is a context used to end the program or cancel requests.
    Ctx context.Context
    // Conf is a configuration object
    Conf Config
)

func main() {
    DB = ... // Connect to the database
    Ctx = ... // Create a context of sorts, perhaps with a timeout or attached to ^C
    Conf = ... // Load a configuration from env vars, file, etc.
    // At this point our code would branch into one of the functions below.
}

func Bar() {
    err := DB.ExecContext(Ctx, "...", Conf.Something, Conf.SomethingElse)
    // check err -- do more stuff
}

func Foo() {
    err := DB.ExecContext(Ctx, "...", Conf.Value, Conf.OtherValue)
    // check err -- do more stuff
}
```

## State and Dependencies  
Global variables usually represent one of two things: *state* or a *dependency*.

### State  
*State* represents some data or value that is used to drive a program's logic.  For example `bytesRemaining` might be a stateful value while reading a file.  Or `hadError` could be a piece of state for an operation that has many steps.  *State* changes over time and periodically the current state is inspected to make decisions.

In our small example above `Ctx` could be considered a stateful value that is used to indicate if the program is canceled or not.

### Dependencies  
*Dependencies* are entities or services a software *depends* on in order to function.  In order to communicate with a database you need a `*sql.DB` thus making it a dependency.

The global variable `DB` is a *dependency* in the above example program -- the program *depends* on it in order to function.

## Why Shouldn't You Use Globals  
There's a handful of reasons why you shouldn't use globals:
1. It's *incredibly easy* to organize your code to *not* use them; if something is easy to do then why not just do it?
2. Global variables hurt readability and maintainability as a program grows; they are a constant source of bugs and logic errors in larger programs.  In concurrent programs -- one of Go's strong points -- they can start to introduce race conditions.
3. Global variables are an impedence to testing your code with unit tests or integration tests.  A key concept in writing tests is to *swap* or *alter* state and dependencies as part of a test's execution; it's hard to do this when code depends on globals.

The third point about testing and testability is a bit hard to digest unless you've actually written test code or have experience with test suites.  The second point about readability and maintainability is experienced by programmers who ignore the warning about global variables and forge ahead stubbornly -- insisting they'll "fix it later" but keep using them "for now."

So let us turn our attention to the first point about how removing them is easy so why not just do it.

## Containers  
There's two simple steps to removing *nearly all* global variables from your code:
1.  Take all your global variables and put them in a container where a container is a `type T struct{}`.
2.  Change all your *functions* to *methods* on this container.

Let's refactor our example to use a container.

```go
package main

import ...

// The global vars are now fields in the container.
type T struct {
    DB *sql.DB
    Ctx context.Context
    Conf Config
}

func main() {
    // Create an instance of the container and assign to its fields.
    t := &T{
        DB: ...,
        Ctx: ...,
        Conf: ...,
    }
    // At this point our code would branch into one of the functions below.
}

// Both Bar() and Foo() are methods on *T -- they can access its fields.
// Change instances of DB, Ctx, and Conf to t.DB, t.Ctx, and t.Conf to access the fields.

func (t *T) Bar() {
    err := t.DB.ExecContext(t.Ctx, "...", t.Conf.Something, t.Conf.SomethingElse)
    // check err -- do more stuff
}

func (t *T) Foo() {
    err := t.DB.ExecContext(t.Ctx, "...", t.Conf.Value, t.Conf.OtherValue)
    // check err -- do more stuff
}
```

1. Instead of a global `var()` section we now have a `type T struct{}` and the variables have become struct fields.
2. Each function is now a method due to having a receiver `func (t *T)`.
3. References to `Ctx`, `Conf`, and `DB` are changed to `t.Ctx`, `t.Conf`, and `t.DB` in order to use the struct fields rather than globals.

One of the first things you should do when writing a new program is declare your `type T struct` -- usually before `func main()` -- and as your program needs more *state* or *dependencies* add them as fields in your container.  In general most of the functions in your package should probably be methods on some type of container.  If you're writing a package with many "free floating" functions then you *may* be making a design mistake unless your package is a library similar to `package strings` or `package math`.

## Pick a Good Name  
`T` isn't a very good name for a container -- I've just been using it as a placeholder for now.  However you *should* pick a good name that semantically describes the contained data.  Since this container is in `package main` and `main` always represents a compiled binary a better name would be any of the following: `Program`, `Application`, `App`, etc.  I like short names where appropriate so let's refactor as `App`:

```go
type App struct {
    DB *sql.DB
    Ctx context.Context
    Conf Config
}

func main() {
    app := &App{
        DB: ...,
        Ctx: ...,
        Conf: ...,
    }
}

func (app *App) Bar() {
    err := app.DB.ExecContext(app.Ctx, "...", app.Conf.Something, app.Conf.SomethingElse)
}

func (app *App) Foo() {
    err := app.DB.ExecContext(app.Ctx, "...", app.Conf.Value, app.Conf.OtherValue)
}
```

Earlier I said one of the first things you should do when writing a program is to create your container -- `type App struct{}` for a `package main`.  But what about other packages?  Well -- do the same thing but name it with the package's purpose in mind.  If you're creating a `package messaging` then maybe your container should be named `Broker` or `Messenger`; if you're creating a database layer then `Store` or `Storage` might be a good name.

For example let's presume we're going to refactor our sample program and move `Bar()` and `Foo()` into `package amazing`.

```go
package amazing

// Amazing is our "container" for package amazing.
type Amazing struct {
    DB *sql.DB
    Ctx context.Context
    Conf Config
}

func (a *Amazing) Bar() {
    err := a.DB.ExecContext(a.Ctx, "...", a.Conf.Something, a.Conf.SomethingElse)
}

func (a *Amazing) Foo() {
    err := a.DB.ExecContext(a.Ctx, "...", a.Conf.Value, a.Conf.OtherValue)
}
```

Now `package main` will import and use `amazing`:
```go
package main

import ... // Assume amazing is imported

type App struct {
    DB *sql.DB
    Ctx context.Context
    Conf Config
    Amazing *amazing.Amazing
}

func main() {
    // Create local vars to promote reuse during initialization.
    db := ... // Create the db
    ctx := ... // Create the context
    conf := ... // Load the config
    app := &App{
        DB: db,
        Ctx: ctx,
        Conf: conf,
        Amazing: &amazing.Amazing{
            DB: db,
            Ctx: ctx,
            Conf: conf,
        },
    }
    // Code branches to app.Amazing.Bar() or app.Amazing.Foo()
}
```

It looks a bit silly to pass the same values into both `app` and `app.Amazing`.  In practicality a larger program will create instances of several different types of containers and each container will need only some or part of the values contained in `App`.

The key points are:
1. Use containers to store state and dependencies.
2. Name your containers appropriately.
3. Pass state and dependencies between containers as appropriate.

## Testing Revisited  
Now that we have `package amazing` I want to quickly revisit one of the points I made about globals hurting testability:

> 3. Global variables are an impedence to testing your code with unit tests or integration tests.  **A key concept in writing tests is to *swap* or *alter* state and dependencies as part of a test's execution;** it's hard to do this when code depends on globals.

Let us presume that `Ctx` is an *optional* field of the `Amazing` type.  In order to test that `package amazing` works with different types of `Ctx` values we might write a small set of tests:

```go
package amazing_test

import ... // snipped

func TestAmazing_Ctx(t *testing.T) {
    db := ... // Get a db
    conf := ... // Load conf
    t.Run("background", func(t *testing.T) {
        a := &amazing.Amazing{
            DB: db,
            Conf: conf,
            Ctx: context.Background(),
        }
        // now do stuff with a and test the results
    })
    t.Run("nil", func(t *testing.T) {
        a := &amazing.Amazing{
            DB: db,
            Conf: conf,
            Ctx: nil,
        }
        // now do stuff with a and test the results
    })
    t.Run("2s deadline", func(t *testing.T) {
        ctx, cancel := context.WithDeadline(context.Background(), 2 * time.Second)
        defer cancel()
        a := &amazing.Amazing{
            DB: db,
            Conf: conf,
            Ctx: ctx,
        }
        // now do stuff with a and test the results
    })
}
```

Each instance of `t.Run(...)` is running a new test where the `Ctx` field of `Amazing` is a different value.  Each test is creating its own instance of `Amazing` and we have a much stronger guarantee that our tests are not interferring with each other.  When a package uses lots of global variables it starts to become unclear in tests just how much of the state should be reset between tests or how one test could affect another.  The net effect of globals is your tests become difficult to maintain or they may lose their determinism -- you never want your tests to be indeterministic.

## Wrapping It Up

Don't.  Use.  Globals.