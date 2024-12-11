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
      ;[note: we discard the /pipe's suffix]
      /pipe[/get[[iter][append]] /call[toks]]
      return[]
    ]
    /set[ [p1] [tok] ]
  ]
]]
```
