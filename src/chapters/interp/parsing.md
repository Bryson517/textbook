# Parsing

You *could* code your own lexer and parser from scratch.  But many
languages include tools for automatically generating lexers and parsers
from formal descriptions of the syntax of a language.  The ancestors of
many of those tools are [lex][lex] and [yacc][yacc], which generate
lexers and parsers, respectively; lex and yacc were developed in the
1970s for C.

As part of the standard distribution, OCaml provides lexer and parser
generators named [ocamllex and ocamlyacc][ocamllexyacc]. There is a more
modern parser generator named [menhir][menhir] available through opam;
menhir is "90% compatible" with ocamlyacc and provides significantly
improved support for debugging generated parsers.

[lex]: https://en.wikipedia.org/wiki/Lex_(software)
[yacc]: https://en.wikipedia.org/wiki/Yacc
[ocamllexyacc]: https://ocaml.org/manual/lexyacc.html
[menhir]: http://gallium.inria.fr/~fpottier/menhir/

## Lexers

Lexer generators such as lex and ocamllex are built on the theory of
deterministic finite automata, which is typically covered in a discrete math or
theory of computation course. Such automata accept *regular languages*, which
can be described with *regular expressions*. So, the input to a lexer generator
is a collection of regular expressions that describe the tokens of the language.
The output is an automaton implemented in a high-level language, such as C (for
lex) or OCaml (for ocamllex).

That automaton itself takes files (or strings) as input, and each character of
the file becomes an input to the automaton. Eventually the automaton either
*recognizes* the sequence of characters it has received as a valid token in the
language, in which case the automaton produces an output of that token and
resets itself to being recognizing the next token, or *rejects* the sequence of
characters as an invalid token.

## Parsers

Parser generators such as yacc and menhir are similarly built on the theory of
automata. But they use *pushdown automata*, which are like finite automata that
also maintain a stack onto which they can push and pop symbols. The stack
enables them to accept a bigger class of languages, which are known as
*context-free languages* (CFLs). One of the big improvements of CFLs over
regular languages is that CFLs can express the idea that delimiters must be
balanced&mdash;for example, that every opening parenthesis must be balanced by a
closing parenthesis.

Just as regular languages can be expressed with a special notation (regular
expressions), so can CFLs. *Context-free grammars* are used to describe CFLs. A
context-free grammar is a set of *production rules* that describe how one symbol
can be replaced by other symbols. For example, the language of balanced
parentheses, which includes strings such as `(())` and `()()` and `(()())`, but
not strings such as `)` or `(()`, is generated by these rules:

* $S \rightarrow (S)$
* $S \rightarrow SS$
* $S \rightarrow \epsilon$

The symbols occurring in those rules are $S$, $($, and $)$. The $\epsilon$
denotes the empty string. Every symbol is either a *nonterminal* or a
*terminal*, depending on whether it is a token of the language being described.
$S$ is a nonterminal in the example above, and ( and ) are terminals.

In the next section we'll study *Backus-Naur Form* (BNF), which is a standard
notation for context-free grammars. The input to a parser generator is typically
a BNF description of the language's syntax. The output of the parser generator
is a program that recognizes the language of the grammar. As input, that program
expects the output of the lexer. As output, the program produces a value of the
AST type that represents the string that was accepted. The programs output by
the parser generator and lexer generator are thus dependent upon on another and
upon the AST type.

## Backus-Naur Form

{{ video_embed | replace("%%VID%%", "NQacOvZbbX4")}}

The standard way to describe the syntax of a language is with a mathematical
notation called *Backus-Naur form* (BNF), named for its inventors, John Backus
and Peter Naur. There are many variants of BNF. Here, we won't be too picky
about adhering to one variant or another. Our goal is just to have a reasonably
good notation for describing language syntax.

BNF uses a set of *derivation rules* to describe the syntax of a language. Let's
start with an example. Here's the BNF description of a tiny language of
expressions that include just the integers and addition:

