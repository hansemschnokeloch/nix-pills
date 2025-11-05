# Les bases du langage {#basics-of-language}

Bienvenue dans le quatrième article. Dans [l'article précédent](03-enter-environment.md), nous avons étudié les environnements Nix. Nous avons installé des logiciels en tant qu'utilisateur, géré leur _profile_, basculé entre les générations et interrogé le _nix store_. Il s'agit des bases de l'administration système avec Nix.

Le [langage Nix](https://nix.dev/manual/nix/stable/language/) est utilisé pour écrire des expressions qui génèrent des dérivations.L'outil [nix-build](https://nix.dev/manual/nix/stable/command-ref/nix-build) sert à construire des dérivations à partir d'une expression. Même en tant qu'administrateur système qui souhaite personnaliser l'installation, il est nécessaire de maîtriser Nix. Utiliser Nix pour vos tâches signifie que vous bénéficiez gratuitement des fonctionnalités que nous avons vues dans les articles précédents.

La syntaxe de Nix est assez inhabituelle, examiner des exemples existants peut vous amener à penser qu'il y a beaucoup de magie en jeu. En réalité, il s'agit principalement d'écrire des fonctions utilitaires pour rendre les choses pratiques.

D'un autre côté, cette syntaxe est idéale pour décrire des paquets, donc apprendre le langage Nix sera rentable lors de l'écriture d'expressions de paquets.

<div class="info">

Important : avec Nix tout est une expression, il n'y a pas de déclarations. C'est habituel avec les langages fonctionnels.

</div>

<div class="info">

Important : les valeurs dans Nix sont immuables.

</div>

## Types de valeurs

Nix 2.0 comprend une commande nommée `nix repl`, qui est un simple outil en ligne de commande permettant d'expérimenter le langage Nix. En fait, Nix est un [langage fonctionnel pur à évaluation paresseuse](https://nix.dev/manual/nix/stable/language/), et pas uniquement un ensemble d'outils pour gérer des dérivations. La syntaxe de `nix repl` est légèrement différente de la syntaxe Nix en ce qui concerne l'affectation des variables, mais cela ne devrait pas être déroutant tant que vous en tenez compte. Je préfère commencer par `nix repl` avant de vous encombrer l'esprit avec des expressions plus complexes.

Lancez `nix repl`. Tout d'abord, Nix prend en charge les opérations arithmétiques de base : `+`, `-`, `*` et `/`. (Pour quitter `nix repl`, utilisez la commande `:q`. L'aide est disponible via la commande `:?`.)

```console
nix-repl> 1+3
4

nix-repl> 7-4
3

nix-repl> 3*2
6
```

Essayer de faire une division avec Nix peut être surprenant.


```console
nix-repl> 6/3
/home/nix/6/3
```

Que s'est-il passé ? Rappelez-vous que Nix n'est pas un langage à usage général, c'est un (langage dédié)[https://fr.wikipedia.org/wiki/Langage_d%C3%A9di%C3%A9] à l'écriture des paquets. La division entière n'a pas vraiment d'utilité pour écrire des expressions de paquets. Nix a analysé `6/3` comme un chemin relatif au répertoire courant. Pour que Nix effectue une division, ajoutez un espace après le `/`. Alternativement, vous pouvez utiliser `builtins.div`.


```console
nix-repl> 6/ 3
2

nix-repl> builtins.div 6 3
2
```

Les autres opérateurs sont `||`, `&&` et `!` pour les booléens, ainsi que des opérateurs relationnels tels que `!=`, `==`, `<`, `>`, `<=`, `>=`. Avec Nix, `<`, `>`, `<=` et `>=` sont peu utilisés. Il existe également d'autres opérateurs que nous verrons au cours de cette série.

Nix dispose de [types](https://nix.dev/manual/nix/stable/language/#overview) entiers, à virgule flottante, chaînes de caractères, chemins, booléens et null. Il y a également des listes, des ensembles (_sets_) et des fonctions. Ces types sont suffisants pour construire un système d'exploitation.

Nix est à [typage fort](https://fr.wikipedia.org/wiki/Typage_fort) mais pas à [typage statique](https://fr.wikipedia.org/wiki/Typage_statique). C'est-à-dire que vous ne pouvez pas mélanger des chaînes de caractères et des entiers, vous devez d'abord effectuer la conversion.

Comme démontré ci-dessus, les expressions seront analysées comme des chemins tant qu'il y a une barre oblique non suivie d'un espace. Par conséquent, pour spécifier le répertoire courant, utilisez `./.` De plus, Nix analyse également les URL de manière spéciale.

Toutes les URL ou tous les chemins ne peuvent pas être analysés de cette manière. En cas d'erreur de syntaxe, il est toujours possible de revenir à des chaînes de caractères simples. Les URL et les chemins littéraux sont pratiques pour une sécurité supplémentaire.

>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
## Identifier

There's not much to say here, except that dash (`-`) is allowed in identifiers. That's convenient since many packages use dash in their names. In fact:

```console
nix-repl> a-b
error: undefined variable `a-b' at (string):1:1
nix-repl> a - b
error: undefined variable `a' at (string):1:1
```

As you can see, `a-b` is parsed as identifier, not as a subtraction.

## Strings

It's important to understand the syntax for strings. When learning to read Nix expressions, you may find dollars (`$`) ambiguous, but they are very important . Strings are enclosed by double quotes (`"`), or two single quotes (`''`).

```console
nix-repl> "foo"
"foo"
nix-repl> ''foo''
"foo"
```

In other languages like Python you can also use single quotes for strings (e.g. `'foo'`), but not in Nix.

It's possible to [interpolate](https://nix.dev/manual/nix/stable/language/string-interpolation) whole Nix expressions inside strings with the `${...}` syntax and only that syntax, not `$foo` or `{$foo}` or anything else.

```console
nix-repl> foo = "strval"
nix-repl> "$foo"
"$foo"
nix-repl> "${foo}"
"strval"
nix-repl> "${2+3}"
error: cannot coerce an integer to a string, at (string):1:2
```

Note: ignore the `foo = "strval"` assignment, special syntax in `nix repl`.

As said previously, you cannot mix integers and strings. You need to explicitly include conversions. We'll see this later: function calls are another story.

Using the syntax with two single quotes is useful for writing double quotes inside strings without needing to escape them:

```console
nix-repl> ''test " test''
"test \" test"
nix-repl> ''${foo}''
"strval"
```

Escaping `${...}` within double quoted strings is done with the backslash. Within two single quotes, it's done with `''`:

```console
nix-repl> "\${foo}"
"${foo}"
nix-repl> ''test ''${foo} test''
"test ${foo} test"
```

## Lists

Lists are a sequence of expressions delimited by space (_not_ comma):

```console
nix-repl> [ 2 "foo" true (2+3) ]
[ 2 "foo" true 5 ]
```

Lists, like everything else in Nix, are immutable. Adding or removing elements from a list is possible, but will return a new list.

## Attribute sets

An attribute set is an association between string keys and Nix values. Keys can only be strings. When writing attribute sets you can also use unquoted identifiers as keys.

```console
nix-repl> s = { foo = "bar"; a-b = "baz"; "123" = "num"; }
nix-repl> s
{ "123" = "num"; a-b = "baz"; foo = "bar"; }
```

For those reading Nix expressions from nixpkgs: do not confuse attribute sets with argument sets used in functions.

To access elements in the attribute set:

```console
nix-repl> s.a-b
"baz"
nix-repl> s."123"
"num"
```

Yes, you can use strings to address keys which aren't valid identifiers.

Inside an attribute set you cannot normally refer to elements of the same attribute set:

```console
nix-repl> { a = 3; b = a+4; }
error: undefined variable `a' at (string):1:10
```

To do so, use [recursive attribute sets](https://nix.dev/manual/nix/stable/language/constructs#recursive-sets):

```console
nix-repl> rec { a = 3; b = a+4; }
{ a = 3; b = 7; }
```

This is very convenient when defining packages, which tend to be recursive attribute sets.

## If expressions

These are expressions, not statements.

```console
nix-repl> a = 3
nix-repl> b = 4
nix-repl> if a > b then "yes" else "no"
"no"
```

You can't have only the `then` branch, you must specify also the `else` branch, because an expression must have a value in all cases.

## Let expressions

This kind of expression is used to define local variables for inner expressions.

```console
nix-repl> let a = "foo"; in a
"foo"
```

The syntax is: first assign variables, then `in`, then an expression which can use the defined variables. The value of the whole `let` expression will be the value of the expression after the `in`.

```console
nix-repl> let a = 3; b = 4; in a + b
7
```

Let's write two `let` expressions, one inside the other:

```console
nix-repl> let a = 3; in let b = 4; in a + b
7
```

With `let` you cannot assign twice to the same variable. However, you can shadow outer variables:

```console
nix-repl> let a = 3; a = 8; in a
error: attribute `a' at (string):1:12 already defined at (string):1:5
nix-repl> let a = 3; in let a = 8; in a
8
```

You cannot refer to variables in a `let` expression outside of it:

```console
nix-repl> let a = (let c = 3; in c); in c
error: undefined variable `c' at (string):1:31
```

You can refer to variables in the `let` expression when assigning variables, like with recursive attribute sets:

```console
nix-repl> let a = 4; b = a + 5; in b
9
```

So beware when you want to refer to a variable from the outer scope, but it's also defined in the current let expression. The same applies to recursive attribute sets.

## With expression

This kind of expression is something you rarely see in other languages. You can think of it like a more granular version of `using` from C++, or `from module import *` from Python. You decide per-expression when to include symbols into the scope.

```console
nix-repl> longName = { a = 3; b = 4; }
nix-repl> longName.a + longName.b
7
nix-repl> with longName; a + b
7
```

That's it, it takes an attribute set and includes symbols from it in the scope of the inner expression. Of course, only valid identifiers from the keys of the set will be included. If a symbol exists in the outer scope and would also be introduced by the `with`, it will _not_ be shadowed. You can however still refer to the attribute set:

```console
nix-repl> let a = 10; in with longName; a + b
14
nix-repl> let a = 10; in with longName; longName.a + b
7
```

## Laziness

Nix evaluates expressions only when needed. This is a great feature when working with packages.

```console
nix-repl> let a = builtins.div 4 0; b = 6; in b
6
```

Since `a` is not needed, there's no error about division by zero, because the expression is not in need to be evaluated. That's why we can have all the packages defined on demand, yet have access to specific packages very quickly.

## Next pill

...we will talk about functions and imports. In this pill I've tried to avoid function calls as much as possible, otherwise the post would have been too long.
