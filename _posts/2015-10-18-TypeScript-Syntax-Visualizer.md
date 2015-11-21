---
layout: post
title: TypeScript Syntax Visualizer
---

In this post I would like to talk about what I have learned from building the [TypeScript Syntax Visualizer](https://github.com/CodeConnect/TypeScriptSyntaxVisualizer). This is a side project of my internship at Code Connect. I learned a lot about TypeScript and Visual Studio from it.

### Main Idea

I would start with why we wanted to build this tool. The main goal of my intern at Code Connect is to find a way to make [ALive](https://comealive.io/) support TypeScript (and thus JavaScript as well since TS is a superset of JS). Since the core part of Alive invloves manipulating syntax tree of user code, it would be nice to have a VS tool that shows us the syntax structure of any TypeScript code.I believe this tool will be helpful for devs interested in TypeScript.

Basically the TypeScript Syntax Visualizer is a tool in Visual Studio very similar to the Syntax Visualizer for C# under `Tool > Other windows > Syntax Visualizer`, except that it is for TypeScript. It shows the syntax structure of TypeScript code, and user can look at it when he/she is trying to get familiar with the syntax of the language. The whole project mainly has two parts: one is to get the syntax tree from user code, and the other part is to put the tree in a tool window of Visual Studio, and allow users to manipulate it (just like how they designed the C# Syntax Visualizer).

### How to get AST?

This is the core of the project. And it is not hard to answer, we should do it using the TypeScript compiler. As documented at [this wiki page](https://github.com/Microsoft/TypeScript/wiki/Using-the-Compiler-API), the TypeScript compiler provides some handy APIs, and we can get the syntax tree esaily using these APIs. To help demonstart the process, I will use `var x: number = 5;` as an example and get the AST out of it.

The particular function we are going to use is `ts.createSourceFile(filename, text, languageVersion, /*setParentPointers*/ true)`. The argument `filename` (does not really matter in this project, but still) should be a string representing an appropriate filename, `text` is a string representing the source text we want to get AST from, and languageVersion should be `ts.ScriptTarget.ES6`. The function returns a syntax node object representing the root of the AST. (it is called SourceFile object to be more accurate, but its still a syntax node. You can read more about TypeScript syntax structure components [here](https://github.com/Microsoft/TypeScript/wiki/Architectural-Overview#data-structures) ) Thus, now we can get the syntax tree by the first four lines of following code. And by recursively visiting its children and children of children, we can know the syntax information about the user code. I also created a function `printAST()` shown below to demonstrate it.

```
/// <reference path="typings/node/node.d.ts" />
import * as ts from "typescript";
var text: string = "var x: number = 5;";
let sourceFile = ts.createSourceFile("test.ts", text, ts.ScriptTarget.ES6, /*setParentNodes */ true);

tion printAST(node: ts.Node, n: number): void {
	var tabs = "";
	for (var i = 0; i < n; i++)
		tabs += "   ";
	console.log(tabs + node.kind.toString() + ": " + node.getText());
	var children = node.getChildren();
	for (var child = 0; child < node.getChildCount(); child++) {
		printAST(children[child], n + 1);
	}
}
printAST(sourceFile, 0);

```

Easy, isnt it? So getting AST is not that hard. However, there is some problem as this process is done in TypeScript. Since we would like to do the VS tool window designing in C#, we need to convert the AST we have got from TypeScript into C#. The approache we discovered to solve this was using a TypeScript processor, which allows us to communicate TypeScript with C#. By this processor, we can have a script containing the TypeScript compiler, and pass user code as string to the script. The script then output the AST of user code in the form of json data back to C#. Now we can successfully get the AST in C#. 

### Design VS tool window

I thought this would be easy but it appeared not like that. At first I did not know the source for the C# Syntax Visualizer is included in [Roslyn on GitHub](https://github.com/dotnet/roslyn/tree/57aaa6c9d8bc1995edfc261b968777666172f1b8/src/Tools/Source/SyntaxVisualizer/SyntaxVisualizerControl), so I tried to create my own xml and controller. Then after I found the source code for the C# one, it definitely became easier to just "copy and paste" how they have designed it.


### Conclusion

To sum up, I would say this is not a hard project, but a interesting one since it involves so many different areas including TypeScript compiler, parsing, Visual Studio, xml stuff, etc. A very good start, I guess. :)
