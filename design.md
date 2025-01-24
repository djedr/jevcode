One possible design is a thin wrapper over JS -- basically a minimal syntactic representation of JS in Jevko.

A parser of such a language as well as a JS code generator are both fairly straightforward. No compile-time envrionment would be required.

After these have enough functionality to bootstrap, the design could be elaborated further, either in JS or in the language under construction. 

Elaboration could involve adding a compile time environment, static analysis, additional language features.

Some example code:

```
const[[pullPrefix] /arrow[[iter]
  const[
    /dict[[value][done]]
    /call[
      /get[[iter][next]]
    ]
  ]
  /call[
    /get[[console][assert]]
    /not[done]
  ]
  /call[
    /get[[console][assert]]
    =[
      /get[[value][type]]
      ['prefix]
    ]
  ]
  return[/get[value][text]]
]]
```

Another one:

```
/call[
  /get[
    /call[
      /get[[span][slice]]
      [i]
    ]
    [trim]
  ]
]
```

That would translate to:

```
span.slice(i).trim()
```

Some funny syntax sugar I just came up with:

```
/get&call[
  /get&call[
    [span][slice][&][i]
  ]
  [trim]
  [&]
]
```

Or even this craziness:

```
/get&call&...[
  [span][slice][&][i][&][trim][&]
]
```

`[&]` here would just flip back and forth between getting and calling.

---

# 2024-12-08

Revised example code:

```
const[[pullPrefix] /arrow[[iter]
  const[
    /dict[[value][done]]
    /pipe[
      /get[[iter][next]]
      /call[]
    ]
  ]
  /assert[/not[done]]
  /assert[
    =[
      /get[[value][type]]
      ['prefix]
    ]
  ]
  return[/get[[value][text]]]
]]
```

`/pipe` here is the interestng bit:

```
/pipe[
  /get[[iter][next]]
  /call[]
]
```

`/pipe` is syntax sugar that works similar to Clojure's `->` threading macro.

It is a universal piping construct. It transforms trees of the form:

```
/pipe[
  expr0[]
  expr1[]
  expr2[]
]
```

Into:

```
expr2[expr1[expr0[]]]
```

-- respecting syntax sugar for 0- and 1-argument calls, i.e.:

```
/pipe[
  expr0[]
  expr1[]
]
```

will become:

```
expr1[expr[0]]
```

whereas:

```
/pipe[
  expr0[]
  expr1[arg1]
]
```

will become:

```
expr1[ expr[0] [arg1] ]
```

-- it will be treated the same as its equivalent desugared form:

```
/pipe[
  expr0[]
  expr1[[arg1]]
]
```

-- this would also become:

```
expr1[ expr[0] [arg1] ]
```

Calls with 2 or more arguments will behave analogously, e.g.:

```
/pipe[
  expr0[]
  expr1[[arg1][arg2]]
]
```

becomes:

```
expr1[expr0[][arg1][arg2]]
```

and so on...

# 2024-12-11

```
const[[parsePipe] /arrow[[iter]
  const[[toks] /list[
    pullPrefixTok[iter]
    /spread[pullSubToks[iter]]
  ]]

  let[ [p1] pullPrefixTok[iter] ]
  while[[true]
    /pipe[
      /get[[toks][unshift]]
      /call[p1]
    ]
    const[ [st] pullSubToks[iter] ]
    if[=[ /get[[st][length]] [1] ]
      const[ [tok] /get[[st][[0]]] ]
      /assert[=[ /get[[tok][type]] ['suffix] ]]
      if[
        !=[
          /pipe[
            /get[[tok][text][trim]]
            /call[]
          ]
          [']
        ]

        ;[special case if we are piping into f[i] -- transform that to f[[i]]]
        /pipe[
          /get[[st][unshift]]
          /call[/dict[
            [type] ['prefix]
            [text] [']
          ]]
        ]
        /pipe[
          /get[[st][push]]
          /call[/dict[
            [type] ['suffix]
            [text] [']
          ]]
        ]
      ]
    ]
    /pipe[
      /get[[toks][push]]
      /call[/spread[st]]
    ]

    const[ [tok] pullTok[iter] ]
    if[=[ /get[[tok][type]] ['suffix] ]
      /assert[=[
        /pipe[/get[[tok][text][trim]] /call[]]
        [']
      ]]
      note: we discard the /pipe's suffix
      /pipe[/get[[iter][append]] /call[toks]]
      return[]
    ]
    /set[ [p1] [tok] ]
  ]
]]
```

