# node-elm-repl

* [Description](#Description)
* [How to use it with Node.js?](#how-to-use-it-with-nodejs)
* [How to use it with CLI?](#how-to-use-it-with-cli)
* [How to contribute?](#how-to-contribute)
* [How it was done?](#how-it-was-done)

# Description

This package is a very specific and weird package.

So if you (yet) don't care about [Elm language](http://elm-lang.org), you shouldn't be interested. Actually, even if you are a big fan of the [Elm language](http://elm-lang.org) (like me, since 0.15), there's totally no guarantee that this package could be interesting to you too.

Why? Because it actually does exactly what [`elm-repl`](https://github.com/elm-lang/elm-repl) does. Even less. It just gets the types of expressions, no values. But... what... changes... everything... is... this package does it completely with node.js!!

And it calculates all the expressions types you asked for just with one single call of `elm-make` (original `elm-repl` currently recompiles everything on every new input).

This makes this REPL three times faster and three times awesomer, since it gives you access to the JSONified structure of the type, instead of just a boring string.

How it got to be so fast and good? Read [How it was done?](#how-it-was-done) below for the nice story and some technical details.

NB: It's work-in-progress, so at some moment you could discover that it parses no records, for example (just for example!) or something even worse. If you find such case, please don't panic, please just follow at least some of the steps described in [Contribute](#how-to-contribute) section.

![Example from CLI](https://raw.githubusercontent.com/shamansir/node-elm-repl/master/img/cli-example.png)

## How to use it with Node.js?

It's not yet published to `npm.js` (if ever will), but you may install it using:

```
npm install git://github.com/shamansir/node-elm-repl.git
```

It's better to ensure that binary parser works with your elm version etc.,
so please first run this in the directory where you've installed the package:

```
npm test
```

Then, in your JS file, and in the same directory where you have `elm-package.json` or where you store your `.elm` modules, if you have any, just do something like:

```javascript
const Repl = require('node-elm-repl');

new Repl({ // options, defaults are listed:
    workDir: '.', // working directory
    elmVer: '0.17.1', // your exact elm-compiler version
    user: 'user', // specify github username you used in elm-package.json
    project: 'project', // specify project name you used in elm-package.json
    projectVer: '1.0.0' // specify project version you used in elm-package.json
}).getTypes(
    [ // imports:
        'List as L',
        'Maybe exposing ( Maybe(..) )'
    ],
    [ // expressions:
        'L.map',
        'L.foldl',
        'Just',
        'Nothing',
        '1 + 1',
        '\\a b -> a + b'
    ]
).then(function(types) { // getTypes returns the Promise which resolves to array
    console.log(Repl.stringifyAll(types).join('\n'));
}).catch(console.error);
```

And when you run this script in the console you should see something like:

```
(a -> b) -> List a -> List b
(a -> b -> b) -> b -> List a -> b
a -> Maybe.Maybe a
Maybe.Maybe a
number
number -> number -> number
```

`Repl` constructor accepts several options:

* `workDir` — specify working directory for execution, for example where the `.elm` files you use are located, or where you have your `elm-package.json`;
* `elmVer` — the exact elm-version you use (default: `'0.17.1'`);
* `user` — your github username specified in `elm-package.json` (default: `'user'`);
* `project` — your github project specified in `elm-package.json` (default: `'project'`);
* `projectVer` — the version of your project from `elm-package.json` (default: `'1.0.0'`);
* `keepTempFile` — for debugging purposes, if _truthy_, then do not delete `.elm` files after compilation;
* `keepElmiFile` — for debugging purposes, if _truthy_, then do not delete `.elmi` files after compilation;

## How to use it with CLI?

CLI interface is a bit different, to use it, you need to create a file with expressions listed.

If you need imports, list them in the first line starting with `;` and splitting them with `;`.

For example (the contents of `src/cli-example`):

```
;List as L;Maybe exposing ( Maybe(..) );String
L.map
L.foldl
Just
Nothing
1 + 1
\a b -> a + b
```

Then run `node src/cli.js --from <your-file-name>`.

And you should get:

```
(a -> b) -> List a -> List b
(a -> b -> b) -> b -> List a -> b
a -> Maybe.Maybe a
Maybe.Maybe a
number
number -> number -> number
```

Repl-CLI accepts several options:

* `--work-dir` — specify working directory for execution, for example where the `.elm` files you use are located, or where you have your `elm-package.json`;
* `--elm-ver` — the exact elm-version you use (default: `'0.17.1'`);
* `--user` — your github username specified in `elm-package.json` (default: `'user'`);
* `--project` — your github project specified in `elm-package.json` (default: `'project'`);
* `--project-ver` — the version of your project from `elm-package.json` (default: `'1.0.0'`);
* `--keep-temp-file` — for debugging purposes, if specified, then do not delete `.elm` files after compilation;
* `--keep-elmi-file` — for debugging purposes, if specified, then do not delete `.elmi` files after compilation;

## How to contribute?

Write a test which fails with `npm test`, file an issue, fork the repository, make a pull request — just any of that or all together will help.

## How it was done?

Elm language has no reflection for the moment, and sometimes you need to know a type of a variable or lambda, or a partial application or... If you are still reading this, you know exactly which cases I mean.

You could use some AST parser for that, but AST is not storing variables-to-types maps, it only allows you to know *what* is this expression, not the types of the variables it was made from, unless you get AST of all the user files and it sounds huge.

You could use `elm-repl` CLI, but it was made for other purposes and for the moment it's quite slow. And anyway, you still need to parse the output.

So I decided to dig into how `elm-repl` works. For the moment, the scenario is:

- Take new input expression;
- Append it to all the imports executed before, so imports will go first;
- Assign a variable to your expression (easter-egg from Evan here);
- Put everything in a temporary `.elm` file;
- Compile this file with `elm-make`;
- Go into `elm-stuff/build-artifacts/blah/blah/` and find the corresponding `.elmi` file there (which is binary!);
- Parse this file with the same Haskell code which compiled it and so extract the variable type from it;
- Also, take the value of this expression by executing compiled `.js` file;

This all takes a lot of time!

So I decided to parse this `.elmi` file with node.js, since its structure turned out not to be very complex (a bit complex, but not too very). You may see some screenshots of reverse-engineering in the process in the `./img` folder.

So when user asks for types several expressions, we put all of them in one single `.elmi` file instead, and each expression gets its own variable and only one `elm-make` call is required to know the types for all of them as a response. Though it also requires to binary-parse this `.elmi` file (in `src/parser.js` you may find this parser), anyway it is much faster than original way.

For the moments, tests for a packs of 10 expressions run from 30ms to 200ms each. More benchmarking later.

_Interesting fact_: If you remove `elm-html` dependency from `elm-package.json` in `test/samples/elm` and disable the corresponding tests for `Html` package, all other tests now run from 10ms to 90ms!

Thanks to Mukesh @mukeshsoni Soni for huge-helping me in my findings, and leading me through trials and errors!

Probably it was a stupid idea, but at least it was fun.
