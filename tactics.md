Tactics
=======

In this chapter, we describe an alternative approach to constructing
proofs, using *tactics*.  A proof term is a representation of a
mathematical proof; tactics are commands, or instructions, that
describe how to build such a proof. Informally, we might begin a
mathematical proof by saying "to prove the forward direction, unfold
the definition, apply the previous lemma, and simplify." Just as these
are instructions that tell the reader how to find the relevant proof,
tactics are instructions that tell Lean how to construct a proof
term. They naturally support an incremental style of writing proofs,
in which you decompose a proof and work on goals one step at a time.

We will describe proofs that consist of sequences of tactics as
"tactic-style" proofs, to contrast with the ways of writing proof
terms we have seen so far, which we will call "term-style"
proofs. Each style has its own advantages and disadvantages. For
example, tactic-style proofs can be harder to read, because they
require the reader to predict or guess the results of each
instruction. But they can also be shorter and easier to
write. Moreover, tactics offer a gateway to using Lean's automation,
since automated procedures are themselves tactics.

Entering Tactic Mode
--------------------

Conceptually, stating a theorem or introducing a ``have`` statement
creates a goal, namely, the goal of constructing a term with the
expected type. For example, the following creates the goal of
constructing a term of type ``p ∧ q ∧ p``, in a context with constants
``p q : Prop``, ``hp : p`` and ``hq : q``:

```lean
theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p :=
  sorry
```

We can write this goal as follows:

```
    p : Prop, q : Prop, hp : p, hq : q ⊢ p ∧ q ∧ p
```

Indeed, if you replace the "sorry" by an underscore in the example
above, Lean will report that it is exactly this goal that has been
left unsolved.

Ordinarily, we meet such a goal by writing an explicit term. But
wherever a term is expected, Lean allows us to insert instead a ``by
<tactics>`` block, where ``<tactics>`` is a sequence of commands,
separated by semicolons or line breaks. We can prove the theorem above
in that way:

```lean
theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p :=
  by apply And.intro
     exact hp
     apply And.intro
     exact hq
     exact hp
```

We often put the ``by`` keyword on the preceding line, and write the
example above as

```lean
theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p := by
  apply And.intro
  exact hp
  apply And.intro
  exact hq
  exact hp
```

The ``apply`` tactic applies an expression, viewed as denoting a
function with zero or more arguments. It unifies the conclusion with
the expression in the current goal, and creates new goals for the
remaining arguments, provided that no later arguments depend on
them. In the example above, the command ``apply And.intro`` yields two
subgoals:

```
    case left
    p : Prop,
    q : Prop,
    hp : p,
    hq : q
    ⊢ p

    case right
    p : Prop,
    q : Prop,
    hp : p,
    hq : q
    ⊢ q ∧ p
```

The first goal is met with the command ``exact hp``. The ``exact``
command is just a variant of ``apply`` which signals that the
expression given should fill the goal exactly. It is good form to use
it in a tactic proof, since its failure signals that something has
gone wrong. It is also more robust than ``apply``, since the
elaborator takes the expected type, given by the target of the goal,
into account when processing the expression that is being applied. In
this case, however, ``apply`` would work just as well.

You can see the resulting proof term with the ``#print`` command:

```lean
# theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p := by
#  apply And.intro
#  exact hp
#  apply And.intro
#  exact hq
#  exact hp
#print test
```

You can write a tactic script incrementally. In VS Code, you can open
a window to display messages by pressing ``Ctrl-Shift-Enter``, and
that window will then show you the current goal whenever the cursor is
in a tactic block. In Emacs, you can see the goal at the end of any
line by pressing ``C-c C-g``, or see the remaining goal in an
incomplete proof by putting the cursor after the first character of
the last tactic. If the proof is incomplete, the token ``by`` is
decorated with a red squiggly line, and the error message contains the
remaining goals.

Tactic commands can take compound expressions, not just single
identifiers. The following is a shorter version of the preceding
proof:

```lean
theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p := by
  apply And.intro hp
  exact And.intro hq hp
```

Unsurprisingly, it produces exactly the same proof term.

```lean
# theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p := by
#  apply And.intro hp
#  exact And.intro hq hp
#print test
```

Multiple tactic applications can be written in a single line by concatenating with a semicolon.

```lean
theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p := by
  apply And.intro hp; exact And.intro hq hp
```

Tactics that may produce multiple subgoals often tag them. For
example, the tactic ``apply And.intro`` tagged the first sugoal as
``left``, and the second as ``right``. In the case of the ``apply``
tactic, the tags are inferred from the parameters names used in the
``And.intro`` declaration. We can structure our tactics using the
notation ``case <tag> => <tactics>``. The following is a structured
version of our first tactic proof in this chapter.

```lean
theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p := by
  apply And.intro
  case left => exact hp
  case right =>
    apply And.intro
    case left => exact hq
    case right => exact hp
```

You can solve the subgoal ``right`` before ``left`` using the ``case``
notation

```lean
theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p := by
  apply And.intro
  case right =>
    apply And.intro
    case left => exact hq
    case right => exact hp
  case left => exact hp
```

Note that Lean hides the other goals inside the ``case`` block. We say
it is "focusing" on the selected goal.  Moreover, Lean signs an error
if the selected goal is not fully solved at the end of the ``case``
block.

For simple sugoals, it may not be worth selecting a subgoal using its
tag, but we may still want to structure the proof. Lean also provides
the "bullet" notation ``. <tactics>`` (or ``· <tactics>``) for
structuring proof.

```lean
theorem test (p q : Prop) (hp : p) (hq : q) : p ∧ q ∧ p := by
  apply And.intro
  . exact hp
  . apply And.intro
    . exact hq
    . exact hp
```

Basic Tactics
-------------

In addition to ``apply`` and ``exact``, another useful tactic is
``intro``, which introduces a hypothesis. What follows is an example
of an identity from propositional logic that we proved in a previous
chapter, now proved using tactics.

```lean
example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) := by
  apply Iff.intro
  . intro h
    apply Or.elim (And.right h)
    . intro hq
      apply Or.inl
      apply And.intro
      . exact And.left h
      . exact hq
    . intro hr
      apply Or.inr
      apply And.intro
      . exact And.left h
      . exact hr
  . intro h
    apply Or.elim h
    . intro hpq
      apply And.intro
      . exact And.left hpq
      . apply Or.inl
        exact And.right hpq
    . intro hpr
      apply And.intro
      . exact And.left hpr
      . apply Or.inr
        exact And.right hpr
```

The ``intro`` command can more generally be used to introduce a variable of any type:

```lean
example (α : Type) : α → α := by
  intro a
  exact a

example (α : Type) : ∀ x : α, x = x := by
  intro x
  exact Eq.refl x
```

You can use it to introduce several variables:

```lean
example : ∀ a b c : Nat, a = b → a = c → c = b := by
  intro a b c h₁ h₂
  exact Eq.trans (Eq.symm h₂) h₁
```

As the ``apply`` tactic is a command for constructing function
applications interactively, the ``intro`` tactic is a command for
constructing function abstractions interactively (i.e., terms of the
form ``fun x => e``).  As the lambda abstraction notation, the
``intro`` tactic allows us to use an implicit ``match``.

