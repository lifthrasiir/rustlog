Title: Summary of Korean Rust Meetup #4
Date: 2015-01-20 11:17

As I've [promised](worklog-2015-01-17.html) before,
this is a summary of the 4th Korean Rust Meetup
hosted by [Korean Rust User Group](http://rust-kr.org/),
where people concentrate on getting things done
no matter they are working on (libraries, Rust PRs and so on).

* * *

[@klutzy](https://github.com/klutzy) was repairing
[suruga](https://github.com/klutzy/suruga)'s experimental X.509 parser.
There was some work on the certificate verification for a while,
but recent macro reforms broke them a lot.
Taking this as an opportunity to fully develop the parser,
the development branch of suruga now got
at least one kind of extensions working.

[@kroisse](https://github.com/kroisse) worked on
the [Rust port](https://github.com/earthreader/rust-earth) of
[libearth](http://libearth.earthreader.org/),
a supporting library for the [Earth Reader](http://earthreader.org/).

I ([@lifthrasiir](https://github.com/lifthrasiir)) added
a [new testing script for Rustdoc outputs](https://github.com/rust-lang/rust/pull/21304).
Actually I wasn't able to send a pull request in time
(the PR was sent hours after the meetup),
as I accidentally omitted `NO_REBUILD=1` and wasted some hours. :S

[@sanxiyn](https://github.com/sanxiyn) got
a [basic infrastructure for code completion](https://github.com/rust-lang/rust/pull/21323).
At the moment it is able to produce
a set of possible field names at given file position.
This will tremendously help the development of IDEs for Rust.

[@simnalamburt](https://github.com/simnalamburt) worked on
[obj-rs](https://github.com/simnalamburt/obj-rs)
which parses Wavefront `.obj` file.
We already have [some existing parsers](https://crates.io/search?q=obj),
but I've been told that
other parsers cannot handle files produced by proprietary softwares,
such as 3ds Max, so their uses are somewhat limited.
It will help many professional game developers I guess.

* * *

There were also some interested people who are simply willing to learn Rust.
This is partly because the meetup was co-located with other meetups
(such as [PyJog](https://ko-kr.facebook.com/pyjog)).
Having 1.0.0 alpha surely was good for them!
I'm looking forward to the next meetup,
which would likely be on the next month (February 2015).

