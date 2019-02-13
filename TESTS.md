
pipe & error handling should be modelled on `cat`.  here are some interesting test cases.

#### missing arguments

* cat

```
# NotHere is a file that doesn't exist

# end of arg list, redirect
$ cat MANIFEST.in README.md setup.py NotHere > /dev/null
cat: NotHere: No such file or directory

# end of arg list, pipe
$ cat MANIFEST.in README.md setup.py NotHere | wc -l
cat: NotHere: No such file or directory
54

# beginning of arg list, pipe (no diff)
$ cat NotHere MANIFEST.in README.md setup.py NotHere | wc -l
cat: NotHere: No such file or directory
54
```

* lolcat w/ tweaked IOError catching:

```
$ ./lolcat -f NotHere MANIFEST.in README.md setup.py | wc -l
[Errno 2] No such file or directory: 'NotHere'
54
```

* original lolcat.py

```
$ lolcat.py -f NotHere MANIFEST.in README.md setup.py | wc -l
[Errno 2] No such file or directory: 'NotHere'
54
```

#### ctrl-c interrupt

* cat: `cat /really/big/file > /dev/null` (really hard to interrupt this; cat is fast)

```
$ cat /really/big/file /another/big/file > /somewhere/on/a/slow/disk
^C
$
```

* lolcat catching KeyboardInterrupt:

```
$ ./lolcat -f /var/log/pacman.log > /dev/null
^C
$
```

* original lolcat.py:

```
# times at ~16s
$ lolcat.py -f /var/log/pacman.log > /dev/null
^CTraceback (most recent call last):
  File "lolcat.py", line 198, in <module>
    sys.exit(run())
  File "lolcat.py", line 193, in run
    lolcat.cat(handle, options)
  File "lolcat.py", line 92, in cat
    self.println(line, options)
  File "lolcat.py", line 105, in println
    self.println_plain(s, options)
  File "lolcat.py", line 125, in println_plain
    self.wrap(self.ansi(rgb)),
  File "lolcat.py", line 63, in ansi
    if r < sep or g < sep or b < sep:
KeyboardInterrupt
```

#### pipes

* cat piping into less/head:

```
$ cat lolcat | less
$
# user presses q to exit less
# no error output, returns to prompt on next line

$ cat lolcat | head -1
#!/usr/bin/env python
$
```

* lolcat w/ overridden SIGPIPE handler:

```
$ ./lolcat -f lolcat | less -R
$
# user presses q to exit less
# no error output, returns to prompt on next line

$ ./lolcat -f lolcat | head -1
#!/usr/bin/env python
$
```

* original lolcat.py

```
$ lolcat.py -f lolcat | less -R
# user presses q to exit less
[Errno 32] Broken pipe
Error in atexit._run_exitfuncs:
Traceback (most recent call last):
  File "lolcat.py", line 22, in reset
    sys.stdout.flush()
BrokenPipeError: [Errno 32] Broken pipe
Exception ignored in: <_io.TextIOWrapper name='<stdout>' mode='w' encoding='UTF-8'>
BrokenPipeError: [Errno 32] Broken pipe

# output is colored correctly, though error output is not reset
$ lolcat.py -f lolcat | head -1
#!/usr/bin/env python
[Errno 32] Broken pipe
Error in atexit._run_exitfuncs:
Traceback (most recent call last):
  File "/jovian/bin/lolcat.py", line 22, in reset
    sys.stdout.flush()
BrokenPipeError: [Errno 32] Broken pipe
Exception ignored in: <_io.TextIOWrapper name='<stdout>' mode='w' encoding='UTF-8'>
BrokenPipeError: [Errno 32] Broken pipe
```

#### binary files

* cat:

```
$ cat lolcat.png | wc -l
69

$ cat lolcat.png | head -1
PNG
```

* lolcat w/ `open(... errors='backslashreplace')` (or other handler):

```
$ ./lolcat -f lolcat.png | wc -l
191

# errors='backslashreplace'
$ ./lolcat -f lolcat.png | head -1
\x89PNG

# errors='replace'
$ ./lolcat -f lolcat.png | head -1
�PNG
```

* original lolcat.py (encoding-related exceptions):

```
$ lolcat.py -f lolcat.png | wc -l
Traceback (most recent call last):
  File "lolcat.py", line 198, in <module>
    sys.exit(run())
  File "lolcat.py", line 193, in run
    lolcat.cat(handle, options)
  File "lolcat.py", line 90, in cat
    for line in fd:
  File "/usr/lib/python3.7/codecs.py", line 322, in decode
(result, consumed) = self._buffer_decode(data, self.errors, final)
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x89 in position 0: invalid start byte
0
```

#### other?