```lean
variable (α : Type) (p q : α → Prop)
example : (∃ x, p x ∧ q x) → ∃ x, q x ∧ p x := by
  intro ⟨w, hpw, hqw⟩
  exact ⟨w, hqw, hpw⟩
```

You can also provide multiple alternatives like in the ``match`` expression.

```lean
variable (α : Type) (p q : α → Prop)
example : (∃ x, p x ∨ q x) → ∃ x, q x ∨ p x := by
  intro
    | ⟨w, Or.inl h⟩ => exact ⟨w, Or.inr h⟩
    | ⟨w, Or.inr h⟩ => exact ⟨w, Or.inl h⟩
```

The ``intros`` tactic can be used without any arguments, in which
case, it chooses names and introduces as many variables as it can. We
will see an example of this in a moment.

The ``assumption`` tactic looks through the assumptions in context of
the current goal, and if there is one matching the conclusion, it
applies it.

```lean
variable (x y z w : Nat)

example (h₁ : x = y) (h₂ : y = z) (h₃ : z = w) : x = w := by
  apply Eq.trans h₁
  apply Eq.trans h₂
  assumption   -- applied h₃
```

It will unify metavariables in the conclusion if necessary:

```lean
# variable (x y z w : Nat)
example (h₁ : x = y) (h₂ : y = z) (h₃ : z = w) : x = w := by
  apply Eq.trans
  assumption      -- solves x = ?b with h₁
  apply Eq.trans
  assumption      -- solves y = ?h₂.b with h₂
  assumption      -- solves z = w with h₃
```

The following example uses the ``intros`` command to introduce the three variables and two hypotheses automatically:

```lean
example : ∀ a b c : Nat, a = b → a = c → c = b := by
  intros
  apply Eq.trans
  apply Eq.symm
  assumption
  assumption
```

Note that names automatically generated by Lean are inaccessible by default. The motivation is to
ensure your tactic proofs do not rely on automatically generated names, and are consequently more robust.
However, you can use the combinator ``unhygienic`` to disable this restriction.

```lean
example : ∀ a b c : Nat, a = b → a = c → c = b := by unhygienic
  intros
  apply Eq.trans
  apply Eq.symm
  exact a_2
  exact a_1
```

You can also use the ``renameI`` tactic to rename the most recent inaccessible names in your context.
In the following example, the tactic ``renameI h1 _ h2`` renames two of the last three hypotheses in
your context.

```lean
example : ∀ a b c d : Nat, a = b → a = d → a = c → c = b := by
  intros
  renameI h1 _ h2
  apply Eq.trans
  apply Eq.symm
  exact h2
  exact h1
```

The ``rfl`` tactic is syntax sugar for ``exact rfl``.
```lean
example (y : Nat) : (fun x : Nat => 0) y = 0 :=
  by rfl
```

The ``repeat`` combinator can be used to apply a tactic several times.

```lean
example : ∀ a b c : Nat, a = b → a = c → c = b := by
  intros
  apply Eq.trans
  apply Eq.symm
  repeat assumption
```

Another tactic that is sometimes useful is the ``revert`` tactic,
which is, in a sense, an inverse to ``intro``.

```lean
example (x : Nat) : x = x := by
 revert x
 -- goal is ⊢ ∀ (x : Nat), x = x
 intro y
 -- goal is y : Nat ⊢ y = y
 rfl
```

Moving a hypothesis into the goal yields an implication:

```lean
example (x y : Nat) (h : x = y) : y = x := by
  revert h
  -- goal is x y : Nat ⊢ x = y → y = x
  intro h₁
  -- goal is x y : ℕ, h₁ : x = y ⊢ y = x
  apply Eq.symm
  assumption
```

But ``revert`` is even more clever, in that it will revert not only an
element of the context but also all the subsequent elements of the
context that depend on it. For example, reverting ``x`` in the example
above brings ``h`` along with it:

```lean
example (x y : Nat) (h : x = y) : y = x := by
  revert x
  -- goal is y : Nat ⊢ ∀ (x : Nat), x = y → y = x
  intros
  apply Eq.symm
  assumption
```

You can also revert multiple elements of the context at once:
```lean
example (x y : Nat) (h : x = y) : y = x := by
  revert x y
  -- goal is ⊢ ∀ (x y : Nat), x = y → y = x
  intros
  apply Eq.symm
  assumption
```

You can only ``revert`` an element of the local context, that is, a
local variable or hypothesis. But you can replace an arbitrary
expression in the goal by a fresh variable using the ``generalize``
tactic.

```lean
example : 3 = 3 := by
  generalize 3 = x
  -- goal is x : Nat ⊢ x = x,
  revert x
  -- goal is ⊢ ∀ (x : Nat), x = x
  intro y
  -- goal is y : Nat ⊢ y = y
  rfl
```

The mnemonic in the notation above is that you are generalizing the
goal by setting ``3`` to an arbitrary variable ``x``. Be careful: not
every generalization preserves the validity of the goal. Here,
``generalize`` replaces a goal that could be proved using
``rfl`` with one that is not provable:

```lean
example : 2 + 3 = 5 := by
  generalize  3 = x
  -- goal is x : Nat ⊢ 2 + x = 5
  admit
```

In this example, the ``admit`` tactic is the analogue of the ``sorry``
proof term. It closes the current goal, producing the usual warning
that ``sorry`` has been used. To preserve the validity of the previous
goal, the ``generalize`` tactic allows us to record the fact that
``3`` has been replaced by ``x``. All we need to do is to provide a
label, and ``generalize`` uses it to store the assignment in the local
context:

```lean
example : 2 + 3 = 5 := by
  generalize h : 3 = x
  -- goal is x : Nat, h : 3 = x ⊢ 2 + x = 5
  rw [← h]
```

Here the ``rewrite`` tactic, abbreviated ``rw``, uses ``h`` to replace
``x`` by ``3`` again. The ``rewrite`` tactic will be discussed below.


More Tactics
------------

Some additional tactics are useful for constructing and destructing
propositions and data. For example, when applied to a goal of the form
``p ∨ q``, we use tactics such as ``apply or.inl`` and ``apply
or.inr``.  Conversely, the ``cases`` tactic can be used to decompose a
disjunction.

```lean
example (p q : Prop) : p ∨ q → q ∨ p := by
  intro h
  cases h with
  | inl hp => apply Or.inr; exact hp
  | inr hq => apply Or.inl; exact hq
```

Note that the syntax is similar to the one used in `match` expressions.
The new subgoals can be solved in any order.

```lean
example (p q : Prop) : p ∨ q → q ∨ p := by
  intro h
  cases h with
  | inr hq => apply Or.inl; exact hq
  | inl hp => apply Or.inr; exact hp
```

You can also use a (unstructured) ``cases`` without the ``with`` and a tactic
for each alternative.

```lean
example (p q : Prop) : p ∨ q → q ∨ p := by
  intro h
  cases h
  apply Or.inr
  assumption
  apply Or.inl
  assumption
```

The (unstructured) ``cases`` is particularly useful when you can close several
subgoals using the same tactic.

```lean
example (p : Prop) : p ∨ p → p := by
  intro h
  cases h
  repeat assumption
```

