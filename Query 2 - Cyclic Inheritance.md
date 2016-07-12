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

Our next function will use the results from the previous function to compute the full inheritance hierarchy for every class. This will also be stored as a map of string to string, so our function signature will be as follows: 

	computeAllSuperclasses := function(parents: map[string] of string): map[string] of string {	

	};

Hopefully at this point you are able to read Boa code fairly well, So I'll just give a brief overview of this function and you can dive in for yourself. 

First, we get a list of every class. We then iterate over that list. 

For every class, we find it's parent. If we have encountered this parent before we exit. Otherwise we add it to the full inheritance and set the current class as the parent of the original class. We continue in this way until we run out of parents. 

Below is the function in its entirety: 

	computeAllSuperclasses := function(parents: map[string] of string): map[string] of string {
	    classes: array of string = keys(parents);
	    out: map[string] of string; 
	    #iterate over every class
	    foreach(i: int; def(classes[i])) {
	        className: string = classes[i];
	        #map of parents. the key is how high in the parentage tree the class is
	        parentMap: map[int] of string; 
	        duplicateMap: map[string] of int; 
	        numParents: int = 0; 
	        currentClass: string = className;
	        #loop until we run out of parents
	        while(true) {
	            #we have found a parent defined in the project that isn't null
	            if(def(parents[currentClass]) && !match("^NULL$", parents[currentClass])) {
	                #add it to the parentMap and set the parent as the class we are considering 
	                parentMap[numParents] = parents[currentClass];
	                currentClass = parents[currentClass];
	                numParents++;
	                #if have hit this class before than stop
	                if(def(duplicateMap[currentClass])) {
	                    break;
	                }
	                duplicateMap[currentClass] = 1; 
	            }
	            #no more parents, exit the loop
	            else {
	                break;
	            }
	        }
	        #build our string of parents 
	        parentString: string = className;
	        #if there are no parents add a null
	        if(len(parentMap) == 0) {
	            parentString = parentString + " -> NULL";
	        }
	        #else add each parent to the string one by one. 
	        else {
	            j: int = 0;
	            while(j < len(parentMap)){
	                parentString = parentString + " -> " + parentMap[j];
	                j++;
	            }
	        }
	        #add this parentage string to the output map
			out[className] = parentString; 
	    }
	    return out; 
	};

So if we consider our original "ClassA extends ClassB", etc. example, the output of this function would be as follows: 

- [ClassA] = ClassA -> ClassB -> ClassC -> ClassD -> ClassA
- [ClassB] = ClassB -> ClassC -> ClassD -> ClassA -> ClassB
- [ClassC] = ClassC -> ClassD -> ClassA -> ClassB -> ClassC
- [ClassD] = ClassD -> ClassA -> ClassB -> ClassC -> ClassD

Now all that's left to do is find the cycles! 

###Step 3: Search through the complete inheritance hierarchy for instances of cyclic inheritance 

This function is pretty simple. It steps through each inheritance string, and if at any point it finds a class with the same name as the starting class, it reports it as a leak. 

	detectCycles := function(parentage: map[string] of string): map[string] of string {
	    cycles: map[string] of string;
	    #iterate over each class in the project 
	    classes: array of string = keys(parentage);
	    foreach(i: int; classes[i]) {
	        currentClass: string = classes[i];
	        #consider every parent of the current class
	        classParentage: array of string = splitall(parentage[currentClass], " -> ");
	        foreach(j: int; classParentage[j]) {
	            #a cycle has been found
	            if(match("^" + currentClass + "$", classParentage[j]) && j > 0) {
	                inheritanceString: string = currentClass; 
	                #add each parent to the inheritance string
	                k: int = 1; 
	                while(k <= j) {
	                    inheritanceString = inheritanceString + " -> " + classParentage[k];
	                    k++; 
	                }
	                cycles[currentClass] = inheritanceString; 
	                break;
	            }    
	        }
	    }
	    return cycles; 
	};

###Full Query

