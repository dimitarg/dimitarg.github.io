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