---
layout: post
title: Yet Another Git Reading List
timestamp: February 28, 2012
---

Git is awesome.  I used Subversion at my first job out of college, which was a
total a pain in the ass.  Collaboration was just irritating.  I was introduced
to Git last summer at Google and have been using it like crazy ever since.  This
is a list of the best resources I've found for learning and becoming proficient
with Git (which is still for me an ongoing journey).

The bare essentials:

- [The Git Parable][parable] by [Tom Preston-Werner][tpw], cofounder of
  [GitHub][gh].  This is a fantastic description in plain English of how Git
  works at a conceptual level.
- [gitref][], a tutorial that covers the most-used Git commands.  It's quite
  thorough and covers the basic workflow, frequent trip-ups, and common
  practices.  Each section has direct links to more detailed documentation
  from the Git man pages and the online version of the book [Pro Git][progit].

Interesting reads:

- [Git for Computer Scientists][gitcs], a short overview of the Git graph model.
- [Uses of git][uses], a blog post outlining the wide range of applications for
  Git outside of traditional version control, such as blogging, file sharing,
  deployment, and others.
- [Git vs Mercurial][vshg], a blog post discussing some of the differences in
  design and philosophy between Git and [Mercurial][hg], the other popular
  distributed VCS.
- [Why Git is Better than X][better] lists some of the reasons why you should
  use Git instead of its competitors.

References:

- [Pro Git][progit], a complete guide and reference written by [Scott Chacon]
  [sc] (the CIO of GitHub and the author of the aforementioned [gitref][]).
  There's also a [print][amazon] version.
- the Git [man pages][man]

If you're not yet using Git, get on it.  And try out [GitHub][gh]--it's free for
open source projects, has great collaboration features,  and is a lot of fun.


[parable]: http://tom.preston-werner.com/2009/05/19/the-git-parable.html
[tpw]: http://tom.preston-werner.com/
[gh]: http://www.github.com
[gitref]: http://gitref.org/
[gitcs]: http://eagain.net/articles/git-for-computer-scientists/
[progit]: http://progit.org/book/
[amazon]: http://www.amazon.com/Pro-Git-Scott-Chacon/dp/1430218339/ref=sr_1_1?ie=UTF8&qid=1330458848&sr=8-1
[sc]: http://scottchacon.com/
[man]: http://schacon.github.com/git/git.html
[vshg]: http://importantshock.wordpress.com/2008/08/07/git-vs-mercurial/
[hg]: http://mercurial.selenic.com/
[uses]: http://devsundar.github.com/2012/02/09/Uses-of-git/
[better]: http://whygitisbetterthanx.com