```text
e ::= i | e + e
i ::= <integers>
```

These rules say that an expression `e` is either an integer `i`, or two
expressions with the symbol `+` appearing between them. The syntax of "integers"
is left unspecified by these rules.

Each rule has the form

```text
metavariable ::= symbols | ... | symbols
```

A *metavariable* is variable used in the BNF rules, rather than a variable in
the language being described. The `::=` and `|` that appear in the rules are
*metasyntax*: BNF syntax used to describe the language's syntax. *Symbols* are
sequences that can include metavariables (such as `i` and `e`) as well as tokens
of the language (such as `+`). Whitespace is not relevant in these rules.

Sometimes we might want to easily refer to individual occurrences of
metavariables. We do that by appending some distinguishing mark to the
metavariable(s). For example, we could rewrite the first rule above as

```text
e ::= i | e1 + e2
```

or as

```text
e ::= i | e + e'
```

Now we can talk about `e2` or `e'` rather than having to say "the `e` on the
right-hand side of `+`".

If the language itself contains either of the tokens `::=` or `|`&mdash;and
OCaml does contain the latter&mdash;then writing BNF can become a little
confusing. Some BNF notations attempt to deal with that by using additional
delimiters to distinguish syntax from metasyntax. We will be more relaxed and
assume that the reader can distinguish them.

## Example: SimPL

As a running example, we'll use a very simple programming language that we call
SimPL. Here is its syntax in BNF:

```text
e ::= x | i | b | e1 bop e2
    | if e1 then e2 else e3
    | let x = e1 in e2

bop ::= + | * | <=

x ::= <identifiers>

i ::= <integers>

b ::= true | false
```

Obviously there's a lot missing from this language, especially functions. But
there's enough in it for us to study the important concepts of interpreters
without getting too distracted by lots of language features. Later, we will
consider a larger fragment of OCaml.

We're going to develop a complete interpreter for SimPL. You can download the
finished interpreter here: {{ code_link | replace("%%NAME%%", "simpl.zip") }}.
Or, just follow along as we build each piece of it.

### The AST

{{ video_embed | replace("%%VID%%", "duTIBuK_fdw")}}

Since the AST is the most important data structure in an interpreter, let's
design it first. We'll put this code in a file named `ast.ml`:

```ocaml
type bop =
  | Add
  | Mult
  | Leq

type expr =
  | Var of string
  | Int of int
  | Bool of bool
  | Binop of bop * expr * expr
  | Let of string * expr * expr
  | If of expr * expr * expr
```

There is one constructor for each of the syntactic forms of expressions in the
BNF. For the underlying primitive syntactic classes of identifiers, integers,
and booleans, we're using OCaml's own `string`, `int`, and `bool` types.

Instead of defining the `bop` type and a single `Binop` constructor, we could
have defined three separate constructors for the three binary operators:

```ocaml
type expr =
  ...
  | Add of expr * expr
  | Mult of expr * expr
  | Leq of expr * expr
  ...
```

But by factoring out the `bop` type we will be able to avoid a lot of code
duplication later in our implementation.

### The Menhir Parser

{{ video_embed | replace("%%VID%%", "-BBbgVhj66s")}}

Let's start with parsing, then return to lexing later. We'll put all the Menhir
code we write below in a file named `parser.mly`. The `.mly` extension indicates
that this file is intended as input to Menhir. (The 'y' alludes to yacc.) This
file contains the *grammar definition* for the language we want to parse. The
syntax of grammar definitions is described by example below. Be warned that it's
maybe a little weird, but that's because it's based on tools (like yacc) that
were developed quite awhile ago. Menhir will process that file and produce a
file named `parser.ml` as output; it contains an OCaml program that parses the
language. (There's nothing special about the name `parser` here; it's just
descriptive.)

There are four parts to a grammar definition: header, declarations, rules, and
trailer.

