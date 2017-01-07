
# rush

[![Go Report Card](https://goreportcard.com/badge/github.com/shenwei356/rush)](https://goreportcard.com/report/github.com/shenwei356/rush)
[![Latest Version](https://img.shields.io/github/release/shenwei356/rush.svg?style=flat?maxAge=86400)](https://github.com/shenwei356/rush/releases)
[![Github Releases](https://img.shields.io/github/downloads/shenwei356/rush/latest/total.svg?maxAge=3600)](https://github.com/shenwei356/rush/releases)

`rush` -- parallelly execute shell commands.

`rush` is a tool similar to [GNU parallel](https://www.gnu.org/software/parallel/)
 and [gargs](https://github.com/brentp/gargs).
 `rush` borrows some idea from them and has some unique features,
  e.g., more advanced embeded strings replacement than `parallel`.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Features](#features)
- [Performance](#performance)
- [Examples](#examples)
- [Usage](#usage)
- [Installation](#installation)
- [Download](#download)
- [Acknowledgements](#acknowledgements)
- [Contact](#contact)
- [License](#license)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


## Features

Major:

- [x] record delimiter (`-D`, default `\n`),
  records sending to every command (`-n`, default `1`),
  and field delimiter (`-d`, default `\s+`).
- [x] keep output order, may use temporary file
- [x] support timeout and retry
- [x] support exit on fist error(s)
- [ ] support continue,
  save status after [capturing ctrl+c](https://nathanleclaire.com/blog/2014/08/24/handling-ctrl-c-interrupt-signal-in-golang-programs/)
- [x] Replacement strings (like GNU parallel):
    - [x] `{#}`, job ID
    - [x] `{}`, full data
    - [x] support positional replacement strings: `{n}`
        - [x] `n`th field in delimiter-delimited data
        - [ ] `n`th matche of regular expression
    - [x] direcotry and file
        - [x] `{/}`, dirname  (`{//}` in GNU parallel)
        - [x] `{%}`, basename (`{/}` in GNU parallel)
        - [x] `{.}`, remove the last extension
        - [x] `{:}`, remove any extension (GNU parallel does not have)
    - [x] combinations:
        - [x] `{%.}`, `{%:}`, basename without extension
        - [x] `{n.}`, `{n/}`, manipulate nth field
- [x] `awk -v` like defined variables
- [ ] appropriate quoting

Minor:

- [x] logging
- [x] dry run
- [x] trim arguments
- [x] verbose

## Performance

Performance of `rush` is similar to `gargs`, and they are both slightly faster than `parallel` (Perl).
See on [release page](https://github.com/shenwei356/rush/releases).

## Examples

1. Simple run, quoting is not necessary

        # seq 1 3 | rush 'echo {}'
        $ seq 1 3 | rush echo {}
        3
        1
        2

1. Read data from file (`-i`)

        $ rush echo {} -i data1.txt -i data2.txt

1. Keep output order (`-k`)

        $ seq 1 3 | rush 'echo {}' -k
        1
        2
        3

1. Timeout (`-t`)

        $ time seq 1 | rush 'sleep 2; echo {}' -t 1
        [ERRO] run command #1: sleep 2; echo 1: time out

        real    0m1.010s
        user    0m0.005s
        sys     0m0.007s

1. Retry (`-r`)

        $ seq 1 | rush 'python unexisted_script.py' -r 1
        python: can't open file 'unexisted_script.py': [Errno 2] No such file or directory
        [WARN] wait command: python unexisted_script.py: exit status 2
        python: can't open file 'unexisted_script.py': [Errno 2] No such file or directory
        [ERRO] wait command: python unexisted_script.py: exit status 2

1. Dirname (`{/}`) and basename (`{%}`)

        $ echo dir/file.txt.gz | rush 'echo {/} {%}'
        dir file.txt.gz

1. Get basename, and remove last (`{.}`) or any (`{:}`) extension

        $ echo dir/file.txt.gz | rush 'echo {%.} {%:}'
        file.txt file

1. Job ID, combine fields index and other replacement strings

        $ echo 123 file.txt | rush 'echo job {#}: {2} {2.}'
        job 1: file.txt file

1. Custom field delimiter (`-d`)

        $ echo a=b=c | rush 'echo {1} {2} {3}' -d =
        a b c

1. Send multi-lines to every command (`-n`)

        $ seq 1 5 | rush -j 3 'echo {1}' -n 2 -k
        1
        3
        5

1. Custom record delimiter (`-D`)

        $ echo -ne ">seq1\nactg\n>seq2\nAAAA\n>seq3\nCCCC"
        >seq1
        actg
        >seq2
        AAAA
        >seq3
        CCCC

        $ echo -ne ">seq1\nactg\n>seq2\nAAAA\n>seq3\nCCCC" | rush -D ">" 'echo FASTA record {#}: name: {1} sequence: {2}' -k -d "\n"
        FASTA record 2: name: seq1 sequence: actg
        FASTA record 3: name: seq2 sequence: AAAA
        FASTA record 4: name: seq3 sequence: CCCC

1. Assign value to variable, like `awk -v` (`-v`)

        $ seq 1  | rush 'echo Hello, {fname} {lname}!' -v fname=Wei -v lname=Shen
        Hello, Wei Shen!

        $ for var in a b; do \
        $   seq 1 3 | rush -k -v var=$var 'echo var: {var}, data: {}'; \
        $ done
        var: a, data: 1
        var: a, data: 2
        var: a, data: 3
        var: b, data: 1
        var: b, data: 2
        var: b, data: 3

## Usage

```
rush -- parallelly execute shell commands

Version: 0.0.4

Author: Wei Shen <shenwei356@gmail.com>

Source code: https://github.com/shenwei356/rush

Usage:
  rush [flags] [command] [args of command...]

Examples:
  1. simple run, quoting is not necessary
      $ seq 1 10 | rush echo {}
  2. keep order
      $ seq 1 10 | rush 'echo {}' -k
  3. timeout
      $ seq 1 | rush 'sleep 2; echo {}' -t 1
  4. retry
      $ seq 1 | rush 'python script.py' -r 3
  5. dirname & basename
      $ echo dir/file.txt.gz | rush 'echo {/} {%}'
      dir file.txt.gz
  6. basename without last or any extension
      $ echo dir/file.txt.gz | rush 'echo {%.} {%:}'
      file.txt file
  7. job ID, combine fields and other replacement strings
      $ echo 123 file.txt | rush 'echo job {#}: {2} {2.}'
      job 1: file.txt file
  8. custom field delimiter
      $ echo a=b=c | rush 'echo {1} {2} {3}' -d =
      a b c
  9. assign value to variable, like "awk -v"
      $ seq 1 | rush 'echo Hello, {fname} {lname}!' -v fname=Wei -v lname=Shen
      Hello, Wei Shen!
  More examples: https://github.com/shenwei356/rush

Flags:
  -v, --assign stringSlice        assign the value val to the variable var (format: var=val)
      --dry-run                   print command but not run
  -d, --field-delimiter string    field delimiter in records, support regular expression (default "\s+")
  -i, --infile stringSlice        input data file, multi-values supported
  -j, --jobs int                  run n jobs in parallel (default value depends on your device) (default 4)
  -k, --keep-order                keep output in order of input
  -n, --nrecords int              number of records sent to a command (default 1)
  -o, --out-file string           out file ("-" for stdout) (default "-")
  -D, --record-delimiter string   record delimiter, supports regular expression (default is "\n") (default "
")
  -r, --retries int               maximum retries (default 0)
      --retry-interval int        retry interval (unit: second) (default 0)
  -e, --stop-on-error             stop all processes on first error(s)
  -t, --timeout int               timeout of a command (unit: second, 0 for no timeout) (default 0)
      --trim string               trim white space in input (available values: "l" for left, "r" for right, "lr", "rl", "b" for both side)
      --verbose                   print verbose information
  -V, --version                   print version information and check for update

```

## Installation

`rush` is implemented in [Go](https://golang.org/) programming language,
 executable binary files **for most popular operating systems** are freely available
  in [release](https://github.com/shenwei356/rush/releases) page.

Just [download](https://github.com/shenwei356/rush/releases) compressed
executable file of your operating system,
and uncompress it with `tar -zxvf xxx.tar.gz` command or other tools.
And then:

1. For Unix-like systems
    1. If you have root privilege simply copy it to `/usr/local/bin`:

            sudo cp rush /usr/local/bin/

    1. Or add the current directory of the executable file to environment variable
    `PATH`:

            echo export PATH=\"$(pwd)\":\$PATH >> ~/.bashrc
            source ~/.bashrc

1. For windows, just copy `rush.exe` to `C:\WINDOWS\system32`.

For Go developer, just one command:

    go get -u github.com/shenwei356/rush/rush


## Download

[rush v0.0.4](https://github.com/shenwei356/rush/releases/tag/v0.0.4)
[![Github Releases (by Release)](https://img.shields.io/github/downloads/shenwei356/rush/v0.0.4/total.svg)](https://github.com/shenwei356/rush/releases/tag/v0.0.4)

OS     |Arch      |File                                                                                                                          |Download Count
:------|:---------|:-----------------------------------------------------------------------------------------------------------------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Linux  |32-bit    |[rush_linux_386.tar.gz](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_linux_386.tar.gz)                    |[![Github Releases (by Asset)](https://img.shields.io/github/downloads/shenwei356/rush/latest/rush_linux_386.tar.gz.svg?maxAge=3600)](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_linux_386.tar.gz)
Linux  |**64-bit**|[**rush_linux_amd64.tar.gz**](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_linux_amd64.tar.gz)            |[![Github Releases (by Asset)](https://img.shields.io/github/downloads/shenwei356/rush/latest/rush_linux_amd64.tar.gz.svg?maxAge=3600)](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_linux_amd64.tar.gz)
OS X   |32-bit    |[rush_darwin_386.tar.gz](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_darwin_386.tar.gz)                  |[![Github Releases (by Asset)](https://img.shields.io/github/downloads/shenwei356/rush/latest/rush_darwin_386.tar.gz.svg?maxAge=3600)](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_darwin_386.tar.gz)
OS X   |**64-bit**|[**rush_darwin_amd64.tar.gz**](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_darwin_amd64.tar.gz)          |[![Github Releases (by Asset)](https://img.shields.io/github/downloads/shenwei356/rush/latest/rush_darwin_amd64.tar.gz.svg?maxAge=3600)](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_darwin_amd64.tar.gz)
Windows|32-bit    |[rush_windows_386.exe.tar.gz](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_windows_386.exe.tar.gz)        |[![Github Releases (by Asset)](https://img.shields.io/github/downloads/shenwei356/rush/latest/rush_windows_386.exe.tar.gz.svg?maxAge=3600)](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_windows_386.exe.tar.gz)
Windows|**64-bit**|[**rush_windows_amd64.exe.tar.gz**](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_windows_amd64.exe.tar.gz)|[![Github Releases (by Asset)](https://img.shields.io/github/downloads/shenwei356/rush/latest/rush_windows_amd64.exe.tar.gz.svg?maxAge=3600)](https://github.com/shenwei356/rush/releases/download/v0.0.4/rush_windows_amd64.exe.tar.gz)

## Acknowledgements

Specially thank [@brentp](https://github.com/brentp)
and his [gargs](https://github.com/brentp/gargs), from which `rush` borrows
lots of ideas.

## Contact

[Create an issue](https://github.com/shenwei356/rush/issues) to report bugs,
propose new functions or ask for help.

## License

[MIT License](https://github.com/shenwei356/rush/blob/master/LICENSE)
