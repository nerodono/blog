+++
title = 'The Software Question'
description = 'Selected works'
date = 2024-12-29T19:59:35+03:00
draft = false
+++

Opinions expressed. The name for that article should be "Selected works", but I'd like to joke about "The Current Thing".

## Nix, NixOS, Guix and Guix
https://nixos.org
https://guix.gnu.org/

Linux distribution that solves the most annoying problems you ever had with your programs and OS - non-determinism ("it works on my machine!") and setting up machines with the same configuration.
The latter is rather annoying, since servers and even home PCs aren't things that set up once and never again, it produces repetitive work (which we all hate!), there's some solutions to that, like something called `Ansible`, `Kubernetes`, `docker`, probably some other container-based solution, however
- They're significantly more error-prone in sense of breaking determinism. While docker supposed to be deterministic, it's easy to find an example of non-deterministically written docker-files, it's really likely that your hand-written dockerfile is non-deterministic, contrary, nix does really good job at preventing that kind of unfortunate results, thus non-deterministic nix setups are rather the exception. I took docker as an example here, it should be applicable to the most container-based solution.
- Most of these solutions configured in yaml. While yaml by itself is not a problem, like, even things explained in [The YAML Document From Hell](https://ruudvanasseldonk.com/2023/01/11/the-yaml-document-from-hell) are not that problematic, actually, the format does pretty good job at explaining complex configs with hierarchy the human-readable way. Thing that can bother you is that these software which uses (and needs) configs in yaml simply implement some kind of programming language on top of it, and that's unfortunate. On the other hand, nix introduces programmable configuration language with the same name - nix, it's basically the programming language, so you are not obligated to struggle with expressing things in crappy language on top of yaml, you get declarative language that kinda looks like json. Bazel, for example, thinks the same way and introduces starlark programmable configuration language, which looks like python.
- Arguably, they're more complex and brain-hurting than NixOS.

The nix as a package manager is superior conceptually, the nixos as an operating system is for nerds, geeks, people who has no better things to do than reading and writing some big configs. Almost everything stated above is applicable to guix as well, however, specifically to guix:
- Name. Guix (Linux) and Guix (Package manager) named equally.
- Guix programmed in GNU dialect of lisp. That lisp is not lazy! So configs written in more understandable style.
- Not so popular. Implies that there's not much software packaged, implies some kind of decentralization.
- More decentralized. Nix and NixOS hosts its primary package repository on GitHub, which owned by the corporation - Microsoft.
- Strict about being "free" and "open source". It's even screaming at you that you need to buy more freedom-respecting hardware on the official install disk (not a joke!). The solution for that is nonguix repos.

Sometimes a mess, \[Specifically to NixOS\] Governance is questionable, the way users write their configs can be confusing due to nix being lazy (it won't evaluate anything until it's really needed), the community is kinda _diverse_ in the negative sense.
Overall, pros outweigh cons, so you should at least try the package manager. Try guix if there some cons like community or governance that is not acceptable for you, either way, guix is interesting piece of work, you should try it unconditionally if you care.

## Rust

Language which I write the most. Solves the most of safety-related problems of languages which designed to run on bare system or for high performance. The key points here is that Rust allows you to not think about safety of your code to some extent, provides some almost nerd-level type system and procedural macros are heavily integrated in the ecosystem, which makes writing performant software with nice UX actually enjoyable.

The cons of language are obvious when you write it enough
- The borrow checker can be rather strict. You'll definitely struggle some time when interaction with borrowing is necessary. Nonetheless, when you experienced enough, you still struggle sometimes! The problems'll be unusual.
- The borrow checker can be dumb. Actually, almost every time it's dumb, it's not like that matters most of the time though. This dummy should be eventually upgraded to `polonius`-based implementation.
- The language foundations aren't specified. This will be a problem if you wish for something like "stable ABI" (this thing has more problems than just lack of specification, but that's out of the scope), or more crazy unsafe tricks.
- Some people complain about large dependency graphs of the rust projects and rust std being small. Some of these are true, like why there's no time formatting facilities in the std? Well, there's barely any functionality for working with time in the standard library, I guess it's time to finally grow balls for the language devs. However, most complains are just whining after running something like `cargo tree | wc -l`. Rust encourages it's developers to split works in crates, this kind of fragmentation is OK, I think we should count authorities, not the dependency graphs, like who made these ten libraries? Let me see.
- `target/` directories are big. Rust stores enormous amount of information in the target directory, for example:
```shell
[nero@lil-maid:~]$ du -sh /nix/store
109G    /nix/store
[nero@lil-maid:~]$ du -sh dev
565G    dev
[nero@lil-maid:~]$ du -sh games
31G     games
```

  The most space here occupies monorepo from my work. The monorepo is rather big though, some C++ folks can find this relatable. Still, unlike the C++ build systems, cargo fails in caching. Yeah, you shitted your filesystem with 565GB of build artifacts and still have to recompile from zero things that are barely affected by the changes, cargo sucks.

- The community is very similar to the nixos one. Keep that in mind.

It's necessary to note, rust borrow checker is not some self-specified substance (the implementation in compiler is the only definition of what borrowing rules are), borrow checker is mere an implementation of aliasing rules, like tree borrows or stacked borrows. The implementation can be and is dumber than the reference rules, sole existence of polonius proves that.

Still, Rust worth your time. It's definitely the progress in tooling and in systems language design, we finally can enjoy writing software without developing some form of paranoia or OCD, this is huge. People get software of better quality, developers are more comfortable, benefits outweigh trade-offs (trade-offs are made mostly for developers, not for the end-user!).

