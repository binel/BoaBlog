# Query 1: Counting If Statements #

This was a exercise query to count the number of if statements in each project. This document will walk you through the code line by line. This is the first post in a series that will hopefully help you learn how to write non-trivial Boa queries yourself. 

First, take a look at the query as a whole. 

	p: Project = input;
	counts: output sum[string] of int;

	visit(p, visitor {
		# only look at the latest snapshot
		before node: CodeRepository -> {
			snapshot := getsnapshot(node);
			foreach (i: int; def(snapshot[i]))
				visit(snapshot[i]);
			stop;
		}
	
		#counts the number of if statements 
		before node: Statement -> {
			if(node.kind == StatementKind.IF) {
				counts[p.project_url] << 1;
			}
		}
	});

##Section 1 : Output 

 The first step in writing a query is figuring out what your outputs are going to be. In this case we are counting the number of something for every project. So we are going to want a different **sum** for every project. To do this we are going to create an output as follows: 

	counts: output sum[string] of int;

Here **counts** is the name of the aggregator (could be anything). **output** is required to tell Boa this is an output aggregator. **sum** means we want to add up whatever we send to the aggregator, and **sum[string]** lets us create different sums with different string indexes. **of int** tells boa that we will be sending integers to the aggregator. 

So, we use this aggregator by saying 

	counts[p.project_url] << 1; 

Which means we are adding **1** to the **sum** indexed by **p.project\_url**, which is the url of the current project. (Notice that the first line aliases p to input, otherwise we would have to say input.project\_url).

## Section 2: Counting 

Now that we  know how to produce an output, we need to actually do some counting. 

In Boa, an if statement is a subcategory of the domain specific type **statement**. So we want to visit every statement in this project and if it is an if statement we want to increment out count. 

To visit a project, we use the following outline 

	visit(p, visitor{
		#visiting code goes here
	}); 

We could also define a visitor elsewhere and use it like this 

	myVisitor := visitor {
		#visiting code goes here
	};

	visit(p, myVisitor);      

---

Now, before we can dive into counting we need to realize something about visitors. When you visit a whole project, you visit the whole *history* of the project. So if we just counted if statements we would count all the if statements through all commits. To avoid these we start out our visitor with this snippet

		# only look at the latest snapshot
		before node: CodeRepository -> {
			snapshot := getsnapshot(node);
			foreach (i: int; def(snapshot[i]))
				visit(snapshot[i]);
			stop;
		}

This will cause the visitor to only visit the most recent version of the project. Use this snippet to avoid over-counting. 

---

Now for counting! 

We want to visit every statement. To do that we place this snippet within our visitor: 

	before s: Statement -> {

	}

**s** is a variable representing the statement we are currently at. 

**before** tells boa that we want to visit this node before any of its children in the AST (abstract syntax tree). It's counterpart is **after**. In most cases you should use **before**. 

We want to eliminate any statements that aren't if statements. So, we write 

	before s: Statement -> {
		if(s.kind == StatementKind.IF) {

		}
	}

notice how we access s.kind. all of the fields of s can be found on the Boa "domain-specific types" page. 

All that is left is to increment our counter! 

	before s: Statement -> {
		if(s.kind == StatementKind.IF) {
			counts[p.project_url] << 1; 
		}
	}

That's all there is to it! Our query as a whole reads as follows: 

	p: Project = input;
	counts: output sum[string] of int;

	visit(p, visitor {
		# only look at the latest snapshot
		before node: CodeRepository -> {
			snapshot := getsnapshot(node);
			foreach (i: int; def(snapshot[i]))
				visit(snapshot[i]);
			stop;
		}
	
		#counts the number of if statements 
		before node: Statement -> {
			if(node.kind == StatementKind.IF) {
				counts[p.project_url] << 1;
			}
		}
	});

Read Query 2: Cyclic Inheritance for a more complex query example. 