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

# 2024-01-01

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

# 2024-01-02

Macros. Here is an idea. Not sure if workable.

A macroexpression is identified by a `#` prefix, e.g. `#if`.

A macroexpression returns tokens.

To evaluate a macroexpression, compile a version M of the program P whose output will be the result of evaluating the expression. This result will be substituted in place of the expression in the "original" program P which will then be compiled to produce the final result R.

Version M would be like P, except all macroexpressions and macrodeclarations would be normal expressions and declarations. The entrypoint of P would be replaced with the context where the macroexpression under evaluation resides. The expression would have access to the entire program, both the macro and the regular functions and variables.

This may be going too far. We shall see.
