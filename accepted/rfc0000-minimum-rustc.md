# Summary
[summary]: #summary

Set a formal policy for the minimum supported Rust version in `rayon` and
`rayon-core`.

- We will support *at least* one year of Rust compilers, meaning that our
  chosen minimum will be at least one year old at the time we release any
  rayon crate requiring that.

- Increases to our minimum supported Rust will also require a minor increase in
  the semantic version of rayon crates.

# Motivation
[motivation]: #motivation

Up to this point, we haven't really had a formal version policy, but we have
been conservative with it.  The initial requirement of Rust 1.12 was set in
[rayon#171], merged in December 2016, where we decided that an increase would
be a breaking change. We had some hesitation about the fact that locking this
down for 1.0 would necessitate raising to 2.0 someday, just for a new compiler
version. This went into `rayon` 0.6.0 and `rayon-core` 1.1.0. Then [rayon#528]
in February 2018 raised this slightly to Rust 1.13, released in `rayon` 1.0 and
`rayon-core` 1.4.0 -- note that this already violated the breaking-change policy
for `rayon-core`.

We haven't updated the minimum since then. In our own code, it's a little sad
not to use newer Rust features, but it has been manageable. We've even been able
to support things like `i128` using feature detection in the build script.

However, it's more difficult for our dependencies, which may not be so
conservative with their minimum Rust. We already need manual [downgrades] in CI
for `docopt` and `lazy_static` just to build on Rust 1.13. We're also stuck
with old versions of `crossbeam` and `crossbeam-deque`, missing out on its newer
enhancements -- see [rayon#586] as an example of user concerns.

With a one year policy on our minimum Rust, this will soon let us raise our
minimum to Rust 1.26, released 2018-05-10, which is supported by `crossbeam` and
`crossbeam-deque` 0.6. It's just a little further to the current `crossbeam` 0.7
requiring Rust 1.28, released 2018-08-02.

[rayon#171]: https://github.com/rayon-rs/rayon/pull/171
[rayon#528]: https://github.com/rayon-rs/rayon/pull/528
[rayon#586]: https://github.com/rayon-rs/rayon/issues/586
[downgrades]: https://github.com/rayon-rs/rayon/blob/047ea91e6450d923ff769179de94a15fcdf7e6aa/.travis.yml#L15-L24

# Guide-level explanation

The only change for Rayon users is that they might have to update their compiler
if they're currently using something older. This minimum is documented in each
crate's `README.md`. If a user is unable to upgrade, they can pin a prior
*minor* version like `rayon = "~1.0"` and `rayon-core = "~1.4"`, or remain on
older rayon versions in their `Cargo.lock`.

There is no effect on the Rayon API. Some newer conditional features like `i128`
ranges can be made unconditional, but this was automatically detected, so
there's still no difference to users.

# Implementation notes

Implementing a new minimum Rust just requires updating the version in the CI
script and `README.md`, and raising our crate minor version in `Cargo.toml`.
After that, we can freely make internal changes to make use of new features,
including dependency updates like `crossbeam`.

# Rationale and alternatives

- The current conservative choice is frustrating for rayon users and developers
  alike, especially being stuck with old `crossbeam`.
- While there's no firm community consensus, updating Rust with a minor semver
  increase seems to be a common pragmatic approach.
  - This also allows the possibility of publishing bug/security fixes as patch
    updates on older minor series.
- If Cargo gains the ability to resolve dependencies based on minimum Rust, per
  [RFC 2495], then that will be a preferred solution. We would have to treat
  that as a one-time bump according to the policy here to get a Cargo that
  understands that field, and from then on we could update freely without
  impacting users of older compilers.

[RFC 2495]: https://github.com/rust-lang/rfcs/pull/2495

# Unresolved questions

We don't really know who will be impacted by raising our minimum. The vast
majority of respondents in the [2018 Rust survey] reported using the current
stable or nightly. 17% reported getting their Rust from Linux distros, but we
don't know which. If we want Rust 1.26, RHEL7 devtools and all Ubuntu LTS have
Rust 1.31; Debian 10 has Rust 1.32, but Debian 9 would be left out with its Rust
1.24; and of course this sample of distros is incomplete.

[2018 Rust survey]: https://blog.rust-lang.org/2018/11/27/Rust-survey-2018.html
