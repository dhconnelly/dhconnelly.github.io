---
title: Why I like lots of languages, and Go is pretty neat too
layout: post
timestamp: March 19, 2012
---

As we approach the release of Go 1, the first "officially-stable" distribution,
there has been an increase in the number of blog posts and interweb dialogues
discussing the pros and cons of Go.  As might be expected, a lot of this comes
in the form of language war pissing matches.  I hate and try not to get involved
in that form of discussion, but since Go explicitly targets supposed failings
of other languages, I suppose it isn't entirely unreasonable for advocates of
those languages to respond aggressively.  This is my take.

I like lots of programming languages.  Instead of talking about why Go is worse
or better than other languages, I'm going to talk about four other languages
and what I like about them.  Then I'll mention a few things that I think Go has
to offer and why I use it.

TL;DR: Language design involves tradeoffs, pick the right tool for the job,
let's all be nice and love programming together instead of engaging in pissing
matches.  Go has nice features that might be worth your time, or maybe not.


Java
----

Almost all of my professional experience is with Java, as a full-time developer
at MIT Lincoln Laboratory and as an intern at Google.  In both of these
environments Java makes sense.

At Lincoln Labs, stability, open-source libraries, and enterprise-friendliness
make Java an easy decision.  LL is essentially a defense research lab, and the
DoD is still worried about problems like: getting decades-old satellites to talk
to new information aggregation systems (look up "stovepipe" systems).  My group
at LL was focused on data integration, data mediation, and "net-centricity".
The Department of Defense thinks Java and SOAP and XML are cool and new, and
let's be honest: outside of the hipster web dev/mobile dev communities, Java and
SOAP and XML are STILL cool and new.  Real Work is being done every day to
migrate legacy systems into the new millenium, and the Java ecosystem is stable
and mature.  Modernization takes years (decades, sometimes) and nobody wants to
do all this work and have to do it again next year just because Java is "too
verbose" or because Node.js is hot shit.  As anyone who has done professional
Java development knows, Java has roughly one brazillion XML libraries and
parsers and bindings generators and validators and ....  As of 2012, this
applies equally well to nearly any problem domain.  Stability, the huge
ecosystem, and industry-standardization are huge wins if you choose Java, and so
for many environments, Java is the Right Choice.  The amazing tooling support
available for Java developers makes this even easier.

At Google, huge teams work on huge codebases.  Please triple the emphasis on
"huge" when you read the previous sentence.  Java, having excellent tools and
hardly any metaprogramming support, makes it ideal for Google-sized projects.
Stealing part of the foreword to [SICP][], I think we can substitute "Java" for
"Pascal":

> Pascal is for building pyramids -- imposing, breathtaking, static structures
> built by armies pushing heavy blocks into place. Lisp is for building
> organisms -- imposing, breathtaking, dynamic structures built by squads
> fitting fluctuating myriads of simpler organisms into place.

To summarize: Java has stability, longevity, libraries, excellent tools, works
well with large teams, and is establishment-friendly.  If I were responsible for
choosing a language in an environment where those things mattered (read: most),
Java would be my choice.


Python
------

I have done some Python professionally, but mostly in support of other tasks as
a convenient scripting language.  Until very recently, though, I did most of my
exploratory programming and grad school work in Python.  It's fun, easy on the
eyes, and very expressive--sometimes called "executable pseudocode".  I can't
offer any comments from personal experience about its scalability, but YouTube
is built in Python.  I suspect that says something significant.

In my experience, Python tooling, while abundant, is pretty disppointing.  Lots
of academic research has been done in static code analysis, but due to Python's
flexibility and dynamic typing, not all of this is easily applicable.  Several
companies offer full-fledged Python IDEs with many of the niceties of Java
environments, but I've never had 100% success getting them to work properly
with all the libraries I want to use.  I suspect this is partly due to the
way Python handles environments and libraries.  Maybe virtualenv solves this
problem.

My ignorance about Python tooling is partly due to another reason why I love
Python: a reduced need for tools.  Python is much less verbose than Java, for
instance, and tends to require far less autocompletion-driven-development.
Emacs is a good-enough Python development environment, which is great, since I
use Emacs for almost everything else.

Also, the easy-on-the-eyes factor actually matters.  Two examples:

For my graduate assistantship, I teach high school math.  (More specifically, I
teach college-level number theory to extremely gifted students at a charter
school--last year, out of a class of 19, two of my students went to MIT, one to
Stanford, one to Yale, and the rest to other top universities.)  We spend a
couple of weeks covering the basics of Python and working on Project Euler.  My
experience is that while some of the students already know how to program (most
of these know Java), the rest are much less scared by Hello World in Python than
in Java.

For a special problems course at my university, I've been working on translating
some of the Common Lisp AI programs from Peter Norvig's classic "Paradigms of
Artificial Intelligence Programming" into Python.  My code is heavily commented,
and I run it through Pycco to make something resembling prose.  The goal is to
produce an educational resource for students encountering AI for the first time
with no background in Lisp.  If you're not familiar with PAIP, it surveys some
fundamental AI techniques by presenting working versions of influential AI
programs: the General Problem Solver, Macsyma, a Prolog implementation, etc.
It's a fantastic hands-on book, but Lisp is dead in undergraduate education.  (I
don't imagine this is a controversial statement if you pay attention to
undergraduate education.)  There's a definite trend towards teaching CS 101 in
Python, and even if you're never programmed in the language, anyone who has read
an algorithms text can read Python.  (By the way, if you don't mind watching the
sausage making, you can find the project on [GitHub][paip].  It's not yet
pretty, as de-Lisp-idiomization of the algorithms requires multiple passes, and
I still have to make forward progress.)

