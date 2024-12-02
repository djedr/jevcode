Translating snippets from https://aosabook.org/en/500L/static-analysis.html

```julia
# A comment about increment
function increment(x::Int64)
  return x + 1
end

increment(5)
```

```
A comment about increment
function[  increment[ Int64[x] ]
  return[+[ [x] [1] ]]
]

increment[5]
```

---

```julia
# Increment x by y
function increment(x::Int64, y::Number)
  return x + y
end

increment(5) # => 6
increment(5,4) # => 9
```

```
Increment x by y
function[  increment[ Int64[x] Number[y] ]
  return[+[ [x] [y] ]]
]

increment[5]   => 6
increment[ [5] [4] ]   => 9
```

---

```julia
function increment(x::Int64)
  if x < 10
    x = x + 50
  else
    x = x + 0.5
  end
  return x
end
```

```
function[  increment[ Int64[x] ]
  if[
    <[ [x] [10] ]  /set[ [x] +[[x][50]] ]
    /set[ [x] +[[x][0.5]] ]
  ]
  return[x]
]
```

---

```julia
function unstable()
  sum = 0
  for i=1:100
    sum += i/2
  end
  return sum
end
```

```
function[  unstable[]
  let[ [sum] [0] ]
  for[
    let[ [i] [1] ]
    <=[ [i] [100] ]
    /set[ [i] +[[i][1]] ]

    /set[  [sum]  +[ [sum] /[[i][2]] ]  ]
  ]
]
```

---

```julia
function stable()
  sum = 0.0
  for i=1:100
    sum += i/2
  end
  return sum
end
```

```
function[  stable[]
  let[ [sum] [0.0] ]
  for[
    let[ [i] [1] ]
    <=[ [i] [100] ]
    /set[ [i] +[[i][1]] ]

    /set[  [sum]  +[ [sum] /[[i][2]] ]  ]
  ]
]
```

---

```julia
function unstable()
  sum = 0
  for i=1:10000
    sum += i/2
  end
  return sum
end
```

```
function[  unstable[]
  let[ [sum] [0] ]
  for[
    let[ [i] [1] ]
    <=[ [i] [10000] ]
    /set[ [i] +[[i][1]] ]

    /set[  [sum]  +[ [sum] /[[i][2]] ]  ]
  ]
]
```

---

```julia
function foo(x,y)
  z = x + y
  return 2 * z
end

code_typed(foo,(Int64,Int64))
```

```
function[  foo[[x][y]]
  const[ [z] +[[x][y]] ]
  return[*[[2][z]]]
]

code_typed[ [foo] /tuple[[Int64][Int64]] ]
```

---

```julia
1-element Array{Any,1}:
:($(Expr(:lambda, {:x,:y}, {{:z},{{:x,Int64,0},{:y,Int64,0},{:z,Int64,18}},{}},
 :(begin  # none, line 2:
        z = (top(box))(Int64,(top(add_int))(x::Int64,y::Int64))::Int64 # line 3:
        return (top(box))(Int64,(top(mul_int))(2,z::Int64))::Int64
    end::Int64))))
```

that's a tough one; something like:

```
/Array[/Dollar[Expr[
  :[lambda]
  /Set[:[x]:[y]]
  /Set[
    /Set[:[z]]
    /Set[
      /Set[:[x][Int64][0]]
      /Set[:[y][Int64][0]]
      /Set[:[z][Int64][18]]
    ]
    /Set[]
  ]
  :[[begin]   none, line 2:
    /set[
      [z]
      Int64[/call[
        top[box]
        [Int64]
        /call[
          top[add_int]
          Int64[x]
          Int64[y]
        ]
      ]]
    ]   line 3:
    return[
      Int64[/call[
        top[box]
        [Int64]
        /call[
          top[mul_int]
          [2]
          Int64[z]
        ]
      ]]
    ]
    Int64[end]
  ]
]]]
```

---

```julia
julia> e = code_typed(foo,(Int64,Int64))[1]
:($(Expr(:lambda, {:x,:y}, {{:z},{{:x,Int64,0},{:y,Int64,0},{:z,Int64,18}},{}},
 :(begin  # none, line 2:
        z = (top(box))(Int64,(top(add_int))(x::Int64,y::Int64))::Int64 # line 3:
        return (top(box))(Int64,(top(mul_int))(2,z::Int64))::Int64
    end::Int64))))
```

```
[jev] /set[  [e]  /get[code_typed[ [foo] /tuple[[Int64][Int64]] ][1]]  ]
:[/Dollar[Expr[
  :[lambda]
  /Set[:[x]:[y]]
  /Set[
    /Set[:[z]]
    /Set[
      /Set[:[x][Int64][0]]
      /Set[:[y][Int64][0]]
      /Set[:[z][Int64][18]]
    ]
    /Set[]
  ]
  :[[begin]   none, line 2:
    /set[
      [z]
      Int64[/call[
        top[box]
        [Int64]
        /call[
          top[add_int]
          Int64[x]
          Int64[y]
        ]
      ]]
    ]   line 3:
    return[
      Int64[/call[
        top[box]
        [Int64]
        /call[
          top[mul_int]
          [2]
          Int64[z]
        ]
      ]]
    ]
    Int64[end]
  ]
]]]
```

---

```julia
julia> names(e)
3-element Array{Symbol,1}:
 :head
 :args
 :typ 
```

```
[jev] names[e]
3-element Array[[Symbol][1]]:
  :[head]
  :[args]
  :[typ]
```

---

```julia
julia> e.head
:lambda

julia> e.typ
Any

julia> e.args
3-element Array{Any,1}:
 {:x,:y}
 {{:z},{{:x,Int64,0},{:y,Int64,0},{:z,Int64,18}},{}}
 :(begin  # none, line 2:
        z = (top(box))(Int64,(top(add_int))(x::Int64,y::Int64))::Int64 # line 3:
        return (top(box))(Int64,(top(mul_int))(2,z::Int64))::Int64
    end::Int64)
```

```
[jev] /get[ [e] [head] ]
:[lambda]

[jev] /get[ [e] [typ] ]
[Any]

[jev] /get[ [e] [args] ]
3-element Array[[Any][1]]:
  /Set[:[x]:[y]]
  /Set[
    /Set[:[z]]
    /Set[
      /Set[:[x][Int64][0]]
      /Set[:[y][Int64][0]]
      /Set[:[z][Int64][18]]
      /Set[]
    ]
    :[[begin]   none, line 2:
      /set[
        [z]
        Int64[/call[
          top[box]
          [Int64]
          /call[
            top[add_int]
            Int64[x]
            Int64[y]
          ]
        ]]
      ]   line 3:
      return[
        Int64[/call[
          top[box]
          [Int64]
          /call[
            top[mul_int]
            [2]
            Int64[z]
          ]
        ]]
      ]
      Int64[end]
    ]
  ]
```

---

```julia
julia> e.args[1] # names of arguments as symbols
2-element Array{Any,1}:
 :x
 :y
```

```
[jev] /get[ [e] [args] [1] ]   names of arguments as symbols
2-element Array[[Any][1]]
  :[x]
  :[y]
```

---

```julia
julia> e.args[2] # three lists of variable metadata
3-element Array{Any,1}:
 {:z}                                     
 {{:x,Int64,0},{:y,Int64,0},{:z,Int64,18}}
 {}  
```

```
[jev] /get[ [e] [args] [2] ]   three lists of variable metadata
3-element Array[[Any][1]]
  /Set[:[z]]
  /Set[/Set[:[x][Int64][0]]/Set[:[y][Int64][0]]/Set[:[z][Int64][18]]]
  /Set[]
```
