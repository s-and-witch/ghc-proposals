.. sectnum::
.. highlight:: haskell

Principles for GHC
==================

This document lays out high-level principles for the evolution of GHC
and the language it compiles. In much the way that we like to write
*properties* of our functions -- instead of just writing down individual
test cases -- this document strives to write *principles* that our proposals
ought to maintain (or build towards).

Articulating these principles helps to guide future proposals: proposals
that maintain the principles in this document are more likely to be accepted,
while proposals that work against these principles are more likely to be rejected.
Note that neither direction is (at all) a guarantee: there will be excpetions
in both directions. Yet by writing down the principles, we can have an informed
discussion of the tradeoffs of accepted or rejecting an individual proposal.

These priniciples are divided into two section:

* `Language design principles <#language-design-principles>`_ (Section 2)
* `GHC stability principles <#GHC-stability-principles>`_ (Section 3)


How to update these principles
------------------------------

When making a proposal, following the `usual guidelines <https://github.com/ghc-proposals/ghc-proposals/#how-to-start-a-new-proposal>`_,
feel free to include a diff against this listing of principles. We can then
discuss the validity of the new principle(s) alongside the concrete proposal for
a change to GHC.

We urge proposers to resist the temptation to propose principles without an
accompanying concrete change to GHC. Debating principles in the abstract does
not seem a productive use of time, though it is possible to imagine exceptions.
For example, the justification for the rejection of a proposal may be distilled into
a new principle here. Accordingly, this document is not (and presumably never will be)
comprehensive: it contains the articulated prinicples guiding GHC's development
but lacks the unarticulated principles, which will be added over time.

All principles in this document are linked back to the PR that introduce them,
so that readers can learn the context in which they were written.

We use a notational convention here where each principle is given an abbreviation,
but we use its full spelling wherever possible. The full spelling is useful for readers,
given the multitude of principles in play. Including the abbreviation is useful in case
the abbreviation is used in other fora (e.g. email, GitHub, etc.) to allow the curious
to find (e.g. with Ctrl+F) the abbreviation within this page.

Language design principles
-------------------

.. _`#281`: proposals/0281-visible-forall.rst
.. _`#378`: proposals/0378-dependent-type-design.rst
.. _`#448`: proposals/0448-type-variable-scoping.rst

Syntax
~~~~~~

Syntactic Unification Principle (SUP)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _`Syntactic Unification Principle`:

**Principle:** In the absence of punning, there is
no difference between type-syntax and term-syntax.

The `Syntactic Unification Principle`_ is a *long term* goal; today, there are many, many violations of this principle. (For example,
``case`` can never appear in types, and ``Int -> Bool`` is meaningless in a term.) However, this principle
should direct future proposals not to make the situation worse, and proposals that bring us closer to
the `Syntactic Unification Principle`_ are to be valued.

*Motivation:* The `Syntactic Unification Principle`_ keeps us forward-compatible with a possible future where the
distinction between term-syntax and type-syntax is removed.

From `#378`_.

Explicit Variable Principle (EVP)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _`Explicit Variable Principle`:

**Principle:**
It is possible to write out all (specified) type arguments in every polymorphic function application,
give the type for every bound variable,
and write a type signature for every expression.
This requires the ability to bring type variables into scope.
These variables can be brought into scope without relying on matching or unification.

Examples::

  const :: a -> b -> a
  const x y = ...    -- there must be some way to name the types of x and y here
  -- using `const (x :: a) (y :: b) = ...` is not powerful enough, because it relies
  -- on matching the pattern signature with the argument type from the type signature
  const @a @b x y = ...  -- this version makes it easier to bind `a` and `b`

  data Ex = forall a. Ex a
  f (Ex x) = ...     -- there must be some way to name the type of x here
  f (Ex @a x) = ...  -- this version does it

  hr :: (forall a. a -> a -> a) -> ...
  hr = ...
  g = hr (\ x y -> ...)     -- there must be some way to name the type of x or y here
  g = hf (\ @a x y -> ...)  -- this version bind the variable

The `Explicit Variable Principle`_ does not hold today:

* Even forgetting about the "no matching" restriction, if we have ``hr2 :: (forall a. F a -> Int) -> ...``,
  where ``F`` is a type family,
  there is no way for a function passed to ``hr2`` to locally bind ``a``.

* With the "no matching" restriction,
  if we have ``hr3 :: (forall a. Maybe (Either (Int, Bool, a) Double) -> ...) -> ...``,
  the only way a function passed to ``hr3`` can bind ``a`` is to repeat the type of the argument to that function.

Once we have the `Explicit Variable Principle`_, there will never be a need for ``Proxy``.

