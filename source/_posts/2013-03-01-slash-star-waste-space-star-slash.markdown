---
layout: post
title: "/* waste space */"
date: 2013-03-01 11:52
comments: true
categories: C, Compilers, Wat
---

{% codeblock At least it is documented lang:c %}
waste()		/* waste space */
{
	waste(waste(waste),waste(waste),waste(waste));
	waste(waste(waste),waste(waste),waste(waste));
	waste(waste(waste),waste(waste),waste(waste));
	waste(waste(waste),waste(waste),waste(waste));
	waste(waste(waste),waste(waste),waste(waste));
	waste(waste(waste),waste(waste),waste(waste));
	waste(waste(waste),waste(waste),waste(waste));
	waste(waste(waste),waste(waste),waste(waste));
}
{% endcodeblock %}

I saw this earlier today while browsing through [one of the earliest recovered C compilers](http://cm.bell-labs.com/cm/cs/who/dmr/primevalC.html).  We're talking prestruct, prepreprocessor madness.  And, apparently, built by [the man](http://en.wikipedia.org/wiki/Dennis_Ritchie) himself.

Modern compilers won't compile it, however.  I hope that someone, someday will make that happen.  As for me though, I can't stop **wat**ing.

You can peruse the source [here](https://github.com/mortdeus/legacy-cc).
