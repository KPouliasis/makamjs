We will start with encoding a version of the simply typed lambda calculus in λProlog. We define two new meta-types to
represent the two sorts of our object language: terms and types. We also define the `typeof`
relation that corresponds to the typing judgement of the language.

```makam
term   : type.
typ    : type.
typeof : term -> typ -> prop.
```

Defining the basic forms of the λ-calculus is very easy, thanks to the support of higher-order
abstract syntax in higher-order logic programming. We can reuse the meta-level function type in
order to implement object-level binding. This is because the meta-level function space is
*parametric* -- that is, the body of a function is a value that can just mention the argument as-is,
instead of being a computation that can inspect the specific value of an argument. Therefore,
meta-level functions exactly represent an object-level binding of a single variable, without
introducing *exotic terms*.

```makam
app    : term -> term -> term.
lam    : typ -> (term -> term) -> term.
arrow  : typ -> typ -> typ.
```

Encoding the typing rule for application as a λProlog *clause* for the `typeof` relation is a
straightforward transliteration of the pen-and-paper version.

```makam
typeof (app E1 E2) T' :-
  typeof E1 (arrow T T'),
  typeof E2 T.
```

In logic programming, the goal of a rule is written first, followed by the premises; the `:-`
operator can be read as "is implied by," and comma is logical conjuction. We use capital letters for
unification variables.

The rule for lambda functions is similarly straightforward: 

```makam
typeof (lam T1 E) (arrow T1 T2) :-
  (x:term -> typeof x T1 -> typeof (E x) T2).
```

There are three things of note in the premise of the rule. First, we introduce a fresh term variable
`x`, through the form `x:term ->`, which can be read as universal quantification. Second, we
introduce a new assumption through the form `typeof x T ->`, which essentially introduces a new rule
for the `typeof` relation locally; this can be read as logical implication. Third, in order to get
to the body of the lambda function to type-check it, we need to apply it to the fresh variable `x`.

With these definitions, we have already implemented a type-checker for the simply typed lambda
calculus, as we can issue queries for the `typeof` relation to Makam:

```makam
typeof (lam _ (fun x => x)) T' ?
>> Yes:
>> T' := arrow T T
```

One benefit of using λProlog instead of rolling our own type-checker is that the occurs check is
already implemented in the unification engine. As a result, a query that would result in an
ill-formed cyclical type with a naive implementation of unification fails as expected.

```makam
typeof (lam _ (fun x => app x x)) T' ?
>> Impossible.
```

Other than supporting higher-order abstract syntax, λProlog also supports polymorphic types and
higher-order predicates, in a matter akin to traditional functional programming languages. For
example, we can define the polymorphic `list` type, and an accompanying `map` higher-order
predicate, as follows:

```
list : type -> type.

nil : list A.
cons : A -> list A -> list A.

map : (A -> B -> prop) -> list A -> list B -> prop.
map P nil nil.
map P (cons X XS) (cons Y YS) :- P X Y, map P XS YS.
```

Using the meta-level list type, we can encode object-level constructs such as tuples and product types 
directly: 

```makam
tuple : list term -> term.
product : list typ -> typ.
```

Similarly we can use the `map` predicate to define the typing relation for tuples. 

```makam
typeof (tuple ES) (product TS) :-
  map typeof ES TS.
```

Executing a query with a tuple yields the correct result:

```makam
typeof (lam _ (fun x => lam _ (fun y => tuple (cons x (cons y nil))))) T ?
>> Yes:
>> T := arrow T1 (arrow T2 (product (cons T1 (cons T2 nil))))
```

So far we have only introduced the predicate `typeof` for typing. In the same way, we can introduce
a predicate for evaluating terms, capturing the dynamic semantics of the language.

```makam
eval : term -> term -> prop.
```

Most of the rules are straightforward, following standard practice for big-step semantics.  We
assume a call-by-value evaluation strategy.

```makam
eval (lam T F) (lam T F).
eval (tuple ES) (tuple VS) :- map eval ES VS.
```

For the beta-redex case, function application for higher-order abstract syntax gives us
capture-avoiding substitution directly: 

```makam
eval (app E E') V'' :-
  eval E (lam _ F), eval E' V', eval (F V') V''.
```