Python is my top choice for rapid development tasks, teaching, web stuff, and
scripting.


C
--

I'm not going to say much here.  I've done a ton of C, but all of it in school.
For someone interested in languages and web stuff, I've taken an awful lot of
systems courses (4, that is, none of which were required for my degrees; I have
a half-written blog post about "learning operating systems: a research paper
course").  This means lots of low-level stuff with pthreads, POSIX shared
memory, schedulers, etc.  I actually like C, a lot.  As they say in K&R, C is a
small language and doesn't need a large book.  They're right--the language and
standard library can fit in your head.  It's nice to sometimes focus on
algorithms and problem solving instead of reading API documentation and gluing
things together.

I just picked up Robert Love's [Linux Kernel Development][lkd] so I can start
hacking on the kernel (put all this education to some use) and try to avoid
forgetting C after I get a real job.  There is, of course, nothing else
currently suitable for Real World kernel programming.  (A nice post about this
showed up on Hacker News yesterday: [Why Systems Programmers Still Use C, and
What to Do About It][c].)


Clojure
-------

As I've mentioned in a previous post, I spent last year writing code in many
different languages.  None of them are worth saying much about.  However, of
those other languages, Clojure was my favorite:

- Lisp is fantastic, for all of the reasons other people have eloquently
  presented for decades.
- Clojure gets free access to many of the benefits I mentioned above for Java.
- Functional programming.  Good for brain stretching, testability, robust
  systems, and other things that I'm sure I forgot.
- The community is fantastic.

I went to Clojure Conj this past November and had a blast.  There is some really
awesome stuff going on in the Clojure community.  They're making Lisp and
functional programming cool and accessible.  However, after spending a month
doing lots of Clojure, I consciously decided to stop until I get a job.  Yes,
I'm interested in all of those companies that make you implement algorithms on a
whiteboard, and I want to spend my free time hacking in something closer to the
"whiteboard problem domain".  You can be outraged if that's your thing, but I'm
not in a hurry.  I just ordered a [new Clojure book][clj] to start reading in
August, when I should know where I'll end up.


Go
--

After putting Clojure aside, I went looking for something more traditional that
was still enjoyable.  I wanted a language:

- designed recently;
- more concise than Java, with syntax for dictionaries;
- with first-class functions;
- built with concurrency in mind;
- that leverages the programming paradigms I already know.

(The third point goes along with one of my New Year's resolutions: become more
focused and mature as a programmer.  I read [The Pragmatic Programmer][TPP]
recently, and am currently reading [The Practice of Programming][tpop].)

Go meets these conditions.  I was initially put off by the "backwards" syntax
for typed declarations and lack of generics.  But after getting over these
initial humps, I've found Go to be fun and enjoyable.

The standard library is modern and shockingly complete.  You can marshall and
unmarshall JSON to/from user-defined data structures.  If you need to analyze
source code, there are packages that implement most of the front end of a Go
compiler--lexing, parsing, manipulating ASTs, etc.  There's built-in image manipulation.  You should [check out the list][packages].

Goroutines and channels make concurrent programming easier.  Some programming
problems are naturally concurrent, but many languages make concurrency a library
problem instead of a language problem.  Syntax support with goroutines and
channels makes this less painful.

I also feel that Go benefits from the background of its authors.  To list three more notable members:

- Rob Pike, coauthor of [The Unix Programming Environment][upe] and [The
  Practice of Programming][tpop] and a member of the UNIX and Plan 9 teams at
  Bell Labs
- Ken Thompson, the inventor of UNIX and a Turing Award recipient, also from
  Bell Labs
- Brad Fitzpatrick, inventor of memcached and LiveJournal

In my opinion, the language and standard library have a solid, pragmatic,
simple, and clean design.  This is of course subjective, but programming in Go
definitely has a certain well-engineered "feel".


Conclusion
----------

I now do all of my exploratory programming in Go.  It's fun.  I like all of the
languages I mentioned earlier, and I feel like the job should always determine
the tool, so I don't want to get involved in cross-language firefights.
Programming is what I love, not language partisanship.

When Go 1 is launched, maybe give it a try.  If it doesn't have feature [X] that
you badly want, then maybe Go is not for you.  And that's OK too.


[SICP]: http://mitpress.mit.edu/sicp/full-text/book/book-Z-H-5.html
[paip]: http://github.com/dhconnelly/paip-python
[lkd]: http://www.amazon.com/Linux-Kernel-Development-Robert-Love/dp/0672329468
[c]: http://www.bitc-lang.org/docs/papers/PLOS2006-shap.html
[clj]: http://www.clojurebook.com/
[faq]: http://weekly.golang.org/doc/go_faq.html#What_is_the_purpose_of_the_project
[psv]: http://www.youtube.com/watch?v=5kj5ApnhPAE
[TPP]: http://amzn.com/020161622X
[tpop]: http://amzn.com/020161586X
[packages]: http://weekly.golang.org/pkg/
[upe]: http://amzn.com/013937681X
