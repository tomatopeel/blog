---
title: "Codewalk #1: Adding a File to IPFS"
date: 2017-06-24T16:48:48+02:00
draft: false
categories: [ "ipfs" ]
tags: [ "ipfs" ]
---
``` bash
$ echo "hello" | ipfs add
added QmZULkCELmmk5XNfCgTnCyFgAVxBRBXyDHGGMVoLFLiXEN QmZULkCELmmk5XNfCgTnCyFgAVxBRBXyDHGGMVoLFLiXEN
```
What's this? The string "hello" is piped through to the STDIN of the `ipfs add` command and some output is returned to us. Let's walk through the code to see what's actually happening here.

``` bash
$ cd $GOPATH/src/github.com/ipfs/go-ipfs
$ grep -rn 'func main()'
...
cmd/ipfs/main.go:61:func main() {
...
```

Here's our entry point.

``` go
// cmd/ipfs/main.go

package main

func main() {
	os.Exit(mainRet())
}

func mainRet() int {
	...
	// parse the commandline into a command invocation
	parseErr := invoc.Parse(ctx, os.Args[1:])
	...
}

...

func (i *cmdInvocation) Parse(ctx context.Context, args []string) error {
	var err error

	i.req, i.cmd, i.path, err = cmdsCli.Parse(args, os.Stdin, Root)
	...
}
```

`args` is just `[]string{"add"}`, what is `Root`?

<small>_Note: I use Emacs with `go-mode.el` with `godef` support, so `C-c C-j` will jump to a function in another file which is extremely useful. I also recommend `backward-forward` and `helm-mode`'s `helm-all-mark-rings`. I also use the [Delve debugger](https://github.com/derekparker/delve)._</small>

```go
// cmd/ipfs/ipfs.go

package main

import (
	"fmt"
	cmds "github.com/ipfs/go-ipfs/commands"
	commands "github.com/ipfs/go-ipfs/core/commands"
)

// This is the CLI root, used for executing commands accessible to CLI clients.
// Some subcommands (like 'ipfs daemon' or 'ipfs init') are only accessible here,
// and can't be called through the HTTP API.
var Root = &cmds.Command{
	Options:  commands.Root.Options,
	Helptext: commands.Root.Helptext,
}
```
Which refers to...
``` go

// core/commands/root.go

var Root = &cmds.Command{
	Helptext: cmds.HelpText{
		// blah blah blah
	},
	Options: []cmds.Option{
		cmds.StringOption("config", "c", "Path to the configuration file to use."),
		cmds.BoolOption("debug", "D", "Operate in debug mode.").Default(false),
		cmds.BoolOption("help", "Show the full command help text.").Default(false),
		cmds.BoolOption("h", "Show a short version of the command help text.").Default(false),
		cmds.BoolOption("local", "L", "Run the command locally, instead of using the daemon.").Default(false),
		cmds.StringOption(ApiOption, "Use a specific API instance (defaults to /ip4/127.0.0.1/tcp/5001)"),
	},
}

```
Cool, now back to `cmdsCli.Parse`
``` go
// commands/cli/parse.go

func Parse(input []string, stdin *os.File, root *cmds.Command) (cmds.Request, *cmds.Command, []string, error) {
	path, opts, stringVals, cmd, err := parseOpts(input, root)
	...
}

func parseOpts(args []string, root *cmds.Command) (
	path []string,
	opts map[string]interface{},
	stringVals []string,
	cmd *cmds.Command,
	err error,
) {
	path = make([]string, 0, len(args)) // path is an empty []string, of len 1
	stringVals = make([]string, 0, len(args))
	optDefs := map[string]cmds.Option{}
	opts = map[string]interface{}{}
	cmd = root
	...
	optDefs, err = root.GetOptions(path) // path is an empty []string, of len 1
	...
}
```