**Header.** The *header* appears between `%{` and `%}`. It is code that will be
copied literally into the generated `parser.ml`. Here we use it just to open the
`Ast` module so that, later on in the grammar definition, we can write
expressions like `Int i` instead of `Ast.Int i`. If we wanted we could also
define some OCaml functions in the header.

```text
%{
open Ast
%}
```

**Declarations.** The *declarations* section begins by saying what the lexical
*tokens* of the language are. Here are the token declarations for SimPL:

```text
%token <int> INT
%token <string> ID
%token TRUE
%token FALSE
%token LEQ
%token TIMES
%token PLUS
%token LPAREN
%token RPAREN
%token LET
%token EQUALS
%token IN
%token IF
%token THEN
%token ELSE
%token EOF
```

Each of these is just a descriptive name for the token. Nothing so far says that
`LPAREN` really corresponds to `(`, for example. We'll take care of that when we
define the lexer.

The `EOF` token is a special *end-of-file* token that the lexer will return when
it comes to the end of the character stream. At that point we know the complete
program has been read.

The tokens that have a `<type>` annotation appearing in them are declaring that
they will carry some additional data along with them. In the case of `INT`,
that's an OCaml `int`. In the case of `ID`, that's an OCaml `string`.

After declaring the tokens, we have to provide some additional information about
*precedence* and *associativity*. The following declarations say that `PLUS` is
left associative, `IN` is not associative, and `PLUS` has higher precedence than
`IN` (because `PLUS` appears on a line after `IN`).

```text
%nonassoc IN
%nonassoc ELSE
%left LEQ
%left PLUS
%left TIMES
```

Because `PLUS` is left associative, `1 + 2 + 3` will parse as `(1 + 2) + 3` and
not as `1 + (2 + 3)`. Because `PLUS` has higher precedence than `IN`, the
expression `let x = 1 in x + 2` will parse as `let x = 1 in (x + 2)` and not as
`(let x = 1 in x) + 2`. The other declarations have similar effects.

Getting the precedence and associativity declarations correct is one of the
trickier parts of developing a grammar definition. It helps to develop the
grammar definition incrementally, adding just a couple tokens (and their
associated rules, discussed below) at a time to the language. Menhir will let
you know when you've added a token (and rule) for which it is confused about
what you intend the precedence and associativity should be. Then you can add
declarations and test to make sure you've got them right.

After declaring associativity and precedence, we need to declare what the
starting point is for parsing the language. The following declaration says to
start with a rule (defined below) named `prog`. The declaration also says that
parsing a `prog` will return an OCaml value of type `Ast.expr`.

```text
%start <Ast.expr> prog
```

Finally, `%%` ends the declarations section.

```text
%%
```

**Rules.** The *rules* section contains production rules that resemble BNF,
although where in BNF we would write "::=" these rules simply write ":". The
format of a rule is

```text
name:
  | production1 { action1 }
  | production2 { action2 }
  | ...
  ;
```

The *production* is the sequence of *symbols* that the rule matches. A symbol is
either a token or the name of another rule. The *action* is the OCaml value to
return if a *match* occurs. Each production can *bind* the value carried by a
symbol and use that value in its action. This is perhaps best understood by
example, so let's dive in.

The first rule, named `prog`, has just a single production. It says that a
`prog` is an `expr` followed by `EOF`. The first part of the production,
`e=expr`, says to match an `expr` and bind the resulting value to `e`. The
action simply says to return that value `e`.

```text
prog:
  | e = expr; EOF { e }
  ;
```


The second and final rule, named `expr`, has productions for all the expressions
in SimPL.

```text
expr:
  | i = INT { Int i }
  | x = ID { Var x }
  | TRUE { Bool true }
  | FALSE { Bool false }
  | e1 = expr; LEQ; e2 = expr { Binop (Leq, e1, e2) }
  | e1 = expr; TIMES; e2 = expr { Binop (Mult, e1, e2) }
  | e1 = expr; PLUS; e2 = expr { Binop (Add, e1, e2) }
  | LET; x = ID; EQUALS; e1 = expr; IN; e2 = expr { Let (x, e1, e2) }
  | IF; e1 = expr; THEN; e2 = expr; ELSE; e3 = expr { If (e1, e2, e3) }
  | LPAREN; e=expr; RPAREN {e}
  ;
```