You can also use the combinator ``tac1 <;> tac2`` to apply ``tac2`` to each
subgoal produced by tactic ``tac1``

```lean
example (p : Prop) : p ∨ p → p := by
  intro h
  cases h <;> assumption
```

You can combine the unstructured ``cases`` tactic with the ``case`` and ``.`` notation.

```lean
example (p q : Prop) : p ∨ q → q ∨ p := by
  intro h
  cases h
  . apply Or.inr
    assumption
  . apply Or.inl
    assumption

example (p q : Prop) : p ∨ q → q ∨ p := by
  intro h
  cases h
  case inr h =>
    apply Or.inl
    assumption
  case inl h =>
    apply Or.inr
    assumption

example (p q : Prop) : p ∨ q → q ∨ p := by
  intro h
  cases h
  case inr h =>
    apply Or.inl
    assumption
  . apply Or.inr
    assumption
```


The ``cases`` tactic can also be used to
decompose a conjunction.

```lean
example (p q : Prop) : p ∧ q → q ∧ p := by
  intro h
  cases h with
  | intro hp hq => constructor; exact hq; exact hp
```

In this example, there is only one goal after the ``cases`` tactic is
applied, with ``h : p ∧ q`` replaced by a pair of assumptions,
``hp : p`` and ``hq : q``. The ``constructor`` tactic applies the unique
constructor for conjunction, ``And.intro``. With these tactics, an
example from the previous section can be rewritten as follows:

```lean
example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) := by
  apply Iff.intro
  . intro h
    cases h with
    | intro hp hqr =>
      cases hqr
      . apply Or.inl; constructor <;> assumption
      . apply Or.inr; constructor <;> assumption
  . intro h
    cases h with
    | inl hpq =>
      cases hpq with
      | intro hp hq => constructor; exact hp; apply Or.inl; exact hq
    | inr hpr =>
      cases hpr with
      | intro hp hr => constructor; exact hp; apply Or.inr; exact hr
```

We will see in [Chapter Inductive Types](./inductive_types.md) that
these tactics are quite general. The ``cases`` tactic can be used to
decompose any element of an inductively defined type; ``constructor``
always applies the first applicable constructor of an inductively defined type,
For example, we can use ``cases`` and ``constructor`` with an existential quantifier:

```lean
example (p q : Nat → Prop) : (∃ x, p x) → ∃ x, p x ∨ q x := by
  intro h
  cases h with
  | intro x px => constructor; apply Or.inl; exact px
```

Here, the ``constructor`` tactic leaves the first component of the
existential assertion, the value of ``x``, implicit. It is represented
by a metavariable, which should be instantiated later on. In the
previous example, the proper value of the metavariable is determined
by the tactic ``exact px``, since ``px`` has type ``p x``. If you want
to specify a witness to the existential quantifier explicitly, you can
use the ``exists`` tactic instead:

```lean
example (p q : Nat → Prop) : (∃ x, p x) → ∃ x, p x ∨ q x := by
  intro h
  cases h with
  | intro x px => exists x; apply Or.inl; exact px
```

Here is another example:

```lean
example (p q : Nat → Prop) : (∃ x, p x ∧ q x) → ∃ x, q x ∧ p x := by
  intro h
  cases h with
  | intro x hpq =>
    cases hpq with
    | intro hp hq =>
      exists x
      constructor <;> assumption
```

These tactics can be used on data just as well as propositions. In the
next two examples, they are used to define functions which swap the
components of the product and sum types:

```lean
def swap_pair : α × β → β × α := by
  intro p
  cases p
  constructor <;> assumption
```

```lean
def swap_sum : Sum α β → Sum β α := by
  intro p
  cases p
  . apply Sum.inr; assumption
  . apply Sum.inl; assumption
```

Note that up to the names we have chosen for the variables, the
definitions are identical to the proofs of the analogous propositions
for conjunction and disjunction. The ``cases`` tactic will also do a
case distinction on a natural number:

```lean
open Nat
example (P : Nat → Prop) (h₀ : P 0) (h₁ : ∀ n, P (succ n)) (m : Nat) : P m := by
 cases m with
 | zero    => exact h₀
 | succ m' => exact h₁ m'
```

The ``cases`` tactic, and its companion, the ``induction`` tactic, are discussed in greater detail in
the [Tactics for Inductive Types](TBD) section.

The ``contradiction`` tactic searches for a contradiction among the hypotheses of the current goal:

```lean
example (p q : Prop) : p ∧ ¬ p → q := by
  intro h
  cases h
  contradiction
```

You can also use ``match`` in tactic blocks.

```lean
example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) := by
  apply Iff.intro
  . intro h
    match h with
    | ⟨_, Or.inl _⟩ => apply Or.inl; constructor <;> assumption
    | ⟨_, Or.inr _⟩ => apply Or.inr; constructor <;> assumption
  . intro h
    match h with
    | Or.inl ⟨hp, hq⟩ => constructor; exact hp; apply Or.inl; exact hq
    | Or.inr ⟨hp, hr⟩ => constructor; exact hp; apply Or.inr; exact hr
```

You can "combine" ``intro h`` with ``match h ...`` and write the previous examples as follows

```lean
example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) := by
  apply Iff.intro
  . intro
     | ⟨hp, Or.inl hq⟩ => apply Or.inl; constructor <;> assumption
     | ⟨hp, Or.inr hr⟩ => apply Or.inr; constructor <;> assumption
  . intro
     | Or.inl ⟨hp, hq⟩ => constructor; assumption; apply Or.inl; assumption
     | Or.inr ⟨hp, hr⟩ => constructor; assumption; apply Or.inr; assumption

```

<a name="structuring_tactic_proofs"></a>Structuring Tactic Proofs
-------------------------

Tactics often provide an efficient way of building a proof, but long sequences of instructions can obscure the structure of the argument. In this section, we describe some means that help provide structure to a tactic-style proof, making such proofs more readable and robust.

One thing that is nice about Lean's proof-writing syntax is that it is possible to mix term-style and tactic-style proofs, and pass between the two freely. For example, the tactics ``apply`` and ``exact`` expect arbitrary terms, which you can write using ``have``, ``show``, and so on. Conversely, when writing an arbitrary Lean term, you can always invoke the tactic mode by inserting a ``begin...end`` block. The following is a somewhat toy example:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) → (p ∧ q) ∨ (p ∧ r) :=
    begin
      intro h,
      exact
        have hp : p, from h.left,
        have hqr : q ∨ r, from h.right,
        show (p ∧ q) ∨ (p ∧ r),
        begin
          cases hqr with hq hr,
            exact or.inl ⟨hp, hq⟩,
          exact or.inr ⟨hp, hr⟩
        end
    end

The following is a more natural example:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) :=
    begin
      apply iff.intro,
        intro h,
        cases h.right with hq hr,
          exact or.inl ⟨h.left, hq⟩,
        exact or.inr ⟨h.left, hr⟩,
      intro h,
      cases h with hpq hpr,
        exact ⟨hpq.left, or.inl hpq.right⟩,
      exact ⟨hpr.left, or.inr hpr.right⟩
    end

