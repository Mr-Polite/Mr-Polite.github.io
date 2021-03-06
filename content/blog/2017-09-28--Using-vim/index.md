---
date: "2017-09-28T09:00:00.009Z"
title: "Using vim"
category: "linux"
---
## How to search in vim
Search command
```
ESC + : + / + [search pattern] + enter
```

Next match
```
n + enter
```

Previous match
```
N + enter
```

## How to navigate in vim
Go forward by word
```
w
```
Go back by word
```
b
```
Move one character left
```
h
```
Move one character right
```
l
```
Move one row down
```
j
```
Move one row up
```
k
```


## How to select, copy and paste in vim
Select a line (up/down arrow key to include more/less lines)
```
V
```
Select texts (letter by letter)
```
v   
```
Select blocks 
```
ctrl + v
```

Then
Delete
```
d
```
Copy (called 'yanking')
```
y
```

Then
Paste after cursor
```
p
```
Paste before cursor
```
P
```

Note: there is no default cutting operation that could be done with one command.

## Lines
open a line below the cursor and start insert mode
```
o
```
open a line above the cursor
```
O
```


## How to undo
```
u
```

## Opening new tabs
```
ESC + : + tabnew
```
To open a new tab,
and 
```
ESC + : + saveas [filename]
```
to save the file.
