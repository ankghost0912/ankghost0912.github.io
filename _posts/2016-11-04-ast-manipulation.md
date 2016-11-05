#Self modifying code using Python (AST Manipulation)

A few days ago, I was working on a project, when I realized that in order to achieve the goals of the project, the code needed to be self modifying. 

There are not many resources out there which would describe how to write self modifying python code, so I decided to create this blog post detailing the method I followed i.e. AST Manipulation.

### What's AST?

AST stands for Abstract Syntax Tree. This is how python statements are represented in memory. So for example, if you write :

` x = 1 + 2` 

That would be represented in python as: 

~~~python
Module: 
   body: Assign
   Assign: Targets, Value
   Targets: Name
   Name: id = x
   Value: BinOp
   BinOp: Left, op, right
   Left: Num(1) 
   op: Add
   right: Num(2)

~~~

It becomes quite clear that this simple statement is represented by a very expressive syntax tree.

###What can I do with AST?

Ah, now we're asking the right questions. Python introduced the `ast` module just to enable curious programmers like you and me to explore and play with the python grammar structures. 

Python is an object oriented language. That means that everyting in python is an object of some class. As seen in the code listing above, the variable `x` belongs to the class `id` which in turn belongs to class `Name`. Smarter readers amongst you, might have figured where I'm going with this- the `ast` module provides these very classes which would come in handy later to explore (and modify) the structures.


Most of the information for playing with `ast` is in the [documentation](https://docs.python.org/3.5/library/ast.html). However, it is not well explained so I will attempt to give an explanation of the most common uses of the module. 


###Traversing and Modifying the AST
`Ast` provides two major set of classes:

* `ast.NodeVisitor`: This is a class used for traversing an AST.
* `ast.NodeTransformer`: This is a class used for transforming the nodes of the AST. 

Both of these classes must be subclassed by the programmer and then the `visit_*` methods be overridden in order to traverse/modify the AST.

The `visit_*` methods are different based on what part of the AST you want to visit. For example, you would write `visit_name` in order to visit the nodes with the class `Name`. 

Here's how you can traverse a tree and print out all the Binary operators in the tree:

~~~python
import ast

class BinOpVisitor(ast.NodeVisitor):
    def __init__(self):
        super.__init__(self)
       
           
    def visit_BinOp(node):
        print 'BinOp:', node.BinOp
             
 
 
 if __name__ == '__main__':
     x = ast.parse('y = 1+2')
     binopv = BinOpVisitor().visit(x)
     
     
~~~

This code introduces some of the methods of the `ast` module. In `main`, x is passed the parsed value of the statement using `parse`. The method `parse` is used for converting a given statement into a python AST. 

The `visit_BinOp` module just prints the binary operator of the corresponding node. 

Since `BinOpVisitor` subclasses from `NodeVisitor` it has access to the `visit` method which can be used to traverse a tree. However, since we've defined `visit_BinOp`, the output just produces the Binary Operator in the expression. Easy right?

However, `ast` also provides us with another method called `walk` which could be used in replacement for above. So the above code can be written equivalently as:

~~~python
x = ast.parse('y = 1+2')
for node in ast.walk(x):
    if isinstance(node, ast.BinOp):
        print 'BinOp:', node.BinOp
~~~ 

####Modifying the AST for fun (and profit):

It's not so much fun as to just see the underlying AST structure. What if we could change it? Well `ast` provides even that functionality to us as well, using `ast.NodeTransformer`. 

Before I describe how we can use the class, I'd like to point out where it can be used. 

Most of the IDE's like PyCharm  use some sort of syntax checking tools to correct the mistakes made by beginner programmers. It is not without bounds of reason to consider that it relies on using AST for that purpose. 


Furthermore, this class can be used to change code at run time by allowing it to read in a script and change the meaning of a particular operator. In this way, the program becomes undeterministic and unreliable. For example, if I change `+` to not mean that "take the left and right operands and return the sum" and change it's meaning to "output a string - hello world", the program would become completely unreliable, but it would not crash. 

There are in essence three phases of Node Transformation: 

* Parse the script into AST.
* Gain access to the node we're trying to change & change it
* Put the changed tree back into it's correct form.

I've already shown how to parse the script into AST. I'll describe how to gain access to a node and change it. After you've subclassed the `ast.NodeTransformer` here's what needs to be written in the `visit_op` (I'll change addition): 

~~~python

def visit_op(self, node):
    old_node = node
    if isinstance(node,ast.Add):
        new_node = ast.Call(func=Name(id='print_hello',
                                      ctx=Load()),
                             args=[])
         ast.copy_location(new_node, old_node)
         ast.fix_missing_locations(new_node)
         return new_node
    return old_node
      

~~~
 
 Suddenly, its a lot of code. However, all it does it compare whether the current node is an instance of ast.Add class and if true, it creates a new node which calls a function `print_hello`. Then the helper function `copy_location` copies the location of the old node to the new one and the helper `fix_missing_location` would fix any missing memory locations.
 
 Compilation to the source code is a simple matter of calling:

~~~python
compile(new_node, '<ast>', 'exec')
~~~
        
        
        
I'd like to thank `/r/python` and to the information found [here](http://www.dalkescientific.com/writings/diary/archive/2010/02/22/instrumenting_the_ast.html). For additional documention on `ast`, [this](https://greentreesnakes.readthedocs.io/en/latest/) is a very good resource as well. Happy hacking!


