---
categories:
  - Others
tags:
  - linux
  - fzf
title: "Effective file finding on Unix with locate, ripgrep and fzf"
slug: "Fzf ripgrep locate"
date: 2017-12-02T12:34:17Z
draft: true
---
##  My experience with Fzf ripgrep and locate

One of the most common task working on a computer is finding stuff. Usually, that either a file or file content.
Since speeding this can really make a difference in terms of productivity, I thought to share my latest workflow.

Say you want to find `myfile.txt` somewhere on the filesystem and edit it, leaving desktop search engines such as baloo out of the equation, I'd usually use the good old [find utils](https://www.gnu.org/software/findutils/manual/html%5Fmono/find.html#Overview)


## Relative vs absolute search
The ideal solution would allow me to differentiate between relative (only search below current directory) and absolute (search from / or ~ etc).
While setting absolute to "/" constitutes a superset of relative, it usually takes longer and can drown you in results.

### GNU locate
```bash
sudo updatedb # only needs to be done once in a while

locate myfile.txt # absolute path variant
locate $PWD | grep myfile.txt # relative path variant
```
Piping through grep is the best I could come up with to simulate relative path search since locate doesn't support excluding paths natively. 
A possible alternative would me [mlocate](https://serverfault.com/questions/454051/how-can-i-view-updatedb-database-content-and-then-exclude-certain-files-paths) using `PRUNENAMES=".bzr .hg .git .svn"` in /etc/updatedb.conf.
This however which requires root previleges, those patterns can't be set as environmental variable as far as I know.
Not very portable (i.e. no local updatedb.conf) and doesn't make the cut for me. 

### GNU find
Classic find is robust and versatile, especially `-exec` allows batch processing.
But can be too slow for only searching on a large codebase.

```bash
find . -name myfile.txt #relative path variant
find ~ -name myfile.txt #absolute path variant
```

### Ripgrep
A third option is ripgrep. 
Having used it originally as a grep alternative (i.e. searching inside files), I only recently realized it's `--files` option making it a file finding alternative.
There is even more options such as fd or the silver searcher etc which I didn't explore here. 

```bash
rg -g myfile.txt --files   #relative path variant
rg -g myfile.txt --files ~ #absolute path variant
```

This slower as locate, faster (however couldn't find a benchmark) as find and doesn't need an index (i.e. root previleges).
As a bonus, ripgrep can also filter out .git files or filter out other patterns (using .ignore files). 

In summary, without having done a formal benchmark, ripgrep seems to make the cut for me at this point.


## adding fuzzy pattern matching with FZF

What if you don't know exactly what you file name is? Most of the time I only remember fractions of the file.
For example I'd know "my" is in it as well as "txt".


```bash
rg -g my*.txt --files ~
```

This clearly calls for some fuzzy finding, which is where fzf comes in.
It basically takes any text stream as input (e.g. STDIN), such as any of the search results above, grep results and what not and lets you fuzzy filter it (that is selecting one line matching the typed in pattern).

A another problem I had is how to pass on the results. Normally I'd have the list of candidate files on the screen that then needs to be copied as argument. Never really found a nifty solution that wasn't fiddely in some way. 



```bash
rg -g my*.txt --files ~ | fzf > out
nvim $(fzf)
nvim ^T
```
The first line uses rg as a prefilter and pipes the output into fzf which then selects one and writes it in out.

The second calls with nvim with the result of an fzf search as parameter. 

The same can be achived by presseing ctrl+T, which is pretty nifty, it's even possible to select multiple files with tab out of the box (similar to `$(fzf -m)`). This solves the second problem above. Just type some command that needs a file as argument and hit Ctrl-T instead of writing out the file name, fzf is filling it for you. 
Couldn't be easier!

Only remaining problem is to tell fzf where to search.
I don't know for certain what plain fzf does by defaultsince the manual is a bit vague, but I reckon it's a find search of the whole file system. 
Does anyone know? Anyway, it always confused but it's easily fixed anyway by providing my own list.
This is where the above section comes in. Turns out, fzf be easily adjusted using any of the above using environmental variables `FZF_CTRL_T_COMMAND` and/or `FZF_DEFAULT_COMMAND`

```bash
export FZF_CTRL_T_COMMAND='rg --files .'
```

```bash
alias ffr="export {FZF_CTRL_T_COMMAND,FZF_DEFAULT_COMMAND}='rg --files .'"
alias ffh="export {FZF_CTRL_T_COMMAND,FZF_DEFAULT_COMMAND}='rg --files $HOME'"
alias ffg="export {FZF_CTRL_T_COMMAND,FZF_DEFAULT_COMMAND}='rg --files /'"
alias ffl="export {FZF_CTRL_T_COMMAND,FZF_DEFAULT_COMMAND}='locate /'"
```
as a memonic, ff=fzf, r=relative, h=home, g=global (last 2 are absolute).

Depending on what kind of search I want, I'd change the undlying fzf search-engine.
Note, the **fasted** would be the last alias (ffl) using locate + fzf but doesn't find files changed after the last `updatedb`.
Once set, just hit ctrl-T to fire up the file finding:


Here you have it:

```bash
nvim C^T
```



## Appendix
excluding files/dirs

-g '!/dir2ignore'

or put it in the .ignore file (containing a ignore list for ripgrep)

echo "dir2ignore/" >> ~/.ignore


