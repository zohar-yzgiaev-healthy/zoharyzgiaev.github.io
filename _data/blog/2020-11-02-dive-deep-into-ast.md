---
template: BlogPost
path: /dive-deep-into-ast
date: 2020-11-02T10:00:58.632Z
title: Dive deep into AST
thumbnail: /assets/header.png
---
![](/assets/header.png)

After a thoughtful discussion with Raziel (my roommate) on the challenge of finding similarities between source codes i've started investigating AST in more-depth. In this blog post I strive to explain what is AST, it's use cases and real world applications. 

# WHAT IS AST?

An Abstract Syntax Tree, or AST for short, is a deeply nested object that represents code. Usually AST will come in a form of a Tree representation of the abstract syntactic structure of a source code. The “abstract” acronym used in a sense that it does not represent every detail appearing in the real syntax, but rather just the structure, and a syntactic construct like an if-condition-then expression may be denoted by means of a single node with three branches.

AST used widely in compilers to represent the structure of program code, but can also be found inside transpilers, linters, code generation tools or in other applications where the understanding of the code is a matter.

The typical implementation of an AST (speaking mostly about OO languages) makes heavy use of polymorphism. The nodes in the AST are typically implemented with a variety of classes, all deriving from a common `AstNodeclass.`

For each syntactical construct in the language you are processing, there will be a class representing that construct in AST, for an example a `VariableNode`node will represent variable names, AssignmentNode will represent an assignment operations, etc. Each node type specifies if that node has children, if so than how many children it has and possibly of what type. For an example, a `ConstantNode`will typically have no children, whereas an `AssignmentNode`will have two children associated to it.

Many times you will encounter the acronym of Concrete Syntax Tree (aka Parse Tree) vs Syntax Tree, let’s clarify the difference between the two. Concrete Syntax tree is a concrete representation of the input. The concrete syntax tree retains all of the information of the input. AST however is an abstract representation of the input. Parents are not present in the AST because the associations are derivable from the tree structure.



Let’s look at a simple AST example of a pseudo-code:



```json
(add 2 (subtract 4 2))
```



For the following pseudo code example the following tree would be generated:



```json
{
  "type": "Program",
  "body": [
    {
      "type": "CallExpression",
      "name": "add",
      "params": [
        {
          "type": "NumberLiteral",
          "value": "2"
        },
        {
          "type": "CallExpression",
          "name": "subtract",
          "params": [
            {
              "type": "NumberLiteral",
              "value": "4"
            },
            {
              "type": "NumberLiteral",
              "value": "2"
            }
          ]
        }
      ]
    }
  ]
}
```



AST can also be transformed using design patterns, most widely the visitor pattern is being used to navigate through the AST, let’s take a look of the following example written in Python:



```python
import ast
class MyVisitor(ast.nodeVisitor):
    def visit_BinaryOp(self, node):
        self.visit(node.left)
        print node.op,
        self.visit(node.right)
    def visit_Num(self, node):
        print node.n
```



nodeVisitor is a base class that walks the abstract syntax tree and calls a visitor function for every node. The nodeVisitor base class implements the visitor design pattern, which is a way of separating an algorithm from an object structure on which it operates. The visitor design patterns allows adding new virtual functions to a family of classes, without modifying the classes. classes. Instead, a visitor class is created that implements all of the appropriate specialisations of the virtual function.

Besides visiting the nodes of the tree we can walk down the tree and allow modification of nodes. Usually modifying the tree nodes will be done by using the nodeTransformer implementation of the transformer design pattern. Basically, the nodeTransformer will walk the AST and use the return value of the visitor methods to replace or remove the old node.

## AST Use Cases

### 1st use case - Compilers: “If you don’t know how compilers work, then you don’t know how computers work”

Most compilers break down into 3 primary stages: Parsing, Transformation, and code generation.