Some related resources:
- [Rust's ugly syntax](https://matklad.github.io/2023/01/26/rusts-ugly-syntax.html) - Why Rust syntax is so ugly?
- [Rust book](https://doc.rust-lang.org/book/) - Start here if you wish to learn
- [Rustonomicon](https://doc.rust-lang.org/nomicon/) - the dark arts of Rust - unsafe.
- [Unsafe Code Guidelines](https://rust-lang.github.io/unsafe-code-guidelines/introduction.html) - Abbreviated as UCG. Informal specification of layouts and how to write unsafe code if you absolutely need the details.
## Haskell
Lazy functional programming language. Contrary to it's self-description to be purely functional language, it's the most useful functional programming language available. Well, kinda strong statement, the **most useful** here is F#, due to being integrated in the dotnet ecosystem, Haskell should be described as the most popular, at least.

Famous for it's expressiveness and big amount of articles like "what a `Monad` is" (something in the category of endofunctors, you know). The community sometimes kinda obsessed with the mathematical fundamentals of the language, while generic user almost every time searches for the applicability of these things to the real tasks.

# Random

When I have no better things to do, I scroll the internet, the next things probably has random inconsistent trails due to this, enjoy.
## Paul Graham articles
This guy is famous for his strong opinions expressed the _strong_ way, my favorite from those I read is:
- [Beating the averages](https://paulgraham.com/avg.html) - popular article (rant) about programming languages power. Introduces the "blub paradox".
- [Revenge of the nerds](https://paulgraham.com/icad.html) - kinda "Beating the averages" written the other way.
- [Writes and write-nots](https://paulgraham.com/writes.html) - ai will divide people even more

Also this guy likes lisp a lot.

## Amos
https://fasterthanli.me/

Rustacean, writes some useful things in the friendly and accessible manner.

- [Building a Rust service with Nix](https://fasterthanli.me/series/building-a-rust-service-with-nix) - series of using nix for rust developers
- [Lies we tell ourselves to keep using Golang](https://fasterthanli.me/articles/lies-we-tell-ourselves-to-keep-using-golang) - nice rant on golang
- [I want off Mr. Golang's Wild Ride](https://fasterthanli.me/articles/i-want-off-mr-golangs-wild-ride) - same as above
- [The HTTP crash course nobody asked for](https://fasterthanli.me/articles/the-http-crash-course-nobody-asked-for) - coping with HTTP

## Without boats
https://without.boats/

Rustacean. Famous for his fair rants on rust's team design choices.

- [Pinned places](https://without.boats/blog/pinned-places/) - Make pinning proper language feature!
- [poll_next](https://without.boats/blog/poll-next/) - revolt against ~~the modern world~~ the `async fn next(&mut self) -> Option<Self::Item>`
- [References are like a jumps](https://without.boats/blog/references-are-like-jumps/) - thoughts on aliasing, shared and mutable state.

## Nico Matsakis. Babysteps
https://smallcultfollowing.com/babysteps/

Heavily influential person on rust language developer. I can call him a scientist maybe. Babysteps blog is full of ideas for the language, so go and read it yourself.

---
- [What color is your function?](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/) - famous article, highlights that async as a language feature is kinda problematic.
- [The PGP problem](https://latacora.micro.blog/2019/07/16/the-pgp-problem.html) - read first, then [The PGP Problem: A Critique](https://articles.59.ca/doku.php?id=pgpfan:tpp).
- [Andrew](https://andrew.internet-landlords.xyz/) - Friend of mine, sometimes writes articles as well. This blog's theme donor.
- [fd](https://github.com/sharkdp/fd) - more convenient alternative to GNU find.
- [ripgrep](https://github.com/BurntSushi/ripgrep) - faster grep.
- [uv](https://github.com/astral-sh/uv) - the actually nice build system for python projects.
- [adaptix](https://github.com/reagento/adaptix) - actually enjoyable library for converting one things to another. Mostly you'll use it for parsing dicts into typed dataclasses. Supports pydantic.
- [Why not pydantic?](https://adaptix.readthedocs.io/en/latest/why-not-pydantic.html) - pydantic is shitty, here's why.
- [Grapevine](https://gitlab.computer.surgery/matrix/grapevine) - faster-evolving [conduit](https://conduit.rs) fork - unstable! Changes can be made in a backward-incompatible manner. Be aware. 
- [Koka lang](https://koka-lang.github.io/koka/doc/index.html) - some nice language from the nerds, supports so-called "algebraic effects"
- [Learn APL](https://xpqz.github.io/learnapl/intro.html) - grow balls and learn APL
- [...but it's unreadable](https://xpqz.github.io/learnapl/intro.html#but-it-s-unreadable) - stop bitching about your skill issues
- [elfo](https://github.com/elfo-rs/elfo) - nice frameworks for making actors-based systems.
- [Idris](https://www.idris-lang.org/) - what if types were values too in a compiled language?
- [Crafting interpreters](https://craftinginterpreters.com/) - nice book which can help you in grasping the language development fundamentals
- [Flutter](https://flutter.dev) - very useful framework for mobile development (mostly) if you hate interacting with java. Written in the most useless language outside the flutter development.
- [Hugo](https://gohugo.io/) - nice static site generator, this site works because of it.
- [Dishka](https://github.com/reagento/dishka) - don't use globals, use at least it.

---
The article is not that short and costed me greater time than I expected, so I'll write some more on that topic later. Maybe something about editors and my projects will be mentioned? We'll see.