Looks like option parsing stuff is coming up, though we've made this `path` slice and we're passing it to `*Command.GetOptions` without having put anything meaningful into it. Presumably the logic here will become clearer as we move through the code.
``` go
// commands/command.go

func (c *Command) GetOptions(path []string) (map[string]Option, error) {
	options := make([]Option, 0, len(c.Options))

	cmds, err := c.Resolve(path) // c is the root command
	...
}

...

func (c *Command) Resolve(pth []string) ([]*Command, error) {
	cmds := make([]*Command, len(pth)+1)
	cmds[0] = c

	cmd := c
	for i, name := range pth {
		cmd = cmd.Subcommand(name)

		if cmd == nil {
			pathS := path.Join(pth[:i])
			return nil, fmt.Errorf("Undefined command: '%s'", pathS)
		}

		cmds[i+1] = cmd
	}

	return cmds, nil // cmds is a []*Command with only the root cmd.
}

...

func (c *Command) GetOptions(path []string) (map[string]Option, error) {
	...
	cmds, err := c.Resolve(path) // c is the root command
	cmds = append(cmds, globalCommand)
}
```

What's `globalCommand`?

``` go
// commands/option.go

// Flag names
const (
	EncShort   = "enc"
	EncLong    = "encoding"
	RecShort   = "r"
	RecLong    = "recursive"
	ChanOpt    = "stream-channels"
	TimeoutOpt = "timeout"
)

// options that are used by this package
var OptionEncodingType = StringOption(EncLong, EncShort, "The encoding type the output should be encoded with (json, xml, or text)")
var OptionRecursivePath = BoolOption(RecLong, RecShort, "Add directory paths recursively").Default(false)
var OptionStreamChannels = BoolOption(ChanOpt, "Stream channel output")
var OptionTimeout = StringOption(TimeoutOpt, "set a global timeout on the command")

// global options, added to every command
var globalOptions = []Option{
	OptionEncodingType,
	OptionStreamChannels,
	OptionTimeout,
}

// the above array of Options, wrapped in a Command
var globalCommand = &Command{
	Options: globalOptions,
}
```

Looks like we're sticking the global options into a phony `Command`, okay...

``` go
// commands/command.go

func (c *Command) GetOptions(path []string) (map[string]Option, error) {
	options := make([]Option, 0, len(c.Options))

	cmds, err := c.Resolve(path)
	if err != nil {
		return nil, err
	}
	cmds = append(cmds, globalCommand)

	for _, cmd := range cmds {
		options = append(options, cmd.Options...)
	}

	optionsMap := make(map[string]Option)
	for _, opt := range options {
		for _, name := range opt.Names() {
			if _, found := optionsMap[name]; found {
				return nil, fmt.Errorf("Option name '%s' used multiple times", name)
			}

			optionsMap[name] = opt
		}
	}

	return optionsMap, nil
}
```

So we're returning this `optionsMap map[string]Option` in order to merge the `globalOptions` options with the `Root *Command`'s options, which becomes `optDefs` here:

``` go
	...
	optDefs, err = root.GetOptions(path)
	if err != nil {
		return
	}

	consumed := false
	for i, arg := range args {
	...
```

A quick look at `args` in [Delve](https://github.com/derekparker/delve):
``` text
(dlv) p args
[]string len: 2, cap: 2, [
	"add",
	"/dev/fd/63",
]
```

`/dev/fd/$n` is a file descriptor for our STDIN.

``` go
	consumed := false
	for i, arg := range args {
		switch {
		...
		// cases for --, --foo, -foo here
		...
		default:
			// arg is a sub-command or a positional argument
			sub := cmd.Subcommand(arg)
			if sub != nil {
				cmd = sub
				path = append(path, arg)
				optDefs, err = root.GetOptions(path)
				if err != nil {
					return
				}

				// If we've come across an external binary call, pass all the remaining
				// arguments on to it
				if cmd.External {
					stringVals = append(stringVals, args[i+1:]...)
					return
				}
			} else {
				stringVals = append(stringVals, arg)
				if len(path) == 0 {
					// found a typo or early argument
					err = printSuggestions(stringVals, root)
					return
				}
			}
		}

```