In fact, there is a ``show`` tactic, which is the analog of the ``show`` keyword in a proof term. It simply declares the type of the goal that is about to be solved, while remaining in tactic mode. Moreover, in tactic mode, ``from`` is an alternative name for ``exact``. With the ``show`` and ``from`` tactics, the previous proof can be written more perspicuously as follows:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) :=
    begin
      apply iff.intro,
        intro h,
        cases h.right with hq hr,
          show (p ∧ q) ∨ (p ∧ r),
            from or.inl ⟨h.left, hq⟩,
          show (p ∧ q) ∨ (p ∧ r),
            from or.inr ⟨h.left, hr⟩,
      intro h,
      cases h with hpq hpr,
        show p ∧ (q ∨ r),
          from ⟨hpq.left, or.inl hpq.right⟩,
        show p ∧ (q ∨ r),
          from ⟨hpr.left, or.inr hpr.right⟩
    end

Alternatively, you can leave off the ``from`` and remain in tactic mode:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) :=
    begin
      apply iff.intro,
        intro h,
        cases h.right with hq hr,
          show (p ∧ q) ∨ (p ∧ r),
            { left, split, exact h.left, assumption },
          show (p ∧ q) ∨ (p ∧ r),
            { right, split, exact h.left, assumption },
      intro h,
      cases h with hpq hpr,
        show p ∧ (q ∨ r),
          { cases hpq, split, assumption, left, assumption },
        show p ∧ (q ∨ r),
          { cases hpr, split, assumption, right, assumption }
    end

The ``show`` tactic can actually be used to rewrite a goal to something definitionally equivalent:

.. code-block:: lean

    example (n : ℕ) : n + 1 = nat.succ n :=
    begin
      show nat.succ n = nat.succ n,
      reflexivity
    end

In fact, ``show`` does a little more work. When there are multiple goals, you can use ``show`` to select which goal you want to work on. Thus both proofs below work:

.. code-block:: lean

    example (p q : Prop) : p ∧ q → q ∧ p :=
    begin
      intro h,
      cases h with hp hq,
      split,
      show q, from hq,
      show p, from hp
    end

    example (p q : Prop) : p ∧ q → q ∧ p :=
    begin
      intro h,
      cases h with hp hq,
      split,
      show p, from hp,
      show q, from hq
    end

There is also a ``have`` tactic, which introduces a new subgoal, just as when writing proof terms:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) → (p ∧ q) ∨ (p ∧ r) :=
    begin
      intro h,
      cases h with hp hqr,
      show (p ∧ q) ∨ (p ∧ r),
      cases hqr with hq hr,
        have hpq : p ∧ q,
          from and.intro hp hq,
        left, exact hpq,
      have hpr : p ∧ r,
        from and.intro hp hr,
      right, exact hpr
    end

As with ``show``, you can omit the ``from`` and stay in tactic mode:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) → (p ∧ q) ∨ (p ∧ r) :=
    begin
      intro h,
      cases h with hp hqr,
      show (p ∧ q) ∨ (p ∧ r),
      cases hqr with hq hr,
        have hpq : p ∧ q,
          split; assumption,
        left, exact hpq,
      have hpr : p ∧ r,
        split; assumption,
      right, exact hpr
    end

As with proof terms, you can omit the label in the ``have`` tactic, in which case, the default label ``this`` is used:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) → (p ∧ q) ∨ (p ∧ r) :=
    begin
      intro h,
      cases h with hp hqr,
      show (p ∧ q) ∨ (p ∧ r),
      cases hqr with hq hr,
        have : p ∧ q,
          split; assumption,
        left, exact this,
      have : p ∧ r,
        split; assumption,
      right, exact this
    end

You can also use the ``have`` tactic with the ``:=`` token, which has the same effect as ``from``:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) → (p ∧ q) ∨ (p ∧ r) :=
    begin
      intro h,
      have hp : p := h.left,
      have hqr : q ∨ r := h.right,
      show (p ∧ q) ∨ (p ∧ r),
      cases hqr with hq hr,
        exact or.inl ⟨hp, hq⟩,
      exact or.inr ⟨hp, hr⟩
    end

In this case, the types can be omitted, so we can write ``have hp := h.left`` and ``have hqr := h.right``. In fact, with this notation, you can even omit both the type and the label, in which case the new fact is introduced with the label ``this``.

Lean also has a ``let`` tactic, which is similar to the ``have`` tactic, but is used to introduce local definitions instead of auxiliary facts. It is the tactic analogue of a ``let`` in a proof term.

.. code-block:: lean

    example : ∃ x, x + 2 = 8 :=
    begin
      let a : ℕ := 3 * 2,
      existsi a,
      reflexivity
    end

As with ``have``, you can leave the type implicit by writing ``let a := 3 * 2``. The difference between ``let`` and ``have`` is that ``let`` introduces a local definition in the context, so that the definition of the local constant can be unfolded in the proof.

For even more structured proofs, you can nest ``begin...end`` blocks within other ``begin...end`` blocks. In a nested block, Lean focuses on the first goal, and generates an error if it has not been fully solved at the end of the block. This can be helpful in indicating the separate proofs of multiple subgoals introduced by a tactic.

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) :=
    begin
      apply iff.intro,
      begin
        intro h,
        cases h.right with hq hr,
        begin
          show (p ∧ q) ∨ (p ∧ r),
            exact or.inl ⟨h.left, hq⟩
        end,
        show (p ∧ q) ∨ (p ∧ r),
          exact or.inr ⟨h.left, hr⟩
      end,
      intro h,
      cases h with hpq hpr,
      begin
        show p ∧ (q ∨ r),
          exact ⟨hpq.left, or.inl hpq.right⟩
      end,
      show p ∧ (q ∨ r),
        exact ⟨hpr.left, or.inr hpr.right⟩
    end

Here, we have introduced a new ``begin..end`` block whenever a tactic leaves more than one subgoal. You can check that at every line in this proof, there is only one goal visible. Notice that you still need to use a comma after a ``begin...end`` block when there are remaining goals to be discharged.

Within a ``begin...end`` block, you can abbreviate nested occurrences of ``begin`` and ``end`` with curly braces:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) :=
    begin
      apply iff.intro,
      { intro h,
        cases h.right with hq hr,
        { show (p ∧ q) ∨ (p ∧ r),
            exact or.inl ⟨h.left, hq⟩ },
        show (p ∧ q) ∨ (p ∧ r),
          exact or.inr ⟨h.left, hr⟩ },
      intro h,
      cases h with hpq hpr,
      { show p ∧ (q ∨ r),
          exact ⟨hpq.left, or.inl hpq.right⟩ },
      show p ∧ (q ∨ r),
        exact ⟨hpr.left, or.inr hpr.right⟩
    end

This helps explain the convention on indentation we have adopted here: every time a tactic leaves more than one subgoal, we separate the remaining subgoals by enclosing them in blocks and indenting, until we are back down to one subgoal. Thus if the application of theorem ``foo`` to a single goal produces four subgoals, one would expect the proof to look like this:

.. code-block:: text

    begin
      apply foo,
      { ... proof of first goal ... },
      { ... proof of second goal ... },
      { ... proof of third goal ... },
      proof of final goal
    end

