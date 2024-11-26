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

