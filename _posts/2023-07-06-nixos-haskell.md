---
title: "Quick and dirty setup for Haskell on NixOS"
date: 2023-07-06T8:00:00Z
published: true
categories:
  - Functional Programming
tags:
  - fp
---

# Usecase

I have a bunch of free time and want to pick up [haskellbook](https://haskellbook.com/) again. Unfortunately, three years have
passed since my last attempt, so I have little recollection of my previous progress. I'm prompted to restart from the very beginning.

In the meantime, I've switched my operating system from Ubuntu and PopOS to NixOS, so I need to figure out how to set up a Haskell
development environment on NixOS. This includes IDE support for my current editor of choice, VS Code.

I'm both a Nix noob and a Haskell noob, so I don't want to mess around too long with this. Anything quick and dirty that works will
fit the bill. I don't have any strong opinions on the "correct" setup either.

# Stack

The first thing I need is a ghc / ghci installation. I know one way to do this is obtain it via a build tool, `stack`. That's also
what the book recommends, so I'm going to go with the flow here.

```
stack
```

```
The program 'stack' is not in your PATH. It is provided by several packages.
You can make it available in an ephemeral shell by typing one of the following:
  nix-shell -p haskellPackages.stack
  nix-shell -p stack
```

Okay. 

As we said, we don't want to mess around with nix expressions right now. Let's go with an ephemeral shell.

```
nix-shell -p stack
stack
```

```
stack ghci

Note: No local targets specified, so a plain ghci will be started with no package hiding or package options.
      
      You are using snapshot: lts-21.1
      
      If you want to use package hiding and options, then you can try one of the following:
      
      * If you want to start a different project configuration than /home/fmap/.stack/global-project/stack.yaml, then you can use stack init to create a new stack.yaml for
        the packages in the current directory. 
        
      * If you want to use the project configuration at /home/fmap/.stack/global-project/stack.yaml, then you can add to its 'packages' field.
      
Configuring GHCi with the following packages: 
GHCi, version 9.4.5: https://www.haskell.org/ghc/  :? for help
Loaded GHCi configuration from /run/user/1000/haskell-stack-ghci/2a3bbd58/ghci-script
ghci> 
```

Cool.

Note that under my current setup (`nixos-unstable`, `nix-channel` managed `nixpkgs`) and at the time of writing, `stack` points to
- `stack 2.9.1`
- `lts-21.1` resolver
- `9.4.5` ghc

This might vary for you depending on your specific channel state or `flake` input. 

# Create project

Let's create a new project that's going to be our workspace for code snippets and exercises while working through the book.

```
stack new haskbook
```

Once this finishes, let's browse around to see what's been created.

```
cd haskbook
code .
```

There's an exampe haskell file created in the project, `Lib.hs`. It is recognised as a haskell source file, but on a plain VSCode
installation, it doesn't even have syntax highlighting. 

Let's work on IDE setup, then.

# `code` extension

A google search for "vscode haskell" yields https://github.com/haskell/vscode-haskell.  The extension name is `haskell.haskell`.
We add it to our IDE setup. In my case, this is managed by `home-manager`:

```nix
programs.vscode = {
  enable = true;
  package = pkgs.vscode;
  extensions = [
    pkgs.vscode-extensions.bbenoist.nix
    pkgs.vscode-extensions.scalameta.metals
    pkgs.vscode-extensions.scala-lang.scala
    pkgs.vscode-extensions.haskell.haskell
  ];
};
```

Activate that change, and let's relaunch `code` again. Open a haskell file, and the extension greets us with

```
Project requires HLS but it isn't installed
```

# Haskell Language Server

HLS stands for `haskell-language-server` This is a (LSP) backend that the IDE extension needs.

If I tried typing that in a console, I get

```
haskell-language-server
The program 'haskell-language-server' is not in your PATH. You can make it available in an
ephemeral shell by typing:
  nix-shell -p haskellPackages.haskell-language-server
```

Okay. Let's launch a shell with `stack` and `haskell-language-server`, and relaunch `code`.

```
nix-shell -p stack haskellPackages.haskell-language-server
```

```
code .
```

I fire up the IDE and open the `Lib.hs` file. I get nothing on hover, so something must be wrong.
Select "Output" in VSCode, and select "Haskell" - the standard output of the extension might point us to an error.

```
2023-07-06 10:36:55.2430000 [client] INFO Activating the language server in working dir: /home/fmap/work/own/haskbook (the workspace folder)
2023-07-06 10:36:55.2450000 [client] INFO run command: haskell-language-server-wrapper --lsp
2023-07-06 10:36:55.2450000 [client] INFO debug command: haskell-language-server-wrapper --lsp
2023-07-06 10:36:55.2450000 [client] INFO server environment variables:
2023-07-06 10:36:55.2660000 [client] INFO Starting language server
No 'hie.yaml' found. Try to discover the project type!
Run entered for haskell-language-server-wrapper(haskell-language-server-wrapper) Version 2.0.0.0 x86_64 ghc-9.2.8
Current directory: /home/fmap/work/own/haskbook
Operating system: linux
Arguments: ["--lsp"]
Cradle directory: /home/fmap/work/own/haskbook
Cradle type: Stack

Tool versions found on the $PATH
cabal:          Not found
stack:          2.9.1
ghc:            9.2.8


Consulting the cradle to get project GHC version...
2023-07-06T10:36:56.298573Z | Debug | executing command: stack setup --silent
2023-07-06T10:36:58.571617Z | Debug | executing command: stack exec ghc -- --numeric-version
Project GHC version: 9.4.5
haskell-language-server exe candidates: ["haskell-language-server-9.4.5","haskell-language-server"]
Launching haskell-language-server exe at:/nix/store/dbyjzdqcvn2q1wha2i1p4bf9jdwm23k2-haskell-language-server-2.0.0.0/bin/haskell-language-server
2023-07-06T10:37:00.589523Z | Debug | executing command: stack setup --silent
2023-07-06T10:37:02.379620Z | Debug | executing command: stack exec ghc -- -v0 -package-env=- -ignore-dot-ghci -e Control.Monad.join (Control.Monad.fmap System.IO.putStr System.Environment.getExecutablePath)
2023-07-06T10:37:04.753173Z | Debug | executing command: stack setup --silent
2023-07-06T10:37:06.926440Z | Debug | executing command: stack exec ghc -- --print-libdir
[0;31mGHC versions don't match![0m
[0;31m[0m
[0;31mExpected: 9.2.8[0m
[0;31mGot:      9.4.5[0m
[Error - 13:37:08] Connection to server is erroring. Shutting down server.
[Error - 13:37:08] Connection to server is erroring. Shutting down server.
```

Right. Apparently `ghc` and `haskell-language-server` must agree on the version, and they don't.
We previously launched a `nix-shell`, let's see what `haskell-language-server` version that brought us.

```
haskell-language-server --version
2023-07-06T10:42:08.734221Z | Info | No log file specified; using stderr.
haskell-language-server version: 2.0.0.0 (GHC: 9.2.8) (PATH: /nix/store/dbyjzdqcvn2q1wha2i1p4bf9jdwm23k2-haskell-language-server-2.0.0.0/bin/.haskell-language-server-9.2.8-unwrapped)
```

Okay. In our case and at the time of writing, that's `haskell-language-server-9.2.8`. But GHC is `9.4.5`, so that's no good.

# Overriding HLS version

A quick web search and we hopefully end up [here](https://haskell4nix.readthedocs.io/nixpkgs-users-guide.html#how-to-install-haskell-language-server). This tells us there's an attribute `supportedGhcVersions` which will let us
override the version.

Summon the incantation:

```
nix-shell -p stack 'pkgs.haskell-language-server.override { supportedGhcVersions = [ "945" ]; }'
```

```
code .
```

HLS will take a bunch of time to fire up, and afterwards we get a working IDE integration:

![alt text](../assets/videos/working-ide.webm "VSCode HLS Integration working.")

Happy hacking!


