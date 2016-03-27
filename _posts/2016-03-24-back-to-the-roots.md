---
---

From now on, no more fancy CSS.
Just plain black-text-on-white-background,
with a focus on semantics and readability instead of fancy looks.

----

Clarification 2016-03-27:  This is about not using
any HTML for layout directly and avoiding elements
in the DOM tree *just* for layout, such as
`<div class="wrap">` and friends.

The HTML should produce a *semantic* representation
of the blog post, **everything else** has to be added
*purely* in CSS.

The layout makes use of the new semantic HTML tags,
such as `<main>` and `<article>`.

When the CSS is not loaded, the site is *not* allowed
to break layout, such as displaying full-width
logos or navigational icons; the plain HTML should
contain no references to images other than
content-relevant ones.
