+++
date = "2015-07-11T15:33:39-07:00"
title = "On discovering Go"
draft = false
+++

**Note:** For a more inspiring and technically informative account of learning Go, you should check out Audrey Lim's brilliant GopherCon 2015 talk on [How a Complete Beginner learned Go as her first backend language in 5 weeks](https://sourcegraph.com/blog/live/gophercon2015/123565059490).

Anyone who's followed (but not muted) me on Twitter in recent months has probably seen at least one of my tweets to @srcgraph. A few days ago, I posted two semi-cryptic tweets that I'm (finally) going to explain in more detail than anyone else should care about:

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">I don&#39;t normally do this, but I promise I&#39;ll only say it once: please consider following / trying <a href="https://twitter.com/srcgraph">@srcgraph</a>. I seriously owe them so much.</p>&mdash; Peggy Li (@_peggyli) <a href="https://twitter.com/_peggyli/status/619669834834415620">July 11, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>


<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/francesc">@francesc</a> <a href="https://twitter.com/srcgraph">@srcgraph</a> it&#39;s actually how I got into Go, and Go is what restored my faith in tech communities. :)</p>&mdash; Peggy Li (@_peggyli) <a href="https://twitter.com/_peggyli/status/619685132882960384">July 11, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

----

Roughly six months ago, I had the opportunity to meet somebody on the Sourcegraph team, so I did what any curious person would: Google "sourcegraph" and decide to install `srclib`. (I have since mostly stopped Googling people / companies.)

Cool, I just need to follow the excellent instructions on [https://srclib.org/install](https://srclib.org/install), at [this commit](https://github.com/sourcegraph/srclib/blob/eb7ef77f0dd949ca8e5fde18becc0b8a34ce11ad/docs/sources/gettingstarted.md) at the time. I already have Go installed, the `src` CLI installs flawlessly, and now I just need the language toolchains (whatever those are). I know I have Python, Node, Ruby, and Go all installed, so shouldn't have any issues here... oh wait, installation error?

The error message doesn't make immediate sense to me, so I try the usual things first: rerun the command in case there was a partial or corrupt download. `rm -rf` the download directory and _then_ rerun again. Rerun from a different directory; maybe I should be in $HOME or $GOPATH, as opposed to whatever my current working directory was. Rerun as root; there's nothing in the output about permission denied, but can't hurt to try? (Note: please don't really blindly `sudo` run unfamiliar commands.)

This is getting nowhere, so there's probably a more "interesting" issue than my inability to copy-paste and execute a command. Time to take a closer look at the root error itself. (Yes, this is usually a logical first or second step, not fifth.)

```
~/go/src/github.com/sourcegraph/srclib-go $ go get github.com/golang/gddo/gosrc
../../golang/gddo/gosrc/client.go:118: syntax error: unexpected range, expecting {
../../golang/gddo/gosrc/client.go:119: syntax error: unexpected {, expecting semicolon or newline or }
../../golang/gddo/gosrc/client.go:122: syntax error: unexpected }
../../golang/gddo/gosrc/client.go:123: non-declaration statement outside function body
../../golang/gddo/gosrc/client.go:124: syntax error: unexpected }
```

Time to troubleshoot! Much of this probably seems obvious to you (or even me) now, but keep in mind that this was my very first time looking at non-trivial (e.g. print hello world or sum the numbers from 1-10) Go source code, and I'd mostly read _about_ Go on Quora and Hacker News. Open the file in vim, and jump to [line 118](https://github.com/golang/gddo/blob/4523d2f070c74ef847157e9aa14137376df63964/gosrc/client.go#L118):

<script type="text/javascript" src="https://sourcegraph.com/R$15595@4523d2f070c74ef847157e9aa14137376df63964===4523d2f070c74ef847157e9aa14137376df63964/.tree/gosrc/client.go/.sourcebox.js?StartLine=118&EndLine=122"></script>

As I try to read and understand Go for what I consider the first time, I have two main thoughts:

* What is the difference between `:=` and `=` anyway? I've used Python, Java, C, and JavaScript and _tiny_ amounts of Ruby and Perl once upon a time, so = seems obvious. But what is :=? It looks like it's being used in a similar manner to =, so what's the difference? (Answer: short variable declarations)

And more strangely...

