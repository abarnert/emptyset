emptyset
========

A stupid module that makes ∅ an empty set literal

For the 69105th time, the idea of having an empty set literal [arose on
python-ideas][emptyset].

  [emptyset]: http://article.gmane.org/gmane.comp.python.ideas/28066

[Terry Reedy suggested][pyu] that the way to do this would be to write a
proprocessor that converted ".pyu" files into ".py" files, but then
[random832 pointed out][random] that this can't be done as a source conversion,
because the whole problem is that there is no Python source code that
evaluates to the empty-set literal bytecode. That is, `[]` compiles to
`BUILD_LIST 0`, `{1}` compiles to `LOAD_CONST 1` then `BUILD_SET 1`, but
nothing compiles to `BUILD_SET 0`. If you follow the thread from there,
you'll see increasingly complicated solutions attempting to get around
that problem.

  [pyu]: http://article.gmane.org/gmane.comp.python.ideas/28159
  [random]: http://article.gmane.org/gmane.comp.python.ideas/28200
    
But, [as I discovered][astset], while there is no _source_ that compiles to
`BUILD_SET 0`, there is a dead-simple _AST_ that compiles to it. And
it's very easy to hook the import loader to compile everything up to
the AST, then do something, then finish compiling.

  [astset]: http://article.gmane.org/gmane.comp.python.ideas/28234

Just run `emptymain.py`, and it should print out this:

    set() is the empty set ∅

(If it fails, or the first part isn't `set()`, then it hasn't
transformed the `∅` in the source code into a valid empty set
literal. If the lasst part isn't `∅`, then it's tampered with the
strings. If both are right, then everything is perfect with the
world, so go take the rest of the day off.)

Implementation details
======================

The character ∅ can legally appear in a string or bytes literal
or a comment, but nowhere else in Python code. So, what we want
to do is replace it everywhere but string and bytes literals (we
don't care about comments) with something that's legal everywhere
we want ∅ to be legal. An identifier seems like exactly what we
want here, so we just need an identifier that doesn't exist in
the source.

So, first we need to find an identifier that doesn't exist
anywhere in the source. Then we want to replace every ∅ in the
source with that identifier. That will affect string and bytes
literals, but we can deal with that below.

Next, we need an AST transformer for the import hook to run.
This can be just a simple `NodeTransformer`. Every `Name` node
is an identifier; if it's our magic identifier, replace it with
the empty set AST. Every `Str` node is a string; if it contains
our magic identifier, change it back. Every `Bytes` node is a
bytes; do the same as with `Str`, except that you'll need to
store `∅` encoded in the source-file encoding, which you stored 
earlier in the import process.

Bugs, hackiness, other caveats
==============================

This requires Python 3.4+, because `importlib` didn't work the same
in 3.3, and didn't exist in 2.7. Similar tricks can be done with
older versions of the importer; [MacroPy][macropy], [Hylang][hy], etc. have
import hooks for older versions (I think 2.7 and 3.2, respectively,
but don't quote me on that).

  [macropy]: https://github.com/lihaoyi/macropy
  [hy]: https://github.com/hylang/hy

As with any import hook, the hook cannot affect your main script, 
only scripts that are imported after the hook is imported.
That's why `emptymain.py` exists: to import the hook from 
`emptymain.py`,  then import the actual script `emptyset.py`.

I'm lazy and didn't store the source-file encoding earlier in the
import process, I just called `importlib._bootstrap.decode_source`
(which is undocumented, in an implementation-details module,
and almost certainly not portable to other Pythons, or even
future CPythons). So, screw bytes literals. If you want them to
work, `decode_source` is only a few lines of code, and you
should be writing it yourself anyway.

Terry's idea was not an import hook that adds the empty set literal
to all .py files after being installed, but some way to add the
literal to .pyu files only. This is doable and easy (see the
Hylang project linked above, which adds a hook for .hy files, 
while leaving .py files alone), but I haven't done it.

I've never used Python 3.4's `importlib`, and things keep changing 
from version to version. (I think it's finally stable, but that's
not much help when I learned on a much earlier version, and I'm
lazy.) And there aren't any good examples out there. Some of what 
I  did is clearly hacky and the wrong way to do it; if someone 
actually  wanted to use this, they'd want to read the docs and do 
it right. (The  `ast` part of the code should be fine, it's the 
import hooking that isn't.)

Finally, the whole idea of adding a Unicode empty set literal 
seems like a bad idea to me, since `set()` is good enough and 
readable and familiar and already working. And, even if you 
needed an empty set literal, the idea that it must compile to an 
actual empty set literal rather than a call to `set` (whether 
for performance, or because you really need to redefine the name 
`set` but also need set literals) makes it even sillier. So, 
really, consider this whole thing a proof of concept, not 
something you should actually fix and use.
