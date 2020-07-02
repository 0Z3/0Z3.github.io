---
layout: page
title: "/home"
---

Ose is an [Open Sound Control](http://opensoundcontrol.org)
implementation and runtime server environment.

Ose provides everything necessary to add OSC support to your project.
Additionally, Ose comes with a small but powerful OSC-based Virtual Machine 
that allows dynamic interaction with your application via an OSC interface.

Ose runs in:
* [web browsers]()
and [node.js](),
* [Max/MSP]()
and other [media programming environments](),
* [Arduino]()
and other [microcontrollers](),
* and lots of other environments.

# Download

Source code is available on GitHub: 
[https://github.com/0Z3/ose](https://github.com/0Z3/ose)

Javascript can be built from the C source code using 
[emscripten](https://emscripten.org/).

It is also available for [nodejs](https://nodejs.org)
via [npm](https://npmjs.org): 
[https://www.npmjs.com/package/osejs](https://www.npmjs.com/package/osejs)

# Build

To build the REPL, on Mac OS X or Linux, you should be able to cd to the 
main directory and type `make debug`. To build the Javascript files,
make sure have [emscripten](https://emscripten.org/) installed, and then
you should be able to run `make js`.

# Introduction

Ose provides a few different interfaces for creating and manipulating OSC data.
The two most important are the *OSC Server*, and the *API*. The API is what 
programmers can use to add OSC support to their applications---it can be used
to assemble and process OSC bundles according to the needs of an application.
The server can be used to provide a level of dynamism to users of an 
application.

We'll start with the server interface and then show how it and the API can be 
used in a variety of applications.

The Ose REPL is a runtime environment for the server, and is the easiest way
to try out the examples below. In a terminal, `cd` to the Ose directory, and 
type `./ose` or `./ose --verbose`.

## OSC Server

The OSC server is really an OSC interface to a Virtual Machine that operates on
OSC bundles. The VM consists of a number of bundles that can be manipulated by
sending OSC messages that the server understands. The most important of these
bundles is the *stack*; we push values onto the stack and apply functions
to them, all by sending OSC messages to the server. 

Some notes about terminology:
* The term *server* doesn't necessarily imply remote or asynchronous 
communication.
* We will continue to refer to the *stack*, but it's 
important to be clear that the stack is just an OSC bundle---there is nothing
special about it, and it is one of many that constitute the context of the 
OSC Server / VM. 
* Keeping with the [OSC spec](http://opensoundcontrol.org/spec-1_0), we refer to
the things that bundles contain as *elements*, which can be either *messages*
or *bundles*. Further, we will refer to the things that *elements* contain
as *items*. 
* The *topmost* element of the stack is the element furthest from the 
*beginning* of the bundle (i.e., the #bundle identifier). 
The *second* element of the stack is the one just below the *topmost*, and so on.

### Values

In order to push values onto the stack, we send messages to the server in
the form of `/<typetag>/<literal>`, where `<typetag>` is either 
`i` for integer, 
`f` for float, 
`s` for string, or
`b` for a binary blob, 
and `<literal>` is a string representation of the value. 

Some examples:
```
/i/10
/f/3.14159
/s/hello world
/b/01FE
```

### Functions

We can operate on values that are on the stack by sending messages of the form
`/!/<function>`. So, if we want to add two numbers together, say 3 and 5, 
we would send the following sequence of messages:
```
/i/3
/i/5
/!/add
```
and the result, 8, would be left on the stack.

### Assignment

In OSC, there are no "bare" values like ints, floats or strings that aren't
part of a message, so when we say that we are 
"pushing the value 3 onto the stack" 
by sending the message `/i/3`, what is really happening is that we are adding 
a message to the stack bundle that has an empty address field and a single 
integer value.

We can give values an address and copy them into the environment by sending
a message of the form `/@/<name>`, where `/name` is the address we would like
to assign to. `/@` pushes all elements on the stack onto a single message, so
the following sequence of messages
```repl
/i/10
/f/3.14159
/s/foo bar
/@/thing
```
would push a message with address `/thing` into the environment, with arguments
10, 3.14159, and "foo bar", and clear the stack. There are a number of ways
to add entries to the environment which we will see later.

### Lookup

Messages that have been placed in the environment through assignment can be 
retreived by replacing the `/@` prefix with `/$`, e.g. `/$/xyz`. 

### Summary

Values can be pushed on the stack by sending literals prefixed by one of either
`/i`, `/f`, `/s`, or `/b`. Even though we keep calling them "values", they're
really just messages with empty addresses. There is nothing special about the 
fact that the address is empty, it's simply that way because it wasn't 
specified.

`/!/<func>`
: means "apply `/<func>` to the elements on the stack."

`/@/<address>`
: means "assign the values on the stack to `/<address>`."

`/$/<address>`
: means "find `/<address>` in the environment and copy it to the stack."

Each of the message prefixes that we have seen so far (`/i`, `/f`, `/s`, `/b`,
`/!`, `/@`, and `/$`) can all be sent on their own, i.e., with no other 
information in the address. If this is done, the server expects to find the 
part of the address that would have followed the prefix on top of the stack. 
For example, sending `/i` to the server will cause the topmost item to be 
converted into an int. Similarly, if you send `/!`, the server will expect to 
find the name of a function on top of the stack.

### Stack Manipulation

The stack manipulation operations were inspired by those of
[Forth](https://en.wikipedia.org/wiki/Forth_(programming_language)), so you'll
find many functions with the same or similar names.

Generally, Ose functions operate on OSC *bundle elements*, which are either
messages, as we have seen, or *bundles*, which we haven't discussed yet but
will see in this section.

#### Reordering and Dropping

The following functions all manipulate the order of the stack:
```
/!/2drop
/!/2dup
/!/2over
/!/2swap
/!/drop
/!/dup
/!/nip
/!/-rot
/!/over
/!/pick/jth
/!/pick/bottom
/!/pick/match
/!/roll/jth
/!/roll/bottom
/!/roll/match
/!/rot
/!/swap
/!/tuck
```

`/!/swap` reorders the top two elements on the stack, while `/!/rot` and 
`/!/-rot` reorder the first and third elements:
```repl
❯ ./ose
# /i/10
STACK:
 [i:10]
# /f/3.14159
STACK:
 [i:10]
 [f:3.141590]
# /s/foo
STACK:
 [i:10]
 [f:3.141590]
 [s:foo]
# /!/swap
STACK:
 [i:10]
 [s:foo]
 [f:3.141590]
# /!/rot
STACK:
 [s:foo]
 [f:3.141590]
 [i:10]
# /!/swap
STACK:
 [s:foo]
 [i:10]
 [f:3.141590]
# /!/-rot
STACK:
 [f:3.141590]
 [s:foo]
 [i:10]
```
`/!/dup` makes a copy of the topmost element, and `/!/drop` throws the topmost 
element away:
```repl
STACK:
 [f:3.141590]
 [s:foo]
 [i:10]
# /!/dup
STACK:
 [f:3.141590]
 [s:foo]
 [i:10]
 [i:10]
# /!/drop
STACK:
 [f:3.141590]
 [s:foo]
 [i:10]
```
`/!/tuck` copies the topmost element below the second element, and `/!/nip`
deletes the second element:
```repl
STACK:
 [f:3.141590]
 [s:foo]
 [i:10]
# /!/tuck
STACK:
 [f:3.141590]
 [i:10]
 [s:foo]
 [i:10]
# /!/nip
STACK:
 [f:3.141590]
 [i:10]
 [i:10]
```

#### Pushing and Popping

The `/!/push` message combines two messages into one, appending the topmost to
the end of the one below it.
```repl
❯ ./ose
# /i/10
STACK:
 [i:10]
# /f/3.14159
STACK:
 [i:10]
 [f:3.141590]
# /s/foo
STACK:
 [i:10]
 [f:3.141590]
 [s:foo]
# /!/push
STACK:
 [i:10]
 [f:3.141590][s:foo]
# /!/push
STACK:
 [i:10][f:3.141590][s:foo]
```

The opposite operation, `/!/pop`, "pops" the topmost item off of the topmost 
element and makes it into its own element.
```
STACK:
 [i:10] [f:3.141590] [s:foo]
# /!/pop
STACK:
 [i:10] [f:3.141590]
 [s:foo]
# /!/swap
STACK:
 [s:foo]
 [i:10] [f:3.141590]
# /!/pop
STACK:
 [s:foo]
 [i:10]
 [f:3.141590]
# /!/swap
STACK:
 [s:foo]
 [f:3.141590]
 [i:10]
```

#### Messages and Bundles

So far, all of the examples have been on messages, but they could also be 
applied to bundles. To make an empty bundle, we can send the message 
`/!/make/bundle`, and then we can push a message into it:
```repl
# /!/make/bundle
STACK:
#bundle
# /i/10
STACK:
#bundle
 [i:10]
# /!/push
STACK:
#bundle
# [i:10]
# /i/20
STACK:
#bundle
 # [i:10]
 [i:20]
# /!/push
STACK:
#bundle
 # [i:10]
 # [i:20]
```

Similarly to messages, we can pop the topmost element off the bundle:
```repl
STACK:
#bundle
 # [i:10]
 # [i:20]
# /!/pop
STACK:
#bundle
 # [i:10]
 [i:20]
```

Pushing and popping always operate on the top two elements, and how they operate
depends on whether those elements are messages or bundles. The rules are as 
follows:
* Both elements are messages: append the items of the topmost message to the
second message
* The second element is a bundle: push the topmost element onto the end of the 
bundle.
* The second element is a message and the topmost element is a bundle: Convert 
the topmost element to a blob and push it onto the end of the message.

#### Grouping and Ungrouping

There are a few different ways to pop all items off of an element in one go.
`/!/pop/all` will produce the same result as the sequence of pops and swaps
above:
```repl
STACK:
 [i:10][f:3.141590][s:foo]
# /!/pop/all
STACK:
 [s:foo]
 [f:3.141590]
 [i:10]

#
```
Note the extra space below the value 10---that's the (empty) address of our 
original message. We often want to drop this, so we could send the message 
`/!/drop`, or we could have sent `/!/pop/all/drop` to do it as part of a 
single operation:
```repl
STACK:
 [i:10][f:3.141590][s:foo]
# /!/pop/all/drop
STACK:
 [s:foo]
 [f:3.141590]
 [i:10]
```

In each of these three examples of popping, we end up with the items reversed on
the stack. If we wanted them in the other order, we could send `/!/unpack`
or `/!/unpack/drop`:
```repl
STACK:
 [i:10][f:3.141590][s:foo]
# /!/unpack
STACK:
 [i:10]
 [f:3.141590]
 [s:foo]
```

The messages on the stack can also be bundled together by sending 
`/!/bundle/all`:
```repl
STACK:
 [i:10]
 [f:3.141590]
 [s:foo]
# /!/bundle/all
STACK:
#bundle
 # [i:10]
 # [f:3.141590]
 # [s:foo]
```

### Operations on Items

The functions described in this section operate on one or more *items*. 
Functions that take more than one argument generally expect each one to be 
the sole item of a message (the addresses of these messages are ignored). 
For example, the `/!/add` function expects the stack to look like this:
```repl
# /i/10
STACK:
 [i:10]
# /i/20
STACK:
 [i:10]
 [i:20]
# /!/add
STACK:
 [i:30]
```
rather than like this:
```repl
# /i/10
STACK:
[i:10]
# /i/20
STACK:
[i:10]
[i:20]
# /!/push
STACK:
[i:10][i:20]
# /!/add
Assertion failed: (n >= 2), function ose_add, file ose_stackops.c, line 2161.
```

#### Math

Ose has a basic set of arithmetic functions. These functions are restricted to 
numerical values of the same type:

`/!/add`
: addition

`/!/sub`
: subtraction

`/!/mul`
: multiplication

`/!/div`
: division

`/!/mod`
: modulo

`/!/pow`
: exponentiation

`/!/neg`
: negation

There are also relational and logical tests which return a 32-bit int with value
0 for false and 1 for true:

`/!/eql`
: equality test

`/!/lte`
: less than or equal

`/!/lt`
: strictly less than

`/!/and`
: logical and

`/!/or`
: logical or

The item of the topmost element on the stack will be the lefthand operand, and 
the second item on the stack with be the right. So, to subtract 3 from 10, i.e. 
`10 - 3 => 7`, we would first push `/i/3`, then `/i/10`, then `/!/sub`:
```repl
❯ ./ose
# /i/3
STACK:
 [i:3]
# /i/10
STACK:
 [i:3]
 [i:10]
# /!/sub
STACK:
 [i:7]
```

As mentioned above, these functions operate only on items of the same type---any
necessary type conversion must be done explicitly. 

#### Type Casting, Coersion, and Reinterpretation

The messages `/i`, `/f`, `/s`, and `/b`, when sent without further address 
components, will convert the top item on the stack to the specified type. 
This is done using C's casting rules where appropriate, and C's standard
library functions such as `sprintf()`, `strtol()`,  `strtod()`, etc. elsewhere.
Some examples:
```repl
❯ ./ose
# /f/3.14159
STACK:
 [f:3.141590]
# /i
STACK:
 [i:3]
# /f
STACK:
 [f:3.000000]
# /s
STACK:
 [s:3.000000]
# /i
STACK:
 [i:3]
```

Often, this is the conversion that makes the most sense, for example, if you are
looking to do an arithmetical operation on two values of different types. 
Sometimes, however, you want to reinterpret a specific bit pattern as a 
different type. In this case, you first convert the type to a blob, 
and then to the new type:
```repl
# /f/3.14159
STACK:
 [f:3.141590]
# /b
STACK:
 [b:<4:40490FFFFFFFD0>]
# /i
STACK:
 [i:1078530000]
```

#### Strings and Blobs

Strings and blobs, unlike ints and floats, have variable length and can be 
combined and split apart in few different ways. 

##### Concatenation

A string may be concatenated onto the end of another string, and similarly, 
two blobs may be concatenated together. 
```repl
# /s/hello
STACK:
 [s:hello ]
# /s/world
STACK:
 [s:hello ]
 [s:world]
# /!/push
STACK:
 [s:hello ][s:world]
# /!/concat/strings
STACK:
 [s:hello world]
```
Similarly, for blobs:
```repl
# /b/00010203
STACK:
 [b:<4:00010203>]
# /b/04050607
STACK:
 [b:<4:00010203>]
 [b:<4:04050607>]
# /!/push
STACK:
 [b:<4:00010203>][b:<4:04050607>]
# /!/concat/blobs
STACK:
 [b:<8:0001020304050607>]
```

Note that unlike many of the functions we have
seen so far that take more than one argument, these functions require the
two items to be concatenated to be the two topmost items of the topmost
message.

##### Decatenation

Decatenation is the opposite of concatenation; you supply a string or a blob,
the number of bytes that should be decatenated, and whether those bytes should
be counted forward from the beginning, or backward from the end.
```repl
# /s/hello world!
STACK:
 [s:hello world!]
# /i/5
STACK:
 [s:hello world!]
 [i:5]
# /!/decat/string/fromstart
STACK:
 [s:hello][s: world!]
# /i/6
STACK:
 [s:hello][s: world!]
 [i:6]
# /!/decat/string/fromend
STACK:
 [s:hello][s: ][s:world!]
# /i/1
STACK:
 [s:hello][s: ][s:world!]
 [i:1]
# /!/decat/string/fromend
STACK:
 [s:hello][s: ][s:world][s:!]
# /!/concat/strings
STACK:
 [s:hello][s: ][s:world!]
# /!/concat/strings
STACK:
 [s:hello][s: world!]
# /!/concat/strings
STACK:
 [s:hello world!]
```

`/!/decat/blob/fromstart` and `/!/decat/blob/fromend` work on blobs in exactly
the same manner as the example above.

##### Splitting

Splitting is like decatenation, although instead of specifying the number of
bytes to count into the string to find the point to make the split, you specify
a *token*---a string of one or more characters---and whether to begin searching
from the start or the end. The split will take place at the point where
the first occurrence of that token was found.

The split functions included here are intended to be used to split apart the 
different components of an OSC address string, and so may behave differently
than split functions in other languages. One notable difference is that they
leave the token attached to the rightmost part of the string.
```repl
# /s//foo/bar/bloo
STACK:
 [s:/foo/bar/bloo]
# /s//
STACK:
 [s:/foo/bar/bloo]
 [s:/]
# /!/split/string/fromstart
STACK:
 [s:/foo]
 [s:/bar/bloo]
 [s:/]
```
Notice also that the stack is left organized in such a way that the split
function can be immediately applied to the remainder of the string to split
it again.

Splitting from the end works similarly:
```repl
# /s//foo/bar/bloo
STACK:
 [s:/foo/bar/bloo]
# /s//
STACK:
 [s:/foo/bar/bloo]
 [s:/]
# /!/split/string/fromend
STACK:
 [s:/bloo]
 [s:/foo/bar]
 [s:/]
```

The split functions only work on strings, not blobs.

##### Joining

Two strings may be joined together with a separater, also a string. The stack 
contain three elements, all messages, with the top two each containing only a 
single string, the topmost being the separator. The join operation is 
essentially:
```repl
/!/swap
/!/push
/!/push
/!/concat/strings
/!/concat/strings
```
```repl
# /s/hello
STACK:
 [s:hello]
# /s/world
STACK:
 [s:hello]
 [s:world]
# /s/
STACK:
 [s:hello]
 [s:world]
 [s: ]
# /!/join/strings
STACK:
 [s:hello world]
```

##### Trimming

Extra whitespace may be trimmed off the start or end of a string as follows:
```repl
# /s/   string with whitespace
STACK:
 [s:   string with whitespace  ]
# /!/trim/string/end
STACK:
 [s:   string with whitespace]
# /!/trim/string/start
STACK:
 [s:string with whitespace]
```

##### String Comparison and Pattern Matching

The `/!/match` function can be used to determine whether the top two strings on 
the stack are the same or not---if the strings are identical, it leaves a 1 on 
the stack, and a 0 otherwise:
```repl
# /s/foo
STACK:
 [s:foo]
# /s/foo
STACK:
 [s:foo]
 [s:foo]
# /!/match
STACK:
 [s:foo]
 [s:foo]
 [i:1]
# /!/clear
STACK:

# /s/foo
STACK:
[s:foo]
# /s/fo0
STACK:
[s:foo]
[s:fo0]
# /!/match
STACK:
[s:foo]
[s:fo0]
[i:0]
```

OSC also supports a small set of 
[regular expressions](http://opensoundcontrol.org/spec-1_0) that can be used to 
do simple pattern matching. The `/!/pmatch` function will pattern match the 
string on top of the stack---the *address*---with the string in the second 
position on the stack---the *pattern*.
```repl
# /s//fo?/bar
STACK:
 [s:/fo?/bar]
# /s//foo
STACK:
 [s:/fo?/bar]
 [s:/foo]
# /!/pmatch
STACK:
 [s:/bar]
 [s:/fo?]
 [i:0]
 [i:1]
#
```
As you can see, `/!/pmatch` leaves four items on the stack which collectively 
describe the results of the pattern match. Starting with the top of the stack
(bottom to top with respect to the display above), the four items are:

`[i:1]`
: indicates that the entire address was matched.

`[i:0]`
: indicates that the entire pattern was not matched.

`[s:/fo?]`
: is the part of the pattern that was matched.

`[s:/bar]`
: is the part of the pattern that was not matched.

#### Parts of Elements

There are times when you may want to operate on the structural parts of a bundle
element as data itself, without incurring Ose's interpretation of it. Bundle
elements, whether they are bundles or messages, have a similar structure. They
both have four parts: a size, an identifier (an address in the case of a 
message, and the string `#bundle\0` in the case of a bundle), a type tag string 
(messages) or a time tag (bundles), and the payload. The following functions 
copy these different parts of a bundle element onto the top of the stack:

`/!/size/elem`
: gets the size of the elem as a 32-bit int.

`/!/address`
: copies the address to a string (this will be `#bundle\0` if the element is a 
bundle.

`/!/tt`
: copies the type tag string if the element is a message, or the time tag if 
the element is a bundle, to a blob.

`/!/payload`
: copies the entire payload section of the element to a blob.

### Queries

#### Counts, Lengths, and Sizes

`/count/elems`
: count the number of elements on the stack.

`/count/items`
: count the number of items in the topmost element on the stack.

`/length/address`
: get the length of the address, not including any NULL bytes or padding.

`/length/tt`
: get the length of the type tag string, or the time tag if the topmost element 
is a bundle.

`/length/item`
: get the length of the topmost item in the topmost element. In the case of a 
string, blob, or other variable-sized item, this value does not include any 
padding or NULL terminating bytes.

`/lengths/items`
: get the length of all items in the topmost element.

`/size/address`
: get the size of the address container, including the NULL byte and any 
padding.

`/size/elem`
: get the size of the topmost element (not including the size field)

`/size/item`
: get the size of the topmost item of the topmost element, including any 
padding.

`/size/payload`
: get the size of the payload.

`/sizes/elems`
: get the sizes of all elements on the stack.

`/sizes/items`
: get the sizes of all items in the topmost element.

`/size/tt`
: get the size of the type tag string (or time tag) including any padding.

### Quoting, Abstraction, Evaluation, and Application

#### Quoting

When we send messages that begin with 
`/i`, `/f`, `/s`, `/b`, `/!`, `/@`, or `/$`,
they are immediately evaluated. In order to suppress evaluation and simply 
push the address onto the stack, you can prepend any message with `/'`
(that's a slash followed by a single quote mark). `/'` means "push everything
after the quote mark onto the stack as an address."
```repl
# /'/i/3
STACK:
/i/3
# /'/i/5
STACK:
/i/3
/i/5
# /'/!/add
STACK:
/i/3
/i/5
/!/add
```

#### `/!/eval`

The eval(uate) function expects a bundle to be on top of the stack---either
by itself as an element, or as a blob that is the topmost item in a 
message. When sent, the eval function copies the current state of each of
the bundles (the input, control, stack, and environment), and clears all
of them except for the environment. It then moves the bundle that was on
top of the stack into the input, and continues evaluation in this context
as normal. When everything in that bundle has been processed, and the
input and control bundles are empty, the original state of each
of the bundles is restored, any results from the evaluation that were left
on the stack are placed on top of the stack, and evaluation continues.
```repl
# /'/i/3
STACK:
/i/3
# /'/i/5
STACK:
/i/3
/i/5
# /'/!/add
STACK:
/i/3
/i/5
/!/add
# /!/bundle/all
STACK:
#bundle
 #/i/3
 #/i/5
 #/!/add
# /!/eval
STACK:
 [i:8]
```

#### `/!/apply`

Apply takes one or more arguments, with the first one being the function to 
be applied to the rest of the elements on the stack. This first argument must
ultimately resolve to a bundle which will be evaluated according to the 
description of `/!/eval` above, or a blob containing a C function pointer,
which we will discuss in the section on importing below. If it is a string,
it will be looked up first in the environment, and then in the list of 
builtin functions, so you may enter functions in the environment that 
shadow builtin ones. If no entry is found in either place, the string is 
simply left on top of the stack.

There is nothing special about the structure of a bundle that will be passed
as the first argument to apply; it will simply be evaluated as any other 
bundle will be, and any local bindings that are performed will be wiped away
when the evaluator restores the environment to its state prior to 
performing the application.

As an example, let's make a function that adds 1 to its argument:
```repl
# /lhs
STACK:
 [s:/lhs]
# /'/!/assign
STACK:
 [s:/lhs]
/!/assign
# /'/$/lhs
STACK:
 [s:/lhs]
/!/assign
/$/lhs
# /i/1
STACK:
 [s:/lhs]
/!/assign
/$/lhs
 [i:1]
# /'/!/add
STACK:
 [s:/lhs]
/!/assign
/$/lhs
 [i:1]
/!/add
# /!/bundle/all
STACK:
#bundle
 # [s:/lhs]
 #/!/assign
 #/$/lhs
 # [i:1]
 #/!/add
 # /@/add1
STACK:

# /i/10
STACK:
 [i:10]
# /!/add1
STACK:
 [i:11]
```
The assignment that happens at the beginning is unnecessary---it's just there 
to show that assignment in a function is just ordinary assignment. We could
have just as easily used the value on the stack directly:
```repl
# /i/1
STACK:
 [i:1]
# /'/!/add
STACK:
 [i:1]
/!/add
# /!/bundle/all
STACK:
#bundle
 # [i:1]
 #/!/add
 # /@/add1
STACK:

# /i/10
STACK:
 [i:10]
# /!/add1
STACK:
 [i:11]
```

### Conditionals and Iteration

#### Conditionals

Conditional execution takes the form of 
```
<condition> 
/!/if 
<then> 
/!/else 
<else> 
/!/end/if
```
where `<condition>` is expected to be a 32-bit int with 0 representing false, 
and non-zero being true. The `/!/else` message and `<else>` clause are optional
and may be omitted.

The "if ... else ... end" construct is different than anything we have seen 
so far in that all elements of the construct must be present in the input
when the `/!/if` message is evaluated---if we just send `/!/if` on its own,
we will get an error. One way to do this is to use the combination of
`/'` and `/!/eval`:
```repl
# /'/i/1
STACK:
/i/1
# /'/!/if
STACK:
/i/1
/!/if
# /'/s/true
STACK:
/i/1
/!/if
/s/true
# /'/!/else
STACK:
/i/1
/!/if
/s/true
/!/else
# /'/s/false
STACK:
/i/1
/!/if
/s/true
/!/else
/s/false
# /'/!/end/if
STACK:
/i/1
/!/if
/s/true
/!/else
/s/false
/!/end/if
# /!/bundle/all
STACK:
#bundle
 #/i/1
 #/!/if
 #/s/true
 #/!/else
 #/s/false
 #/!/end/if
# /!/eval
STACK:
#bundle
 # [s:true]
 ```

#### Iteration

Iteration looks similar to the `/!/if ... /!/end/if` construction we just saw.
It takes the form of
```repl
<number of steps>
/!/dotimes
<stuff to do>
/!/end/dotimes
```

As above, everything everything must be in the input when the loop begins 
running, so we can use a combination of `/'` and `/!/eval` again. This program
reverses a string, by popping one character at a time off the end and pushing
it onto a new string. The inputting of each line using `/'` has been left out
for space.
```repl
STACK:
#bundle
 # [s:]
 # [s:hello world]
 # [i:11]
 #/!/dotimes
 #/i/1
 #/!/decat/string
 #/!/pop
 #/!/rot
 #/!/swap
 #/!/push
 #/!/concat/strings
 #/!/swap
 #/!/end/dotimes
 # /!/eval
STACK:
#bundle
 # [s:dlrow olleh]
 # [s:]
```

### Reading Files and Importing

#### Reading Ose Code From Files

You can write a sequence of addresses in a text file, just as they would
be entered at the REPL prompt. For example, if we had a file called
test.ose with the following contents
```
/s/
/s/hello world
/!/length/item
/!/dotimes
/i/1
/!/decat/string
/!/pop
/!/rot
/!/swap
/!/push
/!/concat/strings
/!/swap
/!/end/dotimes
```
We first push the name of the file onto the stack as a string, and then send 
`/!/repl/import`:
```repl
# /s/test.ose
STACK:
[s:test.ose]
# /!/repl/import
STACK:
[s:dlrow olleh]
[s:]
#
```

#### Importing Compiled C Code

C code can be compiled as a shared library (.so) file and imported in the same
way that we read a file above. Your C code must include a function called
`ose_main` that takes an `ose_bundle` (see the section on the C API below). 
This function will be called when
your library is loaded. Let's say we have the following C code in a file
called `howdy.c`. 
```c
#include "ose_conf.h"

#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#include "ose.h"
#include "ose_context.h"
#include "ose_util.h"
#include "ose_stackops.h"
#include "ose_assert.h"
#include "ose_vm.h"

void howdy(ose_bundle bundle)
{
n	fprintf(stderr, "howdy y'all\n");
}

void ose_main(ose_bundle bundle)
{
	ose_pushMessage(bundle, "/howdy", strlen("/howdy"),
                    1, OSETT_CFUNCTION, howdy);
}
```
When `ose_main` is called, it binds the function `howdy`
to the OSC address `/howdy`. If we load this code using the REPL, the bundle
passed to `ose_main` will be the *environment* bundle, so this function will
now be available to be run by calling `/!/howdy`.

We can compile the C file on Mac OS X with:
```
clang -I. -shared -undefined dynamic_lookup -o howdy.so howdy.c
```
On Linux, it's slightly different:
```
clang -I. -shared -rdynamic -fPIC -o howdy.so howdy.c
```

We can then load it into the REPL environment and run it:
```repl
# /s/howdy.so
STACK:
 [s:howdy.so]
# /!/repl/import
---loaded howdy.so successfully---
STACK:

# /!/howdy
howdy y'all
STACK:

```

## C API

In this section, we describe how to embed Ose in a project, and how to 
run the VM. The examples are in C / C++ and should compile on platforms like
Arduino.

### Allocation, Initialization, and the `osc_bundle` datatype

Ose provides no mechanism for memory allocation---that must be done by the 
host application and may not be changed dynamically. Once the memory has been
allocated, a bundle is created by caling `ose_newBundleFromCBytes()`:
```c
char *bytes = (char *)malloc(4096);
ose_bundle bundle = ose_newBundleFromCBytes(4096, bytes);
```
`ose_bundle` is an opaque data type that may be a pointer, or something else.
This is the object that is passed around and manipulated by the different 
Ose functions. 

Important: if you know the exact size of the bundle you need for your 
application, be aware that the Ose system requires a few bytes of overhead, 
which can be retreived with the macro `OSE_CONTEXT_MAX_OVERHEAD`. 
So, if you know that you need exactly 32 bytes for your bundle, you should 
allocate your memory like this:
```c
const int32_t n = OSE_CONTEXT_MAX_OVERHEAD + 32;
char *bytes = (char *)malloc(n);
ose_bundle bundle = ose_newBundleFromCBytes(n, bytes);
```

You can get the size of the bundle by reading a 32-bit int four bytes "behind"
the bundle:
```c
int32_t size = ose_readInt32(bundle, -4);
```

### Adding and Manipulating the Contents of the Bundle

The easiest way to add a message to a bundle is by calling `ose_pushMessage()`,
which takes the following arguments:
1. the bundle to add the message to,
1. the address of the message,
1. the size of the address 
   (excluding the NULL byte, as calculated by `strlen()`),
1. the number of items (*n*)
1. *n* typetag, data pairs.

Example:
```c
ose_pushMessage(bundle, "/foo", 4, 4,
                OSETT_INT32, 10,
                OSETT_FLOAT, 3.14159,
                OSETT_STRING, "foo",
                OSETT_BLOB, 3, (char *){1, 2, 3});
```

*Anonymous values*---messages with an empty address and a single item---can
be added using 
`ose_pushInt32(bundle, <value>)`
`ose_pushFloat(bundle, <value>)`
`ose_pushString(bundle, <value>)`
`ose_pushBlob(bundle, <length>, <ptr>)`

All of the stack manipulation functions that we saw in the section describing 
the OSC Server / VM are simply wrappers for functions in the C API, so where
we could swap the top two elements on the stack by sending `/!/swap`, in C,
we can call `ose_swap(bundle)` (recall that *the stack* is just a bundle).

### The Ose Context

The bundle that `ose_newBundleFromCBytes` returns is a pointer (potentially
wrapped in a `struct`) to a real OSC bundle, however, that bundle resides 
in a *context*, which is simply an OSC bundle with a particular 
organization. 

When you run `ose_newBundleFromCBytes`, it creates a bundle that begins at the 
beginning of the block of memory you supplied. This bundle contains two 
messages, each with the same organization: they both have 4-byte addresses
(a slash, two characters, and the NULL byte), and contain three ints and two 
blobs. The first of these two blobs contains a bundle, and this is what 
an `ose_bundle` object points to---a bundle disgused as a blob which is 
the fourth item in a message, preceeded by three ints and followed by another
blob. The second blob in these types of *context messages* is simply a 
placeholder---it is there to ensure that the message has a static size, and 
represents the amount of free space available to the bundle contained in the 
first blob of the message.

The first of these two context messages has the address `/st` and is reserved
by the system as a place to put status information. The second context message
has the address `/cx` and contains the bundle that is returned when 
you call `ose_newBundleFromCBytes`.

### The Virtual Machine

The virtual machine is a modified 
[SECD machine](https://en.wikipedia.org/wiki/SECD_machine) and consists of
six bundles called the *input*, *stack*, *environment*, *control*, *dump*,
and *output*. Each of these bundles is contained in a *context message* 
with the same structure described in the previous section, inside the 
`/cx` bundle.

#### Initialization

To initialize the VM, you first allocate a block of memory and call 
`ose_newBundleFromCBytes`, and then pass the resulting bundle to `osevm_init`:
```c
char *bytes = (char *)malloc(4096);
ose_bundle bundle = ose_newBundleFromCBytes(4096, bytes);
ose_bundle osevm = osevm_init(bundle);
```

`osevm_init` will divide up the memory available to the bundle you supplied
into equal portions for each of the six bundles that constitute the VM. 

It's common to need to have a handle on some of the different bundles in 
the VM, so you can get them with the following functions / macros:
```c
ose_bundle vm_input = OSEVM_INPUT(osevm);
ose_bundle vm_stack = OSEVM_STACK(osevm);
ose_bundle vm_env = OSEVM_ENV(osevm);
ose_bundle vm_control = OSEVM_CONTROL(osevm);
ose_bundle vm_dump = OSEVM_DUMP(osevm);
ose_bundle vm_output = OSEVM_OUTPUT(osevm);
```

#### Input to the VM

When the VM is run, it evaluates each message contained in the input bundle,
one at a time. So, in order to do anything, you have to put messages in the 
input bundle, which can either be done directly by using `ose_pushMessage()`
with the input bundle as the first argument, or if you receive a bundle from 
some source like UDP and you want to copy it into the input bundle, you can 
push it in as a blob using `ose_pushBlob`, and then "unpack" it:
```c
int32_t len = 0;
char buf[MAX_LEN];
<receive bundle from somewhere and copy into buf>
ose_pushBlob(vm_input, len, buf);
ose_blobToElem(vm_input);
ose_popAllDrop(vm_input);
```

#### Running the VM

Once you have put some messages in the input bundle, you simply call `osevm_run`
on the full VM (not the input bundle). So, our full example might look like 
this:
```c
char *bytes = (char *)malloc(4096);
ose_bundle bundle = ose_newBundleFromCBytes(4096, bytes);
ose_bundle osevm = osevm_init(bundle);
ose_bundle vm_input = OSEVM_INPUT(osevm);
int32_t len = 0;
char buf[MAX_LEN];
<receive bundle from somewhere and copy into buf>
ose_pushBlob(vm_input, len, buf);
ose_blobToElem(vm_input);
ose_popAllDrop(vm_input);
osevm_run(osevm);
```

At this point, the input, control, and dump bundles should be empty. The stack
may have elements in it, depending on what took place while the VM was running,
and similarly, if anything explicitly moved elements to the output bundle, those
will be there.

What to do at this point will depend on the application and what was done 
during evaluation. You may move the stack to the output and wait for more
input to arrive from some source to run again, you might send the stack to
some destination, you might send the environment somewhere, or reset it, etc.

### Hooks

#### Server Message Hooks

One of the more powerful aspects of the OSC Server / VM is that the functions
that the VM runs in response to the server receiving the builtin messages
(`/i`, `/f`, `/s`, `/b`, `/!`, `/@`, `/$`, and `/'`) are implemented as hooks
that can be overridden at compile-time with your own functions. This allows 
programmers embedding Ose into a particular environment to reconsider what
basic like "lookup" and "assign" mean in that environment. 

In the Arduino example below, we will extend `/$` and `/@` such that if they 
are followed by `/a/<int>` or `/d/<int>`, they will get or set the value of
analog or digital pin number `<int>`, and if not, they will simply lookup 
from or assign to the environment. 

#### Pre- and Post-eval Hooks



### Assertions

### Nonstandard Types

### Compilation Modes (Debug, Release, ...)

## Examples

### Sending OSC to/from the REPL

### Node.js

### Browsers

### Websockets

### Arduino

### Max/MSP
