# Category
A category is a collection of `objects` linked by `arrows`.
A category has two basic properties.
- Ability to compose the arrows ( associativity )
- Existence of an identity arrow for each `object` in the category
## Definition
a formal definition is as follows
- a [[notes/Category Theory/index.md#Class|class]] of $ob(C)$ of objects
- a [[notes/Category Theory/index.md#Class|class]] $mor(C)$ of morphisms or arrows
- a `domain` or a `source class` function $dom : mor(C) \rightarrow ob(C)$
- a `co-domain` or `target class` function $cod : mor(C) \rightarrow ob(C)$
- for every three objects $a,b,c$ -
	 a binary operation called `composition of morphisms` $hom(a,b) \circ hom(b,c) \rightarrow hom(a,c)$
	 here $hom(a,b)$ denotes a subclass of morphisms $f$ such that the domain of $f$ is $a$ and the codomain of $f$ is $b$
$$ hom(a,b) = \{ f \in mor(C) : dom(f) = a \space \& \space cod(f) =b \}$$
- morphisms in this subclass are written as $f : a \rightarrow  b$ 
- and the composition of $f : a \rightarrow b$ , $g : b \rightarrow c$ is written as $f \circ g$ 

## Examples of categories
### Category of sets
- the elements are all the `sets`
- and the morphisms are `set functions` 
- consider the set $\mathbb{C}$ of complex numbers and the set $\mathbb{R}$ of real numbers, the function $f=modulus(x)=|x|$ can be thought of as $f \in hom(\mathbb{C},\mathbb{R})$ 
- similarly if we take arbitrary sets $A=\{1,2,3\}$ and $B=\{2,4,6\}$ and the function $g: 1 \rightarrow 2, 2 \rightarrow 4, 4 \rightarrow 6$ then we can say $g \in hom(A,B)$ 
- this can be extended to all the possible sets ( whether the elements of the sets are integers of rationals or abstract objects themselves, as long as a function exists from one set to another)
- here we can also define the identity arrow for every set $\mathbb{S}$ as $id_{\mathbb{S}}$ which is an arrow that returns the set itself
### Category of Vectors in field F
consider a vector space with field $\mathbb{F}$ , some examples for fields are $\mathbb{R}$ , $\mathbb{C}$ 
- the elements are $\mathbb{F}^n$  where $n$ is the dimension of the vector for field $\mathbb{F}$
- the morphisms are arrows from $\mathbb{F}^n \rightarrow \mathbb{F}^m$ 
- these morphisms can actually be though of as matrix multiplications as well for example for the `object` $\mathbb{F}^n$ we can have the function $f(v \in \mathbb{F}^n) = v \times \mathbb{W}^{n \times m}$ 
- with the signature of the function being $f: \mathbb{F}^n \rightarrow \mathbb{F}^m$  or we can also say that $f \in hom(\mathbb{F}^n,\mathbb{F}^m)$ . 
- multiplying a scalar $s \in \mathbb{F}$ results in an identity morphism as $c \in \mathbb{F},v \in \mathbb{F}^n \implies c v \in \mathbb{F}^n$ 

## Small and large categories 
A category $C$ is called `small` if both $obj(C)$ and $mor(C)$ are actually `sets` and not [[notes/Category Theory/index.md#Class|proper classes]] , and `large` otherwise
# Class
A class defined as `a collection of sets that can be precisely defined by a property`
- The core reason for this distinction is to avoid paradoxes like `Russell's paradox`
- **Set :** A collection of objects that is `small` enough to be a member of other collections, for example the set of all natural numbers $\mathbb{N} = \{1,2,3,...\}$ is a set. It can be member of other sets, like the set of all countable sets
- **Proper Class :** A collection of `objects` that is `too large` to be a set. `It cannot be a member of any other class!`. A proper class contains all the sets that satisfy a specific property
## Example of proper classes
### The class of all sets 
If this collection where a set, then this will lead to the `Russell's paradox` . It is the collection of all sets, but `not a set itself`
### The class of all groups
The collection of all mathematical groups
### The class of all cardinal numbers
- finite cardinal numbers are for example $A = \{1,2,3,4\}$
- infinite cardinal numbers are for example $\mathbb{N}=\{1,2,3,...\}$ having cardinality of $\aleph_{0}$  and we can also have sets with cardinality $\aleph_{1}$ , $\aleph_{2}$ etc. ( note : $|\mathbb{R}| = \aleph_{1}$)
- The set of cardinal numbers is not considered a set itself but rather a class that contains all numbers of all cardinality hence becoming a collection of sets, or rather a `class`

## ZFC vs NBG axiomatic set theory
Von Neumann-Bernays-Godel set theory (NBG) is an axiomatic set theory that is a conservative extension of Zermelo-Frankel-choice set theory (ZFC). NBG introduces the notion of a `class` , which is a collection of `sets` defined by a formula whose quantifiers range only over `sets`. NBG can define classes that are larger than sets, such as the class of all `sets` and the class of all `ordinals`


# Monoid
In abstract algebra, a **monoid** is a set equipped with an associative binary operation and an Identity element
- for the set $S$ we can define the operator $\odot$ as  $(f \in S ) \odot (g \in S) \rightarrow (h \in S)$  
- and the identity element $id$ as $id \odot f = f \odot id = f$ where $f \in S$  