* Why is the developer checking for an error almost every other line? Is this code actually even reliable, or should I just be expecting it to fail? (Answer: the code is perfectly fine and consistent with Go's error handling philosophy.)

My inclination is to give the benefit of the doubt; surely code in a widely-used open-source project wouldn't contain a clear syntax bug, or it would have been reported and fixed already? No matter how silly a logical error there might be, I'm confident that the developer would at least make sure the code _compiles_ at all. But I do quickly check for missing/extra opening or closing braces or parentheses; all good.

I comment out lines 92 - 117 and rerun; nope, the error persists, but at least I've eliminated that (relatively) giant block, which is good. Since this loop uses both `ch` and `files`, I think I can assume those are both well-formed, but I'll throw in a couple of print statements to make sure they're not some null/nil value. Also, head to the playground ([play.golang.org](http://play.golang.org)) to make sure := and <- are both indeed valid tokens. All good there.

Maybe there's an unexpected nil or index-out-of-bounds type issue with the loop causing the body execution to fail? But commenting out lines 119 - 121 doesn't help either, which means that the issue is literally something with `for range files`. (If you've been using Go for awhile, you can probably guess by now.)

Completely stumped and almost ready to give up now, I look at `git blame` and `git log` as a final resort, to see if there were any potentially suspect recent changes. Aha! All I need to see is this commit message (not even the diff): [go 1.4 support](https://github.com/sourcegraph/srclib-go/commit/26ec06a07590eb5311019babedaf0b2b31d3c509)

Okay, this is a minor (not micro) version change to the Go language itself, which generally means not full backwards-compatibility. But still, it's a simple `for` loop! If/else statements and for loops are pretty much the simplest staple control flow structures in most languages, right? But a quick Google search for "golang 1.4" takes me to [https://blog.golang.org/go1.4](https://blog.golang.org/go1.4), where the third paragraph proudly announces:

> The language change is a tweak to the syntax of for-range loops. You may now write "for range s {" to loop over each item from s, without having to assign the value, loop index, or map key.

With a slightly sinking feeling, I ran `go version`... lo and behold, I'm running Go 1.3 (installed back in early or mid December, if I recall correctly; 1.4 was released around the same time). I try uninstalling / deleting my full Go installation and reinstalling a couple of times, to no avail, until I realize `brew install go` is just still picking up 1.3. Indeed, the only thing I had to do was download and install 1.4 from source. Whew, _finally_!

----

Let's quickly review (what I consider to be) two central tenets of software development and distribution:

* Up-to-date documentation (including installation/usage instructions)
* Reproducible builds (this often includes some concept of [preferably versioned] package/dependency management)

This is why I often say (or at least think) that me "discovering" Go was a complete fluke, or at least a lucky accident. In a perfect world, the docs would have said 1.4, or the dependent golang/gddo package would be fully backwards-compatible. I would have peacefully installed it and happily moved along, none the wiser about Go. It's entirely possible that, say, had something like [`gb`](http://getgb.io/) existed at the time, I would not have really gotten hooked on Go, or at least when I did. (Please make sure you keep your docs and dependencies up-to-date and consistent though!)

Instead, I found myself super intrigued by the handful of lines of code I had looked at.  I was immediately struck by and attacted to how readable, approachable, and clear it was. It certainly helps that I have experience with multiple other languages, but I've also skimmed tutorials on, say, OCaml and Haskell, where FP principles themselves made sense but the overabundance of non-alphanumeric tokens in code made my head spin. The terse expressiveness of Scala (compared to Java, at least) and syntactic sugar of Ruby are great for developer productivity or just "code golf", but also mean that production source code isn't always beginner-friendly. In contrast, Go's fairly concise and straighforward syntax and tools like "gofmt" that enforce consistency in code style made the language much easier to digest from day one, and I found myself perusing the [Go Tour](https://tour.golang.org/welcome/1), [Effective Go](https://golang.org/doc/effective_go.html), [Go by Example](https://gobyexample.com/), etc. later that afternoon and weekend. 

But... fast forward to today, and oh, cruel irony! I wanted to include as accurate of an error message as possible, so I decided to install `srclib` inside a Docker container running the golang:1.3 image. Naturally, I then spent several hours trying to figure out [this Docker issue](https://github.com/boot2docker/osx-installer/issues/122) and seriously trying to convince myself this is not the universe changing its mind and signaling that I should quit Go and the Go community. (Solution here: forget boot2docker, an AWS instance is only $2.) I really am not making this up.

---

On the last day of GopherCon 2015, I attended the Code of Conduct &amp; Diversity session and called out a prominent member of the Go team (in front of the person himself) for a discouraging comment I had happened to overhear. About half an hour later, I finally gathered the courage to approach someone from Sourcegraph and awkwardly share an abbreviated version of this story. The second conversation was actually _by far_ the scarier of the two, not least because I was essentially saying, "I'm basically here today because your documentation was a lie, so thank you for that." I seriously expected to be laughed away or outright dismissed, so the polite incredulity was a better response than I could have hoped for.

So there you have it, the real story of how I got started with Go. Frankly, this isn't something I ever planned on sharing, but the tech community needs more diverse voices. I've heard many people (predominantly men) talk about how they moved to Go from C, C++, or Ruby for systems programming or to build microservices. I hope that in some tiny way, this brings a more unique perspective to the table.

In a future post, I'll talk more about how the Go community (even more than the language design itself) "restored my faith in tech communities."
