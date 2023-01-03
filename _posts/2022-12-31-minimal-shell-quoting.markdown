---
layout: post
title:  "Minimal shell quoting"
date:   2023-01-01 22:31:18 +0200
tags:   shell
---
Shell quoting is usually a minor nuisance. You let the shell complete a directory name, and it makes
it escaped as in `~/Library/Application\ Support`. Or you surround an argument with single quotes,
because it has a space, or some characters with special meaning. But it quickly gets out of hand
once single quotes themselves need to be quoted. `set -x` for some Bash script that needs debugging,
and the argument `"'foo'"` (or `\'foo\'`) shows up as `''\''foo'\'''`. It's even worse in Python,
where `print(shlex.quote("'foo'"))` prints `''"'"'foo'"'"''`. I assume this use case is deemed too
rare to care about shorter quoting.

Well, making shorter and human-readable shell quoting for unusual inputs is important enough to me,
so I wrote a command-line tool called [`minq`](https://github.com/shayaviv/minq). It finds the
shortest shell quoting for an input string, while breaking ties by:

1. Preferring single quotes over double quotes, and double quotes over escaping
   - `' a b'` is preferable to `" a b"`
   - `" a b"` is preferable to `\ a\ b`
2. Preferring symmetry of quotes
   - `'c d'` is preferable to `c' d'`

This is obviously a case of a solution looking for a problem. So let's find a problem for it to
solve.

Say I have a `tmux` session named "Work stuff" that I want to attach to. Any of these commands
would work:

* `tmux attach-session -t 'Work stuff'`
* `tmux attach-session -t "Work stuff"`
* `tmux attach-session -t Work\ stuff`

Now, let's say I want to execute this `tmux` command on a remote host via SSH. This won't work:

```console
$ ssh myhost tmux attach-session -t 'Work stuff'
command attach-session: too many arguments (need at most 0)
```

Why? Well, `ssh` concatenates all command arguments (separated by space) and executes them in a
remote shell.  So the remote shell actually executes this invalid command:

```sh
tmux attach-session -t Work stuff
```

To fix our `ssh` command, we have to quote the arguments once again, like this:

```sh
ssh myhost 'tmux attach-session -t "Work stuff"'
```

So far, so good. But what if we want to create an alias for it? Then we have to
quote the arguments a third time, like this:

```sh
alias myalias="ssh myhost 'tmux attach-session -t \"Work stuff\"'"
```

But - this whole triple-quoting thing seems very error-prone - why not let the machine do it?  Let's
write a `quote` command which quotes its arguments, then use it like this to quote the whole thing:

```sh
printf '%s\n' "alias myalias=$(
  quote "$(
    quote ssh myhost "$(
      quote tmux attach-session -t 'Work stuff'
    )"
  )"
)"
```

Neat! To make things interesting, let's implement three alternatives for this `quote` command:

```bash
#!/bin/bash
#
# printf.sh (uses: printf %q)

if [[ $# -gt 0 ]]; then
    printf ' %q' "$@" | tail -c +2  # Trim the first space character
fi
printf '\n'
```

```bash
#!/bin/bash
#
# set-x.sh (uses: set -x)

(set -x; : "$@") 2>&1 | sed 's/^+[^ ]* : *//'  # Trim up to (including) the colon
```

```python
#!/usr/bin/env python3
#
# shlex-join.py (uses: shlex.join)

import sys
import shlex
from itertools import islice

if __name__ == "__main__":
    print(shlex.join(islice(sys.argv, 1, None)))
```

Now, let's line them all up and how they compare to `minq`:

```console
$ for script in printf.sh set-x.sh shlex-join.py minq; do
>   alias quote=$script
>   printf '%s:\n  %s\n' "$script" "alias myalias=$(
>     quote "$(
>       quote ssh myhost "$(
>         quote tmux attach-session -t 'Work stuff'
>       )"
>     )"
>   )"
> done
printf.sh:
  alias myalias=ssh\ myhost\ tmux\\\ attach-session\\\ -t\\\ Work\\\\\\\ stuff
set-x.sh:
  alias myalias='ssh myhost '\''tmux attach-session -t '\''\'\'''\''Work stuff'\''\'\'''\'''\'''
shlex-join.py:
  alias myalias='ssh myhost '"'"'tmux attach-session -t '"'"'"'"'"'"'"'"'Work stuff'"'"'"'"'"'"'"'"''"'"''
minq:
  alias myalias="ssh myhost 'tmux attach-session -t Work\\ stuff'"
```

Nice! Did you notice `minq`'s quoting is even shorter than the quoting I did by hand previously?

Here are a few additional examples that demonstrate how shell quoting can be shortened:

| Raw input string | [`minq`](https://github.com/shayaviv/minq) | `printf.sh` | `set-x.sh` | `shlex-join.py` | 
|-|-|-|-|-|
| `10$` | `10$` | `10\$` |  `'10$'` | `'10$'` |
| `$('foo bar')` | `"\$('foo bar')"` | `\$\(\'foo\ bar\'\)` | `'$('\''foo bar'\'')'` | `'$('"'"'foo bar'"'"')'` |
| `$$$'''` | `'$$$'"'''"` | `\$\$\$\'\'\'` | `'$$$'\'''\'''\'''` | `'$$$'"'"''"'"''"'"''` |

Well, that's all I have for now. Thanks for reading!