1. Parsing is taking a raw code and turning it into a more abstract representation of the code (Parse Tree, Abstract Syntax Tree)
2. Transformation takes the abstract representation and manipulates to do whatever the compiler wants it to. (Using transformation methods on the AST or the parse tree)
3. Code generation - takes the transformed representation of the code and turns it into the new code. (Again, using transformation

Let’s take a look at the [Super Tiny Compiler](https://github.com/jamiebuilds/the-super-tiny-compiler) source code, this compiler compiles lisp-like function calls into C-like function calls. The following snippet code takes an array of tokens and turns it into an AST.



```javascript
function walk() {
  // Inside the walk function we start by grabbing the `current` token.
  let token = tokens[current];
  // We're going to split each type of token off into a different code path,
  // starting off with `number` tokens.
  //
  // We test to see if we have a `number` token.
  if (token.type === "number") {
    // If we have one, we'll increment `current`.
    current++;
    // And we'll return a new AST node called `NumberLiteral` and setting its
    // value to the value of our token.
    return {
      type: "NumberLiteral",
      value: token.value
    };
  }
  // If we have a string we will do the same as number and create a
  // `StringLiteral` node.
  if (token.type === "string") {
    current++;
    return {
      type: "StringLiteral",
      value: token.value
    };
  }
  // Next we're going to look for CallExpressions. We start this off when we
  // encounter an open parenthesis.
  if (token.type === "paren" && token.value === "(") {
    // We'll increment `current` to skip the parenthesis since we don't care
    // about it in our AST.
    token = tokens[++current];
    // We create a base node with the type `CallExpression`, and we're going
    // to set the name as the current token's value since the next token after
    // the open parenthesis is the name of the function.
    let node = {
      type: "CallExpression",
      name: token.value,
      params: []
    };
    // We increment `current` *again* to skip the name token.
    token = tokens[++current];
    // And now we want to loop through each token that will be the `params` of
    // our `CallExpression` until we encounter a closing parenthesis.
    //
    // Now this is where recursion comes in. Instead of trying to parse a
    // potentially infinitely nested set of nodes we're going to rely on
    // recursion to resolve things.
    //
    // To explain this, let's take our Lisp code. You can see that the
    // parameters of the `add` are a number and a nested `CallExpression` that
    // includes its own numbers.
    //
    //   (add 2 (subtract 4 2))
    //
    // You'll also notice that in our tokens array we have multiple closing
    // parenthesis.
    //
    //   [
    //     { type: 'paren',  value: '('        },
    //     { type: 'name',   value: 'add'      },
    //     { type: 'number', value: '2'        },
    //     { type: 'paren',  value: '('        },
    //     { type: 'name',   value: 'subtract' },
    //     { type: 'number', value: '4'        },
    //     { type: 'number', value: '2'        },
    //     { type: 'paren',  value: ')'        }, <<< Closing parenthesis
    //     { type: 'paren',  value: ')'        }, <<< Closing parenthesis
    //   ]
    //
    // We're going to rely on the nested `walk` function to increment our
    // `current` variable past any nested `CallExpression`.
    // So we create a `while` loop that will continue until it encounters a
    // token with a `type` of `'paren'` and a `value` of a closing
    // parenthesis.
    while (
      token.type !== "paren" ||
      (token.type === "paren" && token.value !== ")")
    ) {
      // we'll call the `walk` function which will return a `node` and we'll
      // push it into our `node.params`.
      node.params.push(walk());
      token = tokens[current];
    }
    // Finally we will increment `current` one last time to skip the closing
    // parenthesis.
    current++;
    // And return the node.
    return node;
  }
  // Again, if we haven't recognized the token type by now we're going to
  // throw an error.
  throw new TypeError(token.type);
}
```



As we can see from the sample code above, our defined walk function creates the node depending on the type of the token, each node has both type and value fields associated to it.

Than, let’s take a look at how we can walk through the tree and visit each node:



```javascript
// So we define a traverser function which accepts an AST and a
// visitor. Inside we're going to define two functions...
function traverser(ast, visitor) {
  // A `traverseArray` function that will allow us to iterate over an array and
  // call the next function that we will define: `traverseNode`.
  function traverseArray(array, parent) {
    array.forEach(child => {
      traverseNode(child, parent);
    });
  }
  // `traverseNode` will accept a `node` and its `parent` node. So that it can
  // pass both to our visitor methods.
  function traverseNode(node, parent) {
    // We start by testing for the existence of a method on the visitor with a
    // matching `type`.
    let methods = visitor[node.type];
    // If there is an `enter` method for this node type we'll call it with the
    // `node` and its `parent`.
    if (methods && methods.enter) {
      methods.enter(node, parent);
    }
    // Next we are going to split things up by the current node type.
    switch (node.type) {
      // We'll start with our top level `Program`. Since Program nodes have a
      // property named body that has an array of nodes, we will call
      // `traverseArray` to traverse down into them.
      //
      // (Remember that `traverseArray` will in turn call `traverseNode` so  we
      // are causing the tree to be traversed recursively)
      case "Program":
        traverseArray(node.body, node);
        break;
      // Next we do the same with `CallExpression` and traverse their `params`.
      case "CallExpression":
        traverseArray(node.params, node);
        break;
      // In the cases of `NumberLiteral` and `StringLiteral` we don't have any
      // child nodes to visit, so we'll just break.
      case "NumberLiteral":
      case "StringLiteral":
        break;
      // And again, if we haven't recognized the node type then we'll throw an
      // error.
      default:
        throw new TypeError(node.type);
    }
    // If there is an `exit` method for this node type we'll call it with the
    // `node` and its `parent`.
    if (methods && methods.exit) {
      methods.exit(node, parent);
    }
  }
  // Finally we kickstart the traverser by calling `traverseNode` with our ast
  // with no `parent` because the top level of the AST doesn't have a parent.
  traverseNode(ast, null);
}
```



Next stage would be the transformation which is going to take the AST that we have build and pass it to our traverser function with a visitor and will create a new AST. (Usually, each programming language has it’s own constraints and therefore it’s own AST)



```javascript
function transformer(ast) {
  // We'll create a `newAst` which like our previous AST will have a program
  // node.
  let newAst = {
    type: "Program",
    body: []
  };
  // Next I'm going to cheat a little and create a bit of a hack. We're going to
  // use a property named `context` on our parent nodes that we're going to push
  // nodes to their parent's `context`. Normally you would have a better
  // abstraction than this, but for our purposes this keeps things simple.
  //
  // Just take note that the context is a reference *from* the old ast *to* the
  // new ast.
  ast._context = newAst.body;
  // We'll start by calling the traverser function with our ast and a visitor.
  traverser(ast, {
    // The first visitor method accepts any `NumberLiteral`
    NumberLiteral: {
      // We'll visit them on enter.
      enter(node, parent) {
        // We'll create a new node also named `NumberLiteral` that we will push to
        // the parent context.
        parent._context.push({
          type: "NumberLiteral",
          value: node.value
        });
      }
    },
    // Next we have `StringLiteral`
    StringLiteral: {
      enter(node, parent) {
        parent._context.push({
          type: "StringLiteral",
          value: node.value
        });
      }
    },
    // Next up, `CallExpression`.
    CallExpression: {
      enter(node, parent) {
        // We start creating a new node `CallExpression` with a nested
        // `Identifier`.
        let expression = {
          type: "CallExpression",
          callee: {
            type: "Identifier",
            name: node.name
          },
          arguments: []
        };
        // Next we're going to define a new context on the original
        // `CallExpression` node that will reference the `expression`'s arguments
        // so that we can push arguments.
        node._context = expression.arguments;
        // Then we're going to check if the parent node is a `CallExpression`.
        // If it is not...
        if (parent.type !== "CallExpression") {
          // We're going to wrap our `CallExpression` node with an
          // `ExpressionStatement`. We do this because the top level
          // `CallExpression` in JavaScript are actually statements.
          expression = {
            type: "ExpressionStatement",
            expression: expression
          };
        }
        // Last, we push our (possibly wrapped) `CallExpression` to the `parent`'s
        // `context`.
        parent._context.push(expression);
      }
    }
  });
  // At the end of our transformer function we'll return the new ast that we
  // just created.
  return newAst;
}
```



Lastly, we would need to generate the code by recursively printing each node in the tree into one giant string.



```javascript
function codeGenerator(node) {
  // We'll break things down by the `type` of the `node`.
  switch (node.type) {
    // If we have a `Program` node. We will map through each node in the `body`
    // and run them through the code generator and join them with a newline.
    case "Program":
      return node.body.map(codeGenerator).join("\n");
    // For `ExpressionStatement` we'll call the code generator on the nested
    // expression and we'll add a semicolon...
    case "ExpressionStatement":
      return (
        codeGenerator(node.expression) + ";" // << (...because we like to code the *correct* way)
      );
    // For `CallExpression` we will print the `callee`, add an open
    // parenthesis, we'll map through each node in the `arguments` array and run
    // them through the code generator, joining them with a comma, and then
    // we'll add a closing parenthesis.
    case "CallExpression":
      return (
        codeGenerator(node.callee) +
        "(" +
        node.arguments.map(codeGenerator).join(", ") +
        ")"
      );
    // For `Identifier` we'll just return the `node`'s name.
    case "Identifier":
      return node.name;
    // For `NumberLiteral` we'll just return the `node`'s value.
    case "NumberLiteral":
      return node.value;
    // For `StringLiteral` we'll add quotations around the `node`'s value.
    case "StringLiteral":
      return '"' + node.value + '"';
    // And if we haven't recognized the node, we'll throw an error.
    default:
      throw new TypeError(node.type);
  }
```



Voila! We’ve done writing a simple compiler using AST, as you can see from the above examples understanding AST will help us as developers to understand it programmatically and create applications which relies on this kind of understanding.

### 2nd use case - Babel: compiler for writing next generation JavaScript

Babel which is in a wide use today, also uses AST in order to parse the input code, travers through it and transforms it in order to generate the output code.

Let’s take a look at the parsing stage of Babel:



```javascript
export default function normalizeFile(
  pluginPasses: PluginPasses,
  options: Object,
  code: string,
  ast: ?(BabelNodeFile | BabelNodeProgram)
): File {
  code = `${code || ""}`;
  let inputMap = null;
  if (options.inputSourceMap !== false) {
    // If an explicit object is passed in, it overrides the processing of
    // source maps that may be in the file itself.
    if (typeof options.inputSourceMap === "object") {
      inputMap = convertSourceMap.fromObject(options.inputSourceMap);
    }
    if (!inputMap) {
      try {
        inputMap = convertSourceMap.fromSource(code);
        if (inputMap) {
          code = convertSourceMap.removeComments(code);
        }
      } catch (err) {
        debug("discarding unknown inline input sourcemap", err);
        code = convertSourceMap.removeComments(code);
      }
    }
    if (!inputMap) {
      if (typeof options.filename === "string") {
        try {
          inputMap = convertSourceMap.fromMapFileSource(
            code,
            path.dirname(options.filename)
          );
          if (inputMap) {
            code = convertSourceMap.removeMapFileComments(code);
          }
        } catch (err) {
          debug("discarding unknown file input sourcemap", err);
          code = convertSourceMap.removeMapFileComments(code);
        }
      } else {
        debug("discarding un-loadable file input sourcemap");
        code = convertSourceMap.removeMapFileComments(code);
      }
    }
  }
  if (ast) {
    if (ast.type === "Program") {
      ast = t.file(ast, [], []);
    } else if (ast.type !== "File") {
      throw new Error("AST root must be a Program or File node");
    }
    ast = cloneDeep(ast);
  } else {
    // The parser's AST types aren't fully compatible with the types generated
    // by the logic in babel-types.
    // $FlowFixMe
    ast = parser(pluginPasses, options, code);
  }
  return new File(options, {
    code,
    ast,
    inputMap
  });
}
```

### **3rd use case - Golang GraphQL JSON implementation: “JSON, JSON everywhere”**

In grasp, GraphQL is a query language for your API, and a server-side runtime for executing queries by using a type system you define for your data. Before couple of months I had to implement a backend component with GraphQL in Golang, what got me confused at first is that sometimes your API needs are to return a JSON object according to a custom query. The problems becomes more clear when I understood that in GraphQL we need to define our API types as scalars (boolean, number, string) , list or enums or we have to define custom types but after all they have to return simple scalars. So according to those rules, how can we introduce the JSON type?

Let’s take a look at the following Golang snippet:



```go
import (
    "fmt"
    "github.com/graphql-go/graphql"
    "github.com/graphql-go/graphql/language/ast"
    "github.com/graphql-go/graphql/language/kinds"
)
func parseLiteral(astValue ast.Value) interface{} {
    kind := astValue.GetKind()
    switch kind {
    case kinds.StringValue:
        return astValue.GetValue()
    case kinds.BooleanValue:
        return astValue.GetValue()
    case kinds.IntValue:
        return astValue.GetValue()
    case kinds.FloatValue:
        return astValue.GetValue()
    case kinds.ObjectValue:
        obj := make(map[string]interface{})
        for _, v := range astValue.GetValue().([]*ast.ObjectField) {
            obj[v.Name.Value] = parseLiteral(v.Value)
        }
        return obj
    case kinds.ListValue:
        list := make([]interface{}, 0)
        for _, v := range astValue.GetValue().([]ast.Value) {
            list = append(list, parseLiteral(v))
        }
        return list
    default:
        return nil
    }
}
// JSON json type
var JSON = graphql.NewScalar(
    graphql.ScalarConfig{
        Name:        "JSON",
        Description: "The `JSON` scalar type represents JSON values as specified by [ECMA-404](http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf)",
        Serialize: func(value interface{}) interface{} {
            return value
        },
        ParseValue: func(value interface{}) interface{} {
            return value
        },
        ParseLiteral: parseLiteral,
    },
)
```



As we can see, we introduced a new scalar type which on parseLiteral hook calls the parseLiteral function. The parseLiteral function takes an AST node, check’s its type and returns its corresponding value (as we said before, each AST node has both a type and value fields). By walking through the AST and exposing the node’s type we can understand the inner behaviour of JSON and create the same concept ourselves.

## Summary

Understanding AST and its usage helps us understand core mechanisms applied in different tools such as compilers, code generations, linters, etc. In the upcoming blog post I will explain how to use AST to find similarities between source codes.
