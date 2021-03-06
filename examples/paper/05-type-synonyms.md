<!--
```makam
%use "04-ml-subset.md".
```
-->

Let us proceed to add type synonyms:

```makam
type_synonym : dbind typ T typ -> (typeconstructor T -> program) -> program.

type_synonym_info : typeconstructor T -> dbind typ T typ -> prop.

wfprogram (type_synonym Syn Program') :-
  (t:(typeconstructor T) ->
   type_synonym_info t Syn ->
   wfprogram (Program' t)).
```

Simple enough. How to typecheck them though? We need something like the
conversion rule:

\begin{displaymath}
\inferrule{
  \Gamma \vdash e : \tau \\ \tau =_{\delta} \tau'
}{
  \Gamma \vdash e : \tau'
}
\end{displaymath}

Here $=_{\delta}$ means equality up to expanding type synonyms.

We will need a type equality predicate:

```makam
teq : typ -> typ -> prop.
```

A naive attempt at the conversion rule would be:

```
typeof E T :- typeof E T', teq T T'.
```

However, it is easy to see that this rule leads to divergence:
it does a recursive call to itself.

We can do a bit better. We only need to use the conversion rule in cases where we
already know something about the type `T` of the expression, but our typing rules do
not match that type. In bi-directional typing parlance, instead of analyzing the type
`T` of the expression `E`, we want to synthesize the type starting from a new
meta-variable `T'`, and then check that the two types are equal using `teq`.
So we need to change our rule to only apply in the case where `T` starts with a
concrete type constructor, rather than when it is an uninstantiated meta-variable.

It is typical for a logic programming language to have a predicate that only succeeds
when a specific term is uninstantiated (usually called `var`). In Makam this is
called `refl.isunif` -- the `refl` namespace prefix standing for the fact that we
call these kinds of predicates "reflective", as they give us extra-logical
information about the form of a term (sometimes referred to as "meta-predicates" in
Prolog parlance). Our second attempt thus looks as follows:

```
typeof E T :- not(refl.isunif T), typeof E T', teq T T'.
```

Upon further consideration, we see that this rule leads to an infinite loop as well:
since `teq` should be reflective, for every proof of `typeof E T'` through the other
rules, a new proof using this rule will be discovered, which will lead to another
proof for it, etc. One way to fix it is to make sure that this rule is only used once
at the end, if typing using the other rules fails:

```
typeof, typeof_cases, typeof_conversion : term -> typ -> prop.
typeof E T :-
  if (typeof_cases E T)
  then success
  else (typeof_conversion E T).
typeof_cases (app E1 E2) T' :-
  typeof E1 (arrow T1 T2),
  typeof E2 T1.
...
typeof_conversion E T :-
  not(refl.isunif T), typeof_cases E T', teq T T'.
```

However, this would require changing every typing rule we had. Instead, we can do a
trick, to force the rule to only fire once for each expression `E`, remembering the
fact that we have used the rule already:

```makam
already_in : [A] A -> prop.
typeof E T :-
  not(refl.isunif T),
  not(already_in (typeof E)),
  (already_in (typeof E) -> typeof E T'),
  teq T T'.
```

Also, we need to make sure that we also take the conversion rule into account for
patterns:

```makam
typeof (P : patt A B) S' S T :-
  not(refl.isunif T),
  not(already_in (typeof P)),
  (already_in (typeof P) -> typeof P S' S T'),
  teq T T'.
```

Now let's go and define the actual `teq` predicate. A first approach would be to just
write out each case individually:

```
teq (arrow T1 T2) (arrow T1' T2') :- teq T1 T1', teq T2 T2'.
teq (product TS) (product TS') :- map teq TS TS'.
teq (arrowmany TS T) (arrowmany TS' T') :- teq T T', map teq TS TS'.
teq nat nat.
teq (forall T) (forall T') :- (x:typ -> teq x x -> teq (T x) (T' x)).
teq (tconstr TC Args) (tconstr TC Args') :- map teq Args Args'.
teq (tconstr TC Args) T' :-
  type_synonym_info TC Syn,
  applymany Syn Args T,
  teq T T'.
teq T' (tconstr TC Args) :-
  type_synonym_info TC Syn,
  applymany Syn Args T,
  teq T' T.
```

Only the two last cases are important; the rest is boilerplate that performs
structural recursion through the type. Can we do better than that?

Let us ruminate on a possible solution. We want to handle the case where we have a
constructor applied to a number of arguments generically, so roughly something
like:

```
teq (Constructor Arguments) (Constructor Arguments') :-
  map teq Arguments Arguments'.
```

What we mean here, taking the `arrow` rule as an example, is that `Constructor`
would match with `arrow`, and `Arguments` would get instantiated with the list
of arguments of the constructor. One thing to be careful about though is that
the types of arguments are not all the same. As a result, instead of a homogeneous
list, we need a heterogeneous list. This is simple to represent using the existential
type, `dyn`:

```
dyn : type.
dyn : A -> dyn.
```

So the type of `Arguments` should be `list dyn` rather than `list typ`. The type
of `teq` will need to be changed, so that we can apply it for any different type,
rather than just `typ`:

```
teq : [A] A -> A -> prop.
```

Furthermore, since we have a heterogeneous list, we need a `map` that uses
polymorphic recursion: it needs take a polymorphic function as an argument, so that
it can be used at different types for different elements of the list.

