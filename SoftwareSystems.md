# Writing Software Systems

At the most basic level, an R program, like any other program is a sequence
of instructions written to perform a task. Programs consist of data 
structures, which hold data, and functions, which define things a program
can do. You are already familiar with the native R data structures: vectors,
lists, data frames, etc. And you have already seen the functions that 
access and manipulate these functions. However, as you design your 
own systems on top of R you will eventually want to create your own
data structures.  After these new types are defined you may want 
to create specialized functions that operate on your new data structures. 
In other cases you may want to extend existing systems to take advantage
of your new functionality. This chapter shows you how to build new 
software systems that can "plug into" R's existing functionality
and allows other users to extend your new capabilities.

Data structures are generally associated with a set of functions that
are created to work with them. The data structures and their functions
can be encapsulated to create classes. Classes help us to compartmentalize
conceptually coherent pieces of software. For example, an R vector is
class holding a sequence of atomic types in R. We can create an instance 
a vector using one of R's vector creation routines.

    x <- 1:10
    length(x)

The variable x is an object of type vector. Where the class
describes what the data structure will look like an object is an actual
instance of that type. 
Objects are associated with functions that let us do things like access and
manipulate the data held by an object. In the previous example the
length function is associated with vectors and allows us to find out how many 
elements the vector holds.

R provides three different constructs for programming with classes, also 
called object oriented (OO) programming, S3, S4, and R5. The first two S3 and S4
are written in a style called generic-function OO. Functions that may be
associated with a class are first defined as being generic. Then
methods, or functions associated with a specific class, are defined much
like any other function. However, when an instance of an object is passed
to the generic function as a parameter, it is dispatched to its associated 
method. R5 is implemented in a style called message-passing OO. In this style
a methods are directly associated with classes and it is the object that
determines which function to call.

For the rest of this chapter we are going to explore the use of S3, S4, and
R5 to generate sequences. Along with building a general system
for generating sequences we are going to create classes that generate 
the Fibonacci numbers, one by one. As you probably already know, the 
Fibonacci numbers follow the integer sequence

    0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144...

and are defined by the recurrence

    F(0) = 0
    F(1) = 1
    F(k) = F(k-1) + F(k-2). 

These numbers can easily be generated in R using the familiar vectors and 
functions that you already know. An example of how to do this is 
provided below. It's important to realize that the techniques shown in this
chapter will not allow you to express algorithms your couldn't express 
with R's native data structures and functions. The techniques do allow you
to organize data structures and functions to create a general system
or framework for generating sequences. 

    fibonacci <- function(lastTwo=c()) {
      if (length(lastTwo) == 0) {
        lastTwo <- 1
      } else {
        lastTwo <- c(lastTwo, sum(lastTwo))
        if (length(lastTwo) > 2) {
          lastTwo <- lastTwo[-1]
        }
      }
      return(lastTwo)
    }

    # Get the first 10 fibonacci numbers
    fibs <- fibonacci()
    for (i in 1:10) {
      print(tail(fibs, 1))
      fibs <- fibonacci(fibs)
    }

Creating a general framework for sequences has two advantages. First, it allows
for abstraction. In our example we've defined a vector to hold the last two
values in the Fibonacci sequence along with a function that gets the next
value in the sequence. By realizing that any integer sequence that we might like
to generate can be expressed computationally as data, the last two value
for the Fibonacci sequence, and a function to get the next value. We've 
identified the essential pieces generating sequences. From here we can 
start thinking about the types of things we might like to do with any sequence,
not just the Fibonaccis. Second, we can make our system extensible. That is,
we can write code for other types of sequences
that work within our framework. Extensibility allows you to
create new sequences, like the factorial numbers, based on the abstract 
notion of a sequence. It will even allow others to define their own sequences
that will work within our sequence framework.

## S3

S3 is R's first system for class system. It was first described in the
1992 "White Book" (Chambers & Hastie, 1991) and it is the only object
system used by the base R installation. In this system, new data types or 
classes are built from native types (vector, list, etc.) but they are given 
a `class` attribute. This is a character vector of class names and you should
note that a single object can have multiple types.
Recalling that in the last section the data data needed to create a
Fibonacci sequence was a vector of size two, we can create a new
data type, called FibonacciData to hold these values:

    # Create a FibonacciData object using attributes
    x <- vector(mode="integer")
    attr(x, "class") <- "FibonacciData"
    x

    # using the structure function

    x <- structure(vector(mode="integer"), class="FibonacciData")
    x

    # using the class function

    x <- vector(mode="integer")
    class(x) <- "FibonacciData"
    class(x)
    # [1] "FibonacciData"

