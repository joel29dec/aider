
# Improving GPT-4's codebase understanding with ctags

GPT-4 is extremely useful for "self-contained" coding tasks,
like generating brand new code or modifying a pure function without dependencies.

But it's difficult to use GPT-4 to modify or extend
a large, complex pre-existing codebase.
To modify such code, GPT needs to understand the dependencies and APIs
which interconnect all of its subsystems.
Depending on the coding task, it may not be obvious
how to determine which parts of the codebase are relevent to solving the task.
Even assuming we can identify all the needed context, we must
ensure that it fits within GPT-4's 8k-token
context window.

To address these issues, `aider` has
a new experimental feature that utilizes `ctags` to provide
GPT with a **concise map of your whole git repository** including
all declared variables and functions with call signatures.
This *repo map* enables GPT to better comprehend, navigate
and edit the code in larger repos.

To get a sense of how effective this can be, this
[chat transcript](https://aider.chat/examples/add-test.html)
shows GPT-4 creating a black box test case, **without being given
access to the source code of the function being tested or any of the
other code in the repo.**
Using only the meta-data in the repo map, GPT is able to figure out how to
call the method to be tested, as well as how to instantiate multiple
class objects that are required to prepare for the test.


## The problem: code context

GPT-4 is great at "self contained" coding tasks, like writing or
modifying a pure function with no external dependencies. These work
well because you can send GPT a self-contained request like "write a
Fibonacci function" or "rewrite the loop using list
comprehensions". These changes require no context beyond the code
being discussed.

Most real code is not pure and self-contained, it is intertwined with
code from many different files in a repo.
If you ask GPT to "switch all the print statements in class Foo to
use the BarLog logging system", it needs to see the code in the Foo class
with the prints, and it also needs to understand how the project's BarLog
logging system works.

A simple solution is to **send the entire codebase** to GPT along with
each change request. Now GPT has all the context! But this won't work
for even moderately
sized repos that won't fit in the 8k-token context window.

An
improved approach is to be selective, and **hand pick which files to send**.
For the example above, you could send the file that
contains Foo and the file that contains the BarLog logging subsystem.
This works pretty well, and is supported by `aider`: you
can manually specify which files to "add to the chat".

But it's not ideal to have to manually identify the right
set of files to add to the chat. 
Some changes may need context from many files.
And you might still overrun
the context window if you need to add many files for context.

## Using a repo map to provide context

The latest version of `aider` sends a **repo map** to GPT along with
each change request. The map contains a list of all the files in the
repo, along with the symbols which are defined in each file. Callables
like functions and methods also include their signatures.

Here's a
sample of the map of the aider repo, just showing the maps of
[main.py](https://github.com/paul-gauthier/aider/blob/main/aider/main.py)
and
[utils.py](https://github.com/paul-gauthier/aider/blob/main/aider/utils.py)
:

```
aider/
   ...
   main.py:
      function
         main (args=None, input=None, output=None)
      variable
         status
   ...
   utils.py:
      function
         do_replace (fname, before_text, after_text, dry_run=False)
         find_original_update_blocks (content)
         quoted_file (fname, display_fname, number=False)
         replace_most_similar_chunk (whole, part, replace)
         show_messages (messages, title=None)
         strip_quoted_wrapping (res, fname=None)
         try_dotdotdots (whole, part, replace)
      variable
         DIVIDER
         ORIGINAL
         UPDATED
         edit
         separators
         split_re
   ...
```

Mapping out the repo like this provides some benefits:

  - GPT can see variables, classes, methods and function signatures from everywhere in the repo. This alone may give it enough context to solve many tasks. For example, it can probably figure out how to use the API exported from a module just based on the details shown in the map.
  - If it needs to see more code, GPT can use the map to figure out by itself which files it needs to look at. GPT will then ask to see these specific files, and `aider` will automatically add them to the chat context (with user approval).

Of course, for large repositories, even just their map might be too large
for the context window.  However, this mapping approach opens up the
ability to collaborate with GPT-4 on larger codebases than previous
methods.  It also reduces the need to manually curate which files to
add to the chat context, empowering GPT to autonomously identify
relevant files for the task at hand.

## Using ctags to make the map

Under the hood, `aider` uses
[universal ctags](https://github.com/universal-ctags/ctags)
to build the
map. Universal ctags can scan source code written in many
languages, and extract data about all the symbols defined in each
file.

For example, here is the `ctags --fields=+S --output-format=json` output for the `main.py` file mapped above:

```json
{
  "_type": "tag",
  "name": "main",
  "path": "aider/main.py",
  "pattern": "/^def main(args=None, input=None, output=None):$/",
  "kind": "function",
  "signature": "(args=None, input=None, output=None)"
}
{
  "_type": "tag",
  "name": "status",
  "path": "aider/main.py",
  "pattern": "/^    status = main()$/",
  "kind": "variable"
}
```

The repo map is built using this `ctags` data.
Rather then sending the data to GPT using verbose json, `aider`
formats the map as a sorted,
hierarchical tree. This is a format that GPT can easily understand and which efficiently conveys the map data using a
minimal number of tokens.

## Example chat transcript

This
[chat transcript](https://aider.chat/examples/add-test.html)
shows GPT-4 creating a black box test case, **without being given
access to the source code of the function being tested or any of the
other code in the repo.** Instead, GPT is operating solely off 
the repo map.

Using only the meta-data in the map, GPT is able to figure out how to call the method to be tested, as well as how to instantiate multiple class objects that are required to prepare for the test.

GPT makes one reasonable mistake writing the first version of the test, but is
able to quickly fix the issue after being shown the `pytest` error output.

## Try it out

To use this experimental repo map feature:

  - Install [aider](https://github.com/paul-gauthier/aider#installation).
  - Install [universal ctags](https://github.com/universal-ctags/ctags).
  - Run `aider` with the `--ctags` option inside your repo.
  