- The first production, `i = INT`, says to match an `INT` token, bind the
  resulting OCaml `int` value to `i`, and return AST node `Int i`.

- The second production, `x = ID`, says to match an `ID` token, bind the
  resulting OCaml `string` value to `x`, and return AST node `Var x`.

- The third and fourth productions match a `TRUE` or `FALSE` token and return
  the corresponding AST node.

- The fifth, sixth, and seventh productions handle binary operators. For
  example, `e1 = expr; PLUS; e2 = expr` says to match an `expr` followed by a
  `PLUS` token followed by another `expr`. The first `expr` is bound to `e1` and
  the second to `e2`. The AST node returned is `Binop (Add, e1, e2)`.

- The eighth production, `LET; x = ID; EQUALS; e1 = expr; IN; e2 = expr`, says
  to match a `LET` token followed by an `ID` token followed by an `EQUALS` token
  followed by an `expr` followed by an `IN` token followed by another `expr`.
  The string carried by the `ID` is bound to `x`, and the two expressions are
  bound to `e1` and `e2`. The AST node returned is `Let (x, e1, e2)`.

- The last production, `LPAREN; e = expr; RPAREN` says to match an `LPAREN`
  token followed by an `expr` followed by an `RPAREN`. The expression is bound
  to `e` and returned.

The final production might be surprising, because it was not included in the BNF
we wrote for SimPL. That BNF was intended to describe the *abstract syntax* of
the language, so it did not include the concrete details of how expressions can
be grouped with parentheses. But the grammer definition we've been writing does
have to describe the *concrete syntax*, including details like parentheses.

There can also be a *trailer* section after the rules, which like the header is
OCaml code that is copied directly into the output `parser.ml` file.

### The Ocamllex Lexer

