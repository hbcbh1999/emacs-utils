A Guided Tour of an Implementation 
==================================
of Clojure Destructuring Bind
=============================
in Emacs Lisp
-------------

### Introduction ###

Clojure is a new (relatively) Lisp variant which has attracted a lot
of attention due to its modern features and close relationship with
the JVM.  Emacs Lisp is a lisp so antiquated that it attracts more
ridicule than plaudits.  This document describes an attempt to bring
some of the nicer syntactical (and one semantic) features from Clojure
into Elisp.

### Destructuring Bind ###

Clojure owes a certain debt to the statically typed functional
languages.  It's emphasis on Lazy values references languages like
Haskell, and is destructuring bind forms refer to the pattern matching
(efficiently) afforded by static type systems like those in Standard
ML and its variants.  Destructuring bind has, of course, appeared in
places like Lua and Python, where simple tuples can be destructured,
but Clojure's support is somewhat more extensive in that it supports
generic destructuring of sequences and tables.  In Scheme (for
instance) we might wish to swap two numbers held in the first and
second slot of a list.

    (define swap (lst) 
     (list (cadr lst) (car lst)))

In Clojure we could specify how to extract values from the arguments
of a function right in the function definition:

    (defn swap [[a b]] (list a b))

(Clojure's binding forms are by convention represented by vectors.)
Here SWAP takes only one argument, but it binds two names.  The inner
`[a b]` expression indicates that the single argument is expected to
be a list, and that its first and second values should be bound to the
symbols `a` and `b` respectively.   Clojure's destructuring bind
supports recursive binding forms.  If, for instance, we wanted to
write a function which accepts a single list, whose second element is
a list, whose first value we wish to extract, we could write:

    (defn extract [[_ [a]]] a)

Where we have nested the destructuring deep into the sequence to pull
out `a`.  Error checking aside, this is a pretty nice feature,
particularly because Clojure _also_ allows destructuring on tables,
which have their own source-level representation.  To write a function
which pulls out the values in a table located at keys `:x` and `:y`
and returns them in a flat list, we can write:

    (defn extract [{a :x b :y}] (list a b))

Suppose the value at `x` were a list and we wished to get its second
element:

    (defn extract [{[_ part] :x}] part)

Would do the trick.  We can combine, recursively, table and sequence
destructuring syntax.  This project is an implementation of this
feature in Emacs Lisp.

### Bonus Material: Recur ###

Tail call optimization is a controversial subject in some arenas.  The
feature was somewhat famously kept out of Python by its "dictator for
life."  Scheme implementations, on the other hand, are required to
support this feature.  Clojure takes a middle path (more for reasons
related to the JVM than any political sensitivity): tail calls to
"oneself" can be made by virtue of an explicit form, `recur`, which
resembles a function call but can only be invoked from tail position,
and which reuses the current stack frame instead of creating a new
one.  This allows many basic algorithms which depend on tail calls for
elegant expression to be written naturallly in Clojure.

This library also allows the use of a `recur` special form, statically
checked to be only from tail position.  

### Syntactic & Other Notes ###

Clojure supports tables at the level of source code via a curly braces
notation:

    {:x 10 :y 11} 

Would be the table with keys `:x`, `:y` pointing to 10 and 11
respectively.  Unfortunately, Emacs Lisp does not provide facilities
to extend the reader, which would could use to create a syntax for
tables if we were using (say) Common Lisp.  We'll be using a mildly
ad-hoc solution.  My standard library provides functions to create
tables succinctly:

    (tbl! :x 10 :y 11)

Is equivalent to the above Clojure.  For destructuring we will use a
vector to represenent sequences (`[a b c]`) and a vector with a
special head token to represent tables (`[:: a :x b :y]`).  That is,
`::` indicates an expression represents a table, rather than sequence
destructuring.  

Additionally, we'll allow the table syntax to destructure
association-lists, since these are common table surrogates in other
lisp dialects and because they can be made persistant (as data
structures) more easily than Emacs Lisp's tables.  This has no impact
on the syntax for destructuring bind, however.  

Finally, this implementation uses a few non-standard special forms
from my standard library which bear remarking upon.  The form
`let-seq`, defined in `utils.el` is a simple form for destructuring
lists.  It creates a context where a series of symbols are bound to
the values in a list:

    (let-seq (a b c) (list 3 2 1) 
      (list a b c))

Evaluates to '(1 2 3). 

The form `let-tbl` allows a very simple form of table destructuring.  
    
    (let-tbl 
     ((x :a)
      (y :b))
     (tbl! :a 10 :b 11)
     (+ x y))

Evaluates to 21.

Both `let-seq` and `let-tbl` are used to parse the destructuring
syntax.  Destructuring bind itself does not expand into them,
basically because destructuring can have recursive structures and
these forms are "flat".

The implementation makes use of other functions in
the `utils.el` library, but these are the most conspicuous departures
from recognizeable lisp.

### Implementation Sketch ###

Any destructuring bind expression will ultimately expand to a `let*`
form in Emacs Lisp.  Eg: 

    (dlet [[x y] (list 1 2)]
       (+ x y))

Expands to something not unlike:

    (let* ((_ (list 1 2))
           (x (elt _ 0))
           (y (elt _ 1)))
        (+ x y))

(Where `_` in the actual expansion would be a fresh symbol for the
purposes of macro hygeine, probably generated with `gensym`.)

The task of this project is to parse the clojure-style forms and
convert them into a series of symbol/value pairs for a `let*`
expression.  

Two sub-parsers are defined in the files `parse-seq-binder.el` and
`parser-table-binder.el`, for sequences and tables respectively.
These parsers check and extract the syntax of their respective types,
but do not recursively parse sub-parts.  Recursive parsing is
supported at a higher level, essentially by trampolining.  

Both sub-parsers are ad-hoc parsers which fold series of state
dependent functions over the sequence of tokens representing the
binding expression.  Ultimately they return a list of important
information about each parser which allows the equivalent pure emacs
lisp `let` forms to be constructed.  

The action of these parsers is coordinated in `defn.el`'s
`handle-binding` function which handles detecing which type of binder
needs to be interpreted and dispatching to the appropriate parser.
Recursive parsing of binding forms is supported at this level.  

Emacs Lisp enforces a max-nesting depth and so the slightly more
obvious strategy of creating nested-`let*`'s is put aside in favor of
passing around an accumulator for the entire sequence of binding
pairs.  Hence, the functions `handle-binding`, `handle-tbl-binding`
and `handle-seq-bindind` sport an additional argument `previous-lets`
so that a list of `let*` binders can be collected.  

Once a sequencec of symbol/value pairs is constructed, the rest of the
work of the macro is essentially bookkeeping.  `fn` is the form which
does most of the work, since `defn` expands in terms of `fn`.  The
body of the expanded `fn` is a lambda expression.  This expression
takes any number of arguments and checks this number against the
arities of the binders it was defined with.  If it finds a match, it
invokes the appropriate body, each body wrapped in a `let*` form whose
binding expressions were generated by parsing and expanding the
appropriate destructuring expression.  

The body of the function is expanded in slightly different ways
depending on whether the function tries to call itself recursively
using `recur`.

#### Recur Implementation ####

Tail recursion is similar to a loop in that it essentially indicates
that we re-execute a set of forms in a context where variables have
been bound to new values.  This library supports tail recursion by
using a codewalker to find and appropriately expand `recur` forms in
the body of a `recur` supporting form, enclosing the entire body in a
while loop.  An invisible loop sentinal controls whether we "recur"
when execution terminates or whether we rebind the appropriate
variables (using a `setq` expression) and execute the body again.

Because we are destructively updating the variables in our recursive
context, it is particularly important that we check whether `recur` is
invoked from tail position.  If we were to expand a `recur` form in
any context where there were susbsequent expressions which depended on
the values in the scope, these expressions would behave anomalously.
Hence, we need a codewalker to _check_ each occurances of `recur` is
in tail position.  

Given an expression `expr`, a tail position is any position for which
there are no subsequent expressions.  A few examples are in order:

    (progn a b c)

In the above, `c` is in tail position.  

    (progn a b (recur c))

would be a legal place for recur to occur, but:

    (progn (recur a) b c)

Would produce an error because `(recur a)` occurs before `b` or `c`
are evaluated.  In an `if` statement, `tail-ness` is preserved by
either branch of the statement.  So if an `if` statement is in tail
position, either branch is also in tail position, because they are
mutually exclusive.  Similar reasoning applies to `cond` and other
special forms.

The codewalker implemented in the second half of `defn.el` bears this
basic idea out, covering the entire beastiary of basic Emacs Lisp
forms.  Because Emacs Lisp is extensible via macros our codewalker
cannot understand all possible un-expanded Emacs Lisp fragments.
Hencec this library calls `macroexpand-all` on code before it is
walked in order boil down any Elisp into a form containing only
primitive expressions.  The macro `dsetq` is then used to expand
`recur` into an expression which sets the local variables to their new
values.

If the codewalker detects a `recur` in non-tail position, it produces
an error.

#### Note on Recur and Lexical Scope ####

This version of recursion will not handle lexically scoped variables
correctly because it mutates the context for recursion without
considering whether a lambda expression or other artifact is hanging
onto a reference to some value.  Supporting the correct behavior would
require a much more complex codewalkder, and, in any case the need is
somewhat obviated by the requirement that Emacs Lisp programmers
explicitely indicate a desire for lexical scope using a `lexical-let`
expression.  Each `lexical-let` then closes over its own version of
the variables in scope.  So even though we are employing mutation:

    (setq closures (dloop [i 0
                              acc nil]
                          (if (= i 10) (reverse acc)
                            (recur (+ i 1) 
                                   (cons
                                    (lexical-let ((x i))
                                      (lambda () x)) acc)))))

Which collects a series of `lambda`'s with different values of `i`,
the following code produces the correct values:

    (funcall (elt closures 0)) -> 0
    (funcall (elt closures 4)) -> 4

etc.

Emacs Lisp will support real lexical scope in the next release of
emacs.  How that bodes for this project is unknown.

### Example Usage ###

Destructuring bind lets us construct some fairly concise
implementations of certain functions.  The function `prod,` which
takes a list and returns the produce of its elements, might be
implemented with `(lambda (lst) (reduce #'* lst))`, of course, but it
makes a nice example:

    (require 'defn)
    (defn prod
     ;;; prod will have two bodies, differentiated by arity
     ([[head & tail :as list] acc]
      (if list (recur tail (* acc head)) acc))
     ([list]
      (prod list 1)))

This example demonstrates dispatch based on arity, destructuring of a
sequence (first argument, first body) and recursion.

    (prod '(1 2 3 4 5 6 7)); -> 5040

The form `defn` always compiles its innards, so the resulting code is
pretty zippy.  

Suppose we are using an association list to represent people in some
kind of database.  People have first and last names, but they might
not have a middle name, so both:

    '((:first-name "George") (:last-name "Washington"))
    '((:first-name "Timofey") (:middle-name "Pavlovich") (:last-name
      "Pnin"))

Are legitimate "persons" (neglecting that "Pavlovich" is a patronym
rather than a proper middle name).  How might we write a function to
extract the middle name succintly?

    (defn extract-middle-name 
          [[:: middle :middle-name :or 
           (alist>>
            :middle-name :NMI)]]
         middle)

    (extract-middle-name '((:first-name "Timofey") 
                           (:middle-name "Pavlovich") 
                           (:last-name "Pnin"))); -> "Pavlovich"

    (extract-middle-name '((:first-name "George") (:last-name
    "Washington"))); -> :NMI 

