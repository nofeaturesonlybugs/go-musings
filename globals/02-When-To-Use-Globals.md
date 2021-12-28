# When to Use Globals
In the prior article I stressed the idea that you shouldn't use globals.  Yet they are in the language so there must be a time and place for their usage.  The focus in this article is to help unravel the confusion that is created by the paradox of a language having globals and nearly every programmer using the language saying, *"Don't use globals!"*

## Immutability  
It's important to understand what is meant by `immutability` or for something to be `immutable` in the context of when it's acceptable to use globals.  Simply put if something is `immutable` it doesn't change -- it always has the same state, the same values, the same attributes, the same properties, etc.

It's also worth noting that the opposite of `immutable` is `mutable` -- if something is `mutable` then it is likely to change.

## State, Dependencies, and Immutability  
Previously when discussing why not to use globals our attention was drawn to *state* and *dependencies* -- which are the two things most novice or beginner programmers assign to globals.  *State* and *dependencies* are frequently not `immutable` -- rather they are typically `mutable`.

When either *state* or *dependencies* are stored globally you run the risk of distinct objects sharing state and behaving erroneously.  Object `A` might update a value in global state that causes object `B` to start creating errors; or if a global dependency is altered, replaced, or swapped by one area of your code another area may still be tied to the old dependency and create bugs that are hard to track and isolate.

When programmers say, *"Don't use globals!"* what they're really saying is *"Don't use globals for state or dependencies."*  However it's the `mutable` nature of *state* and *dependencies* that makes them ill-suited as global values.

Therefore when programmers say, *"Don't use globals!"* what they're really, really saying is *"Don't use globals for `mutable` data."*

1. Don't use globals for state.  
    + Put state in a container and pass it around.
2. Don't use globals for dependencies.  
    + Put dependencies in a container and pass them around.
3. You *may* use globals for `immutable` data...  
    + ...after extremely careful consideration and/or benchmarking shows a global value is a good fit.

That ends the academic discussion around when to use globals; the rest of this document presents some example cases and why they are appropriate global usage.

## `Immutable` Calculated Data  
`Immutable` calculated data is any piece of data your program needs to calculate once and will then use repeatedly during its execution.  It is `immutable` thus it will never be recalculated or altered *while* the program is running.  Further any tests you write will not have a need to alter or mutate this data -- in other words your test cases don't mutate this data to increase code coverage or for dependency injection.

Let's look at a small example function and refactor it to use a global correctly.

```go
// IsHttpPrefix returns true if str has case-insensitive prefix "http".
func IsHttpPrefix(str string) (bool, error) {
	re, err := regexp.Compile("(?i)^http.*")
	if err != nil {
		return false, err
	}
	return re.MatchString(str), nil
}
```

At first glance this is a good function, yes?  We're taking advantage of Go's ability to return multiple values from a function and we're passing the `error` upwards for the caller to handle.  However the only reason this function will even return an error is if our regexp is invalid and can not compile.  The particular regular expression is a string literal -- `"(?i)^http.*"` is hard-coded into our program and will **never** change while the program is executing.  Further there is no mechanism by which a unit test or other test software will alter the regular expression.  Therefore this regular expression represents `immutable` data.

On a performance note the regular expression is compiled every time `IsHttpPrefix` is called.  Compiling a regular expression is not free -- it costs some CPU and memory -- so constantly recompiling the same `immutable` expression is wasted resources.  *Performance tuning and micro-optimizations are topics unto themselves; when in doubt write benchmarks and make decisions from qualitative measurements.*

Regarding the `immutability` of the regexp and the wasted resources this is a good candidate for using a global.

Ideally we would use a `const` expression however none of the following are valid Go code:
```go
const re, err = regexp.Compile("(?i)^http.*")
// yields compile errors:
//  multiple-value regexp.Compile() in single-value context
//  missing value in const declaration
```

```go
const re = regexp.MustCompile("(?i)^http.*")
// yields compile error:
//  const initializer regexp.MustCompile("(?i)^http.*") is not a constant
```