```
dyn.map : (forall A. [A] A -> A -> prop) -> list dyn -> list dyn -> prop.
```

This is in contrast to a type like `[A] (A -> A -> prop) -> list dyn -> list dyn ->
prop`, which would instantiate the type `A` to the type of the first element of the
list, making further applications to different types impossible.

Makam currently does not provide higher-rank types as the above type suggests
-- nor do any \lamprolog implementations that we are aware of. Instead, it provides
a way to side-step this issue, through a predicate that replaces polymorphic
type variables with fresh variables, allowing it to be instantiated with new types.
This is called `dyn.call`, and `dyn.map` can be defined in terms of that:

```
dyn.call : [B] (A -> A -> prop) -> B -> B -> prop.
dyn.map : (A -> A -> prop) -> list dyn -> list dyn -> prop.
dyn.map P [] [].
dyn.map P (HD :: TL) (HD' :: TL') :- dyn.call P HD HD', dyn.map P TL TL'.
```

(`dyn.call` is itself defined in terms of a more fundamental predicate `dyn.duphead`
that creates a fresh version of a single polymorphic constructor with fresh
type variables.)

Based on these, the only thing missing is a way to actually check whether a term
is a ground term that can be decomposed into a constructor and a list of arguments.
Makam provides this in the form of the `refl.headargs` predicate:

```
refl.headargs : B -> A -> list dyn -> prop.
```

(Other Prolog implementations also provide predicates towards the same effect;
for example, SWI-Prolog provides `compound_name_arguments` which is quite similar.
Though such predicates are not typical in other \lamprolog implementations, they
should not be viewed as a hack: we could always define these within the language
if we maintained a discipline, where we added a rule to `refl.headargs` for every
constructor that we introduce. For example:
```
arrowmany : list typ -> typ -> typ.
refl.headargs (arrowmany TS T) [arrowmany, [dyn TS, dyn T]].
```
Maybe taking extra care to check that we are not instantiating a unification
by using `refl.isunif`.)

We are now ready to proceed to defining the boilerplate generically. We will do this
as a reusable higher-order predicate for structural recursion, that we will use to
implement `teq`. We will define it in open recursion style, providing the predicate
to use on recursive calls as an argument:

```makam
structural_recursion : [B] (A -> A -> prop) -> B -> B -> prop.

structural_recursion Rec X Y :-
  refl.headargs X Constructor Arguments,
  dyn.map Rec Arguments Arguments',
  refl.headargs Y Constructor Arguments'.
```

We also need to handle built-in types, such as the meta-level `int` and `string`
types, in case they are used as an argument with other constructors: 

```makam
structural_recursion Rec (X : string) (X : string).
structural_recursion Rec (X : int) (X : int).
```

And last, we need to handle the case of the meta-level function type as well:

```makam
structural_recursion Rec (X : A -> B) (Y : A -> B) :-
  (x:A -> structural_recursion Rec x x -> structural_recursion Rec (X x) (Y x)).
```

We are done! Now we can define `teq` using `structural_recursion`, through
an auxiliary predicate called `teq_aux`. We only need to define the non-trivial
cases for it, using `structural_recursion` for the rest, while tying the
open recursion knot at the same time:

```makam
teq_aux : [A] A -> A -> prop.

teq T T' :- teq_aux T T'.

teq_aux T T' :-
  structural_recursion teq_aux T T'.

teq_aux (tconstr TC Args) T' :-
  type_synonym_info TC Synonym,
  applymany Synonym Args T,
  teq_aux T T'.

teq_aux T' (tconstr TC Args) :-
  type_synonym_info TC Synonym,
  applymany Synonym Args T,
  teq_aux T' T.
```

Other than minimizing the boilerplate, the great thing about using
`structural_recursion` is that no adaptation needs to be done when we add any new
constructor to our `typ` datatype -- even if that uses new types that we have not
defined before. For example, we did not have to take any special provision to handle
types we defined earlier such as `dbind` -- everything works out thanks to the
reflective predicates we are using. (Mention something about the expression problem?)

The one form of terms that `structural_recursion` does not handle are uninstantiated
unification variables. We find that leaving that as something that we handle whenever
we define a new predicate that uses `structural_recursion` works fine. In this case,
`teq` is only supposed to be used with ground terms, so it is fine if we fail when we
encounter a unification variable.

Let's try an example out:

```makam
ascribe : term -> typ -> term.
typeof (ascribe E T) T :- typeof E T.

wfprogram (
  (type_synonym (dbindnext (fun a => dbindbase (product [a, a])))
  (fun bintuple => 
  
  main (lam (tconstr bintuple [product [nat, nat]])
            (fun x => 
    case_or_else x
    (patt_tuple [patt_tuple [patt_wild, patt_wild], patt_tuple [patt_wild, patt_wild]])
    (dbindbase (tuple []))
    (tuple [])
  ))
))) ?
>> Yes.
```

Let's make sure we don't diverge on type error:

```makam
ascribe : term -> typ -> term.
typeof (ascribe E T) T :- typeof E T.

wfprogram (
  (type_synonym (dbindnext (fun a => dbindbase (product [a, a])))
  (fun bintuple => 
  
  main (lam (tconstr bintuple [product [nat, nat]])
            (fun x => 
    case_or_else x
    (patt_tuple [patt_tuple [patt_wild], patt_tuple [patt_wild, patt_wild]])
    (dbindbase (tuple []))
    (tuple [])
  ))
))) ?
>> Impossible.
```
