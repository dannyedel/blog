---
title: Find the first *good* commit with `git bisect`
---

Normally, `git bisect` is used to fight *regressions*,
such as the classic "It was working fine yesterday!" problems.
When this is due to a change in the code that was uploaded without
consulting the unit tests,
`git bisect` will easily find it.

## Standard regression-finding workflow

### Start the bisecting process

```bash
git bisect start
```


### Verify master's broken-ness

Just because someone says `master` is broken on their machine,
doen't mean the problem shows up on yours.
Reproduce the problem and tell `git bisect` about `master`'s state.

```bash
git checkout master
make && make test || git bisect bad
```

### It worked yesterday

We need to figure out the exact commit ID (SHA)
of the 'yesterday' commit where it worked.
For fun, let's asumme that it **actually worked** 'yesterday'.

Assuming the developers have their clocks set right, this
command will output (only) the SHA1 (`%H`) of the latest
commit (`-1`) that is at least one day old.

```bash
OKAY=$(git log master -1 --before '1 day ago' --format=format:%H)
```

Check that this commit actually passes the test suite,
and tell `git bisect` about it.

```bash
git checkout $OKAY
make && make test && git bisect good
```

### Let the computer do the rest

`git bisect` uses binary search to find the breaking revision
in logarithmic time, which is really efficient.
To give an idea what logarithmic means,
you can bisect a **million** revisions in about 20 steps.

Any binary search algorithm *requires* a 'sorted' input sequence.

For our case, this assumes that there is ***one*** clear
point in history where **everything before** is good,
and **everything since** is bad.

While this assumption is often good enough for regressions,
"The testsuite passes" is a very complex definition of "good",
since an old (fixed in the meantime) failure of a test completely
unrelated to the current problem will affect the binary search.

For cherry-picking specific fixes it might be better to limit
the definition of "good" to "this one test passes",
and bisect multiple times over the different tests
in case of other testsuite errors.

One can also tell git bisect that the current revision is untestable,
for example because it cannot be compiled.
This is accomplished by returning 125 to the bisect process.

```bash
git bisect run bash -c 'make || exit 125 ; run-that-one-test'
```

Now lay back and let the computer do it's thing.  It will take
roundabout `log2(number_of_revisions)` compiles and test suite runs.
Obviously, the compile time can be massively reduced by employing
`ccache` or similar, since a lot of files' content might not have
changed, even though their timestamps might get touched.

Also, if you later need to bisect the same revisions for the other
test, `ccache` can make the compile step near-instant.

`git bisect` will soon tell you about (and check out) the first bad
commit.  When you're done call `git bisect reset` to leave bisecting.

## Backporting fixes

The above workflow is great for identifying a regression.
However, when we need to backport a problem fix from master
to an older stable version, we need
to find the first *good* commit, which requires us to invert the logic.

In recent git versions, one can change the terminology,
which helps to avoid confusion.


```bash
git bisect start --term-old=broken --term-new=fixed
git bisect broken v1.5
git bisect fixed master
```

However, for `git bisect run` the return codes still have the mapping
to "old" and "new":

* `0` (normally, test suite success) means "old",
  which is now mapped to "broken".
* `125` means "skip", no surprises here.
* `1-127 except 125` (normally, test suite failed) means "new",
  which is now mapped to "fixed".

So we have to invert a bit.  Bash's operator `!` comes in handy.

```bash
git bisect run bash -c 'make || exit 125 ; ! run-that-one-test'
```

This will create the following mapping:

* Build failed: 125 (skip current revision)
* Test suite passes (exit zero): Inverted by `!` to `1`,
  meaning "new" (mapped to "fixed")
* Test suite failed (exit nonzero): Inverted by `!` to `0`,
  meaning "old" (mapped to "broken")