# 2024-12-12

Now with more M-expressions!

```
const[parsePipe; /arrow[iter;
  const[toks; /list[
    pullPrefixTok[iter]
    /spread[pullSubToks[iter]]
  ]]

  let[ p1; pullPrefixTok[iter] ]
  while[true;
    /get[toks;unshift][p1]
    const[ st; pullSubToks[iter] ]
    if[=[ /get[st;length] 1 ]
      const[ tok; /get[st;0] ]
      /assert[=[ /get[tok;type] 'suffix' ]]
      if[
        !=[
          /get[tok;text;trim][]
          ''
        ]

        ;[special case if we are piping into f[i] -- transform that to f[i]]
        /get[st;unshift][/dict[
          type; 'prefix'
          text; ''
        ]]
        /get[st;push][/dict[
          type; 'suffix'
          text; ''
        ]]
      ]
    ]
    /get[toks;push][/spread[st]]

    const[ tok; pullTok[iter] ]
    if[=[ /get[tok;type] 'suffix' ]
      /assert[=[
        /get[tok;text;trim][]
        ''
      ]]
      note: we discard the /pipe's suffix
      /get[iter;append][toks]
      return[]
    ]
    /set[ p1; tok; ]
  ]
]]
```

# 2024-12-15

This M-expression-like syntax:

```
export[
  const[ astToCode; /arrow[ ast;
    const[ /dict[type] ast ]
    const[ gen; /get[generators;get][type] ]
    if[=[ gen; undefined ]
      return['?todo?']
    ]
    return[gen[ast]]
  ]]
]
```

would compile to this:

```js
export const astToCode = (ast) => {
  const {type} = ast
  const gen = generators.get(type)
  if (gen === undefined) {
    return '?todo?'
  }
  return gen(ast)
}
```

The same could be generated from this even more regular/primitive syntax:

```
export[
  const[[astToCode] /arrow[[ast]
    const[ /dict[type] [ast] ]
    const[ [gen] /pipe[
      /get[[generators][get]]
      /call[type]
    ]]
    if[=[[gen][undefined]]
      return['?todo?]
    ]
    return[gen[ast]]
  ]]
]
```

# 2024-12-16

Format strings.

```
f'Hello, $[name]!'
```

Conditionals.

```
[ atom[x]? x; T? ff[car[x]] ]
```

Reviving M-expressions as K-expressions.

K-exp. KEXP.

M-exp was:

```
label[ff;λ[[x];[atom[x] → x; T → ff[car[x]]]]]
```

K-exp is:

```
label[ff; lambda[x;
  [ atom[x]? x; T? ff[car[x]] ]
]]
```

J-exp is:

```
label[[ff] lambda[[x]
  cond[ if[atom[x]] [x] if[T] ff[car[x]] ]
]]
```

JS is:

```
const ff = (x) => atom(x)? x: true? ff(car(x)): undefined
```

# 2024-12-18

Strings.

