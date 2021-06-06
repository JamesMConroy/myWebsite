+++
title = "Solving Exercism's Bash Grep Challenge"
date = "2021-03-06"
author = "James Conroy"
cover = ""
tags = ["bash", "grep", "scripting"]
showFullContent = false
+++

# Reimplementing `grep`

One of exercism.io's challenges is to reimplement `grep` in bash.
Here's a walk through of how I solved it.


## Solution 1

``` bash
#/bin/env bash

grep "${@}"
exit 0
```

This solution probably isn't in the spirit of the challenge, but it does satisfy all the tests.

Time to go home.

## Solution 2

For the same reasons why you shouldn't use a word in its own definition, reimplementations shouldn't use the original tool.
Which means I have to actually think about what my script needs to do.

My `grep.sh` needs to:

- take a number of files
- take a regex

Once it has all the arguments `grep.sh` needs to check each line in each file to see if the line contains the regex.

Those are not the only requirements, the script needs to accept several flags too, but these are the core functions.

``` bash
#!/usr/bin/env bash

while read line
do
  if [[ ${line} =~ ${1} ]]
  then
    echo ${line}
  fi
done < ${2}
```

This script takes a regex and a file, and compares each line against the regex to see if it matches.
If the line matches it is printed to `stdout`.

There, our basic functionality is complete, and we can submit the soluti...

> 25 tests, 23 failures

Well that's not good. Guess I have to make those pass first.

## Adding Functionality one Test at a Time

One of the nice things about test driven development, is that you always know what is broken and have a list of what still needs to be implemented.
In this case I'm going to go top to bottom making each test pass before moving on to the next.

## Flags

For the challenge exercism.io wants the script to implement several flags.

- -n: Print the line numbers of each matching line.
- -l: Print only the names of files that contain at least one matching line.
- -i: Match line using a case-insensitive comparison.
- -v: Invert the program -- collect all lines that fail to match the pattern.
- -x: Only match entire lines, instead of lines that contain a match.

This is a good opportunity to use the builtin `getopts`.
Since the script needs to accept several flags, a regex, and a list of file names it would be unwieldy to roll my own solution.

Before that I want to encapsulate our current script in its own function.

``` bash
findMatch() {
  while read line
    do
      if [[ ${line} =~ ${1} ]]
        then
          echo ${line}
    fi
      done < ${2}
}

findMatch "${@}"
```
Running the test suit on this shows the same passing and failing tests, so I'm good to continue.

### Print the Line Number of Matches

To implement the `-n` option I keep track of how many lines I check and print the number of the current line with the line if the `$lineNumber` variable is set.

``` bash
#!/usr/bin/env bash

findMatch() {
  number=0
  while read line
  do
    (( number+=1))
    if [[ ${line} =~ ${1} ]]
    then
      if $lineNumber
      then
        echo "${number}:${line}"
      else
        echo ${line}
      fi
    fi
  done < ${2}
}

lineNumber=false
while getopts "nlivx" arg
do
  case "${arg}" in
    n) lineNumber=true ;;
    # other options not implimented yet
  esac
  shift $((OPTIND - 1))
done

regex="${1}"
findMatch "${regex}" "${2}"
```
> 25 tests, 20 failures

We are making progress.

### Case Insensitive Flag

For this I'm going to use the shell option `nocasematch`.

From bash's manpage:

> nocasematch:
> If  set,  bash  matches patterns in a case-insensitive fashion when performing matching while executing case or [[ conditional commands, when performing pattern substitution word expansions, or when filtering possible completions as part of programmable completion.

To enable the functionality I just added `i) shopt -s nocasematch ;;` to the `getops` switch statement.

> 25 tests, 18 failures

### Print Matching File Names

Like the `-n` flag I implement this by setting a var and changing the behavior of `findMatch()`.
This time to print the filename and break if it finds at least one match.

``` bash
findMatch() {
  number=0
  while read line
  do
    (( number+=1))
    if [[ ${line} =~ ${1} ]]
    then
      if $fileName
      then
        echo ${2}
        break
      fi
      if $lineNumber
      then
        echo "${number}:${line}"
      else
        echo ${line}
      fi
    fi
  done < ${2}
}
```

### Match the Entire Line

I am going to change the regex passed to `findMatch()` to implement this flag.

``` bash
regex="${1}"
if $entireLine
then
  regex="^${regex}$"
fi
shift 1

findMatch "${regex}" "${@}"
```

The `regex="^${regex}$"` is adding the line start (`^`) and line end (`$`) characters to the regex.

This should be a simple change and running the tests confirms that ... more tests are failing,

## Squashing Bugs

Running the script manually to see what is happening shows some errors when it is passed 3 flags.
Specifically `./grep.sh: line 4: ${2}: ambiguous redirect`.

A good way to add `set -xv` to your script.
You can place it either at the start, or just at your problem areas.

Setting `-v` tells bash to print each line in your script as it is read.
The `-x` option will print the expansion of each command after.
These two commands let you see just what your script is ding during execution.
Think of it like print statement debugging on steroids.

In my case the script was setting the regex to the third flag and the filename to what should of been the pattern.
Looking through my script the culprit was `shift $((OPTIND-1))`, which should be placed just after the while loop.

Once the line was changed the tests started passing.

# Refactoring

Since we now have another set of tests passing and we were just debugging, this is a good time to refactor.
Specifically the `findMatch()` function is too long, unwieldy, and implementing the last option would be repetitive right now.

First off I want to improve the functions readability by assigning the positional arguments to named variables.

``` bash
findMatch() {
  regex="${1}"
  file="${2}"
```

Next I want the function to construct a variable to return instead of just echoing out the current line.

``` bash
number=0
  while read line
  do
    output=""
    (( number+=1))
    if [[ "${line}" =~ ${regex} ]] && ! $invertMatch
    then
      # Break early if we only want to know the files with lines that match
      if $fileName
      then
        echo ${file}
        break
      fi

      if $lineNumber
      then
        output="${number}:"
      fi
      output+="${line}"
      echo ${output}
    fi
  done < ${file}
}
```
This lets us get rid of some else clauses and will make adding in the last flag's functionality easier.

The latest refactor I want to do is getting rid of all those global variables.
They make it more difficult to add functionality, are messy, and are using good variable names I could use other places.

To get rid of them I'm going to have getopts construct a string of the letter options and pass that to `findMatch()`.
Once in `findMatch()` I'm going to check the string to see if it contains the letter for flow control.
Some steps are still better performed outside of `findMatch()`, like setting `shopt - s nocasematch`, so they can stay where they are.

``` bash
#!/usr/bin/env bash

findMatch() {
  regex="${1}"
  options="${2}"
  file="${3}"

  number=0
  while read line
  do
    output=""
    (( number+=1))
    if [[ "${line}" =~ ${regex} ]]
    then
      # Break early if we only want to know the files with lines that match
      if [[ ${options} =~ l ]]
      then
        echo ${file}
        break
      fi
      if [[ ${options} =~ n ]]
      then

        output="${number}:"
      fi
      output+="${line}"
      echo ${output}
    fi
  done < ${file}
}

options=""
entireLine=false
while getopts "nlivx" arg
do
  case "${arg}" in
    n) options+=n ;; # Line numbers
    l) options+=l ;; # only print file names
    i) shopt -s nocasematch ;; # ignore case
    #v) invertMatch=true ;; # invert match
    x) entireLine=true ;;  # match the entire line
  esac
done
shift $((OPTIND-1))

regex="${1}"
if $entireLine
then
  regex="^${regex}$"
fi
shift 1

findMatch "${regex}" "${options}" "${@}"

```
A quick check with bats shows the same number of passing tests as before, so we haven't broken anything.

### Invert Matches

The last option to implement is the invert match flag.

Like the others we add `v` to the `options` variable and pass it to `findMatch()`.

Now that we have the inverse flag we want to print the line if the line contains the regex xor the invert flag is set.
So print if the line matches and the invert flag is not set or print the line if the regex doesn't match and the invert flag is set.

I achieved this by changing the if statement to:

``` bash
if ( [[ "${line}" =~ ${regex} ]] && [[ ! ${options} =~ v ]] ) ||
   [[ ! "${line}" =~ ${regex} ]] && [[ ${options} =~ v ]] )
```

## Multiple Files

Now that all the flags are working the script need to be able to handle multiple files.
Right now the script just checks the first file given.

You can fairly easily run `findMatch()` on multiple files by calling it in a loop like:

``` bash
for file in ${@}
do
  findMatch "${regex}" "${options}" "${file}"
done
```

Unfortunately we're not done yet.
When searching through multiple files `grep` prints the file name before every line of output.
This can be achieved the same way I printed the line numbers before matches with the `n` flag.
We just need to check for multiple file arguments and add a new letter to out `options` string.

``` bash
if [[ ${options} =~ f ]]
then
  output+="${file}:"
fi

...

if [[ ${#} > 1 ]]
then
  options+=f
fi
```

With that all the tests are passing and the script is ready for final refactoring.
In this case I'm just going to wrap up all the logic not in `findMatch()` into a `main()` function at the start of the script.
This will let you look at the script's logic in the order it is used, without having to wade through a long function definition first.


You can find all the code on my [github](https://github.com/JamesMConroy/grep.sh).