*Motivation:*
As GHC supports more and more type-level programming,
the ability to write out type signatures, arguments, and annotations has become increasingly important.
With ``-XScopedTypeVariables``, GHC allows us to bring type variables into scope,
but often requires us to do so by cumbersome matching.
If we have a type ``Maybe (Either (Int, Bool, a) Double)``,
that's a lot to type just to be able to, say, bind ``a``. The EVP says we do *not* have to resort to matching, ever.

From `#448`_.

Visibility Orthogonality Principle (VOP)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _`Visibility Orthogonality Principle`:

**Principle**: Whether an argument is visible or
invisible should affect only its visibility, not other properties.

A consequence of the `Visibility Orthogonality Principle`_ is that these two programs should have the same meaning::

   f1 :: forall a -> ...
   f1 blah1 = ...

   g1 = ... f1 blah2 ...

   -------

   f2 :: forall a. ...
   f2 @(blah1) = ...

   g2 = ... f2 @(blah2) ...

The only difference between these is the visibility.

Put another way: two programs that are the same except for visibility markers (such as
the ``.`` vs ``->`` in a ``forall`` or the presence or absence of a ``@``) should desugar
to the same Core program.

Currently, the design for `#281`_ (along with the design for ``-XTypeApplications``)
violates the `Visibility Orthogonality Principle`_, because the visibility marker ``@`` also affects the difference between
term-syntax and type-syntax. However, given the `Syntactic Unification Principle`_, we strive to uphold the `Visibility Orthogonality Principle`_ when
there is an absence of punning.

*Motivation:* Visibility should be just that: a superficial property that describes
(only) whether an argument is visible in the user-written source code.

Name resolution and scoping
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Lexical Scoping Principle (LSP)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _`Lexical Scoping Principle`:

**Principle**:

a. For every appearance of an identifier,
it is possible to determine whether that appearance is a *binding site* or an *occurrence* without examining the context.

b. For every *occurrence* of an identifier,
it is possible to uniquely identify its *binding site*, without involving the type system.

The `Lexical Scoping Principle`_ is almost true today, with the following nuances:

1. Template Haskell splices may need to be run before completing name resolution (and running those splices requires type-checking them).

2. The `deprecated mechanism <https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/exts/duplicate_record_fields.html#selector-functions>`_ for disambiguating duplicate record fields violates the `Lexical Scoping Principle`_ by requiring the type system.

3. In a pattern signature,
   if we have ``f (x :: Maybe a)``,
   the ``a`` is an occurrence if ``a`` is already in scope,
   and it is a binding site otherwise.

4. In a type signature, any out-of-scope variable is implicitly bound.
   This is not technically a violation of this principle
   (the seemingly-unbound identifier in the type signature is always an occurrence),
   but it's worth noting here.

*Motivation:*
These principles mean that we can understand the binding structure of a program without relying on type inference,
important both for the implementation of GHC and the sanity of programmers.
Furthermore, it allows readers to identify which variables should be brought newly into scope without tracking the list of variables already in scope.
This last point becomes even more poignant if we consider the possibility of mixing the term-level and type-level namespaces (`#270`_) and need to think about clashes between type variables and imported term variables.

\(a) from `#448`_;
\(b) from `#378`_.

Explicit Binding Principle (EBP)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _`Explicit Binding Principle`:

**Principle**:
Through the right combination of extensions and/or warning flags,
it is possible for a Haskell programmer to ensure that all identifiers in a program have an explicit binding site.

Examples::

   id :: a -> a    -- the variable `a` has no explicit binding site, but we can write `forall a.` to provide one

   f :: (Bool, Bool) -> Bool
   f (x :: (b, b)) = ...   -- the variable `b` is bound to `Bool` by this
                           -- pattern signature. But either the first b is a binding
                           -- site, in violation of the Lexical Scoping Principle (a),
                           -- or there is no explicit binding site, in violation of
                           -- the Explicit Binding Principle.

*Motivation:*
The `Explicit Binding Principle`_ allows programmers to control exactly how variables come into scope.
It also prevents the possibility of typos that accidentally introduce new variables.

From `#448`_.

Contiguous Scoping Principle (CSP)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _`Contiguous Scoping Principle`:

**Principle**: The region of a program for which an identifier
is in scope is contiguous.

The `Contiguous Scoping Principle`_ is *not* respected by Haskell 2010 nor some of GHC's extensions.
Here are some places where it is violated:

1. ``do``\ -notation.
   Example: ``do (x, (f x -> Pat)) <- action; blah``.
   ``x`` is in scope in its pattern, to the right of its binding site, but then not in ``action``.
   It is in scope again in ``blah``.
   Example of potential confusion: ``f x = do x <- x; g x``.

#. List comprehensions.
   Example: ``[ (x, y) | x <- thing1, y <- thing2, condition3 ]``.
   The variable ``y`` is in scope in ``condition3`` and the ``(x, y)`` at the beginning, but nowhere else.
   Example of potential confusion:
   ``f x y = [ (x, y) | x <- y, y <- x ]``.