Now let's see how the lexer generator is used. A lot of it will feel familiar
from our discussion of the parser generator. We'll put all the ocamllex code we
write below in a file named `lexer.mll`. The `.mll` extension indicates that
this file is intended as input to ocamllex. (The 'l' alludes to lexing.) This
file contains the *lexer definition* for the language we want to lex. Menhir
will process that file and produce a file named `lexer.ml` as output; it
contains an OCaml program that lexes the language. (There's nothing special
about the name `lexer` here; it's just descriptive.)

There are four parts to a lexer definition: header, identifiers, rules, and
trailer.

**Header.** The *header* appears between `{` and `}`. It is code that will
simply be copied literally into the generated `lexer.ml`.

```text
{
open Parser
}
```

Here, we've opened the `Parser` module, which is the code in `parser.ml` that
was produced by Menhir out of `parser.mly`. The reason we open it is so that we
can use the token names declared in it, e.g., `TRUE`, `LET`, and `INT`, inside
our lexer definition. Otherwise, we'd have to write `Parser.TRUE`, etc.

**Identifiers.** The next section of the lexer definition contains
*identifiers*, which are named regular expressions. These will be used in the
rules section, next.

Here are the identifiers we'll use with SimPL:

```text
let white = [' ' '\t']+
let digit = ['0'-'9']
let int = '-'? digit+
let letter = ['a'-'z' 'A'-'Z']
let id = letter+
```

The regular expressions above are for whitespace (spaces and tabs), digits (0
through 9), integers (nonempty sequences of digits, optionally preceded by a
minus sign), letters (a through z, and A through Z), and SimPL variable names
(nonempty sequences of letters) aka ids or "identifiers"&mdash;though we're now
using that word in two different senses.

FYI, these aren't exactly the same as the OCaml definitions of integers and
identifiers.

The identifiers section actually isn't required; instead of writing `white` in
the rules we could just directly write the regular expression for it. But the
identifiers help make the lexer definition more self-documenting.

**Rules.** The rules section of a lexer definition is written in a notation that
also resembles BNF. A rule has the form

```text
rule name =
  parse
  | regexp1 { action1 }
  | regexp2 { action2 }
  | ...
```
Here, `rule` and `parse` are keywords. The lexer that is generated will attempt
to match against regular expressions in the order they are listed. When a
regular expression matches, the lexer produces the token specified by its
`action`.

Here is the (only) rule for the SimPL lexer:

```text
rule read =
  parse
  | white { read lexbuf }
  | "true" { TRUE }
  | "false" { FALSE }
  | "<=" { LEQ }
  | "*" { TIMES }
  | "+" { PLUS }
  | "(" { LPAREN }
  | ")" { RPAREN }
  | "let" { LET }
  | "=" { EQUALS }
  | "in" { IN }
  | "if" { IF }
  | "then" { THEN }
  | "else" { ELSE }
  | id { ID (Lexing.lexeme lexbuf) }
  | int { INT (int_of_string (Lexing.lexeme lexbuf)) }
  | eof { EOF }
```

Most of the regular expressions and actions are self-explanatory, but a couple
are not:

* The first, `white { read lexbuf }`, means that if whitespace is matched,
  instead of returning a token the lexer should just call the `read` rule again
  and return whatever token results. In other words, whitespace will be skipped.

* The two for ids and ints use the expression `Lexing.lexeme lexbuf`. This calls
  a function `lexeme` defined in the `Lexing` module, and returns the string
  that matched the regular expression. For example, in the `id` rule, it would
  return the sequence of upper and lower case letters that form the variable
  name.

* The `eof` regular expression is a special one that matches the end of the file
  (or string) being lexed.

Note that it's important that the `id` regular expression occur nearly last in
the list. Otherwise, keywords like `true` and `if` would be lexed as variable
names rather than the `TRUE` and `IF` tokens.

### Generating the Parser and Lexer

Now that we have completed parser and lexer definitions in `parser.mly` and
`lexer.mll`, we can run Menhir and ocamllex to generate the parser and lexer
from them. Let's organize our code like this:
```text
- <some root folder>
  - dune-project
  - src
    - ast.ml
    - dune
    - lexer.mll
    - parser.mly
```

In `src/dune`, write the following:
```text
(library
 (name interp))

(menhir
 (modules parser))

(ocamllex lexer)
```

That organizes the entire `src` folder into a *library* named `Interp`. The
parser and lexer will be modules `Interp.Parser` and `Interp.Lexer` in that
library.

Run `dune build` to compile the code, thus generating the parser and lexer. If
you want to see the generated code, look in `_build/default/src/` for
`parser.ml` and `lexer.ml`.

### The Driver

Finally, we can pull together the lexer and parser to transform a string into an
AST. Put this code into a file named `src/main.ml`:

```ocaml
open Ast

let parse (s : string) : expr =
  let lexbuf = Lexing.from_string s in
  let ast = Parser.prog Lexer.read lexbuf in
  ast
```

This function takes a string `s` and uses the standard library's `Lexing` module
to create a *lexer buffer* from it. Think of that buffer as the token stream.
The function then lexes and parses the string into an AST, using `Lexer.read`
and `Parser.prog`. The function `Lexer.read` corresponds to the rule named
`read` in our lexer definition, and the function `Parser.prog` to the rule named
`prog` in our parser definition.

Note how this code runs the lexer on a string; there is a corresponding function
`from_channel` to read from a file.

We could now use `parse` interactively to parse some strings.  Start utop
and load the library declared in `src` with this command:

```console
$ dune utop src
```

Now `Interp.Main.parse` is available for use:

```ocaml
# Interp.Main.parse "let x = 3110 in x + x";;
- : Interp.Ast.expr =
Interp.Ast.Let ("x", Interp.Ast.Int 3110,
 Interp.Ast.Binop (Interp.Ast.Add, Interp.Ast.Var "x", Interp.Ast.Var "x"))
```

That completes lexing and parsing for SimPL.