While it is true that a class is simply an attribute it is recommended that 
when you access and modify class information you use the `class` function. It
communicates your intent more clearly, making your code easier to read. 
Furthermore, it is often better to create a function to create 
instances of a class, rather than simply attaching attributes ad-hoc.
The functions below are called __constructors__ and they create an object
of type `SequenceData` and an object of type `FibonacciData`, which is
also of type `SequenceData`.

    SequenceData <- function(x=NULL) {
      r <- structure( vector(mode="integer"), class="SequenceData" )
      if (!is.null(x)) {
        r <- x
      }
      r
    }

    FibonacciData <- function(x=NULL) {
      r <- SequenceData(x)
      class(r) <- c("FibonacciData", class(r))
      r
    }

By defining data types we can create special functions,
called methods that behave differently depending on the type of the object
passed to the method. For example, let's say that we want to be able 
to handle the generation of all any type integer sequences with a method, called
`nextNum`. The `nextNum` function will return a an object, which
could be a `FibonacciData` object, and from the returned object we
get get the next value in the sequence. This is easily accomplished 
by creating __generic functions__, which will allow us to define 
a `nextNum` and `value` method for different types of sequences. 

    nextNum <- function(x) {
      UseMethod("nextNum", x)
    }

    value <- function(x) {
      UseMethod("value", x)
    }

Both of these generic functions take a single parameter `x` and pass the
name of the function and the parameter to the `UseMethod` function. The 
first argument of `UseMethod` registers the `nextNum` and `value` functions
as generic functions; essentially letting R know that they are generic 
functions and calls to `nextNum` and `value` need to be handled as such.
The second argument to UseMethod says that specific methods will be called,
or __dispatched__, based on the type of the variable `x`.
Now that the generic function has been defined we can define methods, 
called `nextNum` and `value` which each take an object of type 
`SequenceData` or `FibonacciData` and performs the appropriate operation.

    nextNum.SequenceData <- function(x) {
      stop("You can't call nextNum on an abstract SequenceData type")
    }

    value.SequenceData <- function(x) {
      stop("You can't call value on an abstract SequenceData type")
    }

    nextNum.FibonacciData <- function(x) {
      # The class of the return vector needs to be "FibonacciData".
      # We can do this by passing it to the constructor.
      FibonacciData(c(tail(x, 1), ifelse(!length(x), 1, sum(x))))
    }

    value.FibonacciData <- function(x) {
      ifelse(length(x) == 0, 0, tail(x, 1))
    }

A method name starts with the corresponding generic function name, followed by
a ".", followed by the type of the parameter.
The `UseMethod` function uses the class of `x` to figure out which method to 
call. If `nextNum` or `value` is called and `x` has more than one class, 
as it does in this case `UseMethod` will look for methods in the
same order that the classes appear in the class attribute. It should
be noted in this example that the `SequenceData` type 
categorizes a broad range of things, in this case sequences.
It also allows us to define but not implement operations which can be 
performed on any sequence. The `FibonacciData` type is a specific type
of `SequenceData`, and needs to implement its own methods for `nextNum` and
`value`.  When this is complete we can use `FibonacciData` objects
much like the familiar data structures and functions.

Technical note: After `UseMethod` has
found the correct method it uses the same evironment as the generic
function. So any assignment or evaluations that were made before the
call to `UseMethod` will be accessible to the method.


    a <- FibonacciData()
    fibs <- rep(NA, 10)
    for (i in 1:10) {
      fibs[i] <- value(a)
      a <- nextNum(a)
    }
    print(fibs)
    # [1]  0  1  1  2  3  5  8 13 21 34

As mentioned before, the base R installation makes heavy use of S3 methods, just
like the ones we've been creating. This means that we can create methods for 
standard R functions, allowing our new data types to act the same as R's native
types. In the example below we'll create a new method for R's `print` function,
which takes as an argument a `SequenceData` object and prints its value.

    print.SequenceData <- function(x, ...) {
      print(value(x))
      return(invisible(x))
    }

    fib <- FibonacciData()
    print(fib)

In this case an object of type `FibonacciData` is created, which also
has type `SequenceData`. The `print(fib)` generic function call dispatches
to the `print.SequenceData` method. In this method, the `value()` method
is called, which is dispatched to `value.Fibonacci` since it appears first
in the parameters vector of classes. This functionality is called polymorphism
and it allows us to create the `print.SequenceData` method based on an
__abstract__ type `SequenceData`.  However, the method works as expected
when it passed a __concrete__ type, in this case a `FibonacciData` object.

## Advanced S3 

## S4

## Closures

## R5