Another reasonable convention is to enclose *all* the remaining subgoals in indented blocks, including the last one:

.. code-block:: lean

    example (p q r : Prop) : p ∧ (q ∨ r) ↔ (p ∧ q) ∨ (p ∧ r) :=
    begin
      apply iff.intro,
      { intro h,
        cases h.right with hq hr,
        { show (p ∧ q) ∨ (p ∧ r),
            exact or.inl ⟨h.left, hq⟩ },
        { show (p ∧ q) ∨ (p ∧ r),
            exact or.inr ⟨h.left, hr⟩ }},
      { intro h,
        cases h with hpq hpr,
        { show p ∧ (q ∨ r),
            exact ⟨hpq.left, or.inl hpq.right⟩ },
        { show p ∧ (q ∨ r),
            exact ⟨hpr.left, or.inr hpr.right⟩ }}
    end

With this convention, the proof using ``foo`` described above would look like this:

.. code-block:: text

    begin
      apply foo,
      { ... proof of first goal ... },
      { ... proof of second goal ... },
      { ... proof of third goal ... },
      { ... proof of final goal ....}
    end

Both conventions are reasonable. The second convention has the effect that the text in a long proof gradually creeps to the right. Many theorems in mathematics have side conditions that can be dispelled quickly; using the first convention means that the proofs of these side conditions are indented until we return to the "linear" part of the proof.

Combining these various mechanisms makes for nicely structured tactic proofs:

.. code-block:: lean

    example (p q : Prop) : p ∧ q ↔ q ∧ p :=
    begin
      apply iff.intro,
      { intro h,
        have hp : p := h.left,
        have hq : q := h.right,
        show q ∧ p,
          exact ⟨hq, hp⟩ },
      intro h,
      have hp : p := h.right,
      have hq : q := h.left,
      show p ∧ q,
        exact ⟨hp, hq⟩
    end

.. _tactic_combinators:

Tactic Combinators
------------------

*Tactic combinators* are operations that form new tactics from old ones. A sequencing combinator is already implicit in the commas that appear in a ``begin...end`` block:

.. code-block:: lean

    example (p q : Prop) (hp : p) : p ∨ q :=
    begin left, assumption end

This is essentially equivalent to the following:

.. code-block:: lean

    example (p q : Prop) (hp : p) : p ∨ q :=
    by { left, assumption }

Here, ``{ left, assumption }`` is functionally equivalent to a single tactic which first applies ``left`` and then applies ``assumption``.

In an expression ``t₁; t₂``, the semicolon provides a *parallel* version of the sequencing operation: ``t₁`` is applied to the current goal, and then ``t₂`` is applied to *all* the resulting subgoals:

.. code-block:: lean

    example (p q : Prop) (hp : p) (hq : q) : p ∧ q :=
    by split; assumption

This is especially useful when the resulting goals can be finished off in a uniform way, or, at least, when it is possible to make progress on all of them uniformly.

The *orelse* combinator, denoted ``<|>``, applies one tactic, and then backtracks and applies another one if the first one fails:

.. code-block:: lean

    example (p q : Prop) (hp : p) : p ∨ q :=
    by { left, assumption } <|> { right, assumption}

    example (p q : Prop) (hq : q) : p ∨ q :=
    by { left, assumption } <|> { right, assumption}

In the first example, the left branch succeeds, whereas in the second one, it is the right one that succeeds. In the next three examples, the same compound tactic succeeds in each case.

.. code-block:: lean

    example (p q r : Prop) (hp : p) : p ∨ q ∨ r :=
    by repeat { {left, assumption} <|> right <|> assumption }

    example (p q r : Prop) (hq : q) : p ∨ q ∨ r :=
    by repeat { {left, assumption} <|> right <|> assumption }

    example (p q r : Prop) (hr : r) : p ∨ q ∨ r :=
    by repeat { {left, assumption} <|> right <|> assumption }

The tactic tries to solve the left disjunct immediately by assumption; if that fails, it tries to focus on the right disjunct; and if that doesn't work, it invokes the assumption tactic.

Incidentally, a tactic expression is really a formal term in Lean, of type ``tactic α`` for some ``α``. Tactics can be defined and then applied later on.

.. code-block:: lean

    meta def my_tac : tactic unit :=
    `[ repeat { {left, assumption} <|> right <|> assumption } ]

    example (p q r : Prop) (hp : p) : p ∨ q ∨ r :=
    by my_tac

    example (p q r : Prop) (hq : q) : p ∨ q ∨ r :=
    by my_tac

    example (p q r : Prop) (hr : r) : p ∨ q ∨ r :=
    by my_tac

With a ``begin...end`` block or after a ``by``, Lean's parser uses special mechanisms to parse these expressions, but they are similar to ordinary expressions in Lean like ``x + 2`` and ``list α``. (The annotation ``[...]`` in the definition of ``my_tac`` above invokes the special parsing mechanism here, too.) The book `Programming in Lean <https://leanprover.github.io/programming_in_lean/>`__ provides a fuller introduction to writing tactics and installing them for interactive use. The tactic combinators we are discussing here serve as casual entry points to the tactic programming language.

You will have no doubt noticed by now that tactics can fail. Indeed, it is the "failure" state that causes the *orelse* combinator to backtrack and try the next tactic. The ``try`` combinator builds a tactic that always succeeds, though possibly in a trivial way: ``try t`` executes ``t`` and reports success, even if ``t`` fails. It is equivalent to ``t <|> skip``, where ``skip`` is a tactic that does nothing (and succeeds in doing so). In the next example, the second ``split`` succeeds on the right conjunct ``q ∧ r`` (remember that disjunction and conjunction associate to the right) but fails on the first. The ``try`` tactic ensures that the sequential composition succeeds.

.. code-block:: lean

    example (p q r : Prop) (hp : p) (hq : q) (hr : r) :
      p ∧ q ∧ r :=
    by split; try {split}; assumption

Be careful: ``repeat {try t}`` will loop forever, because the inner tactic never fails.

In a proof, there are often multiple goals outstanding. Parallel sequencing is one way to arrange it so that a single tactic is applied to multiple goals, but there are other ways to do this. For example, ``all_goals t`` applies ``t`` to all open goals:

.. code-block:: lean

    example (p q r : Prop) (hp : p) (hq : q) (hr : r) :
      p ∧ q ∧ r :=
    begin
      split,
      all_goals { try {split} },
      all_goals { assumption }
    end

In this case, the ``any_goals`` tactic provides a more robust solution.
It is similar to ``all_goals``, except it fails unless its argument
succeeds on at least one goal.

.. code-block:: lean

    example (p q r : Prop) (hp : p) (hq : q) (hr : r) :
      p ∧ q ∧ r :=
    begin
      split,
      any_goals { split },
      any_goals { assumption }
    end

The first tactic in the ``begin...end`` block below repeatedly splits
conjunctions:

.. code-block:: lean

    example (p q r : Prop) (hp : p) (hq : q) (hr : r) :
      p ∧ ((p ∧ q) ∧ r) ∧ (q ∧ r ∧ p) :=
    begin
      repeat { any_goals { split }},
      all_goals { assumption }
    end

In fact, we can compress the full tactic down to one line:

.. code-block:: lean

    example (p q r : Prop) (hp : p) (hq : q) (hr : r) :
      p ∧ ((p ∧ q) ∧ r) ∧ (q ∧ r ∧ p) :=
    by repeat { any_goals { split <|> assumption} }

The combinators ``focus`` and ``solve1`` go in the other direction. Specifically, ``focus t`` ensures that ``t`` only effects the current goal, temporarily hiding the others from the scope. So, if ``t`` ordinarily only effects the current goal, ``focus { all_goals {t} }`` has the same effect as ``t``. The tactic ``solve1 t`` is similar, except that it fails unless ``t`` succeeds in solving the goal entirely. The ``done`` tactic is also sometimes useful to direct the flow of control; it succeeds only if there are no goals left to be solved.

Rewriting
---------

The ``rewrite`` tactic (abbreviated ``rw``) and the ``simp`` tactic were introduced briefly in :numref:`calculational_proofs`. In this section and the next, we discuss them in greater detail.

The ``rewrite`` tactic provides a basic mechanism for applying substitutions to goals and hypotheses, providing a convenient and efficient way of working with equality. The most basic form of the tactic is ``rewrite t``, where ``t`` is a term whose type asserts an equality. For example, ``t`` can be a hypothesis ``h : x = y`` in the context; it can be a general lemma, like ``add_comm : ∀ x y, x + y = y + x``, in which the rewrite tactic tries to find suitable instantiations of ``x`` and ``y``; or it can be any compound term asserting a concrete or general equation. In the following example, we use this basic form to rewrite the goal using a hypothesis.

.. code-block:: lean

    variables (f : ℕ → ℕ) (k : ℕ)

    example (h₁ : f 0 = 0) (h₂ : k = 0) : f k = 0 :=
    begin
      rw h₂, -- replace k with 0
      rw h₁  -- replace f 0 with 0
    end

In the example above, the first use of ``rw`` replaces ``k`` with ``0`` in the goal ``f k = 0``. Then, the second one replaces ``f 0`` with ``0``. The tactic automatically closes any goal of the form ``t = t``. Here is an example of rewriting using a compound expression:

.. code-block:: lean

    example (x y : ℕ) (p : ℕ → Prop) (q : Prop) (h : q → x = y)
      (h' : p y) (hq : q) : p x :=
    by { rw (h hq), assumption }

Here, ``h hq`` establishes the equation ``x = y``. The parentheses around ``h hq`` are not necessary, but we have added them for clarity.

Multiple rewrites can be combined using the notation ``rw [t_1, ..., t_n]``, which is just shorthand for ``rewrite t_1, ..., rewrite t_n``. The previous example can be written as follows:

.. code-block:: lean

    variables (f : ℕ → ℕ) (k : ℕ)

    example (h₁ : f 0 = 0) (h₂ : k = 0) : f k = 0 :=
    by rw [h₂, h₁]

By default, ``rw`` uses an equation in the forward direction, matching the left-hand side with an expression, and replacing it with the right-hand side. The notation ``←t`` can be used to instruct the tactic to use the equality ``t`` in the reverse direction.

.. code-block:: lean

    variables (f : ℕ → ℕ) (a b : ℕ)

    example (h₁ : a = b) (h₂ : f a = 0) : f b = 0 :=
    begin
      rw [←h₁, h₂]
    end

In this example, the term ``←h₁`` instructs the rewriter to replace ``b`` with ``a``. In the editors, you can type the backwards arrow as ``\l``. You can also use the ascii equivalent, ``<-``.

Sometimes the left-hand side of an identity can match more than one subterm in the pattern, in which case the ``rewrite`` tactic chooses the first match it finds when traversing the term. If that is not the one you want, you can use additional arguments to specify the appropriate subterm.

.. code-block:: lean

    import data.nat.basic

    example (a b c : ℕ) : a + b + c = a + c + b :=
    begin
      rw [add_assoc, add_comm b, ←add_assoc]
    end

    example (a b c : ℕ) : a + b + c = a + c + b :=
    begin
      rw [add_assoc, add_assoc, add_comm b]
    end

    example (a b c : ℕ) : a + b + c = a + c + b :=
    begin
      rw [add_assoc, add_assoc, add_comm _ b]
    end

In the first example above, the first step rewrites ``a + b + c`` to ``a + (b + c)``. Then next applies commutativity to the term ``b + c``; without specifying the argument, the tactic would instead rewrite ``a + (b + c)`` to ``(b + c) + a``. Finally, the last step applies associativity in the reverse direction rewriting ``a + (c + b)`` to ``a + c + b``. The next two examples instead apply associativity to move the parenthesis to the right on both sides, and then switch ``b`` and ``c``. Notice that the last example specifies that the rewrite should take place on the right-hand side by specifying the second argument to ``add_comm``.

By default, the ``rewrite`` tactic affects only the goal. The notation ``rw t at h`` applies the rewrite ``t`` at hypothesis ``h``.

.. code-block:: lean

    import data.nat.basic

    -- BEGIN
    variables (f : ℕ → ℕ) (a : ℕ)

    example (h : a + 0 = 0) : f a = f 0 :=
    by { rw add_zero at h, rw h }
    -- END

The first step, ``rw add_zero at h``, rewrites the hypothesis ``a + 0 = 0`` to ``a = 0``. Then the new hypothesis ``a = 0`` is used to rewrite the goal to ``f 0 = f 0``.

The ``rewrite`` tactic is not restricted to propositions. In the following example, we use ``rw h at t`` to rewrite the hypothesis ``t : tuple α n`` to ``v : tuple α 0``.

.. code-block:: lean

    def tuple (α : Type*) (n : ℕ) :=
      { l : list α // list.length l = n }

    variables {α : Type*} {n : ℕ}

    example (h : n = 0) (t : tuple α n) : tuple α 0 :=
    begin
      rw h at t,
      exact t
    end

Note that the rewrite tactic can carry out generic calculations in any algebraic structure. The following examples involve an arbitrary ring and an arbitrary group, respectively.

.. code-block:: lean

    import algebra.ring

    example {α : Type*} [ring α] (a b c : α) :
      a * 0 + 0 * b + c * 0 + 0 * a = 0 :=
    begin
      rw [mul_zero, mul_zero, zero_mul, zero_mul],
      repeat { rw add_zero }
    end

    example {α : Type*} [group α] {a b : α} (h : a * b = 1) :
      a⁻¹ = b :=
    by rw [←(mul_one a⁻¹), ←h, inv_mul_cancel_left]

Using the type class mechanism described in :numref:`Chapter %s <type_classes>`, Lean identifies both abstract and concrete instances of the relevant algebraic structures, and instantiates the relevant facts accordingly.

.. _using_the_simplifier:

Using the Simplifier
--------------------

Whereas ``rewrite`` is designed as a surgical tool for manipulating a goal, the simplifier offers a more powerful form of automation. A number of identities in Lean's library have been tagged with the ``[simp]`` attribute, and the ``simp`` tactic uses them to iteratively rewrite subterms in an expression.

.. code-block:: lean

    import data.nat.basic

    variables (x y z : ℕ) (p : ℕ → Prop)
    variable  (h : p (x * y))

    example : (x + 0) * (0 + y * 1 + z * 0) = x * y :=
    by simp

    include h
    example : p ((x + 0) * (0 + y * 1 + z * 0)) :=
    by { simp, assumption }

In the first example, the left-hand side of the equality in the goal is simplified using the usual identities involving 0 and 1, reducing the goal to ``x * y = x * y``. At that point, ``simp`` applies reflexivity to finish it off. In the second example, ``simp`` reduces the goal to ``p (x * y)``, at which point the assumption ``h`` finishes it off. (Remember that we have to ``include h`` explicitly because it is not explicitly mentioned.) Here are some more examples with lists:

.. code-block:: lean

    import data.list.basic

    variable {α : Type*}

    open list

    example (xs : list ℕ) :
      reverse (xs ++ [1, 2, 3]) = [3, 2, 1] ++ reverse xs :=
    by simp

    example (xs ys : list α) :
      length (reverse (xs ++ ys)) = length xs + length ys :=
    by simp [add_comm]

This example uses facts about lists that are found in Lean's `mathematics library <https://github.com/leanprover/mathlib>`_, which we need to explicitly `import`.

As with ``rw``, you can use the keyword ``at`` to simplify a hypothesis:

.. code-block:: lean

    import data.nat.basic

    -- BEGIN
    variables (x y z : ℕ) (p : ℕ → Prop)

    example (h : p ((x + 0) * (0 + y * 1 + z * 0))) :
      p (x * y) :=
    by { simp at h, assumption }
    -- END

Moreover, you can use a "wildcard" asterisk to simplify all the hypotheses and the goal:

.. code-block:: lean

    import data.nat.basic

    -- BEGIN
    variables (w x y z : ℕ) (p : ℕ → Prop)

    local attribute [simp] mul_comm mul_assoc mul_left_comm
    local attribute [simp] add_assoc add_comm add_left_comm

    example (h : p (x * y + z * w  * x)) : p (x * w * z + y * x) :=
    by { simp at *, assumption }

    example (h₁ : p (1 * x + y)) (h₂ : p  (x * z * 1)) :
      p (y + 0 + x) ∧ p (z * x) :=
    by { simp at *, split; assumption }
    -- END

For operations that are commutative and associative, like multiplication on the natural numbers, the simplifier uses these two facts to rewrite an expression, as well as *left commutativity*. In the case of multiplication the latter is expressed as follows: ``x * (y * z) = y * (x * z)``. The ``local attribute`` command tells the simplifier to use these rules in the current file (or section or namespace, as the case may be). It may seem that commutativity and left-commutativity are problematic, in that repeated application of either causes looping. But the simplifier detects identities that permute their arguments, and uses a technique known as *ordered rewriting*. This means that the system maintains an internal ordering of terms, and only applies the identity if doing so decreases the order. With the three identities mentioned above, this has the effect that all the parentheses in an expression are associated to the right, and the expressions are ordered in a canonical (though somewhat arbitrary) way. Two expressions that are equivalent up to associativity and commutativity are then rewritten to the same canonical form.

.. code-block:: lean

    import data.nat.basic

    variables (x y z w : ℕ) (p : ℕ → Prop)

    local attribute [simp] mul_comm mul_assoc mul_left_comm
    local attribute [simp] add_assoc add_comm add_left_comm

    -- BEGIN
    example : x * y + z * w  * x = x * w * z + y * x :=
    by simp

    example (h : p (x * y + z * w  * x)) : p (x * w * z + y * x) :=
    begin simp, simp at h, assumption end
    -- END

As with the rewriter, the simplifier behaves appropriately in algebraic structures:

.. code-block:: lean

    import algebra.ring

    variables {α : Type*} [comm_ring α]

    local attribute [simp] mul_comm mul_assoc mul_left_comm
    local attribute [simp] add_assoc add_comm add_left_comm

    example (x y z : α) : (x - x) * y + z = z :=
    begin simp end

    example (x y z w : α) : x * y + z * w  * x = x * w * z + y * x :=
    by simp

As with ``rewrite``, you can send ``simp`` a list of facts to use, including general lemmas, local hypotheses, definitions to unfold, and compound expressions. The ``simp`` tactic does not recognize the ``←t`` syntax that ``rewrite`` does, so to use an identity in the other direction you need to use ``eq.symm`` explicitly. In any case, the additional rules are added to the collection of identities that are used to simplify a term.

.. code-block:: lean

    def f (m n : ℕ) : ℕ := m + n + m

    example {m n : ℕ} (h : n = 1) (h' : 0 = m) : (f m n) = n :=
    by simp [h, h'.symm, f]

A common idiom is to simplify a goal using local hypotheses:

.. code-block:: lean

    variables (f : ℕ → ℕ) (k : ℕ)

    example (h₁ : f 0 = 0) (h₂ : k = 0) : f k = 0 :=
    by simp [h₁, h₂]

To use all the hypotheses present in the local context when simplifying, we can use the wildcard symbol, ``*``:

.. code-block:: lean

    variables (f : ℕ → ℕ) (k : ℕ)

    -- BEGIN
    example (h₁ : f 0 = 0) (h₂ : k = 0) : f k = 0 :=
    by simp *
    -- END

Here is another example:

.. code-block:: lean

    import data.nat.basic

    example (u w x y z : ℕ) (h₁ : x = y + z) (h₂ : w = u + x) :
      w = z + y + u :=
    by simp [*, add_assoc, add_comm, add_left_comm]

The simplifier will also do propositional rewriting. For example, using the hypothesis ``p``, it rewrites ``p ∧ q`` to ``q`` and ``p ∨ q`` to ``true``, which it then proves trivially. Iterating such rewrites produces nontrivial propositional reasoning.

.. code-block:: lean

    variables (p q r : Prop)

    example (hp : p) : p ∧ q ↔ q :=
    by simp *

    example (hp : p) : p ∨ q :=
    by simp *

    example (hp : p) (hq : q) : p ∧ (q ∨ r) :=
    by simp *

The next example simplifies all the hypotheses, and then uses them to prove the goal.

.. code-block:: lean

    import data.nat.basic

    variables (u w x x' y y' z : ℕ) (p : ℕ → Prop)

    example (h₁ : x + 0 = x') (h₂ : y + 0 = y') :
      x + y + 0 = x' + y' :=
    by { simp at *, simp * }

One thing that makes the simplifier especially useful is that its capabilities can grow as a library develops. For example, suppose we define a list operation that symmetrizes its input by appending its reversal:

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

Then for any list ``xs``, ``reverse (mk_symm xs)`` is equal to ``mk_symm xs``, which can easily be proved by unfolding the definition:

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

    -- BEGIN
    theorem reverse_mk_symm (xs : list α) :
      reverse (mk_symm xs) = mk_symm xs :=
    by { unfold mk_symm, simp }
    -- END

Or even more simply,

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

    -- BEGIN
    theorem reverse_mk_symm (xs : list α) :
      reverse (mk_symm xs) = mk_symm xs :=
    by simp [mk_symm]
    -- END

We can now use this theorem to prove new results:

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

    theorem reverse_mk_symm (xs : list α) :
      reverse (mk_symm xs) = mk_symm xs :=
    by simp [mk_symm]

    -- BEGIN
    example (xs ys : list ℕ) :
      reverse (xs ++ mk_symm ys) = mk_symm ys ++ reverse xs :=
    by simp [reverse_mk_symm]

    example (xs ys : list ℕ) (p : list ℕ → Prop)
        (h : p (reverse (xs ++ (mk_symm ys)))) :
      p (mk_symm ys ++ reverse xs) :=
    by simp [reverse_mk_symm] at h; assumption
    -- END

But using ``reverse_mk_symm`` is generally the right thing to do, and it would be nice if users did not have to invoke it explicitly. We can achieve that by marking it as a simplification rule when the theorem is defined:

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

    -- BEGIN
    @[simp] theorem reverse_mk_symm (xs : list α) :
      reverse (mk_symm xs) = mk_symm xs :=
    by simp [mk_symm]

    example (xs ys : list ℕ) :
      reverse (xs ++ mk_symm ys) = mk_symm ys ++ reverse xs :=
    by simp

    example (xs ys : list ℕ) (p : list ℕ → Prop)
        (h : p (reverse (xs ++ (mk_symm ys)))) :
      p (mk_symm ys ++ reverse xs) :=
    by simp at h; assumption
    -- END

The notation ``@[simp]`` declares ``reverse_mk_symm`` to have the ``[simp]`` attribute, and can be spelled out more explicitly:

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

    -- BEGIN
    attribute [simp]
    theorem reverse_mk_symm (xs : list α) :
      reverse (mk_symm xs) = mk_symm xs :=
    by simp [mk_symm]
    -- END

    example (xs ys : list ℕ) : reverse (xs ++ mk_symm ys) = mk_symm ys ++ reverse xs :=
    by simp

    example (xs ys : list ℕ) (p : list ℕ → Prop)
        (h : p (reverse (xs ++ (mk_symm ys)))) :
      p (mk_symm ys ++ reverse xs) :=
    by simp at h; assumption

The attribute can also be applied any time after the theorem is declared:

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

    -- BEGIN
    theorem reverse_mk_symm (xs : list α) :
      reverse (mk_symm xs) = mk_symm xs :=
    by simp [mk_symm]

    attribute [simp] reverse_mk_symm

    example (xs ys : list ℕ) :
      reverse (xs ++ mk_symm ys) = mk_symm ys ++ reverse xs :=
    by simp

    example (xs ys : list ℕ) (p : list ℕ → Prop)
        (h : p (reverse (xs ++ (mk_symm ys)))) :
      p (mk_symm ys ++ reverse xs) :=
    by simp at h; assumption
    -- END

Once the attribute is applied, however, there is no way to remove it; it persists in any file that imports the one where the attribute is assigned. As we will discuss further in :numref:`attributes`, one can limit the scope of an attribute to the current file or section using the ``local attribute`` command:

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

    theorem reverse_mk_symm (xs : list α) :
      reverse (mk_symm xs) = mk_symm xs :=
    by simp [mk_symm]

    -- BEGIN
    section
    local attribute [simp] reverse_mk_symm

    example (xs ys : list ℕ) :
      reverse (xs ++ mk_symm ys) = mk_symm ys ++ reverse xs :=
    by simp

    example (xs ys : list ℕ) (p : list ℕ → Prop)
        (h : p (reverse (xs ++ (mk_symm ys)))) :
      p (mk_symm ys ++ reverse xs) :=
    by simp at h; assumption

    end
    -- END

Outside the section, the simplifier will no longer use ``reverse_mk_symm`` by default.

You can even create your own sets of simplifier rules, to be applied in special situations.

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

    theorem reverse_mk_symm (xs : list α) :
      reverse (mk_symm xs) = mk_symm xs :=
    by simp [mk_symm]

    -- BEGIN
    run_cmd mk_simp_attr `my_simps

    attribute [my_simps] reverse_mk_symm

    example (xs ys : list ℕ) :
      reverse (xs ++ mk_symm ys) = mk_symm ys ++ reverse xs :=
    by simp with my_simps

    example (xs ys : list ℕ) (p : list ℕ → Prop)
      (h : p (reverse (xs ++ (mk_symm ys)))) :
        p (mk_symm ys ++ reverse xs) :=
    by simp with my_simps at h; assumption
    -- END

The command ``run_cmd mk_simp_attr `my_simps`` creates a new attribute ``[my_simps]``. (The backtick is used to indicate that ``my_simps`` is a new name, something that is explained more fully in `Programming in Lean <https://leanprover.github.io/programming_in_lean/>`__.) The command ``simp with my_simps`` then adds all the theorems that have been marked with attribute ``[my_simps]`` to the default set of theorems marked with attribute ``[simp]`` before applying ``[simp]``, and similarly with ``simp with my_simps at h``.

Note that the various ``simp`` options we have discussed --- giving an explicit list of rules, using ``at`` to specify the location, and using ``with`` to add additional simplifier rules --- can be combined, but the order they are listed is rigid. You can see the correct order in an editor by placing the cursor on the ``simp`` identifier to see the documentation string that is associated with it.

There are two additional modifiers that are useful. By default, ``simp`` includes all theorems that have been marked with the attribute ``[simp]``. Writing ``simp only`` excludes these defaults, allowing you to use a more explicitly crafted list of rules. Alternatively, writing ``simp without t`` filters ``t`` and removes it from the set of simplification rules. In the examples below, the minus sign and ``only`` are used to block the application of ``reverse_mk_symm``.

.. code-block:: lean

    import data.list.basic

    open list

    variables {α : Type*} (x y z : α) (xs ys zs : list α)

    def mk_symm (xs : list α) := xs ++ reverse xs

    theorem reverse_mk_symm (xs : list α) :
      reverse (mk_symm xs) = mk_symm xs :=
    begin unfold mk_symm, simp end

    -- BEGIN
    attribute [simp] reverse_mk_symm

    example (xs ys : list ℕ) (p : list ℕ → Prop)
        (h : p (reverse (xs ++ (mk_symm ys)))) :
      p (mk_symm ys ++ reverse xs) :=
    by { simp at h, assumption }

    example (xs ys : list ℕ) (p : list ℕ → Prop)
        (h : p (reverse (xs ++ (mk_symm ys)))) :
      p (reverse (mk_symm ys) ++ reverse xs) :=
    by { simp [-reverse_mk_symm] at h, assumption }

    example (xs ys : list ℕ) (p : list ℕ → Prop)
        (h : p (reverse (xs ++ (mk_symm ys)))) :
      p (reverse (mk_symm ys) ++ reverse xs) :=
    by { simp only [reverse_append] at h, assumption }
    -- END

Exercises
---------

#. Go back to the exercises in :numref:`Chapter %s <propositions_and_proofs>` and :numref:`Chapter %s <quantifiers_and_equality>` and redo as many as you can now with tactic proofs, using also ``rw`` and ``simp`` as appropriate.

#. Use tactic combinators to obtain a one line proof of the following:

   .. code-block:: lean

       example (p q r : Prop) (hp : p) :
       (p ∨ q ∨ r) ∧ (q ∨ p ∨ r) ∧ (q ∨ r ∨ p) :=
       by sorry