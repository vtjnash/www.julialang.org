---
layout: post
title: 'Shell escaping: 2 truths and a lie'
author: <a href="http://github.com/vtjnash/">Jameson Nash</a>
---

[Put This In Your Pipe]:  {% post_url blog/2013-04-08-put-this-in-your-pipe %}
[`Distributed`]: https://docs.julialang.org/en/v1/stdlib/Distributed/#Distributed.WorkerConfig

# What is escaping?
{:.no_toc}

This is a very common task when manipulating any sort of data. But what is it?
And how do we know when we need it?

String escaping is the mechanism for avoiding whole classes of problems when
working with data exchange, including metacharacter breakage, command injection,
and rendering errors. In the networking world, it's also an aspect of framing
and stream decoding.

Sound complicated? It shouldn't be! It just requires careful understanding of
the list of decoders between the source and destination, and working backwards.

Subsequently, I'm going to just use the term "decoder" as the broadest class of
transformation. Note that while this could mean encryption, it also applies to
any other textual transform, including parsers and compressors. They don't even
need to be lossless--they just need to be deterministic.

For example, when passing arguments through a [shell](#posix-shell) to a
target program, the user needs to take various steps to ensure the string
arrives intact, and that the shell doesn't get confused by the presence of an
unexpected meta-character.

As explained in a much earlier post titled [Put This In Your Pipe], Julia tries
to help you avoid this by avoiding passing the data through the shell.

This ensures that variables spliced into the command with `$` are not further
processed, since the complete meta-character interpretation step already
occurred while parsing the Julia code.

But sometimes, this isn't possible.

For example, when launching [`Distributed`]() jobs with `exeflags`, we know the
arguments are going to be passed to `ssh` and then to the user's shell (usually
some variant on `sh`), before getting to the command line parser of Julia on the
target machine. This reveals one strategy for dealing with these though: define
an API that accepts raw strings and encodes them internally. But what if you are
in charge of writing that code? Well, that's the situation I found myself in.

However, as I was doing research into the proper steps for handling `powershell`
(also known as `pwsh`), I found that most resources fell short of explaining the
proper and necessary steps for handling arbitrary user input. Instead they
focused only on the ability for a programmer to write arbitrary code.
While also useful to know various alternatives, they would over-complicate the
matter for implementing it mechanically.

I feel it's important to note before jumping in that there are many possible
encodings which all will yield the same output. The purpose here is generally
to pick the strategy that will yield the most reasonable output with the fewest
rules.

# Strategy
{:.no_toc}

Now let's lay out our strategy. Remember that when I use the term "decoder", it
applies to any sort of transform to the input data.

1. Start with the input text. This may be one argument, or a whole list of them.
2. Identify the list of decoders that the text will need to pass through. This
   might not be the same for each argument, although it usually is.
3. Reverse that list (for each argument).
4. For each item in that list of decoders, apply the corresponding encoder from
   the list below.
5. Add any additional arguments required at this stage of the pipeline.

Be aware that some encoders may take the whole argument list at once (consider
`tar`), while others might only take one argument, but could output multiple
results (consider `zipsplit`). Thus some pipelines are impossible.

# Table of Contents:
{:.no_toc}

* TOC will be placed here
{:toc}

# String?

To encode "string" data, it typically requires knowing exactly which parser will be used.
Many cases have very similar parsers, so some are able to handle by selecting a
conservative superset.

For the Unicode strings itself, there are numerous encodings. Common ones
include UTF-8 and UTF-16. I will not cover specific here, as libraries to handle
these are readily available.

## Out-of-band

One elegant strategy for dealing with framing is simply to pass the delimiters
out-of-band. When calling a program on a Unix machine, the list of arguments
gets passed as an actual list (`char **argv`). This let's us use the identify
transform, and essentially just ignore it--the same is not true on Windows,
which we will cover later.

## C string

[`Base.escape_string`]: https://docs.julialang.org/en/v1/base/strings/#Base.unescape_string

General encoding of Unicode strings, such as for embedding into a C program
file, is implemented in the [`Base.escape_string`] function. This is implemented
as follows:

Backslashes and quotes (`\` and `"`) are escaped with a backslash (`\\` and
`\"`). Non-printable characters are escaped either with their standard C escape
codes, `"\0"` for NUL (if unambiguous), unicode code point (`"\u"` prefix) or
hex (`"\x"` prefix).

## Julia string

