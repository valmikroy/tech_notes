# Vim Notes

Set of commands used in vim

### NerdTree 

```
i  : split open horizontal 
gi : Same as i, but leave the cursor on the NERDTree
s  : split open vertical
gs : Same as s, but leave the cursor on the NERDTree
o  : replace existing buffer with new file

r : refresh file tree pane
m : get filesystem menu, press ‘a’ to add child node, ending with / will create a directory

x : close tree hierarchy in Nerdtree pane 

\ : Nerdtree toggle on/off
,nf : Move focus to Nerdtree
```



### Folds

```
set foldmethod=syntax  : fold with syntax
set foldnestmax=1      : fold level 
zo and zc              : open and close fold
zR and zM              : open and close all folds
,ef and ,df            : enable and disable folding
```



### vim-go

```
[[ and ]]  : to beginning and end of the function 
,w         : toggle tag bar
,ww        : save existing file
,tt        : run go test 
,g         : focus 
```



### Ultisnips

```
C-Space : complete snippet
Tab     :  move forward and fill in the blanks
C-x     : backward trigger (never worked for me)
```