The best we can do in Go is to put the compiled regexp into a global variable.  Note also that we are using `regexp.MustCompile()` which does not return an `error` but instead panics in lieu of returning the `error`.  The potential panic is acceptable here because it is something we will catch and fix during development between iterations of `go build` and program execution.

```go
var reIsHttpPrefix = regexp.MustCompile("(?i)^http.*")
```

We can now rewrite `IsHttpPrefix()` to use the global variable and -- as a bonus in this instance -- simplify its signature because the only portion of the function that *could* cause an error has been removed.

```go
// reIsHttpPrefix is a precomputed regexp for IsHttpPrefix().
var reIsHttpPrefix = regexp.MustCompile("(?i)^http.*")

// IsHttpPrefix returns true if str has case-insensitive prefix "http".
func IsHttpPrefix(str string) bool {
	return reIsHttpPrefix.MatchString(str)
}
```

## Defaults  
Another good use case for global values is when providing default values.  This is more likely to arise when you are creating a package acting as a library.

Consider you are writing a `package fantastic`.

```go
// Config contains configuration data for a Fantastic.
type Config struct {
    Something string
    Other int
}

// Fantastic is the fantastic thing this package does.
type Fantastic struct {
    // Config is used to configure a Fantastic instance.
    Config Config
}

// DoIt does something.
func (f *Fantastic) DoIt() {
    // uses f.Config
}
```

It will be used as:
```go
func main() {
    fant := &fantastic.Fantastic{
        Config: fantastic.Config{
            Something: "Hi",
            Other: 42,
        }
    }
    fant.DoIt()
}
```

But what if *most* instances of `Fantastic` can use a standard default configuration; we might provide it as a global.

```go
package fantastic

var DefaultConfig = Config{
    Something: "Hi",
    Other: 42,
}
```

```go
package main

func main() {
    fant := &fantastic.Fantastic{
        Config: fantastic.DefaultConfig,
    }
    fant.DoIt()
}
```

We can take this idea further and even make `Config` optional by turning it into a pointer.
```go
// Fantastic is the fantastic thing this package does.
type Fantastic struct {
    // Config is used to configure a Fantastic instance.
    //
    // Config=nil means DefaultConfig is used.
    Config *Config
}

// DoIt does something.
func (f *Fantastic) DoIt() {
    // use Config or DefaultConfig
    conf := f.Config
    if conf == nil {
        conf = &Config{}
        *conf = DefaultConfig // Shallow copies DefaultConfig into Conf
    }
    // Now we use only local var conf for configuration values.
}
```

And now clients don't even have to provide it.
```go
package main

func main() {
    fant := &fantastic.Fantastic{}
    fant.DoIt()
}
```

## Exported Globals vs Private Globals  
Any time you decide to use a global variable you should consider carefully if it should be exported or not.  The regular expression example is a good case for making the global unexported or private; there's low likelihood anything outside the package will need to see it to understand how the package works.

On the other hand the example demonstrating default values or configuration is a good case for exporting the global value.  If the default global is declared as `var defaultConfig = ...` then it is not visible outside the package, does not show up in documentation, and users of your package will need to dig in your source code to find the default configuration values.  Naming it `DefaultConfig` ensures it shows up in documentation and is easier to find at a glance.

## Being a Good Samaritan  
Now let us address the large elephant in the room.  

    Variables -- global or local -- are by definition variable and have no intrinsic property of being immutable.

In other words there's no such thing as an *immutable global variable*.  Any piece of code is free to mutate any global variable and there's no protection offered by the compiler or any other tool.

The entire concept of *immutable global variable* hinges on everyone using the code to be good samaritans; or said differently everyone has to act in good faith and *agree* not to mutate the variable.

## Wrapping It Up
Remember and heed the warnings about *state* and *dependencies*:
> 1. Don't use globals for state.  
    + Put state in a container and pass it around.
> 2. Don't use globals for dependencies.  
    + Put dependencies in a container and pass them around.

Which can be summarized as:
> Don't use globals for `mutable` data.

However:
> 3. You *may* use globals for `immutable` data...  

...as long as you remember *it's not really immutable* and we're all just sort of pretending it is.
