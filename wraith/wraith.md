- Feature Name: Wraith
- Start Date: 2020-10-18
- RFC PR: [racket/racket2-rfcs#0000](https://github.com/racket/racket2-rfcs/pull/0000)
- Authors: Alex Knauth and Christopher Lemmer Webber, based on ideas from Arne Babenhauserheide

# Summary
[summary]: #summary

Wraith is a syntax proposal that eliminates parentheses but has a clean translation to s-expression syntax.

# Motivation
[motivation]: #motivation

Many users are overwhelmed by parenthetical syntax and would like an alternative.  Proposals such as Honu, Shrubbery are desirable in the sense of providing an alternative syntax that may be available to such users, but involve syntax transformations that involve significantly larger machinery.  Meanwhile, s-expression syntax is well understood, studied, supported in the Racket community.  This proposal expands on ideas from [Wisp](http://srfi.schemers.org/srfi-119/srfi-119.html) ([examples](https://dustycloud.org/blog/wisp-lisp-alternative/)) but with some refinements.

<!-- TODO: Expand -->

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Consider the following Racket program, taken from the Racket
[quick introduction](https://docs.racket-lang.org/quick/index.html):

``` racket
define (add-drawing p)
  define drawer
    make-pict-drawer p
  new canvas%
      parent f
      style '(border)
      paint-callback
        lambda (self dc)
          drawer dc 0 0
```

This corresponds to the following Racket code:

``` racket
(define (add-drawing p)
  (define drawer
    (make-pict-drawer p))
  (new canvas%
       [parent f]
       [style '(border)]
       [paint-callback
        (lambda (self dc)
          (drawer dc 0 0))]))
```

The intuition here that a newline with indentation corresponds to
a nested expression:

``` racket
define drawer                             |  (define drawer
  make-pict-drawer p                      |    (make-pict-drawer p))
```

But how to handle many arguments that aren't composing a new expression?
A leading `.` period can be used to continue a line:

``` racket
list 1 2 3                                |  (list 1 2 3
   . 4 5 6                                |        4 5 6)
```

Keywords are implicitly considered to be continuing arguments in the
previous expression:

``` racket
standard-cat 100 90                       |  (standard-cat 100 90
             #:happy? #t                  |                #:happy? #t)
```

In general, indentation level does not matter super strongly; as long
as "greater" than the previous, it is "nested into the parent
expression":

``` racket
standard-cat                              |  (standard-cat
  . 100 90                                |   100 90
  #:happy? #t                             |   #:happy? #t)
```

Parentheses can be used to build a new nested expression.  For
instance:

``` racket
define drawer (make-pict-drawer p)        |  (define drawer (make-pict-drawer p))
```

``` racket
define (display-excitement str reasons r2)|  (define (display-excitement str reason)
  format "SO EXCITED about ~a (~a, ~a)!!" |    (format "SO EXCITED about ~a (~a, ~a)!!"
         string-upcase str                |            (string-upcase str)
         . reason r2                      |            reason r2))
```

Lines within a parenthetical expression still
follow expressions still generally follow Wraith's rules
(note that this is a departure from Sweet Expressions and Wisp!):

``` racket
let* [animal "dog"                        |  (let* [(animal "dog")
      noise "barks"                       |         (noise "barks")
      player-hears                        |         (player-hears
        format "the ~a says: ~a!!!"       |          (format "the ~a says: ~a!!!"
               . animal noise]            |                  animal noise))]
  displayln player-hears                  |    (displayln player-hears))
```
However, that let still looks busy.
If we're deviating from s-expressions, is there a way to make
it less nested?

In traditional Racket S-Expression syntax, `()` and `[]` have only
been differentiated by convention.
In Wraith syntax, `[]` has a special meaning which can be described
as "a wrapped set of wrapped expressions".
This makes aesthetically more appealing `let` and `for` syntax examples.

``` racket
for [pet '("cat" "dog" "horse")]          |  (for [(pet '("cat" "dog" "horse"))]
  printf "I love my ~a!\n" pet            |    (printf "I love my ~a!\n" pet))
```

``` racket
define (counting-letters-song letters)    |  (define (counting-letters-song letters)
  for [letter letters                     |    (for [(letter letters)
       number (in-naturals 1)]            |          (number (in-naturals 1))]
    printf "I like ~a, it's number ~a!"   |      (printf "I like ~a, it's number ~a!"
      . letter number                     |         letter number)
    newline                               |      (newline))
  displayln "Singing a letters song!"     |    (displayln "Singing a letters song!"))
```

The first line of the bracketized expression is always closed at the
end of the line. To use a multi-line expression, it must be on its own line.

``` racket
define (greeter name)                     |  (define (greeter name)
  let ((to-say                            |    (let ((to-say
          format "Hey there ~a! :D"       |           (format "Hey there ~a! :D"
                 . name))                 |                   name)))
    displayln to-say                      |      (displayln to-say)))
```

``` racket
define (greeter name)                     |  (define (greeter name)
  let [                                   |    (let (
       to-say                             |          (to-say
          format "Hey there ~a! :D"       |           (format "Hey there ~a! :D"
                 . name]                  |                   name)))
    displayln to-say                      |      (displayln to-say)))
```

New lines of content within parenthetical/bracketed expressions must
have deeper indentation than the line with the opening parenthesis.
In other words, the following is an error and not permitted:

``` racket
;; Not allowed!
define a-list
  '(1 2
 3)
```

Finally, many potential users have grown up being taught mathematics
using infix syntax.  As such, a switch to prefix notation is seen as a
major requirement for understandability.  This is a reasonable
request.  However, infix notation also often introduces "order of
operation problems", which confuse students and mathematicians
everywhere and are generally solved by putting parentheses back around
operations.  Thus Wraith partially borrows from
[SRFI 105](https://srfi.schemers.org/srfi-105/srfi-105.html)
for curly infix operations but only permits one kind of infix operation
per curly-grouping:

``` racket
define (double x)                      |  (define (double x)
  . {x * 2}                            |    (* x 2))
```

``` racket
define (squared-minus-one x)           |  (define (squared-minus-one x)
  . {{x * x} - 1}                      |    (- (* x x) 1))
```

``` racket
. {1 + 2 + {9 / 3}}                    |  (+ 1 2 (/ 9 3))
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

<!-- This is the technical portion of the RFC. Explain the design in sufficient detail that: -->

<!-- - Its interaction with other features is clear. -->
<!-- - It is reasonably clear how the feature would be implemented. -->
<!-- - Corner cases are dissected by example. -->

<!-- The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work. -->

There are 4 ways of grouping expressions together in Wraith:
  1. Indentation
      ```
      a b_1 ... b_m
        c_1
        ...
        c_n
      ```
  2. Parentheses
      ```
      (a b_1 ... b_m
        c_1
        ...
        c_n)
      ```
  3. Square Brackets
      ```
      [a_1 b_1_1 ... b_1_m
        c_1_1
        ...
        c_1_n
       ...
       a_l b_l_1 ... b_l_m
        c_l_1
        ...
        c_l_n]
      ```
  4. Curly Braces
      ```
      {a_1 op a_2 ... op a_n}
      ```

Indentation applies to expressions both outside and inside parens and brackets. However it only considers leading indentation.

Within an indented expression:
```
a b_1 ... b_m
  c_1
  ...
  c_n
```
The indentation of each `c` expression must be greater than the indentation of the `a` expression

For example, here the `d` is not associated with `a` because it does not have greater indentation than `a`:
```
a b_1   | (a b_1)
d       | d
```

And here the `d` is associated with `c_1` because it has greater indentation than `c_1`:
```
a b_1   | (a b_1
  c_1   |    (c_1
   d    |      d))
```

The indentation does not group anything together if `a` is a `#:` keyword or a `.` dot.

Wraith only considers leading indentation, because indentation within
a line cannot be calculated exactly in the general case.

For example, the indentation of this opening parenthesis depends on
geopolitics, because regional-indicator symbols can be two or one
letters wide depending on whether the region is defined on the system
at the current time:

🇩🇩 (a
   a_or_b?)

# Drawbacks
[drawbacks]: #drawbacks

<!-- Why should we *not* do this? -->

Wraith does not include algebra-style function calls `f(x,y)`, and does not include any precedence in infix notation, so `{x + y * z}` is not allowed.

Wraith also does not allow you to write extra parentheses anywhere around any expression without changing meaning, so `(x)` is different from `x`.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

<!-- - Why is this design the best in the space of possible designs? -->
<!-- - What other designs have been considered and what is the rationale for not choosing them? -->
<!-- - What is the impact of not doing this? -->

# Prior art
[prior-art]: #prior-art

<!-- Discuss prior art, both the good and the bad, in relation to this proposal. -->
<!-- A few examples of what this can include are: -->

<!-- - For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had? -->
<!-- - For community proposals: Is this done by some other community and what were their experiences with it? -->
<!-- - For other teams: What lessons can we learn from what other communities have done here? -->
<!-- - Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background. -->

<!-- This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture. -->
<!-- If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages. -->

<!-- Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC. -->
<!-- Please also take into consideration that Racket sometimes intentionally diverges from common language features. -->

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!-- - What parts of the design do you expect to resolve through the RFC process before this gets merged? -->
<!-- - What parts of the design do you expect to resolve through the implementation of this feature before stabilization? -->
<!-- - What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->

# Future possibilities
[future-possibilities]: #future-possibilities

<!-- Think about what the natural extension and evolution of your proposal would -->
<!-- be and how it would affect the language and project as a whole in a holistic -->
<!-- way. Try to use this section as a tool to more fully consider all possible -->
<!-- interactions with the project and language in your proposal. -->
<!-- Also consider how the this all fits into the roadmap for the project -->
<!-- and of the relevant sub-team. -->

<!-- This is also a good place to "dump ideas", if they are out of scope for the -->
<!-- RFC you are writing but otherwise related. -->

<!-- If you have tried and cannot think of any future possibilities, -->
<!-- you may simply state that you cannot think of anything. -->

<!-- Note that having something written down in the future-possibilities section -->
<!-- is not a reason to accept the current or a future RFC; such notes should be -->
<!-- in the section on motivation or rationale in this or subsequent RFCs. -->
<!-- The section merely provides additional information. -->