If you can understand this query then I think you are ready to start writing non-trivial Boa queries of your very own.

	p: Project = input; 
	#used for debugging   
	#logging: output collection[string] of string; 
	cyclicInheritance: output collection[string] of string; 
	
	#Takes a package name and class name as input and returns the full concatenated name
	formatClassName := function(packageName: string, className: string) : string {
	    fullName: string; 
	    #case where the packageName is an empty string
	    if(match("^$", packageName)) {
	        fullName =  className;
	    }
	    else {
	        fullName = packageName + "." + className;    
	    }
	    return fullName; 
	};
	
	#takes a project as input and returns a map[string] of string, where
	#the keys are the classes in the project and the values are the superclasses of the keys
	#if a class has no superclass than the value will be the string "NULL"
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
	
	
	
	#will find the total parentage for every class. The input is a map[string] of string where the keys are 
	#classes and the values are their superclasses. Returns a map[string] of string. 
	#Example: Class A extends Class B, and Class B extends Class C.
	#   Input: <Class A, Class B>, <Class B, Class C>, <Class C, NULL>.
	#   Output: <Class A, "Class A -> Class B -> Class C">,
	#           <Class B, "Class B -> Class C">,
	#           <Class C, "Class C -> NULL">
	computeAllSuperclasses := function(parents: map[string] of string): map[string] of string {
	    classes: array of string = keys(parents);
	    out: map[string] of string; 
	    #iterate over every class
	    foreach(i: int; def(classes[i])) {
	        className: string = classes[i];
	        #map of parents. the key is how high in the parentage tree the class is
	        parentMap: map[int] of string; 
	        duplicateMap: map[string] of int; 
	        numParents: int = 0; 
	        currentClass: string = className;
	        #loop until we run out of parents
	        while(true) {
	            #we have found a parent defined in the project that isn't null
	            if(def(parents[currentClass]) && !match("^NULL$", parents[currentClass])) {
	                #add it to the parentMap and set the parent as the class we are considering 
	                parentMap[numParents] = parents[currentClass];
	                currentClass = parents[currentClass];
	                numParents++;
	                #if have hit this class before than stop
	                if(def(duplicateMap[currentClass])) {
	                    break;
	                }
	                duplicateMap[currentClass] = 1; 
	            }
	            #no more parents, exit the loop
	            else {
	                break;
	            }
	        }
	        #build our string of parents 
	        parentString: string = className;
	        #if there are no parents add a null
	        if(len(parentMap) == 0) {
	            parentString = parentString + " -> NULL";
	        }
	        #else add each parent to the string one by one. 
	        else {
	            j: int = 0;
	            while(j < len(parentMap)){
	                parentString = parentString + " -> " + parentMap[j];
	                j++;
	            }
	        }
	        #add this parentage string to the output map
			out[className] = parentString; 
	    }
	    return out; 
	};
	
	#Takes as input a representation of the complete parentage of each class and returns any cycles found.
	#Example: Class A extends Class B, and Class B extends Class A.
	#   Input: <Class A, "Class A -> Class B -> Class A">,
	#          <Class B, "Class B -> Class A">
	#   Output: <Class A, "Class A -> Class B -> Class A">
	detectCycles := function(parentage: map[string] of string): map[string] of string {
	    cycles: map[string] of string;
	    #iterate over each class in the project 
	    classes: array of string = keys(parentage);
	    foreach(i: int; classes[i]) {
	        currentClass: string = classes[i];
	        #consider every parent of the current class
	        classParentage: array of string = splitall(parentage[currentClass], " -> ");
	        foreach(j: int; classParentage[j]) {
	            #a cycle has been found
	            if(match("^" + currentClass + "$", classParentage[j]) && j > 0) {
	                inheritanceString: string = currentClass; 
	                #add each parent to the inheritance string
	                k: int = 1; 
	                while(k <= j) {
	                    inheritanceString = inheritanceString + " -> " + classParentage[k];
	                    k++; 
	                }
	                cycles[currentClass] = inheritanceString; 
	                break;
	            }    
	        }
	    }
	    return cycles; 
	};
	
	
	exists(m: int; match("^java", lowercase(p.programming_languages[m]))) {
	    parents: map[string] of string = findParentClasses(p);
	    parentage: map[string] of string = computeAllSuperclasses(parents);
	    cycles: map[string] of string = detectCycles(parentage);
	    
	    cycleArray: array of string = values(cycles);
	    foreach(i: int; cycleArray[i]) {
	        cyclicInheritance[p.project_url] << cycleArray[i];
	    }
	}