---
---

I am experimenting with smaller line-lengths,
combined with justification, and 
automatic(!) hyphenation by the browser, to improve
readability.  I just don't like super-long line lengths
for reading.

Check the source code of the pages wherever words are
hyphenated to confirm this is not done at the source level.

Or just resize the viewport width.

Reference: [mozilla developer documentation on hyphens][1]

-----

{:lang="de"}
Das Grundprinzip sollte eigentlich auch auf Deutsch funktionieren,
zum Testen gibt's hier das allseits beliebte
Donaudampfschifffahrtskapitänsmützenhalterversicherungsvertreterbetriebsratssitzungsorganisationskomitee.

-----

Update 2016-03-24:  See [the insane amount of changes][2] that
were required to get this working.


[1]: https://developer.mozilla.org/en/docs/Web/CSS/hyphens
[2]: https://github.com/dannyedel/blog/commit/1205d203a307038417f7ea73bc0207cc3661eff3
