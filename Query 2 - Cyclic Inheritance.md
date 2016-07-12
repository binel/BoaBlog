# Query 2 : Finding Cyclic Inheritance #

---
###Disclaimer: 

I wrote this query when I was new to Boa. Were I to write it again, there are several things I would do differently. 

---
This query is quite a bit more complex than Query 1. Before we dive into the code it would be a good idea to define what it is we are looking for. 

##Section 1: What is Cyclic Inheritance? 

Say we have the following code in Java

	public class ClassA extends ClassB {}

Then we say that ClassB is a parent of ClassA, or that ClassB is a superclass of ClassA. 

Now suppose in this same project we have 

	public class ClassB extends ClassC {}

Then ClassC is a superclass of ClassA. Further, since ClassB was a superclass of ClassA, ClassC is *also* a superclass of ClassA.

Below is all the class relationships we have defined so far. A -> B means that A extends B, or B is a superclass of A.

- ClassA -> ClassB -> ClassC
- ClassB -> ClassC
- ClassC

To see what Cyclic Inheritance is (if you haven't already guessed), let's introduce one more class into our project.

	public class ClassD extends ClassA {}

If we try to write out our class relationships again you can see that we now run into a rather serious problem. 

- ClassA -> ClassB -> ClassC -> ClassD -> ClassA -> ClassB -> ClassC -> ...

We have fallen into an infinite loop. 

If you try to write Java code like this it will fail to compile, but there are some projects in our dataset that exhibit this property. So, let's find them. 

##Section 2: Overview 

Since this query if fairly long, I'm going to give a broad overview of it's operation.

1. Find the immediate parent of each class 
2. Compute the complete inheritance hierarchy for each class 
3. Search through the complete inheritance hierarchy for instances of cyclic inheritance 

For the first step, we need to be able to find the parent of each class. Classes in Boa are a type of **Declaration**. Declarations can also be things like abstract classes or interfaces. Each **Declaration** in Boa potentially has an array of **parents**.

However, those parents aren't **Declarations**, they are represented as **Types**, which only have **name** and **type** fields. Importantly, these **Types** don't contain any information on their parents. This Boa design choice makes our query a little bit more complex, and is why step 1 and 2 are separate. 

One more note before we jump into the code: Boa is unable to mix non-basic types. So, for example, having an 

	map[string] of array of string

is disallowed. To work around this limitation we creating psuedo-arrays out of strings. This will become apparent in Section 4.

##Section 3: The Query 

Since we have broken our query up into three steps, let's create a function for each step. 

###Step 1: Find the immediate parent of each class

Our first function is going to find the immediate parent of every class in the project. Our function is going to take a project as input and return a map of strings to strings. So our function signature will look like this 

	findParentClasses := function(p: Project) : map[string] of string {

	};

We are going to be visiting the project, so we should add a visitor and the snippet to only view the most recent snapshot. We will also add a variable to store our output.

	
	findParentClasses := function(p: Project): map[string] of string {
	    parents: map[string] of string; 
	    visit(p, visitor {
	        # only look at the latest snapshot
	    	before node: CodeRepository -> {
				snapshot := getsnapshot(node);
	        	foreach (i: int; def(snapshot[i])) {
	        		visit(snapshot[i]);
	        	}
	        	stop;
	    	}
	
		}
	};

Now we need to visit every class. But we need to have the fully-qualified name of the class. To get that we have to start at the namespace level and iterate over ever class in that namespace. That section of the visitor looks like this:

    before node: Namespace -> {
        foreach(i: int; node.declarations[i]) {
            declaration: Declaration = node.declarations[i];
            fullClassName: string = formatClassName(node.name, declaration.name);

	}

The formatClassName function is a helper function to format the fully-qualified class name nicely. 

We only want to consider declarations that are classes, and for each of that declarations parents we only want to consider the parent if it is also a class. 


    if(declaration.kind == TypeKind.CLASS) {
        foreach(j: int; declaration.parents[j]) {
            #if it extends another class, add that class as a parent
            if(declaration.parents[j].kind == TypeKind.CLASS) {
                #format the parent's full classname including the package 
                parentFullClassName: string = formatClassName(node.name, declaration.parents[j].name);
                parents[fullClassName] = parentFullClassName;
            }
        }

Here you can see that we get the parent's fully qualified name, and the key and the value into our map. Our function as a whole looks like this: 

	findParentClasses := function(p: Project): map[string] of string {
	    parents: map[string] of string; 
	    visit(p, visitor {
	        # only look at the latest snapshot
	    	before node: CodeRepository -> {
				snapshot := getsnapshot(node);
	        	foreach (i: int; def(snapshot[i])) {
	        		visit(snapshot[i]);
	        	}
	        	stop;
	    	}
	    	
	        before node: Namespace -> {
	            foreach(i: int; node.declarations[i]) {
	                declaration: Declaration = node.declarations[i];
	                #format the name of this class including the package 
	                fullClassName: string = formatClassName(node.name, declaration.name);
	                #if it is a class look to see if it extends another class 
	                if(declaration.kind == TypeKind.CLASS) {
	                    foreach(j: int; declaration.parents[j]) {
	                        #if it extends another class, add that class as a parent
	                        if(declaration.parents[j].kind == TypeKind.CLASS) {
	                            #format the parent's full classname including the package 
	                            parentFullClassName: string = formatClassName(node.name, declaration.parents[j].name);
	                            parents[fullClassName] = parentFullClassName;
	                        }
	                    }
	                    #if, after looking at all parents, none of them are a class, we say parent is null
	                    if(!haskey(parents, fullClassName)) {
	                        parents[fullClassName] = "NULL";
	                    }
	                }
	            }    
	        }
	    });  
	    
	    return parents; 
	};

###Step 2: Compute the complete inheritance hierarchy for each class



###Step 3: Search through the complete inheritance hierarchy for instances of cyclic inheritance 