---
layout: page
title: "MiniJava"
date: 2019-12-01
parent: Original Website
back_to_top: true
---

In my first quarter as a junior in the Allen school (Autumn 2019), I took a compilers course. The class was very theory heavy, but as a complement to lecture, we built a functional compiler which translates a subset of Java ([MiniJava](http://www.cs.tufts.edu/~sguyer/classes/comp181-2006/minijava.html)) to executable x86-64 instructions. MiniJava does not include Strings or generics (thank goodness), but does do fun stuff with inheritance.  

I was surprised on the first day when my TA from CSE 333, [Kristofer Wong](https://www.linkedin.com/in/kristofertwong/), sat down next to me and asked if I wanted to be partners. He later convinced me that the two of us should swap from CSE 401 (the undergraduate version of the course) to M501, the masters version of the class. The distinction for the masters credit was relatively simple: choose a nontrivial feature to develop independently. Initially I though register  allocation would be fun, right up to the moment I discovered graph coloring is an NP complete problem.

Instead, we went with Kris's idea - method overloading. Thankfully, the [Javadoc](https://docs.oracle.com/javase/specs/jls/se11/html/jls-15.html#jls-15.12) is pretty explicit about what behavior is expected, so we had a very well written spec to implement. The scanner and parser for our compiler were generated by [Jflex](https://www.jflex.de/) and [CUP](http://www2.cs.tum.edu/projects/cup/), so we were able to skip right into the meat of semantic analysis as soon as we figured out how to be clients of actual software tools.

Thankfully, we were allowed to write our compiler in Java (meaning Java was turning itself into x86). This dramatically simplified things, and allowed us to make use of OOP for our type system. We used three abstract classes to  differentiate between Objects and Primitives. The (written in Java) inheritance tree is shown below*:  

_SemanticType_ (abstract)     
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_SemanticPrimitive_ (abstract)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SemanticBoolean  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SemanticInteger  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_SemanticObject_ (abstract)  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SemanticIntegerArray  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;SemanticClass  
    
_* "Semantic" prefix included to avoid collisions with Java_

    
Below is a (very rough) pseudo code version of our analysis  

    Given a parse tree π, and a type system τ:  
    
        Build up a record of types from declared classes in π  
        Make note of all declared field types, and method return types  
        
        ∀ classes C ∈ π
            ∀ methods M ∈ C        
                ∀ nodes Ν ∈ M
                    if (N is not a leaf node)  
                        recurse  
                        use new info for further analysis  
                    else (Ν is a leaf node λ)  
                        return τ equivalent of λ 

Where `λ` nodes are literals (int, boolean) and non-`λ` nodes are any nontrivial features of the language (variable use, addition, loops, etc)

Looking back at our Inheritance tree, it's fairly obvious what most of the nodes are _supposed_ to be, and what behaviours should be permitted on them. The only real twist is the SemanticClass Class. The best description I can give is that every instance of SemanticClass represents a declared (written) class in whatever MiniJava program is being compiled.

```java
public class SemanticClass extends SemanticObject {
    // Majority of this Class omitted
    
    /**
     * A pointer to the class that this SemanticClass extends (null if does not
     * extend anything)
     */
    private SemanticClass parent;
    
    /**
     * The name of the class.
     */
    private String name;
    
    /**
     * An environment containing information about the variables declared
     * at the top of the class.
     */
    public Environment fields;
    
    /**
     * Similar to @fields, this field keeps track of the methods defined
     * in this class (and their variables/params).
     */
    public MethodList vtable;
    
    // Majority of this Class omitted
}
```

The `parent` pointer is filled in after a first pass to collect the names of all Classes which exist in the program. Once inheritance relations are checked and recorded, instance variables and methods can be examined. Since our type system is quite small, most checks are very straightforward. The really exciting stuff comes in when you start to analyze method invocations. 

When a parse tree π is being parsed, There is no guarantee for the order of the classes being parsed relative to their inheritance relations. In other words, if  `B` extends `A`, we could parse `B` first and it would be impossible to tell which methods from A were being overloaded/overridden. To circumvent this issue, we included another Java class `Method` which would keep track of its `Signature`, the name, return type, and argument list of a given method. 

```java
public class Method {
    // Majority of this Class omitted
    /**
     * The variables declared at the beginning of the method (the first elements
     * of which are the parameters).
     */
    public Environment vars;

    /**
     * The signature of the method.
     */
    public Signature signature;
    // Majority of this Class omitted
}
```

Instances of `Method` are created and added to the `vtable` field of a `SemanticClass` as it's equivalent in π is being parsed. Then, vtables are constructed when they are needed (this lazy evaluation could certainly be improved for optimal performance). In step one of the method below, you see this behavior in action. Three empty lists are passed to a method in an instance of `SemanticClass`, which are then passed all the way up to the top of the inheritance chain before the following actions are taken:
```java
/**
 * Constructs the vtable for this instance of SemanticClass. If this
 * instance extends a SemanticClass, this method will invoke itself
 * on the parent.
 *
 * @param indexes a List (which should initially be empty) that will be
 *                filled with the Signatures of all methods known to this
 *                SemanticClass (including inherited methods). The indexes
 *                of the Signatures in @indexes will correspond to an index
 *                in @vtable.
 * @param vtable a List which will be filled with all the Methods known to
 *               this SemanticClass. To determine the index of a method in
 *               the vtable, find the index of its Signature in @indexes.
 * @param classNames a list containing the name of the class that a method
 *                   in @vtable (at the same index) was generated from.
 */
public void constructVtable(List<Signature> indexes,
                            List<Method> vtable,
                            List<String> classNames) {
    int methodIndex;
    
    if (this.getParent() != null) {
        this.getParent().constructVtable(indexes, vtable, classNames);
    }
     
    // this.getSignatures() gets the methods declared in THIS class.
    for (Signature currSig : this.getSignatures()) {
        methodIndex = this.indexOfMethod(currSig);
        
        if (indexes.contains(currSig)) {
            // If this method signature has already been seen, just update
            // the pointer at its offset (this is how the compiler handles
            // overriding).
            vtable.set(indexes.indexOf(currSig), 
                       this.vtable.methods.get(methodIndex));
            classNames.set(indexes.indexOf(currSig), this.getName());
        } else {
            // This method signature is new to the vtable, so append it and
            // move on to the next one.
            indexes.add(currSig);
            vtable.add(this.vtable.methods.get(methodIndex));
            classNames.add(this.getName());
        }
    }
}
```

Once a parent `SemanticClass` finishes its work, the child can start up, performing the same logic on all methods that were added to it when its π equivalent was parsed. The result of this work is three lists:    

1. All the signatures known to a given class _(this is really just a helper for indexing)_
2. Pointers to the `Methods` that were inherited/overloaded/declared in the `SemanticClass`
3. The name of the class that the method was declared in  

These lists are aligned, meaning the information in list 1 at index `i` is about the same `Method` as the information in lists 2 and 3 at index `i`. The class names (list 3) are crucial for codegen, because method labels are prefixed with their class names (since multiple classes could have the same method signature).

Once we have all this knowledge about the methods known to a class, we can attempt to resolve an invocation. One last detail concerning the "magnitude" of a match - if a declared method signatures matches an invocation, the match is considered to be magnitude μ, where μ can be expressed as   
    
    Σ {number of steps down inheritance tree paramActual is from paramExpected} 
      where both params are are non-primitives
    
Now, lets assemble all our knowledge into the resolution of a method invocation in MiniJava.

```java
/**
 * Attempts to resolve @s to a method inside this SemanticClass. The methods
 * to be considered in this resolution are only those immediately available
 * to @this (all inherited/overloaded/overridden methods). If a method has
 * been overridden, only the most recent version will be considered.
 *
 * @param inv the invocation signature to resolve.
 * @return an instance of the Method @s resolves to, and null if resolution
 *         cannot be completed.
 */
public Method resolveInvocation(Signature inv) {
    // The minimum magnitude seen so far, the number of Signatures which
    // match with that magnitude, and the index of the first seen match of
    // magnitude @minMag.
    int minMag    = Integer.MAX_VALUE,
        numMinMag = 0, tmpMag,
        index     = 0, tmpIndex;
        
    List<Signature> indexes = new ArrayList<>();
    List<Method> vtable     = new ArrayList<>();
    List<String> classNames = new ArrayList<>();

    // 1. Get all methods known to this class (with appropriate overloading
    //    and overriding)
    this.constructVtable(indexes, vtable, classNames);

    // 2. Filter out all matching Signatures
    tmpIndex = 0;
    for (Signature s : indexes) {
        // Expected Signature on right side of .equalsSubclass()
            // note the special equals method which allows actual params to 
            // be subclasses of the expected parameters.
        if (inv.equalsSubclass(s)) {
            tmpMag = determineMagnitude(inv, indexes.get(tmpIndex));
            if (tmpMag == minMag) {
                // Potential error
                numMinMag++;
            } else if (tmpMag < minMag) {
                // New minimum magnitude, reset all bookkeeping info.
                numMinMag = 1;
                minMag    = tmpMag;
                index     = tmpIndex;
            }
        }

        tmpIndex++;
    }

    // 3. Check that only one method matched as close as the min. mag.
    if (numMinMag == 1) { return vtable.get(index); }

    return null;
}
```

A closing thought to the reader - for the purposes of this project (and Java in general), return type was not considered in comparing  the signature of two `Method` instances. However, I believe that return type could be another factor in overloading, i.e. the following would be legal syntax:

```java
public class Foo {
    public int m1() {
        return 1;
    }
    
    @Default
    public boolean m1() {
        return true;
    }
    
    public Bar m1() {
        return new Bar();
    }    
}
``` 

The above example would be relatively trivial to perform type inference on to determine which declaration should be used, and the annotation should resolve any instances where inference isn't sufficient (such as when the result of the method is unused). However, as always, subclassing introduces a headache to this potential feature. Consider the following:

```java
public abstract class Animal {}

public class Dog extends Animal {}

public class Cat extends Animal {}

public class Foo {
    @Default
    public Dog m1() {
        return new Dog();
    }
    
    public Cat m1() {
        return new Cat();
    }
}

public class Example {
    public static void main(String[] args) {
        Foo f = new Foo();
        
        example1(f);
        example2(f);
    }
    
    public static void example1(Foo f) {
        Animal kitty = f.m1();
    }
    
    public static void example2(Foo f) {
        List<Animal> pets = new ArrayList<>();

        pets.add(f.m1());
    }
}
```

I believe that my proposed annotation system would address the ambiguity - use the default when the user was not clear! However, this option does not leave a client with the ability to access the non-default m1 in Foo. This is somewhat unsatisfactory, and it seems a more complex method of annotations would be needed to properly address this issue. But, I've probably run out of time to propose hypothetical scenarios. At any rate, it would certainly be interesting to see how a modern language would manage to implement this additional layer of complexity to overloading.

Thank you for reading this far! Feel free to send me an email if you have any questions about what I've written here. My email should be listed in my resume, linked on my home page :)

---