#. Arrow notation.
   Example: ``proc x -> do y <- task1 -< input1; task2 -< input2``.
   The variable ``x`` is in scope in ``input1`` and ``input2`` but not in ``task1`` or ``task2``.
   Example of potential confusion: ``f x = proc x -> x -< x``.
   The two ``x``\ s at the end refer to *different* variables.

#. ``-XScopedTypeVariables``.
   Example: ``f :: forall a. a -> a; x :: Int; f y = (y :: a)``.
   The type variable ``a`` is in scope in the definition of ``f`` but not in the type signature for ``x``.

#. GADT header variables.
   Example of potential confusion:
   ``data G a where MkG :: a Int -> G Bool deriving C a``.
   The ``a`` in the type of ``MkG`` is completely unrelated to the ``a`` toward the beginning and in the deriving clause.

There may be others beyond this.
The goal here is *not* to establish the `Contiguous Scoping Principle`_, but to be mindful of new violations.

*Motivation:*
The `Contiguous Scoping Principle`_ makes programs easier to read,
in that a reader can add a variable to their internal tracking of in-scope variables then remove that variable from their in-scope set just once.

From `#448`_.

Semantics
~~~~~~~~~

Predictable Erasure Principle (PEP)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. _`Predictable Erasure Principle`:

**Principle**: The programmer knows, for sure, which bits of the program will be
retained at runtime, and which will be erased.

The `Predictable Erasure Principle`_ is true today: types are erased, while terms are retained.

From `#378`_.

User experience
~~~~~~~~~~~~~~~

Opt-In Principle (OIP)
^^^^^^^^^^^^^^^^^^^^^^

.. _`Opt-In Principle`:

**Principle**: Users who do not opt into an advanced feature will
not be affected by it.

This principle is violated in various ways today: it is easy for GHC to generate error messages that refer to
advanced features even when writing simple code. In addition, the existence of advanced features likely slow
down GHC even when those features are not active. Yet this principle is important to keep in mind going forward,
as we hope not to make the current situation worse.

From `#378`_, slightly generalized.



GHC stability principles
--------------------------

The Haskell ecosystem has a built-in tension between stability and innovation.

One the one hand, breaking changes impose heavy costs:

* Users keep having to update their code
* Library authors keep having to update their code
* Updating to a later version of a library can in turn force taking on new dependencies.

These difficulties add friction, discourage adoption, and make Haskell an unpredictable investment.

On the other hand, GHC has always sought to be a laboratory for innovation.  If the stability guarantees are too onerous, we risk imposing a different set of costs:

* Volunteers may get discouraged by the hoops they have to jump through to get a change agreed.
* The language may become stuck in a local optimum, because moving to a better design would require breaking changes.
* GHC may become encrusted with cruft and workarounds that exist to preserve existing, but undesirable, behaviour.

We can't navigate this tension with simple all-or-nothing rules.  We have to take each case on its merits, aware of both sets of costs.  Rather than starting the debate from scratch each time, we can have some general rules, or principles, to guide us.  The `GHC base library proposal <https://github.com/haskellfoundation/tech-proposals/blob/main/proposals/accepted/051-ghc-base-libraries.rst>`_ made a start in this direction; this section discusses GHC itself.

Assumptions
~~~~~~~~~~~~

We assume:

* That we have adopted the  `GHC base library proposal <https://github.com/haskellfoundation/tech-proposals/blob/main/proposals/accepted/051-ghc-base-libraries.rst>`_, which establishes the ``ghc-experimental`` package.
* That we have adopted some form of `GHC Proposal #601 <https://github.com/ghc-proposals/ghc-proposals/pull/601>`_, which allows us to designate extension flags as Experimental or Stable.  (Maybe other categories too, but at least these two.)  Moreover, we have actually categorised existing extensions, at least far enough to identify the Stable ones.  This is still a work in progress.

Terminology
~~~~~~~~~~~~~~~

We define a **stable Haskell package** as follows. A stable Haskell package

* Does not use Experimental extensions.
* Does not use experimental features.  (Examples: ``INCOHERENT`` pragmas, use of the Javascript or Wasm back end, builds on non-tier-1 platforms.)
* Does not use ``-Werror`` in its default build configuration.
* Explicitly specifies a language edition (``Haskell98``, ``GHC2021``), in the source code or the build configuration.
* Depends only on stable Haskell packages.

Boot packages shipped with GHC will be designated as stable or not in the documentation:

* ``base`` is stable, despite depending on unstable packages.
* Any GHC-internal packages (e.g. ``ghc-internal``, ``ghc-prim``, ``ghc-bignum``) are not stable and may change API without notice and between minor releases.
* The ``ghc`` package is not stable. (There is a separate project to define a more stable GHC API, but that is out of scope for this document.)
* ``ghc-experimental`` is not stable, but should aim to minimise breaking changes as far as feasible, since it is user-facing.
* ``template-haskell`` is not stable, because it contains AST datatypes which need to be changed as the language evolves.

Stability (GR1)
~~~~~~~~~~~~~~~~

**General rule (GR1)**.  *A stable Haskell package that works today should continue to work in subsequent releases of GHC (using the same compiler flags of course).*

Notes and clarifications:

* The general goal of "works" includes both "compiles successfully" and "runs successfully, as fast as it did before".  That is a very demanding goal, so it has the status of a highly-sought-after aspiration rather than an unbreakable promise.

* *The base library*.  Achieving (GR1) depends crucially on the stability of the base package; but that is already under the careful management of the Core Libraries Committee.  However ``base`` does, and should, evolve, so achieving (GR1) really depends on achieving the "reinstallable ``base``" goal.  That is, you should be able to compile a package with (say) GHC 9.16, using a version of ``base`` that exposes precisely the API of the ``base`` that came with GHC 9.12.

* *Warnings*

  * ``-Werror`` is excluded because otherwise (GR1) would be broken whenever we add a warning.
  * Similarly ``-Werror=flag`` is excluded to allow warnings to fire in more cases without breaking (GR1).
  * It's fine for a stable package to use ``-Werror`` in a CI build, to help the author find warnings.  But not in the default configuration, used by others when installing the package.
  * There is no stability guarantee that a later GHC will emit the same warnings as an earlier GHC.  A notable case in point is deprecations, where a later GHC may advise authors to (say) import a function from a different module.  But in general, warnings should not be regarded as stable.

* *Language editions*.

  * A stable package should specify an explicit language edition because subsequent releases of GHC might change the default language edition.  So if the package does not pin a specific language edition, it might then fail when compiled with a later release.
  * Language editions like ``GHC202x`` should use only Stable extensions.

* (GR1) is restricted to Stable extensions, and excludes dependence on ``ghc-experimental``.  That leaves the "experimental" space open to more rapid change.

* For new language extensions, (GR1) is trivially satisfied if the change is gated behind an extension flag.

* (GR1) is certainly broken if, say, we turn a warning into an error.


* We will need to enumerate the "experimental features" mentioned above.

Exceptions (GR2)
~~~~~~~~~~~~~~~~

**General rule (GR2)**.   *We may break (GR1), but only with a compelling reason, and with careful consideration of impact.*

Compelling reasons include

* To fix a security loophole.
* To fix an outright bug in GHC.  It is possible that some code might accidentally rely on that bug; but we can't use that as a reason to grandfather the bug indefinitely.  (Imagine that 2+77 = 80.)  There is a judgement call here, about what a "bug" is.
* To fix a design flaw in the language.   The switch to simple subsumption is an example of such a change, albeit one that was not well handled at the time.

This list is not exhaustive, but the emphasis is on "compelling reason", bearing in mind the costs that any change imposes on users.

Fixing an outright bug, or correcting (what we now consider to be) a design flaw in the language,counts as "compelling" reasons, even though they threaten stability.

* We want Haskell to be a beautiful, elegant language, not one riddled with inconsistencies, grandfathered in simply because fixing the inconsistency would risk some breakage.
* Similarly, we acknowledge the possibility of making major changes, if we truly believe that they will make the language better, and/or open the door to other improvements.
* Yet we should not make such changes lightly, because they impose real costs.
* Choices should be informed by the amount of breakage, e.g. by compiling thousands of packages in Hackage.
* Changes, especially significant changes, should be introduced gradually: the subject of (GR3).

Deprecations (GR3)
~~~~~~~~~~~~~~~~~~~~

**General rule (GR3)**.  *If we break (GR1), for a compelling reason (GR2), we should whenever possible provide a deprecation cycle, as well as copious publicity, so that users are alerted (via depreciation warnings) to the upcoming change, have time to adapt, and can raise concerns.*

A deprecation cycle should give at least one GHC major release for authors to make changes.  In th case of changes that are more difficult to accommodate, we may consider a second cycle.

Note that (GR3) is not a reason to say that (GR1) is unimportant.  A deprecation cycle defers costs; it does not reduce or eliminate them.
However, deprecation cycles are important.  They give users and library authors time to adapt to changes, without a mad panic.

Even changes to Experimental extensions should seek to follow (GR3), but with a significantly lower "bar".

Mechanisms for checking stability
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Having crisper definitions opens up the possibility of also providing mechanical support, through which a user can declare their intent to use only stable packages.  This is the subject
of `GHC Proposal #617 <https://github.com/ghc-proposals/ghc-proposals/pull/617>`_.