```
['abc]  ->  `abc`
['abc[n]def]  ->  `abc\ndef`
['abc\ndef]  ->  `abc\\ndef`
['$`]  ->  `\$\``
```

# 2024-12-19

```
x['...] -> x[['...]]
```

# 2024-12-20

``````
x[`'...'`]
``````

# 2024-12-21

Winter Solstice.

```
/str[abc[n]def]
```

|
v

```
'abc\ndef'
```

So instead of `['abc[n]def]` we'd have `/str[abc[n]def]` -- less special-casing.

# 2024-12-22

The disadvantage is `f['abc]` becomes `f[/str[abc]]` but less and less it seems to matter.

# 2024-12-23

Think I'll just allow heredocs.

```
/str[`'...'`]
```

Some version at least.

# 2024-12-24

Compile-time execution.

Makes a preprocessor/macro system.

Can be based around tokens/trees (Jev-specific) or strings (universal).

Could have both.

Would need heredocs to be universal.

# 2024-12-25

Another take on heredocs.

``````
/str`[...]`
/str``[...]``
etc.
e.g.:
/str`[ unbalanced brackets: ][ ]`
``````

# 2024-12-29

or...

```
/str`'...'`
```

# 2024-12-30

maybe also

```
/str[...[...]...$[x]...] -> `...[...]...${x}...`
```

# 2024-12-31

or maybe

```
/str['...']   -> `...`
/str[['...']] -> `...`
/str[='...'=] -> `...`
```

ruminating...

# 2025-01-01

it'll be

```
/str['...']
/str[`'...'`]
/str[``'...'``]
/str[```'...'```]
```

etc.

Then we'd also have:

```
/tstr['...$[...]...']
```

for template strings.

To think about: should `['...]` be special-cased?
E.g. make it `['...']` and have it be just a JS string + optional `$[...]` (if any substitution found, then it'll compile to a template, else regular string).

What happens to `f['...']` then? Perhaps it should be transformed into `f[['...']]` at tokenizer-level?

# 2025-01-02

Macros. Here is an idea. Not sure if workable.

A macroexpression is identified by a `#` prefix, e.g. `#if`.

A macroexpression returns tokens.

To evaluate a macroexpression, compile a version M of the program P whose output will be the result of evaluating the expression. This result will be substituted in place of the expression in the "original" program P which will then be compiled to produce the final result R.

Version M would be like P, except all macroexpressions and macrodeclarations would be normal expressions and declarations. The entrypoint of P would be replaced with the context where the macroexpression under evaluation resides. The expression would have access to the entire program, both the macro and the regular functions and variables.

This may be going too far. We shall see.

# 2025-01-03

However only expressions which return tokens can be macroexpanded/preceded with `#`. If an expression which does not evaluate to a stream of tokens is evaluated as a macro, an error is raised.

Example code that uses a macro:

```
const[ [isOdd] /arrow[[n] 
  ...
]]

const[ [f] /arrow[[n]
  return[*[ [n] [2] ]]
]]

const[ [g] /arrow[[m]
  const[ [h] /arrow[[p]
    return[+[ [p] [2] ]]
  ]]

  const[ [x] #ifx[
    isOdd[f[5]]
    @['hello]
    @['hello $[h[3]]]
  ]]

  return[x]
]]
```

When compiling

```
#ifx[
  isOdd[f[5]]
  @['hello]
  @['hello $[h[3]]]
]
```

we first transform the program into something like:

```
const[ [isOdd] /arrow[[n] 
  ...
]]

const[ [f] /arrow[[n]
  return[*[ [n] [2] ]]
]]

const[ [g] /arrow[[m]
  const[ [h] /arrow[[p]
    return[+[ [p] [2] ]]
  ]]

  return[ifx[
    isOdd[f[5]]
    @['hello]
    @['hello $[h[3]]]
  ]]
]]

g[]
```

then we compile and execute that to obtain the result:

```
['hello 5]
```

then we substitute this result in place of the macro:

```
const[ [isOdd] /arrow[[n] 
  ...
]]

const[ [f] /arrow[[n]
  return[*[ [n] [2] ]]
]]

const[ [g] /arrow[[m]
  const[ [h] /arrow[[p]
    return[+[ [p] [2] ]]
  ]]

  const[ [x] ['hello 5] ]

  return[x]
]]
```

this is then compiled to produce the final macroexpanded result.

# 2025-01-04

No sigils needed.

```
/macro[f[x]] instead of #f[x]
/quote['hello] instead of @['hello]
etc.
```

More verbose -- yes. Optimizing that though proves counterproductive.

# 2025-01-06

I mean optimizing for being the least verbose is counterproductive.

Interesting to think about: where are the points of diminishing returns, how to balance dimensions which are at odds? Ultimately some provisional decisions must push things forward. To see how things go and gather more data, if for no other reason.

It's incredible that anything can be accomplished at all.

# 2025-01-07

Losing momentum is real. And so is sustaining it.

# 2025-01-08

A tree of freely diverging ideas. Or is it a network?

# 2025-01-09

```
ctx.fillRect(130, 190, 40, 60)
ctx.fillRect[ [130] [190] [40] [60] ]
/call[
  /get[ [ctx] [fillRect] ]
  [130] [190] [40] [60]
]
```

Going further would be too far.

# 2025-01-10

Forgot to update the year. Haha. :D xD

Corrected.

Some weird JS translated to Jev.

```js
[] == ![] // true
```

```
==[ /list[] /not[/list[]] ]   true
```

or

```
/loose-equal[ /list[] /not[/list[]] ]   true
```

Maybe.

BTW perhaps `=` should have it's own semantics.

```js
!!"false" === !!"true" // true
```

```
=[ /not[/not[/str['false']]] /not[/not[/str['true']]] ]   true
```

# 2025-01-11

Rather not support class creation.

```js
class Person extends Entity {
  name;

  constructor(name) {
    this.name = name;
  }

  introduceSelf() {
    console.log(`Hi! I'm ${this.name}`);
  }
}
```

But it could look like

```
class[ [Person] extends[Entity]
  [name]

  constructor[name] [
    /set[
      /get[[this][name]]
      [name]
    ]
  ]

  introduceSelf[] [
    /log[/str['Hi! I'm $[/get[[this][name]]]']]
  ]
]
```

Maybe.

Private fields and methods could be:

```
class[ [A] extends[B]
  /private[field]

  /private-method[someMethod][] [
    /log[/str['Hello?']]
  ]
]
```

# 2025-01-12

A little fun with C.

```
#include[stdio.h]

/function[
  /return-type[int]
  main[
    /arg[ [argc] /type[int] ]
    /arg[ [argv] /type[/ptr[/ptr[char]]] ]
  ]
  printf[/str['Hello, World!']]
  return[0]
]
```

# 2025-01-19

Yet another encoding(s) for HTML/XML.

```html
<input type="checkbox" id="scales" name="scales" checked />
```

```
\input/[].type[checkbox].id[scales].name[scales].checked/[]
```

```
\input/[].type[checkbox].id[scales].name[scales].[checked]
```

# 2025-01-20

```js
const input = document.createElement('input')
input.setAttribute('type', 'checkbox')
input.setAttribute('id', 'scales')
input.setAttribute('name', 'scales')
input.setAttribute('checked', '')
```

```
const[ [input] /call[
  /get[[document][createElement]]
  /str['input']
]]
/call[
  /get[[input][setAttribute]]
  /str['type'] /str['checkbox']
]
/call[
  /get[[input][setAttribute]]
  /str['id'] /str['scales']
]
/call[
  /get[[input][setAttribute]]
  /str['name'] /str['scales']
]
/call[
  /get[[input][setAttribute]]
  /str['checked'] /str['']
]
```

# 2025-01-21

this is what an LLM generated for me to translate:

```py
def generate_fibonacci_sequence(n):
if n < 0:
raise ValueError("Input must be a non-negative integer")
if n == 0 or n == 1:
return [n]
fib_sequence = [0, 1]
while fib_sequence[-1] + fib_sequence[-2] < n:
fib_sequence.append(fib_sequence[-1] + fib_sequence[-2])
return fib_sequence
```

not even valid!

let's try and fix:

```py
def generate_fibonacci_sequence(n):
  if n < 0:
    raise ValueError("Input must be a non-negative integer")
  if n == 0 or n == 1:
    return [n]
  fib_sequence = [0, 1]
  while fib_sequence[-1] + fib_sequence[-2] < n:
    fib_sequence.append(fib_sequence[-1] + fib_sequence[-2])
  return fib_sequence
```

better.

now let's translate:

```
def[ generate_fibonacci_sequence[n]
  if[
    <[ [n] [0] ]
    raise[ValueError[/str['Input must be a non-negative integer']]]
  ]
  if[
    or[
      =[ [n] [0] ]
      =[ [n] [1] ]
    ]
    return[/list[n]]
  ]
  /set[ [fib_sequence] /list[[0][1]] ]
  while[
    <[
      +[
        /get[
          [fib_sequence]
          [-1]
        ]
        /get[
          [fib_sequence]
          [-2]
        ]
      ]
      [n]
    ]

    /call[
      /get[[fib_sequence][append]]
      +[
        /get[
          [fib_sequence]
          [-1]
        ]
        /get[
          [fib_sequence]
          [-2]
        ]
      ]
    ]
  ]
  return[fib_sequence]
]
```

# 2025-01-22

More spelled out versions of a HTML element.

```html
<input type="checkbox" id="scales" name="scales" checked />
```

```
/element[
  /tag[input]
  /attr[type[checkbox]]
  /attr[id[scales]]
  /attr[name[scales]]
  /attr[checked]
  /self-closing[]
]
```

```
/element[
  /tag[input]
  /attributes[
    type[checkbox]
    id[scales]
    name[scales]
    [checked]
  ]
  ;[no children[] means self-closing]
]
```

# 2025-01-23

More AI-generated code.

```py
import random

def generate_random_name():
    first_names = ["Alice", "Bob", "Charlie", "Diana", "Ethan", "Fiona"]
    last_names = ["Smith", "Johnson", "Williams", "Jones", "Brown", "Davis"]
    
    first_name = random.choice(first_names)
    last_name = random.choice(last_names)
    
    return f"{first_name} {last_name}"

# Generate and print a random name
random_name = generate_random_name()
print(random_name)
```

```
import[random]

def[generate_random_name[]
  /set[ [first_names] /list[
    /str['Alice'] /str['Bob']   /str['Charlie'] 
    /str['Diana'] /str['Ethan'] /str['Fiona']
  ]]
  /set[ [last_names] /list[
    /str['Smith'] /str['Johnson'] /str['Williams'] 
    /str['Jones'] /str['Brown']   /str['Davis']
  ]]
  
  /set[ [first_name] /call[
    /get[[random][choice]]
    [first_names]
  ]]
  /set[ [last_name] /call[
    /get[[random][choice]]
    [last_names]
  ]]

  return[/fstr['{[first_name]} {[last_name]}']]
]

;[Generate and print a random name]
/set[ [random_name] generate_random_name[] ]
print[random_name]
```

# 2025-01-24

Let's try and translate this: https://rosettacode.org/wiki/Find_the_intersection_of_two_lines#Kotlin

```kotlin
// version 1.1.2

class PointF(val x: Float, val y: Float) {
    override fun toString() = "{$x, $y}"
}

class LineF(val s: PointF, val e: PointF)

fun findIntersection(l1: LineF, l2: LineF): PointF {
    val a1 = l1.e.y - l1.s.y
    val b1 = l1.s.x - l1.e.x
    val c1 = a1 * l1.s.x + b1 * l1.s.y

    val a2 = l2.e.y - l2.s.y
    val b2 = l2.s.x - l2.e.x
    val c2 = a2 * l2.s.x + b2 * l2.s.y

    val delta = a1 * b2 - a2 * b1
    // If lines are parallel, intersection point will contain infinite values
    return PointF((b2 * c1 - b1 * c2) / delta, (a1 * c2 - a2 * c1) / delta)
}

fun main(args: Array<String>) {
    var l1 = LineF(PointF(4f, 0f), PointF(6f, 10f))
    var l2 = LineF(PointF(0f, 3f), PointF(10f, 7f))
    println(findIntersection(l1, l2))
    l1 = LineF(PointF(0f, 0f), PointF(1f, 1f))
    l2 = LineF(PointF(1f, 2f), PointF(4f, 5f))
    println(findIntersection(l1, l2))
}
```

```kotlin
;[version 1.1.2]

class[ PointF[
  val[[x][Float]] 
  val[[y][Float]]
]
  override[fun[toString[] /istr['{${x}, ${y}}']]]
]

class[LineF[
  val[[s][PointF]]
  val[[e][PointF]]
]]

fun[ findIntersection[
  [l1]:[LineF]
  [l2]:[LineF]
] :[PointF]
    val[ [a1] -[[l1.e.y][l1.s.y]] ]
    val[ [b1] -[[l1.s.x][l1.e.x]] ]
    val[ [c1] +[[a1 * l1.s.x][b1 * l1.s.y]] ]

    val[ [a2] -[[l2.e.y][l2.s.y]] ]
    val[ [b2] -[[l2.s.x][l2.e.x]] ]
    val[ [c2] +[[a2 * l2.s.x][b2 * l2.s.y]] ]

    val[ [delta] -[[a1 * b2][a2 * b1]] ]
    ;[If lines are parallel, intersection point will contain infinite values]
    return[PointF[
      /[[b2 * c1 - b1 * c2][delta]]
      /[[a1 * c2 - a2 * c1][delta]]
    ]]
]

fun[ main[[args]:[Array[String]]]
    var[ [l1] LineF(PointF(4f, 0f), PointF(6f, 10f)) ]
    var[ [l2] LineF(PointF(0f, 3f), PointF(10f, 7f)) ]
    println[findIntersection[l1, l2]]
    /set[ [l1] LineF(PointF(0f, 0f), PointF(1f, 1f)) ]
    /set[ [l2] LineF(PointF(1f, 2f), PointF(4f, 5f)) ]
    println[findIntersection[l1, l2]]
]
```
