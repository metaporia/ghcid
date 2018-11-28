# ghcid [![Hackage version](https://img.shields.io/hackage/v/ghcid.svg?label=Hackage)](https://hackage.haskell.org/package/ghcid) [![Stackage version](https://www.stackage.org/package/ghcid/badge/nightly?label=Stackage)](https://www.stackage.org/package/ghcid) [![Linux Build Status](https://img.shields.io/travis/ndmitchell/ghcid/master.svg?label=Linux%20build)](https://travis-ci.org/ndmitchell/ghcid) [![Windows Build Status](https://img.shields.io/appveyor/ci/ndmitchell/ghcid/master.svg?label=Windows%20build)](https://ci.appveyor.com/project/ndmitchell/ghcid)

Either "GHCi as a daemon" or "GHC + a bit of an IDE". To a first approximation, it opens `ghci` and runs `:reload` whenever your source code changes, formatting the output to fit a fixed height console. Unlike other Haskell development tools, `ghcid` is intended to be _incredibly simple_. In particular, it doesn't integrate with any editors, doesn't depend on GHC the library and doesn't start web servers.

_Acknowledgements:_ This project incorporates significant work from [JPMoresmau](https://github.com/JPMoresmau), who is listed as a co-author.

### Using it

Run `stack install ghcid` or `cabal update && cabal install ghcid` to install it as normal. Then run `ghcid "--command=ghci Main.hs"`. The `command` is how you start your project in `ghci`. If you omit `--command` then it will default to `stack ghci` if you have the `stack.yaml` file and `.stack-work` directory, default to `ghci` if you have a `.ghci` file in the current directory, and otherwise default to `cabal repl`.

Personally, I always create a `.ghci` file at the root of all my projects, which usually [reads something like](https://github.com/ndmitchell/ghcid/blob/master/.ghci):

    :set -fwarn-unused-binds -fwarn-unused-imports
    :set -isrc
    :load Main

After that, resize your console and make it so you can see it while working in your editor. On Windows you may wish to pass `--topmost` so the console will sit on top of all other windows. On Linux, you probably want to use your window manager to make it topmost or use a [tiling window manager](http://xmonad.org/).

### Vim API

* `Ghcid <args`: runs ghci command from config, or default, `stack ghci`
* `GhcidKill` : kills ghcid process--necessary when a new dependency or module
   has been added
* `GhcidRestart` : runs `Ghcid | GhcidKill`

### What you get

On every save you'll see a list of the errors and warnings in your project. It uses `ghci` under the hood, so even relatively large projects should update their status pretty quickly. As an example:

    Main.hs:23:10:
        Not in scope: `verbosit'
        Perhaps you meant `verbosity' (imported from System.Console.CmdArgs)
    Util.hs:18:1: Warning: Defined but not used: `foo'

Or, if everything is good, you see:

    All good

Please [report any bugs](https://github.com/ndmitchell/ghcid/issues) you find.

### Editor integration

There are a few plugins that integrate Ghcid into editors, notably:

* [VS Code](plugins/vscode/)
* [nvim](plugins/nvim/)
* [Emacs](plugins/emacs/)

### Usage tips

* If you have a `.ghcid` file in the current folder, or a parent folder, the contents of that file will be used as command line arguments. For example, if you always want to pass `--command=custom-ghci` then putting that in a `.ghcid` file will free you from writing it each time.
* There is an article on [auto-reloading threepenny-gui apps during development](https://binarin.ru/post/auto-reload-threepenny-gui/).
* There are a list of [general tips for using Ghcid](http://www.parsonsmatt.org/2018/05/19/ghcid_for_the_win.html).

### FAQ

* _This isn't as good as full IDE._ I've gone for simplicity over features. It's a point in the design space, but not necessarily the best point in the design space for you. For "real" IDEs see [the Haskell wiki](http://www.haskell.org/haskellwiki/IDEs).
* _If I delete a file and put it back it gets stuck._ Yes, that's a [bug in GHCi](https://ghc.haskell.org/trac/ghc/ticket/9648). If you see GHCi getting confused just kill `ghcid` and start it again.
* _I want to run my tests when files change._ You can pass any `ghci` expression with the `--test` flag, e.g. `--test=:main`, which will be run whenever the code is warning free (or pass `--warnings` for when the code is merely error free).
* _I want to run arbitrary commands when arbitrary files change._ This project reloads `ghci` when files loaded by `ghci` change. If you want a more general mechanism something like [Steel Overseer](https://github.com/schell/steeloverseer) or [Watchman](https://facebook.github.io/watchman/) will probably work better.
* _I want syntax highlighting in the error messages._ One option is to use Neovim or Emacs and run the terminal in a buffer whose file type is set to Haskell. Another option is to pipe `ghcid` through [source-highlight](https://www.gnu.org/software/src-highlite/) (`ghcid | source-highlight -s haskell -f esc`).
* _I'm not seeing pattern matching warnings._ Ghcid automatically appends `-fno-code` to the command line, which makes the reload cycle about twice as fast. Unfortunately GHC 8.0 and 8.2 suffer from [bug 10600](https://ghc.haskell.org/trac/ghc/ticket/10600) which means `-fno-code` also disables pattern matching warnings. On these versions, either accept no pattern match warnings or use `-c` to specify a command line to start `ghci` that doesn't include `-fno-code`. From GHC 8.4 this problem no longer exists.
* _I get "During interactive linking, GHCi couldn't find the following symbol"._ This problem is a manifestation of [GHC bug 8025](https://ghc.haskell.org/trac/ghc/ticket/8025), which is fixed in GHC 8.4 and above. Ghcid automatically appends `-fno-code` to the command line, but for older GHC's you can supress that with `--test "return ()"` (to add a fake test) or `-c "ghci ..."` to manually specify the command to run.
* _I only see source-spans or error messages on errors/warnings after the first load._ Due to limitations in `ghci`, these flags are only set _after_ the first load. If you want them to apply from the start, pass them on the command line to `ghci` with something like `-c "ghci -ferror-spans -fdiagnostics-color=always".
* _I want to match on the file/line/column to get jump-to-error functionality in my editor._ You will variously see `file:line:col:message`, `file:line:col1-col2:msg` and `file:(line1,col1)-(line2,col2):message`, as these are the formats GHC uses. To match all of them you can use a regular expression such as `^(\\S*?):(?|(\\d+):(\\d+)(?:-\\d+)?|\\((\\d+),(\\d+)\\)-\\(\\d+,\\d+\\)):([^\n]*)`.
