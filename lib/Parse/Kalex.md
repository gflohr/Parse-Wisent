# NAME

Kalex - A lexical scanner generator for Perl


# SYNOPSIS

From the command-line:

    $ kalex input.l
    $ kalex --help

Or from Perl:

    $parser = Parse::Kalex->new(%options);
    $parser = Parse::Kalex->newFromArgv(\@ARGV);
    $parser->scan or exit 1;
    $parser->output or exit 1;

<!--TOC-->
# TABLE OF CONTENTS
   * [DESCRIPTION](#description)
   * [INTRODUCTION](#introduction)
   * [EXAMPLES](#examples)
      * [Basic Example](#basic-example)
      * [Counting Lines and Characters](#counting-lines-and-characters)
   * [FORMAT OF THE INPUT FILE](#format-of-the-input-file)
      * [Format of the Definitions Section](#format-of-the-definitions-section)
         * [Name Definitions](#name-definitions)
         * [Start Condition Definitions](#start-condition-definitions)
         * [Indented Text](#indented-text)
         * [%{ CODE %} Sections](#code-sections)
         * [%top Sections](#top-sections)
         * [Comments](#comments)
         * [%Option Directives](#option-directives)
            * [%option yywrap/yynowrap](#option-yywrap-yynowrap)
      * [Format of the Rules Section](#format-of-the-rules-section)
         * [Rules](#rules)
      * [Format of the User Code Section](#format-of-the-user-code-section)
      * [Comments in the Input](#comments-in-the-input)
   * [PATTERNS](#patterns)
      * [Submatches](#submatches)
      * [Interpolation](#interpolation)
      * [ANCHORS ("^" and "$")](#anchors-and)
      * [Multi-Line Patterns](#multi-line-patterns)
      * [(?x:pattern)](#x-pattern)
      * [(?n:pattern)](#n-pattern)
   * [HOW THE INPUT IS MATCHED](#how-the-input-is-matched)
      * [How It Really Works](#how-it-really-works)
         * [Captures](#captures)
         * [Backreferences](#backreferences)
      * [Performance Considerations](#performance-considerations)
      * [Alternations](#alternations)
   * [ACTIONS](#actions)
      * [ECHO](#echo)
      * [YYBEGIN](#yybegin)
      * [YYPUSH](#yypush)
      * [YYPOP](#yypop)
      * [REJECT](#reject)
      * [yymore()](#yymore)
      * [yyless()](#yyless)
      * [yyrecompile()](#yyrecompile)
      * [unput()/yyunput()](#unput-yyunput)
      * [input()/yyinput()](#input-yyinput)
   * [THE GENERATED SCANNER](#the-generated-scanner)
      * [Scanner File Layout](#scanner-file-layout)
      * [Namespaces](#namespaces)
      * [Variable Scopes](#variable-scopes)
      * [Strictness](#strictness)
      * [The Lexing Subroutine `yylex()`](#the-lexing-subroutine-yylex)
   * [REENTRANT SCANNERS](#reentrant-scanners)
      * [Setting the Perl Package](#setting-the-perl-package)
      * [Scanner Instance](#scanner-instance)
      * [Variable an Method Names](#variable-an-method-names)
         * [Method Names](#method-names)
         * [Variable Names](#variable-names)
         * [Defining Your Own Variables](#defining-your-own-variables)
   * [FREQUENTLY ASKED QUESTIONS](#frequently-asked-questions)
      * [Quantifier Follows Nothing In Regex](#quantifier-follows-nothing-in-regex)
      * [Unknown regexp modifier "/P" at](#unknown-regexp-modifier-p-at)
      * [Can't Use String ("...") As a Hashref](#can-t-use-string-as-a-hashref)
      * [Why Does My Debugger Not Work?](#why-does-my-debugger-not-work)
   * [DIFFERENCES TO FLEX](#differences-to-flex)
      * [Functions and Variables](#functions-and-variables)
      * [No yywrap() By Default](#no-yywrap-by-default)
      * [BEGIN is YYBEGIN](#begin-is-yybegin)
      * [YYPUSH and YYPOP](#yypush-and-yypop)
      * [The Best Match Is Not Necessarily the Longest](#the-best-match-is-not-necessarily-the-longest)
      * [Name Definitions Define Perl Variables](#name-definitions-define-perl-variables)
      * [REJECT is Less Expensive](#reject-is-less-expensive)
      * [Code Following REJECT is Allowed](#code-following-reject-is-allowed)
      * [unput() Arguments Have Arbitrary Length](#unput-arguments-have-arbitrary-length)
   * [COPYRIGHT](#copyright)
   * [SEE ALSO](#see-also)

# DESCRIPTION

Kalex is the scanning counterpart of [Kayak](Kayak.md).  It can be
used to generate lexical analyzers (also known as tokenizers or
scanners) for all kinds of parsers.

Its command-line interface `kalex` is a Perl equivalent to `flex(1)`.
This manual is on purpose structured in a similar fashion as the flex
manual so that you can easily compare features.

# INTRODUCTION

Kalex reads the given scanner description from the given input sources,
or standard input if no input sources were specified.  The description 
mainly consists of *rules* with a regular expression to match on the 
left-hand side and an optional action to execute for each match on 
the right-hand side. The action consists of arbitrary Perl code.  
The match text and possibly captured sub matches are available as 
Perl variables.

Most of this manual assumes that you create a scanner as a 
standalone Perl script `lex.yy.pl` from a scanner description
`NAME.l`.  All variables and functions in the generated scanner belong
to the `main`.  This default type of scanners is really best 
suited for one-shot solutions and quick experiments with kalex.

For any serious application you will rather create an 
object-oriented, reentrant scanner.  If you already have experience
with `lex`, `flex` or similar scanner generators for other languages,
you should probably read the section [Reentrant
Scanners](#reentrant-scanners) first.  For learning the general
concepts behind scanner generators, using the default mode is probably
simpler.

# EXAMPLES

## Basic Example

Take this input file `japh.l`:

```lex
%%
Guido                      print "just another Perl hacker";
%%
yylex;
```

Generate and run the scanner.

```sh
$ kalex japh.l
$ echo "I am Guido" | perl lex.yy.pl
I am just another Perl hacker
```

The command `kalex japh.pl` has compiled the scanner description 
`japh.l` into a Perl scanner `lex.yy.pl`.  This scanner copies its 
input verbatim to the output but replaces every occurence of the
string "Guido" to "Just another Perl hacker".

## Counting Lines and Characters

The following example is taken from the flex manual:

```lex
    my ($num_lines, $num_chars) = (0, 0);
%%
\n      ++$num_lines; ++$num_chars;
.       ++$num_chars;
%%
yylex;
print "# of lines = $num_lines, # of characters= $num_chars\n";
```

This scanner counts the number of characters and the number of lines in its
input. It produces no output other than the final report on the number of
lines and characters in the input stream.

# FORMAT OF THE INPUT FILE

The overall format of the kalex input file is:

```lex
definitions
%%
rules
%%
user code
```

All sections can be empty and the user code section is optional.  The
smallest valid input to kalex is therefore a lone `%%`.  That will
produce a scanner which copies its standard input to standard output.

## Format of the Definitions Section

In the definitions section you can define various properties and 
aspects of the scanner.

### Name Definitions

A name definition takes the following form:

```
name definition
```

`name` must be a valid Perl identifier.  Perl identifiers may start
with one of the letters "a" to "z", "A" to "Z" or the underscore "_",
followed by an arbitrary number of these characters or the digits
"0" to "9".

Non-ASCII characters are also allowed but it depends on your version
of Perl and your user code whether such identifiers are accepted
by Perl.  Try `perldoc utf8` for details.

The definition must be a valid regular expression fragment.
Whitespace inside of the fragment must either be backslash escaped or
part of a character class:

```
VARIABLE   foo\ bar[ ]baz
```

The pattern is the string "foo bar baz".  The first space character
is escaped, the second one is part of a character class.

You can reference the variable in a rule like this:

```lex
DIGIT [0-9]
CHAR [a-zA-Z]
%%
${CHAR}${DIGIT}       print "coordinate $^N\n";
.|\n
```

You can omit the curly braces if the character following the variable
name cannot be part of a valid variable name.

```lex
DIGIT [0-9]
CHAR [a-zA-Z]
%%
$CHAR$DIGIT           print "coordinate $^N\n";
.|\n
```

Using variable references, capturing parentheses, or back references
inside definitions will lead to undefined behavior of the scanner.
All of the following definitions must be avoided:

```lex
HAS_VARIABLE   Name: \$name
HAS_CAPTURE    $#([0-9+);
HAS_BACKREF    (["']).*?\1
```

Non-capturing parentheses (that are parentheses followed by a
question mark "?") are allowed:

```
TAG          <[a-z]+(?: [a-z]+=".*?"])>
```

Comments (`/* ... */`) after the definition are allowed and are
discarded.  They are *not* copied to the generated scanner.  Note
that they will possibly confuse syntax highlighters because comments
are not allowed after name definitions for flex and lex.

### Start Condition Definitions

In the definitions section, you can declare an arbitrary number
of start conditions in one of the following forms:

```lex
%s COND1 COND2
%x XCOND1 XCOND2
```

The form `%s` declares an *inclusive* start condition, the form
`%x` declares an *exclusive* start condition.  See the section
[Start Conditions](#start-conditions) below for more information
on start conditions.

The same restrictions for possible names apply as for [Name
Definitions](#name-definitions) above.

You can place comments after start conditions.

### Indented Text

All indented text in the definitions section is copied verbatim to
the generated scanner.  If you generate a [reentrant
scanner](#reentrant-scanners), the
text is inserted right after the `package` definition in the generated
code.

### %{ CODE %} Sections

All text enclosed in `%{ ...%}` is also copied to the output without
the enclosing delimiters, but the enclosed text must be valid Perl
code.

### %top Sections

A `%top` section can contain arbitrary Perl code:

```lex
%top {
    use strict;

    my $foo = 1;
    my $bar = 2;
}
%%
RULE...
```

The enclosed code will be placed at the top of the file, outside
of a possible `package` statement for reentrant parsers.

Multiple `%top` sections are allowed.  Their order is preserved.

### Comments

C-style comments (`/* ... */`) must start in the first column of a line.
There are converted to a Perl comment and copied to the output.

C-style comments that do *not* start in the first column are treated as
[indented text](#indented-text) and are copied verbatim to the output,
where they will almost inevitably cause a syntax error.  Use Perl
style comments in indented text!

### %Option Directives

Options are defined with the `%option` directive

```lex
%option noyywrap outfile="parse.pl"
```

Boolean options can be preceded by "no" to negate them.  Options
that take a value receive the value in a single- or double-quoted
string.  Escape sequences like `\n` are only expanded in
double-quoted strings. (FIXME! Not implemented!)

The following options are supported;

#### %option yywrap/yynowrap

Activates or deactivates the yywrap mechanism.  See
[The yywrap() Function](#the-yywrap-function) below.  The
default is false.

## Format of the Rules Section

### Rules

The rules section consists of an arbitrary number of rules defined
as:

```lex
<SC1,SC2,SC3>pattern action
```

The first part of the rule is an optional comma-separated list of 
start conditions enclosed in angle brackets.  If present, the
rule is only active in one of the listed start conditions.

See [Start Conditions](#start-conditions) below for more information
on start conditions.

The pattern can be almost any Perl regular expression.  See
[Patterns](#patterns) below for more information.

The third optional part is an action.  In any of the following
two forms:

```lex
$(DIGIT)+\.($DIGIT)    {
                           return FLOAT => "$_[1].$_[2]";
                       }
\n                     return NEWLINE => "\n";
```

Instead of `{ ... }` you can also use `%{ ... %}`.

See [Actions](#actions) below for more information on actions.

Since start conditions and actions are optional, a rule can also
consist of a pattern only.

## Format of the User Code Section

The user code section is copied verbatim to the generated scanner.

If the scanner is not reentrant, it will be preceded by

```perl
package main;

no strict;
```

That means that you should put a `use strict;` at the beginning
of your user code section if you want to enable strictness.

## Comments in the Input

Kalex supports C style comments, that is everything inside `/* ... */` usually
gets copied verbatim to the output but is converted to a Perl style comment:

```C
/* Lexical scanner for the XYZ language. */
```

That C style comment becomes:

```Perl
# Lexical scanner for the XYZ language.
```

Kalex should accept comments everywhere flex accepts comments.  If not,
please report it as a bug.  Notable differences to flex are:

* Comments are allowed after start condition declarations.
* Comments are allowed after [name definitions](#name-definitions).

These comments are, however, considered comments on the kalex input and are discarded in the output.

# PATTERNS

The patterns used in the [rules section](#format-of-the-rules-section) are
just regular expressions.  And since Kalex is written in Perl, it is no
wonder that they are *Perl* regular expressions.  You can learn everything 
you want to know about regular expressions with the command `perldoc perlre`
or online at https://perldoc.perl.org/perlre.html.

There are two notable differences to Perl that are both owed to the fact that
the Kalex input is not a Perl program but a description that produces a Perl
program.

## Submatches

The variables `$1, $2, $3, ... $n` that hold captured submatches should not
be used.  They are present but will most probably not contain what you
expect.  The same applies to the magical arrays `@-` and `@+`.

Instead of `$1, $2, $3, ... $n` you can use `$_[1], $_[2], $_[3], ... $_[n]`
in actions.

You can, however, use back references without problems, for example:

```lex
("|').*?\1
```

Even if `$1` will not hold a single or double quote in the above example,
you can refer to it with `\1'.  Actually, the real regular expression is
modified a little bit before being passed to Perl, and the back references
are automatically fixed up to point to the correct submatch.

## Interpolation

In Perl programs, regular expressions are subject to variable interpolation.
For most practical purposes, you can achieve the same effect with [name
definitions](#name-definitions).  You can still interpolate other variables
or even code with `@{[...]}`  but the behavior will most probably look
arbitrary to you.

It is not really arbitrary.  In fact, variable interpolations and code
will be evaluated in the context of a method in the package 
`Parse::Kalex::Lexer` or whatever other package you have specified on the
command-line with the option `--lexer` but you should not rely on that
because this implementation detail may change in the future.

Specifically, keep in mind that the following does *not* work:

```
%%
    my $digit = '[0-9]';
$digit+\.$digit+            return FLOAT, $yytext;
%%
```

The variable `$digit` is lexically scoped to the routine `yylex()` but the
regular expression is compiled in another scope where there is no
variable `$digit` defined.

On the other hand, this will work as expected:

```
DIGIT [0-9]
%%
$DIGIT+\.$DIGIT+            return FLOAT, $yytext;
%%
```

## ANCHORS ("^" and "$")

All kalex patterns are compiled with the `/m` modifier.  That means that
`^` stands for the beginning of a line or the beginning of input, and
`$` stands for the end of a line or the end of input.  See the section
[How the Input Is Matched](#how-the-input-is-matched) for more information.

## Multi-Line Patterns

If the pattern begins with a tilde `~` the following input is treated as a
multi-line pattern.  Example:

```lex
%%
~{
    [1-9][0-9]+       # the part before the decimal point
    \(?:    
    \.                # the decimal point
    [0-9]+            # the fractional part.
    )?                # the fractional part is optional.
}gsx                  ECHO
```

The tilde has the same effect as if Perl had seen the matching operator
`m` in Perl code.
The first character after the tilde `~` is the delimiter, in this case an
opening curly brace.  All nesting delimiters - that are curly braces, 
square brackets, angle bracktes, and parentheses - can be nested.

After the trailing delimiter, you can add all modifiers that Perl support.

See `perldoc perlre` for more information.  Just imagine that instead of 
`~PATTERN` you would have written `$variable =~ mPATTERN`.

## (?x:pattern)

The `x` flag is currently not recognized by kalex.  If you use it inside
a regular [pattern](#pattern), the pattern will still end at the first
whitespace character that is not escaped or outside of a character class.

Inside [multi-line patterns](#multi-line-patterns) it works as expected.

## (?n:pattern)

The `n` modifier introduced in Perl 5.22 is currently not recognized.
It will mess up the match counting and fixup of backreferences.  Do not
use it!

# HOW THE INPUT IS MATCHED

The generated scanner matches its input against the patterns provided in
the rules section, that are valid for the current [start
condition](#start-conditions), stopping at the first match.  The
starting position of the match `pos()` is set to where the last
match left off.

If a rule matches but the match is empty, you will create  an endless 
loop unless you change the start condition in the action code or return.
Currently, there is no warning about empty matches.

The matched text is availabe in the variable `$yytext` (resp.
`$_[0]->{yytext}`) or in the variable `$^N`.  The difference is that
`$^N` will only contain the matched text for the current rule while 
`$yytext` may contain prefixed text resulting froma a preceding
invocation of [`yymore()`](#yymore).

Then the [action](#actions) for the matching rule is executed.  Remember
that there is always a default rule appended to the user supplied rules:

```lex
.|\n    ECHO
```

Because of that, the smallest valid scanner description looks like this:

```lex
%%
```

The [definitions](#format-of-the-definitions-section) and [rules 
section](#format-of-the-definitions-section) are empty, and the [user 
code section](#format-of-the-user-code-section) is missing in this case.
The generated scanner will therefore copy its entire input to the
output.

## How It Really Works

The above description comes close to the actual behavior but is actually
not true.  Take the following scanner definition as an example:

```lex
%%
[ a-zA-Z]+                     ECHO;
.|\n                              /* discard */
```

Kalex will translate that into a regular expression which will roughly look
like this in Perl:

```perl
qr{\G([^a-zA-Z0-9 ])(?:{$r = 0})|(.|\n)(?:{$r = 1})|(.|\n)(?:{$r = 2})}
```

It creates a long regular expression with alternations, where each 
alternation corresponds to a rule.  After each alternation, it inserts
a little code snippet that is needed for finding out which rule had
matched.  The code is actually not `$r = N` but rather reads 
`$self->{__yymatch} = [ ... ]` where the elipsis stands for data that
helps doing the rest of the job faster.

If you are using [start conditions](#start-conditions), then such a
regular expression is generated for each of them.  They differ in the
combination of active rules for each start condition.

### Captures

You are allowed to capture submatches with parentheses.  Kalex keeps
track of them so that it can provide you the submatches in the variables
`$_[1]`, `$_[2]`, ..., no matter at which position in the input file
the rule appears.

Caveat: The relatively new `/n` modifier which prevents the grouping
metacharacters `()` from matching is currently ignored.  Do not use it!

### Backreferences

Likewise, backreferences (`\1, \2, ... \n`) are also modified in the
regular expression before it is being compiled to point at the correct
submatch.

## Performance Considerations

Optimizing your scanner usually boils down to two simple rules:

1) Rules that often match should preferably appear early in the input.
2) Longer matching regexes are faster than regexes with short matches.

Rule 1 is often hard to follow and can introduce bugs if you are not 
careful enough.

Example for rule 2: You want to create a scanner that strips off all
HTML markups (we ignore HTML comments for simplicity):

Bad:

```lex
%s MARKUP
%%
<               YYBEGIN('MARKUP')
>               YYBEGIN('INITIAL')
<MARKUP>.|\n    /* discard */
.|\n            ECHO;
```

Good:

```lex
%%
\<.*?>         /* discard */
[^<]+          ECHO;
```

That does exactly the same as before but it matches the larget possible
chunks of data.  That means it does less matches, and the action code
gets executed less often.  The "bad" example above instead matches 
one character at a time.

The last rule of the "bad" example is not needed because it is identical
to the default rule.

## Alternations

Keep in mind that every rule in the input becomes an alternation in the
generated regular expressions:

```lex
%%
([-a-zA-Z]+)|([0-9]+)                yyprint(">>>$yytext<<<");
```

An equivalent but probably more readable description would look like this:

```lex
%%
[-a-zA-Z]+                           |
[0-9]+                               yyprint(">>>$yytext<<<");
```

Not that the first form is a real challenge for an average Perl hacker but
the second one is simply clearer.  The action `|` for the first rule 
means "same as the following".

# ACTIONS

Each rule can have an *action* which is arbitrary Perl code immediately
following the [pattern](#patterns).  Remember that whitespace outside
of character classes (`[...]`) in patterns has to be properly escaped.

If the action is empty, the matched text will be discarded.  The following
example will delete all occurences of the word bug from the input:

```lex
%%
bugs?
```

All other input is passed through because of the default rule.

The following example from the flex manual compresses multiple spaces and
tabs into a single space character, and throws away whitespace found at 
the end of a line:

```lex
%%
[ \t]+$       /* ignore this token */
[ \t]+        print ' ';
```

You do not need a trailing semi-colon in the action as it is automatically
added but it also doesn't hurt.

If the action code spans multiple lines, you have to enclose it in
curly braces `{ ... }`.  The form `%{ ... %}` is also allowed:

```lex
%%
[-+]?[0-9]+.[0-9]+  {
                        print "float: $yytext\n";
                    }
[-+]?[0-9] /* alternative: */ %{
                                  print "integer: $yytext\n";
                              %} /* end of action */
.|\n                # Throw away everything else.
```

Note how you can put C-Style comments before and after actions.
Perl style comments are treated as code and are copied verbatim
to the scanner.

An action consisting solely of a pipe symbol means "execute the
action for the following rule":

```lex
[-a-zA-Z]+        /* fall through */ |
                                       /* fall through */
[0-9]+\.[0-9]+                       |
[0-9]+                               yyprint(">>>$yytext<<<");
```

Note that you cannot put comments after the pipe symbol because it cannot
be distinguished from legitimate Perl code.  Comments before the pipe
symbol or above the line are okay.

Actions can contain arbitrary Perl code including `return` statements to
return a value to whatever routine called `yylex()`. Each time `yylex()`
is called it continues processing tokens from where it last left o  until 
it either reaches the end of input or executes a `return`.

These functions/methods are defined by the scanner:

## ECHO

Use `$_[0]->ECHO()` in a [reentrant scanner](#reentrant-scanners).

`ECHO` copies `$yytext` to the scanner's output.

## YYBEGIN

Use `$_[0]->YYBEGIN()` in a [reentrant scanner](#reentrant-scanners).

This method is the equivalent of `BEGIN` for flex scanners.  It 
has been renamed to `YYBEGIN` for kalex because `BEGIN` is a reserved
word in Perl.

`YYBEGIN('FOOBAR')` puts the scanner into the start condition `FOOBAR`
and replaces the current start condition stack with `(FOOBAR)`.

The argument to `YYBEGIN` is a string!  Calling it with an undeclared
start condition name will cause a run-time error.

The start condition `0` is the same as `'INITIAL'`.

## YYPUSH

Use `$_[0]->YYPUSH()` in a [reentrant scanner](#reentrant-scanners).

`YYPUSH('FOOBAR')` puts the scanner into the start condition `FOOBAR`
and pushes `FOOBAR` onto the start condition stack.  You can fall
back to the previous start condition with [`YYPOP`](#yypop).

The argument to `YYPUSH` is a string!  Calling it with an undeclared
start condition name will cause a run-time error.

## YYPOP

Use `$_[0]->YYPOP()` in a [reentrant scanner](#reentrant-scanners).

`YYPOP` will remove the last pushed start condition from the start
condition stack and put the scanner back into the condition it was
before the last call to [`YYPUSH`](#yypush).

Calling `YYPOP()` if the start condition stack has only one element,
will cause a run-time error.

## REJECT

Use `$_[0]->REJECT()` in a [reentrant scanner](#reentrant-scanners).

`REJECT` pushes back the last matched text onto the input and matches
again, but skipping the rule that matched last.  So to say, it
picks the second best rule.

Example from the flex documentation:

```lex
    my $word_count = 0
%%
frob        special(); REJECT;
[^ \t\n]    ++$word_count;
```

This scanner calls the function `special()` whenever a word starts with
"frob".  The call to `REJECT` ensures that it is also counted as a
word.

Calling `REJECT` more than once in one action is an error and leads to an
undefined scanner behavior.  However, multiple uses of `REJECT` in
different rules are allowed, and `REJECT` will then skip the current
rule for the next match, and all rules rejected immediately before.  See
this example from the flex documentation:

```lex
%%
abcd         |
abc          |
ab           |
a            ECHO; REJECT;
.|\n         /* eat up any unmatched character */
%%
```
This scanner prints out "abcdabcaba" for all occurences of "abcd" in the
output.  It first matches "abcd", prints it out, and then repeats the
matching but this time with rule 1 omitted.  The second best rule is then
for "abc", and the same happens.  The next best rules are then "ab", and
"a", until the fifth time only the last rule matches that discards the
input and implicitely resets the rejected rule set to empty, so that the
next occurrence of "abcd" will start the procedure over.

Using `REJECT` in flex scanners is somewhat frowned upon because it slows
down the entire scanner.  Kalex scanners work differently and you suffer
from only a mostly negligible performance penalty.

## yymore()

Use `$_[0]->yymore` in a [reentrant scanner](#reentrant-scanners).

Normally, the variable `$yytext` gets overwritten after each match.
Calling `yymore()` from an action has the effect that the matched text
will be appended to `$yytext` instead for the next match.

The following example will extract the contents of double-quoted strings
and allows backslash escaping for arbitrary characters.

```lex
%s QUOTE
%%
<QUOTE>[^\\"]+     yymore;
<QUOTE>\\(.)       substr $yytext, -2, 2, $_[1]; yymore;
<QUOTE>"           {
                       YYBEGIN('INITIAL');
                       chop $yytext;
                       yyprint "$yytext\n";
                   }
"                  YYBEGIN('QUOTE');
.|\n
```

Every double quote makes the scanner enter the start condition `QUOTE`.
Inside `QUOTE` all sequences of characters other than a backslash or
double-quote are matched.  But the call to `yymore` prevents `$yytext`
to be overwritten.  Instead, the next match is appended.

If a backslash is encountered, the backslash is replaced with the 
escaped character, and `$yytext` is shortened by one character.  Again,
the call to `yymore()` prevents `$yytext` from being overwritten in the
next match.

Finally, if an unescaped quote is encountered, it is `chomp`ed off of
`$yytext` and the contents of `$yytext` is copied to the output with
`yyprint`.

Extracting quote-like constructs in this manner is maybe more 
straightforward than the well-known Friedl-style regex for the same 
purpose because you extract and unescape simultaneously.

## yyless()

Use `$_[0]->yyless` in a [reentrant scanner](#reentrant-scanners).

The function `yyless(n)` causes kalex to start the next match at
the nth character of `$yytext`

```lex
foobarbaz   ECHO; yyless(3); 
[a-z]       ECHO;
```

When the above scanner sees the string "foobarbaz" in the input,
it first copies it to the output, then moves [`$yypos`](#yypos)
back 6 characters (9 characters length of [`$yytext`](#yypos)
minus 3).

A call of `yyless(0)` causes the entire match to be pushed back to
the input.  This will result in an endless loop until you have changed
the start condition or other matching parameters.

There is no bounds checking for `n`.  If it is greater than the length
of the current match, you continue matching *before* the last match.
Likewise, a negative value of `n` will skip parts of the input
altogether.

Note that [`$yytext`](#yytext) gets updated accordingly.

## yyrecompile()

Use `$_[0]->yyrecompile` in a [reentrant scanner](#reentrant-scanners).

[Name definitions](#name-definitions) are Perl variables (scalars) of the 
same name that are lexically scoped to the lexing function [`yylex()`](#yylex):

```lex
START_TAG <
END_TAG   >
%%
${START_TAG}([a-z]+)${END_TAG}
```

The function `yylex() will contain code similar to this:

```perl
sub yylex() {
    my $START_TAG = '<';
    my $END_TAG = '>';

    while (1) {
        # Match the input ...
    }
}
```

You are free to assign to these variables in [actions](#actions), but 
that will not change the regular expressions matched against.  You have 
to call `recompile()` in order to signal that change to kalex.

A real-world example for that is the [Template
Toolkit](http://www.template-toolkit.org/), a very popular template engine
for Perl.  The Template Toolkit processes everything within the 
tags `[% ... %]` as template code.

But how can you write about templates in a template? The solution used
by the template toolkit is very handy:

```html
<h1>[% title %]</h1>

A condition can be written like this:

[% TAGS [- -] %]
<code>
[% IF condition %]
html code ....
[% END %]
</code>
[- TAGS [% %] -]
```

The template directive `TAGS` changes the opening and closing delimiters
to arbitrary strings.

Switching between mini scanners with start conditions does not help here
because the new delimiters are arbitrary, user-supplied strings.  What
you need, is a *self-modifying* scannner;

```lex
WS [ \t]
NWS [^ \t]
OT \[\% 
CT \%\]
%%
${OT}${WS}*TAGS${WS}+($NWS+)${WS}+($NWS+)${WS}*${CT} {
            $OT = quotemeta $_[1];
            $CT = quotemeta $_[2];
            yyrecompile;
        }
${OT}(.*?)${CT}       yyprint "Template code: $_[1]\n";
.|\n
%%
yylex;
```

The two [name definitions](#name-definitions) contain the initial
delimiters.  If the scanner sees the template directive `TAGS` in the
input stream, it takes the following two strings as the new delimiters.
The call to `yyrecompile()` will make that change known to the scanner,
so that the patterns get recompiled.

This is less expensive than it sounds.  In reality, kalex creates a 
fingerprint of the current set of name definitions and caches the
compiled ruleset for that particular fingerprint.  That means that
you pay the price for new variables only once.

In the above example, the patterns are recompiled the first time the
`TAGS` directive is used with two new delimiters.  If a certain set
of delimiters had already been used, the ruleset is simply replaced with
one from the cache.

**Important**: Do not forget the call to `quotemeta`, when a name 
definition should stand for a literal string!

## unput()/yyunput()

Use `$_[0]->yyunput()` in a [reentrant scanner](#reentrant-scanners).

A call to `unput(STRING)` will insert "STRING" into the input stream
at the current matching position.  If your input stream is a variable
(if [`$yyin`](#yyin) is a reference to a scalar) the variable the
`$yyin` points to is modified!

The following scanner will convert all Perl comments into C comments.

```lex
%%
#(.*)       unput(' */'); unput($_[1]); unput('/* ');
/\*.*?\*/   ECHO;
.
```

Note that `unput()` messes up the internal location tracking.  The next
time the input matches after `yylex` is called, the scanner sets up the
location in such a way that it is accurate again, once all unput characters
have been read.  Consequently, if the next match consists exclusively of
unput characters, both start and end point of the location should not be
taken serious.  If the match consists of unput characters and of characters
from the input source, the end point is correct, the start point is not. 

## input()/yyinput()

Use `$_[0]->yyinput()` in a [reentrant scanner](#reentrant-scanners).

The function `input()` moves the match pointer one character forward,
in other words, that character is skipped in the input stream.  You can
use `yyinput()` as an alias for `input()`.  [Reentrant 
scanners](#reentrant-scanners) *only* support the method `yyinput()` but
not `input()`.

If you pass a positive integer as an argument, the pointer is forwarded
that many characters at once.  An invalid number or 0 passed as an
argument turns the call into a no-op.

The function returns the portion of the input stream that was skipped.

The following scanner will print out all backslash-escaped characters in
the input:

```lex
%%
\\       my $c = yyinput(1); yyprint "\\$c\n";
.|\n
```

# THE GENERATED SCANNER

Kalex generates a scanner file `lex.yy.pl` resp. `lex.yy.pm` for a
[reentrant scanner](#reentrant-scanners).  You can change the name
of the output file with the command-line option `--outname`.

## Scanner File Layout

The layout of the scanner file is as follows:

```perl
# %top{ ... } sections.

# Code from definitions sections

# Boilerplate scanner code.

sub yylex {
    # More boilerplate scanner code.

    my (NAME_DEFINITIONS) = (...);

    # Leading code from action section.

     while ($yyinput =~ /\Gpattern/) {
         # Boilerplate code.

         # Action code.
     }
}

# User code section.
```

## Namespaces

All variable names (without the leading siglets `$`, `%`, `@`, or `&`),
function or method names used by Kalex start with "yy", "_yy", or
"__yy" in either lower- or uppercase.  Do not use such names!

Additionally, the function names `ECHO`, `REJECT`, `input`, and
`unput` are used by Kalex ([reentrant scanners](#reentrant-scanners)
use `yyinput/yyunput` instead of `input/unput`).

## Variable Scopes

Almost all variables, functions, and methods are scoped to the scanner
package, by default `main`.  Non-reentrant scanners also use the
package name `Parse::Kalex::Lexer`.

All [name definitions](#name-definitions) become Perl variables of the
same name that are scoped to `yylex()`.

The only other documented variables scoped to `yylex()` are `$yyself`
and, of course, `@_`.

## Strictness

By default, all code that comes from the scanner definition, is
preceded by `no strict` to disable strictness.  You can enable strictness
either globally with [`%option strict`](#option-strict) or with
the [command-line option](#command-line-interface) `--strict`.
You can also enable
strictness (or other Perl pragmas) locally by placing `use strict` inside
the code in the scanner definition file.  The section [Scanner File
Layout](#scanner-file-layout) should help you to avoid repetitions.

## The Lexing Subroutine `yylex()`

The subroutine roughly looks like this:

```perl
sub yylex {
    my ($yyself) = @_;

    my (NAME_DEFINITIONS ...);
    
    if (first_run) {
      # Initialize name definitions.
    }

    # Code preceding the first rule in the rules section.

    while ($yyinput =~ /\G(pattern1|pattern2|...)/) {
        # Setup $yyttext and more variables.

        goto YYRULEn;
YYRULE1:  action1; next;
YYRULE2:  action2; next;
...
    }
}
```

When `yylex()` is called, it enters an endless loop matching the global
input source until either the end of input is reached and no other
source is supplied via  [`yywrap`](#yywrap) or one of the actions
executes a `return` statement.

If the end of input is reached, the scanner behavior is undefined, until
you either call [`yyrestart`](#yyrestart) or just point [`$yyin`](#yyin]
to a new input source.

If `yylex()` returns from one of the actions, the scanner state does
not change and resumes where it left off, when it is called again.

If you have enabled [strictness](#strictness) in the rules section,
you can declare variables either in the code preceding the first
rule, or in any action.  They will then be declared in all following
actions.

Note that the code from the [user code
section](#format-of-the-user-code-section) is placed *after* the
routine `yylex()`.  If you want to invoke them without parentheses,
you have to declare them in one of the code sections.

# START CONDITIONS

The scanner is in one scanner state at a time, called a *start
condition*.  Rules active for a certain start condition are prefixed
with `<SC>` where `SC` stands for the name of the start condition.

```lex
<DQSTRING>[^"]*               return STRING => $yytext;
```

If the above scanner is in the start condition `DQSTRING`, it will
consume everything until the next double quote and return the text
matched.  The next rule will be active in the start conditions
`INITIAL` and `DQSTRING`:

```lex
<INITIAL,DQSTRING>\(.)
```

You declare start conditions in the [definitions
section](#format-of-the-definitions-section) using unindented lines
starting with either `%s` or `%x` followed by a space-separacted
list of condition names.  Start conditions declared with `%s` are
*inclusive* start conditions; those declared with `%x` are *exclusive*.

## Inclusive and Exclusive Start Conditions

If the current start condition is an inclusive one, all rules marked
with that start conditions and all rules which have no start condition
at all are active.  If the current start condition is exclusive, 
only the rules marked with that start condition are active, those without
a start condition are inactive.

Example:

```lex
%s MARKUP SCRIPT
%x PRE
%%
[ \t\r\n]+        yyprint " ";
<PRE>[ \t\r\n]+   ECHO;
```

The above scanner will collapse all sequences of whitespace into one
single space character, except, when in start condition `PRE`.  Note
that `PRE` is an *exclusive* start condition declared with `%x`.
If it had been declared as an *inclusive* start condition, the
first rule that collapses white space would have always been active,
even in start condition `PRE`.  Exclusive start conditions can be
used for mini-scanners that are completely independent from the rest
of the scanners.

## Switching Start Conditions

Calling `YYBEGIN(CONDITION)` switches to start condition `CONDITION`.
You can also use [`yy_push_state()`](#yy_push_state), 
[`yy_pop_state()`](#yy_pop_state), and 
[`yy_top_state()`](#yy_top_state), to manipulate a stack of start
conditions.

## INITIAL

The scanner starts in the special start condition `INITIAL` which is
present in every scanner.  The start condition `0` is a synonym for
`INITIAL`.

## The Catch-All Start Condition `*`

The catch-all start condition `<*>` stands for *every* start condition,
even exclusive one:

```lex
<*>[ \t\r\n]+       /* discard  */
```

This scanner discards all whitespace in all scanner states.  Note that
you cannot combine `*` with other start conditions (which would not
make sense anyway).

## Getting the Current Start Condition

You can get the current start condition with any of 
[`YYSTATE/$_[0]->YYSTATE`](#YYSTATE), 
[`YY_START/$_[0]->YY_START`](#YY_START), or
[`yy_top_state/$_[0]->yy_top_state`](#yy_top_state), they are all
equivalent.  Note, however, that start conditions are non-negative
integers.  

# REENTRANT SCANNERS

By default, kalex generates a scanner that is scoped to the Perl
`main` package.  Other than generally being considered a design flaw
it has the disadvantage that you cannot have more than one kalex
scanner in your Perl program.

A better approach is to encapsulate the entire scanner state in one
Perl reference.  That allows you to have as many scanner instances
as you want inside the same program.  All of these scanners are
then *reentrant*.

Object-oriented is somewhat a synonym for reentrant in this case.
But reentrant kalex scanners deliberately sacrifice best OO
practices, not only for compatibility with classical lex/flex behavior
but also for the idea of allowing you to mess around with the
internals of the scanner as long as you know what you are doing.
Calling these scanners reentrant instead of object-oriented is an
inexpensive defense against all kinds of accusations from the OO police
for these design flaws or decisions.

## Setting the Perl Package

There are two ways defining the namespace for the kalex methods
and instance variables.

You can use a [`%option` directive](#option-directives):

```lex
%option package=Smellovision::Parse::Lexer
```

The other possibility is to use the command-line option `-p` or 
`--package`: 

```shell
$ kalex --package="Smellovision::Parse::Lexer"
```

The command-line option has precedence over the `%option` directive.

## Scanner Instance

You can access the scanner instance in actions either as `$_[0]`
or as `$yyself`.  The scanner instance is a hash reference.

You can define arbitrary properties and methods of the instance.  To
avoid name conflicts, do not use any names that start with "yy",
"_yy", or "__yy" and their respective uppercase versions.  The
method names `ECHO` and `REJECT` are also already defined.

## Variable an Method Names

### Method Names

Instead of functions, you have to invoke methods of the object
`$yyself` in reentrant scanners.  The names are the same as for the 
corresponding functions.  Examples:

```perl
$yyself->ECHO;
$yyself->yyless(3);
```

### Variable Names

All variables defined in functional scanners become instance variables
for example `$yyself->{yytext}` instead of just `$yytext`.

The only variables defined in actions are `$yyself`, `@_` and all
[name definitions](#name-definitions).

### Defining Your Own Variables

You can define own variables in actions but you should avoid all names
that start with "yy", "_yy", or "__yy" and their respective uppercase
versions.

Just as with the functional interface, all actions share the same
scope.  You have to keep that in mind when using `my`, `local`, or
pragmas like `strict`, `warnings` and so on.


# FREQUENTLY ASKED QUESTIONS

## Quantifier Follows Nothing In Regex

The exact error message is mostly something like:

```
Quantifier follows nothing in regex; marked by <-- HERE in m/* <-- HERE ...
```

Most probably you have used a C style comment inside Perl code, for
example:

```lex
[^ \t]+                   yyprint " "; /* collapse whitespace */
```

That looks correct but Kalex has no (reliable) way of finding out that the
Perl code ends after the semi-colon.  If you want to place a comment after
an action, you have several choices:

```lex
[^ \t]+                   { yyprint " "; } /* collapse whitespace */
[^ \t]+                   %{ yyprint " "; %} /* collapse whitespace */
[^ \t]+                   yyprint " "; # collapse whitespace
```
All of them work.  In brief: Either enclose the Perl code in balanced
braces, or use a Perl comment.

## Unknown regexp modifier "/P" at

It is usually *reported* before [Quantifier Follows Nothing in
Regex](#quantifier-follows-nothing-in-regex) but actually appears
after it.  And it has the same reason.  You are using C-style comments
after one-line actions, see [above](#quantifier-follows-nothing-in-regex).

If you look into the generated source file, you understand the error
message.  It may look like this:

```perl
#line 6 "test.l"
YYRULE0: ECHO    /* some illegal comment; next;

#line 345 "lib/Path/To/Scanner.pm"
YYRULE3: $self->ECHO;; next;
```

The misplaced comment is misinterpreted as a pattern match, and that match
often ends at path references in the source file.

## Can't Use String ("...") As a Hashref

When creating a [reentrant scanner](#reentrant-scanners), you have to call
*methods*, not *functions* from [actions](#actions):

```lex
%%
    /* Wrong!  */
foo                      yyless(1);
    /* Right!  */
bar                      $_[0]->yyless(1)

```

## Why Does My Debugger Not Work?

At least [Devel::ptkdb](search.cpan.org/~aepage/Devel-ptkdb) has
difficulties handling source files generated by Kalex.  This is caused
by `#line` directives in the generated code.  These directives are needed
so that error messages and warnings do not point into the generated source
file but rather to their origin, normally the [scanner
description](#format-of-the-input-file).  Unfortunately, `Devel::ptkdb`
is getting confused by the directives.

You can work around the problem by using [`%option
noline`](#option-noline) or the [command-line 
option](#command-line-interface) `--noline` which suppresses these
line directives in the generated file.

# DIFFERENCES TO FLEX

## Functions and Variables

The following table gives an overview of various functions and variables
in flex and kalex.

<table>
  <thead>
    <tr>
      <th rowspan="2">flex</th>
      <th colspan="2">kalex</th>
      <th rowspan="2">Meaning</th>
    </tr>
    <tr>
      <th>non-reentrant</th>
      <th>reentrant</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><code>BEGIN</code></td>
      <td><code>YYBEGIN</code></td>
      <td><code>$_[0]->YYBEGIN</code></td>
      <td>set <a href="#start-conditions">start condition</a></td>
    </tr>
    <tr>
      <td><code>ECHO</code></td>
      <td><code>ECHO</code></td>
      <td><code>$_[0]->ECHO</code></td>
      <td>write <a href="#yytext">$yytext</a> to <a href="#yyout">$yyout</a>
    </tr>
    <tr>
      <td><code>REJECT</code></td>
      <td><code>REJECT</code></td>
      <td><code>$_[0]->REJECT</code></td>
      <td>see <a href="REJECT">REJECT</a></td>
    </tr>
    <tr>
      <td>-</td>
      <td><code>YYPOP()</code></td>
      <td><code>$_[0]->YYPOP()</code></td>
      <td>pop the last pushed <a href="#start-conditions">start condition</a> off the stack</td>
    </tr>
    <tr>
      <td>-</td>
      <td><code>YYPUSH()</code></td>
      <td><code>$_[0]->YYPUSH()</code></td>
      <td>push a <a href="#start-conditions">start condition</a> on the stack</td>
    </tr>
    <tr>
      <td><code>yy_flex_debug</code></td>
      <td><code>$yy_kalex_debug</code></td>
      <td><code>$_[0]->{yy_kalex_debug}</code></td>
      <td>enable <a href="#debugging">debugging</a></td>
    </tr>
    <tr>
      <td><code>yyleng</code></td>
      <td>-</td>
      <td>-</td>
      <td>length of the match, use <code>length $yytext</code> instead</td>
    </tr>
    <tr>
      <td><code>yylex()</code></td>
      <td><code>yylex()</code></td>
      <td><code>$_[0]->yylex()</code></td>
      <td><a href="#how-the-input-is-matched">return the next token from the input stream</a></td>
    </tr>
    <tr>
      <td><code>yymore()</code></td>
      <td><code>yymore()</code></td>
      <td><code>$_[0]->yymore()</code></td>
      <td>append the next match to the current value of <a href="#yytext"><code>$yytext</code></a>
    </tr>
    <tr>
      <td><code>-</code></td>
      <td><code>yyrecompile()</code></td>
      <td><code>$_[0]->yyrecompile()</code></td>
      <td>re-evaluate all <a href="#name-definitions">name definitions</a> and re-compile the patterns as needed</a>
    </tr>
    <tr>
      <td><code>yytext</code></td>
      <td><code>$yytext</code></td>
      <td><code>$_[0]->{yytext}</code></td>
      <td>the currently matched input, possibly prepended by the previous value if <a href="#yymore">yymore()</a> had been called.
    </tr>
  <tbody>
</table>

## No yywrap() By Default

Kalex uses yywrap() in exactly the same manner as flex but it assumes
by default that you want to scan just one input stream and does not
attempt to invoke yywrawp() unless you explicitely specify it with

```lex
%option yywrap
```

That avoids the necessity to add `%option noyywrap` to your input
files for the normal use case.

## BEGIN is YYBEGIN

Because `BEGIN` is a compile phase keyword in Perl, it is called `YYBEGIN`
resp. `$self->YYBEGIN()` in kalex.

## YYPUSH and YYPOP

Start conditions in kalex can be stacked.

This feature can sometimes provide elegant solutions.  Most of the time it
is a recipe for trouble because it is very easy to get lost.

## The Best Match Is Not Necessarily the Longest

The best match (the one that is selected) in is always the longest match.
If there is more than one rule that produces a match of that length, the
one that comes first in the input file is used.

In Kalex, the first rule that produces a match is selected.  The length
of the match does not matter.

Take the following lexer definition as an example.

```lex
%%
a                          /* discard */
a+                         ECHO;
.|\n                       /* discard */
%%%
```

If you feed the string "aaah" into a flex lexer with that definition, it will
print "aaa", a Kalex lexer will remain silent.

The Kalex lexer will pick the first rule three times because it comes first.
The second rule is effectively useless.

A flex lexer will pick the second rule once, because it produces the longer
match.

## Name Definitions Define Perl Variables

Name definitions are identical in kalex and flex but the way you use them
in patterns differ.  In kalex you use regular Perl syntax:

```lex
DIGIT [0-9]
%%
\&#${DIGIT}*;
```

You can also assign to them inside actions but then you have to call
`yyrecompile()` resp. `$lexer->yyrecompile()` from within the scanner
so that the regular expressions are updated.

## REJECT is Less Expensive

Using [`REJECT`](#reject) in only one action slows down the whole scanner
with flex, even those rules that do not call `REJECT`.  Using `REJECT`
in kalex rules only has a very small performance penalty, and you pay
the price only once per occurrence.

The price is that all patterns have to be re-compiled, with the rejected
rule, and possibly previously rejected rules omitted.  But the pattern
set for that particular combination of rejected rules is cached so that
the next `REJECT` will be almost for free.

## Code Following REJECT is Allowed

All code following `REJECT` in flex is discarded.  In kalex scanners, you
can call REJECT wherever you want, not just as the last statement of your
action.

Note however that calling REJECT multiple times within one action leads to
an undefined scanner behavior.

## unput() Arguments Have Arbitrary Length

Flex scanners expect a single character as the argument to `unput()`.
In kalex scanner actions you can unput strings of arbitrary length.

# COPYRIGHT

Copyright (C) 2018 Guido Flohr <guido.flohr@cantanea.com>,
all rights reserved.

# SEE ALSO

kalex(1), perl(1)