The function `extract-middle-name` demonstrates the table
destructuring syntax and the use of an `:or` clause, which specifies
an alternative expression to destructurin the event that any
destructuring of the input fails.  It can be used to provide default
values for variables.  

### If I'd Known Then What I Know Now ###

This project is a little more than a year old, and in the interveining
time, I've learned a few new things which would have changed,
somewhat, the implementation.  For one, the more complete
understanding of the destructuring bind syntax which I developed while
creating this library would probably have informed a more elegant
implementation.  

More concretely, I know recognize that parsing the binding expressions
is really a job for less ad-hoc parsing strategy.  Subsequent to this
project, I produced an implementation of monadic parser combinators
based on SMUG, for Common Lisp.  If I were to re-implement this
project now, I'd use a combinator approach for parsing.  For another
project I've written a Common-Lisp lambda list parser using parser
combinators, which is in `simplified-lambda-list-parser.el` which
should convey the style of what a more informed implementation might
look like (although CL-lambda lists are simpler than Clojure
destructuring forms).  See also `monad-parse.el` and `monads.el`.

The question of `recur` is separate, basically, from the destructuring
bind problem, and handling the two things together is responsible for
some of the mess of this implementation.  I've subsequently factored
out `recur` support into a separate library.  A rewrite would almost
certainly concern itself almost completely with destructuring bind,
and use the `recur.el` library to support self-recursion.  The
implementation in `recur.el` is also a little bit cleaner, and
supports a better error checking model than the one in this file.
I've included it, if the reader is interested.  There are also some
stylistic differences - the codewalker dispatches to a function for
each form encountered rather than doing processing in place, for
instance.  This makes for a more readable implementation.