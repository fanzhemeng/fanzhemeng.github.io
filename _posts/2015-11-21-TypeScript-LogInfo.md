---
layout: post
title: TypeScript LogInfo
---

In this post I would like to share my approache to log TypeScript variable declaration. This is a proof of concept for [ALive](https://comealive.io/) for TypeScript.

### Trivial case attempt

My first attempt was to make some changes to TypeScript compiler, so that TypeScript code
`var x: number = 5;` 
can be transpiled to JavaScript
`var x ===== 5;`

Since it is hard to rewrite AST for TypeScript, we decided to do the rewriting in the emitting step of transpiling TypeScript to JavaScript.

```
const source = "var x: number = 5;";
let result = ts.transpile(source, { module: ts.ModuleKind.CommonJS });
```

I started with the code above, and put a breakpoint on the second line to see how transpiling worked inside the compiler. Note that the function `ts.transpile()` is provided as compiler API [here](https://github.com/Microsoft/TypeScript/wiki/Using-the-Compiler-API). After clicking step over and then jumping to the compiler file (typescript.js), I searched for word "emitVariableDeclaration" (Thanks to TypeScript Syntax Visualizer we have built now I know how they called it), then I found a function with this name. Thus, this is the function I needed to make the change. Using the debugger to step through the function, I found where the assignment mark (i.e. "=") is emitted. 

On line 7 from the bottom in the function, `emitOptional(" = ", initializer);`. Here inside function `emitOptional()` they are using a function `write()`. I did not understand what it was for at first. Later when I asked Josh for help, we found out that `write()` is a method of the `writer` for emitting recording, and the compiler builders were using destructuring to simplify the code (from `writer.write()` to `write()`). I think this `write()` function records the JavaScript code rewriten from TypeScript so the JavaScript can be output as file or returned by functions later.

Here, we just need to change `emitOptional(" = ", initializer);` to `emitOptional(" ===== ", initializer);`, and the trivial case is good to go.


### Building Result Classes

I basically followed the strucutre of those classes for C#, and made some small changes necessary for the language. You can look at the file AliveResultClasses.ts for these classes.


### LogInfo for `var x: number = 5;`

This is a trivial case that we want to use to prove the doability of Alive for TypeScript. Here, we want to rewrite TypeScript (main.ts)

```
var x: number = 5;
```

to JavaScript

```
LoggingType.MethodStart();
var x = LoggingType.LogInfo(5);
LoggingType.MethodEnd();
```

Note that it is harder to do this than the one I described at the start, since in this case I needed to first attach all the Alive result classes (in JavaScript) to the transpiled JavaScript code. And then add `LoggingType.MethodStart()` and `LoggingType.MethodEnd()` around the line, then also add the `LoggingType.LogInfo()` around `5`.

I used `fs.readFileSync()` (learned from [here](https://github.com/Microsoft/TypeScript/wiki/Using-the-Compiler-API)) to read the transpiled JavaScript AliveResultClasses.js to the compiler, then used `write()` to attach the classes to the emitted code. It is done inside function `createFileEmitter()` and right before `emitSourceFile(root);` (on line 30201 in my local typescript.js, in which I have made some changes, but it should be around that line) So now we got all classes in it.

Now we need to add `MethodStart()` and `MethodEnd()`. Using debugger again, I found out the whole `var x: number = 5;` is emitted in function `emitVariableStatement()`. Then I stepped through the function, found the function `tryEmitStartOfVariableDeclarationList()` is where the keyword `var` is emitted, and this is the place I should insert `LoggingType.MethodStart();`. (line 32522 on my local typescript.js)

I did not inserted `LoggingType.MethodEnd();` in my local typescript.js, since it is pretty similar to `MethodStart()` and it does not really matter for one line of code. But it is definitely necessary for further expansion. 

Now we need to insert `LoggingType.LogInfo()`. I did it in function `emitVariableDeclaration()` (same function as the trivial case at the start of this summary). I changed `emitOptional(" = ", initializer);` to `emitOptional("= LoggingType.LogInfo('x', ", initializer)` and add `write(")")` after it. (on line 33572 on my local compiler typescript.js) I know this code looks not good, and may not work in more complex situation, but this demonstarted that Alive for TypeScript is doable. For more complex cases, we need to discover some better way of inserting all these code.

Now, we can get back to main.ts, and after running 

```
const source = "var x: number = 5;";
let result = ts.transpile(source, { module: ts.ModuleKind.CommonJS });
```

variable result is the string which contains definition of all result classes, and the three lines we were looking for. So variable result is a string looks like:

```
"/* 
all result classes here
*/

LoggingType.MethodStart();
var x = LoggingType.LogInfo(5);
LoggingType.MethodEnd();"
```

we now can run this script (in string) by `eval(result);`. But since we wanted to get some feedback from it, we appended "LoggingType.RootResult" to result (i.e. `result = result + "\nLoggingType.RootResult"`) to make the JavaScript script return the root result of LoggingType. 

So now the main.ts looks like:

```
/// <reference path="Scripts/typings/node/node.d.ts" />
/// <reference path="AliveResultClasses.ts" />

import {readFileSync} from "fs";
import * as ts from "typescript";

const source = "var x: number = 5;";
let result = ts.transpile(source, { module: ts.ModuleKind.CommonJS });
result = result + "\nLoggingType.RootResult;";
console.log(result);
var evalResult = eval(result);

var rootResult = <ExecutionResult>evalResult;
var r: MethodIterationResult = <MethodIterationResult>rootResult.Children[0];
console.log(r.ToStringRecursive());

```

The last three lines tries to look into the root result returned by the script, by doing that we can check if we implemented the classes and the insertion process correctly.

This is my approache to log `var x: number = 5;`.


### Small tricks learned for dealing with TypeScript

- if error "Build: cannot be found module 'typescript'" shows up when starting debugging, try clean the project (this deletes the generated .js files), and then Ctrl+S to save the changes of all .ts files (this re-generates the .js files). Click start now and it should be good to go.
- method overload in TypeScript is quite different, see [here](http://stackoverflow.com/questions/12688275/method-overloading) for some explanation

