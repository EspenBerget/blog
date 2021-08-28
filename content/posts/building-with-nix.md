---
title: "Building with Nix"
date: 2021-08-24T21:52:48+02:00
draft: false
---

Part of my intent with the [last post][prev] was to create something I could try to ship as a Nix derivation.

I really like the ide of a purely functional package manager. 
Particularly one that has the reach of Nix. 

## About Nix
In the development cycle we are plagued by dependencies not only in production, but also in development. 
Setting up a project can be just as much of a hassel as making it run on the clients computers, or servers. 
Further the increase in production environment choices means more to learn for the developers. Company wants Azure, well time to read the docs, new company uses docker and kubernetes... time to read the docs. 
Would be nice to have a more uniform way of dealing with all this. A language for building software, and a environment of atomic, uniform, and consistent development, where "it works on my machine" is not just solved in production, but also in development. 

Nix offers all this. It's usability stretch from building a simple app, like this tutorial to configuring an entire OS (see NixOS).

Nix itself is a purely functional, lazy evaluated language. It is the heart of the nix ecosystem, which also provides `nix-build` for building, `nix-shell` to run and try your program in a consistent manner. If your program have a dependency found on your computer, nix-shell still won't use it, instead you must state its dependency and install it in nix store. This means that you wont ship code with "incomplete dependencies".

There is also a `nix-env` which is similar to `apt`, `npm`, `brew`, etc... 

### Nix Language
The few components of the nix language I deem important for this tutorial are sets and functions.

A set in nix looks like this `{ a = 1; b = 2; }`. Important to note that values are separated by a semicolon, and all values must be followed by one!
Sets are frequently used to passed attributes to functions.

A function in nix looks lif `f = a: b: a + b`, well this is really two functions. A function takes an argument followed by a colon and a function body.
This one her is assigned to `f` and take a value, returns a function that takes a value and adds the two values together. 

A more typical example of a function is 
```nix
let 
    add = { a, b }: a + b;
in 
    add {a = 1; b = 2;}
```
This one takes a set and adds the values. You see how it is called in the last line. 

Finally we should mention derivations. This is what we will make in order to have our program build in the standard environment. Nix is all about derivations, they hold the necessary information for nix to build, and make sure all dependencies are met. 


## Using Nix
So just getting started with Nix for our backend app is no issue.
We will just write a simple `default.nix` file. 

##### **default.nix**
```nix
with (import <nixpkgs> {});     # 1.

stdenv.mkDerivation {           # 2.
    pname = "DenoServer";       # 3.
    version = "1.0.0";          # 3.

    buildInputs = [ deno ];     # 4.

    shellHook = ''              
        deno run --allow-net --allow-write --allow-read index.js
    '';                         # 5.
}
```
> ##### To understand more of mkDerivation go to [this site][derivation]

1. This is a simple way off getting access to the nix ecosystem. We simply import all nix packages, and make them accessible to our environment using `with`. You might think this is overkill, all 80,000 packages! But remember that nix is lazy, it does not import anything right away, instead it only gives us what we use, when we use it. This is merly a declarative statement, saying "make sure all thing are accessible if needed".

2. This is the important bit, making a derivation. Remember that they are the meat of nix development. We pass a set to the function which must contain some thing required for the derivation.

3. These two are more for logistical purposes. We need a name and a version. We could also use the `name` attribute like `name = "DenoServer-1.0.0";`. 

4. This is the reason I thought this app would be a simple start. We only need to state one dependency, namely deno. Dependencies are given in the `buildInputs` attribute in the set passed to `mkDerivation`.

5. Finally we want the `nix-shell` command to start the deno server. So we pass a command argument to the `shellHook` attribute. The double ticks are nix's way of passing multi line strings.


Now running `nix-shell` in the backend folder should make nix build a project from scratch, import all dependencies (you will see more that deno, because deno and every other program have some dependencies them self). 
Lastly you will see the line `Starting server on localhost:8001`.

Our server now runs on any nix supported platform, without us needing to think about what must be installed, nix handles that.

## Building with Nix
The first example shows that nix can setup a deno environment and run it, but it not really a good use of nix's powers. 
All we do is make it easier to install and run deno, given that the user have nix, if not, we made it worse! 

So instead lets build a derivation with nix instead. Remember nix is all about derivations. Before I make a builder with nix, our directory is becoming a bit tangled with js files and nix files. Would be nice to separate them, give the project a focus. DevOps in one folder, dev in the other. Two simple commands would do. 
```bash
$ mkdir src
$ mv *.js src/
```
There, now the js source (dev stuff) is in `src` and nix files (devops) is in the top folder.

Now to the derivation building.

##### **default.nix**
```nix
with (import <nixpkgs> {});             

stdenv.mkDerivation {                   
    pname = "DenoServer";               # 1.
    version = "1.0.0";                  # 1.

    buildInputs = [ deno ];             # 1.

    src = ./src;                        # 2.

    DENO_DIR = ".cached";               # 3.

    buildPhase = ''                     
        deno compile -o server \
            --allow-net --allow-read --allow-write \
             index.js 
    '';                                 # 4

    installPhase = ''
        mkdir -p $out/bin
        mv server $out/bin
    '';                                 # 5

    shellHook = ''
        $out/bin/server
    '';                                 # 6.
}
```

1. All of these are the same as before.

2. This informs nix about our src directory, fittingly the attribute is  named src.

3. This is a neat thing about mkDerivation. Often the builder is a bash session, and the attributes are passed to it as environment variables. Turns out deno need this one to be set in order to work, and we can simply set it her. No hassel! 

4. Here we take advantage of deno's compile command. It is experimental though, but to good for my use case to ignore. It will create a binary file that we will call later, we named this file `server`.
            
5. Next we install our new binary in a folder within the derivation called bin. The derivation path is stored in $out. After we made this folder we move the server binary there.

6. Finally if we choose to run `nix-shell` it will now start the server.

Now running `nix-build` in the root of our project will create the derivation and add it to our nix store. This also gives us a result folder which if we run `./result/bin/server` will start the server. 

## Conclusion
Nix is easier to set up then expected. This is not the full extend of nix's capabilities by far, but if you want to learn more check out the [nix pils][pils] tutorial series, and learn the language at [a tour of nix][tour].

For me this was a first derivation and I think nix is my goto tool in the future. 

[prev]: (https://espenberget.github.io/trying-deno-and-building-a-server-api/)
[pils]: (https://nixos.org/guides/nix-pills/index.html)
[tour]: (https://nixcloud.io/tour/?id=1)
[derivation]: (http://blog.ielliott.io/nix-docs/stdenv-mkDerivation.html)