[`repr`]: https://docs.julialang.org/en/v1/base/strings/#Base.repr-Tuple{Any}

Encoding Julia strings can be done by calling the [`repr`] function. This is
similar to other [Unicode string](#c-string), but also escapes the `$` character
with a backslack (`\$`).

## Julia "raw" strings

In Julia, another option to represent strings is to use the so-called "raw
string" format. The description of this is fairly lengthy, but the textual
examples tend to be very straight-forward. It is also the same algorithm we'll
see later in [CommandLineToArgv](#commandlinetoargv). The algorithm is:

All quotation marks must be escaped, and any backslashes that precede them.
Only when a sequence of backslashes precedes a quote character. Thus, 2n
backslashes followed by a quote encodes n backslashes and the end of the literal
while 2n+1 backslashes followed by a quote encodes n backslashes followed by a
quote character. Anywhere else, each n backslashes simply encodes for n
backslashes.

# HTML

[`encodeURIComponent`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent
[`HTTP.escapeuri`]: https://juliaweb.github.io/HTTP.jl/stable/public_interface/#HTTP.URIs.escapeuri
[`Node.textContent=`]: https://developer.mozilla.org/en-US/docs/Web/API/Node/textContent
[`Element.setAttribute`]: https://developer.mozilla.org/en-US/docs/Web/API/Element/setAttribute
[`HTTP.escapehtml`]: https://juliaweb.github.io/HTTP.jl/stable/public_interface/#HTTP.Strings.escapehtml

The web is such an important factor in programming, I felt this was worth
including. I've put this under one label, but it's important to realize there
are actually more than 3 distinct cases here that must be handled.

 * URLs (URIs, parameter strings): the escape character here is `%`, and the
   special characters are all _bytes_ (UTF-8) except "A-Z a-z 0-9 - _ . ! ~ * '
   ( )". Each byte that is not one of those characters is replaced by `%xx`,
   where `xx` is the hex value of the byte being escaped. Ref JavaScript
   [`encodeURIComponent`] and Julia [`HTTP.escapeuri`].
 * HTML body: the escape character here is `&`, and the special characters are
   "< > &", with replacements "&lt; &gt; &amp;", respectively. Ref JavaScript
   [`Node.textContent=`] and Julia [`HTTP.escapeuri`].
 * HTML attributes: the escape character here is also `&`, but the special
   characters are `&` and either `'` or `"` (depending on the choice of
   surrounding delimiter). The escaped replacements for the latter two are
   `&#39;` and `&quot;`. Additionally, it may need to be wrapped in single or
   double quotes to escape white space. Since this uses the same escape
   character as HTML body, it is possible to write an encoder that will be valid
   for both. Alternatively, white-space (and any other characters) may be
   escaped instead by replacing it with `&#nnnn;` or `&#xhhhh;`, where `nnnn` is
   the decimal value of the code point (character, not byte like in URLs) or
   `hhhh` is the hex value. Ref JavaScript [`Element.setAttribute`].
 * JavaScript (JSON, JSONP): This can use same handling as [C
   string](#c-string), excluding malformed characters which would typically use
   the `\xhhhh` form. As described above, is sufficient to only escape `\` and
   `"` plus any control codes (characters below 0x20).

# Base64

If you have control over some parts of the decoding pipeline, a popular strategy
is to base64-encode the input string early. This can turn many future stages
(until the decode) into the identity function, reducing the risk of accidentally
hitting a buggy decoder or forgetting to handle a stage.

As with Unicode encoding, there are many libraries and other good resources on
this, so I will not go into further detail.

# RegEx

There are many special characters in regular expressions. One strategy would be
to enumerate all of these ("[({})]+.\*?\\" etc.) and escape them with a
backslash. An alternate strategy is to wrap the verbatim string in `\Q`
(quoted) ... `\E` (end). That only leaves one character pairing to escape,
`\E`, for which the escape sequence is the replacement string is the six
character sequence `\\E\QE`.

# Unix derivatives

Unix programs tend to use a wide and diverse set of decoders. However, encoding
only requires picking the common subset that will be get consistent handling.

## Posix Shell

[`Base.shell_escape_posixly`]: https://github.com/JuliaLang/julia/blob/v1.2.0/base/shell.jl#L232
[`join`]: https://docs.julialang.org/en/v1/base/strings/#Base.join

The posix shell (`sh`) has spawned many numerous variants, such as `bash` (GNU
Bourne-Again SHell), `tcsh` (C shell), `busybox`, `ksh` (KornShell), and `zsh`
(Z shell), (you can find the list for your machine in `/etc/shells`).
Additionally, it's spawned clones in many languages, including Julia ("\`\`"
command literal syntax). These each have distinct and varied ways in which they
will interpret special characters, and different orderings to their
interpretation and expansion phases.

Fortunately, however, the goal of string escaping is to ensure all of those
complex and varied phases will be inactivate on our arguments. And further, they
all treat `'` as defining a raw string literal.

This gives a very simple, complete rule: wrap each argument in a pair of `'`'s,
and replace any internal `'`'s with the escape sequence `'\''`.

In Julia, this functionality is provided by the [`Base.shell_escape_posixly`]
function. It additionally implements some printing simplifications, such as
detecting when the `'` aren't necessary at all or could be substituted
preferentially for `"`. Those provide nicer output, but aren't required.

This is actually the combination of two steps in the pipeline:
 * Joining the arguments into a single string, by simply concatenating them
   with spaces (For example, by using [`join`]).
 * Encoding each argument separately, to hide any shell-metacharacters from
   later stages in the pipeline.
They are combined into one step for convenience since there's no loss of
generality to provide such an interface.

## Posix programs

In addition to the intermediate shell, it's necessary to consider how the target
application will parse the command line. This is not a step you may normally
think of being in the pipeline, but for stronger robustness, it may be
necessary to consider carefully[^1] these nuances.

 * The `PATH` variable, as used by `execvp` to find programs, uses `:` as the
   delimiter between entries. As such, you can't have a path with an embedded
   `:` in the search PATH as the decoder has no escape sequence for it.
 * Some programs (such as `find`) will output lists of file paths separated by
   white space. As such, you would get incorrect or ambiguous answers if a file
   name happened to contain white space. Some such programs define an alternate
   output mode intended for such uses that uses a nul character as a separate
   instead (which is not valid to appear in a file path). For `find`, this
   option is named `-print0`.
 * Most programs will interpret an argument starting with a `-` as giving a
   command line option rather than a value. For many programs, this means you
   should put a `--` in the argument list, to first terminate option parsing
   before giving the user argument. For example, `rm -- $filename`.

[^1]: In fact, if there is a concern of adversarial input, it may be better to
    entirely avoid passing the data to the external program in this way, since
    it can be hard to realize all of the edge cases. Instead, you may want to
    consider [Base64 encoding](#base64), passing the value in a [side
    channel](#out-of-band) such as `stdin`, or using the builtin features of
    the current language instead of a posix tool.

# Windows

On Windows, the situation is a bit different from on Unix. When transitioning
between these systems, it's important to be aware of this, because it impacts
how some of the stages in the pipeline operate.

In particular, the command line arrives at the target application as a single
string, and is then responsible for interpreting it. Unlike with `posix`, each
program is expected to implement their own shell and glob capabilities (if
relevant), since this interface means that the shell cannot do it.

So unlike in Unix, where we observe that the shell does significant
preprocessing of the input and turns the command input into a list of arguments
and then does further processing on that, on Windows we observe that the shell
only does some initial preprocessing to identify line breaks.

## CommandLineToArgv

Adding to the complication on Windows, there is not one standard decoder, but
at least three. Unfortunately, also none of them implement what they document,
despite (or because) the documentation for what they should do is very simple,
clear, and precise.

Fortunately, for our purposes, we only need to know how to avoid triggering any
of the surprise behaviors. And for that, knowing just the simple rules is
generally sufficient.

For your edification, though, I'll tell of the more common quirks you may
encounter. They are:
 - "Standard" Windows programs (such as DOS commands)
 - Mingw-w64 programs and `CommandLineToArgv`
 - MSVC and .Net programs
 - User-supplied

Firstly, the documentation uses the same format as [Julia "raw"
strings](#julia-raw-strings) described before[^win1]:

> All quotation marks must be escaped, and any backslashes that precede them.
Only when a sequence of backslashes precedes a quote character. Thus, 2n
backslashes followed by a quote encodes n backslashes and the end of the literal
while 2n+1 backslashes followed by a quote encodes n backslashes followed by a
quote character. Anywhere else, each n backslashes simply encodes for n
backslashes.

That means that this: `app "a""b c"` should be expected to be interpreted as
the single argument `ab c`.

The `CommandLineToArgv` adds an undocumented additional rule that a
double-quote immediately before a closing double-quote is preserved
(but the second double-quote still closes the string).

That means that this: `app "a""b c"` gets interpreted as the argument pair
`a"b` and `c`.

Most other programs (including the MSVC parser and .Net parser) appears to use
an almost similar rule to that, although their implementations are separate and
their behaviors are undocumented, so I can't say for certain if they are
exactly the same. Unfortunately, this shortcoming then seems to have crept into
the documentation and implementation of spawning new processes from .Net, again
describing an almost similar rule to what actually exists[^win2]. The rule
these programs appear to use is that any pair of double-quotes inside a
double-quoted string also encode for a single double-quote.

That means that this: `app "a""b c"` is interpreted as the single argument `a"b c`.

What do all these have in common? They all have the four special characters
`"`, `\`, space, and tab. But you can also write your own: in fact, there's
even one alternate one provided by default that adds some additional special
characters to implement filename globbing[^win3]. We're going to ignore that
one, however, since the documentation is very limited (it just says to go read
the documentation for specifics), and on initial testing, it appears to be
incapable of being used for our purposes as it fails to implement any sort of
escape sequence.

So our final algorithm to avoid all of that is fairly straightforward, if not
precisely simple. This can also be found in various other online resources such
as the Microsoft blog[^win4]. And we've already seen it in
[before](#julia-raw-strings). In brief[^win5]:

  1. Double all `\` characters leading up to a `"` or the end of the string.
  2. Replace all `"` characters with `\"`.

[^win1]: <https://docs.microsoft.com/en-us/cpp/cpp/parsing-cpp-command-line-arguments>
[^win2]: <https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.processstartinfo.argumentlist?view=netcore-3.0>
[^win3]: <https://docs.microsoft.com/en-us/cpp/cpp/customizing-cpp-command-line-processing>
[^win4]: <https://blogs.msdn.microsoft.com/twistylittlepassagesallalike/2011/04/23/everyone-quotes-command-line-arguments-the-wrong-way>
[^win5]: Actually, there's one more quirk in argument parsing which is that
    sometimes the first argument gets parsed with a yet different set of rules.
    This is justified on the basis that the OS isn't expected to be able to
    deal with filenames containing special characters (such as `"` or a
    terminating `\`), and the first argument is conventionally the path used to
    launch the program.  Some of those considerations are mentioned talked
    about more in the MSDN documentation for [CreateProcessW](), although we
    can't do anything about it here. Anyways, forget I mentioned it.
[CreateProcessW]: https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw

## Cmd (not DOS)

This one is actually quite simple, but won't fit into you mental model at all
if you try to understand this like it is a posix shell. This is greatly
exacerbated by the tendency of many online resources to talk about this in
terms of the posix model with "arguments" and "splitting." But Cmd doesn't have
any concept of either of those. This means it actually has quite a simple
escaping algorithm. A true DOS prompt would be more primitive than this, but
hardly ever is seen in practice now, so I didn't spend time researching it.

Each character that might be special must be preceded by the escape character
`^`. The list of special characters is `%`, `!`, `^`, `\`, `"`, `<`, `>`, `&`, and
`|`. (many lists get this wrong, and either miss some or list others like `?`
that are meta characters of a different "program", such as `find` or `for`).

It's also valid to prefix every character with `^`, but typically more
cumbersome.

## Batch files

Batch files are similar to Cmd, but has a few surprises of it's own.

First is that the special characters are the same as Cmd, but
you uses `%%` instead of `^%` to write a literal `%`
character. (`^%%` is also valid, but not `^%^%`)
All other special characters are preceded by `^`, like before.

Also the argument parser for batch files is very limited and can hold some nasty surprises for arbitrary input.

  - The argument list (%1 to %9) is split on white space (spaces, tabs, and newlines).
  - Wrap an arguments in `"`'s to keep it as one argument, including white spaces.
  - The quotes will be preserved in the argument.
  - There's no escape character (no way to write an unbalanced `"`).
  - A use of an argument that ends in `^` inside the script will _quote_ the next character (newline, space, etc.) in the script, potentially completely alternating the meaning of that command and the next one.

## Powershell and pwsh

### Powershell parser

### Powershell cmdlets

### Powershell -Command

### Powershell special rules

### Powershell special commands


# Other
(e.g. strategy for writing your own)


---
