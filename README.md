An experimental repository for collecting Rayon RFCs. Rayon RFCs are
proposals to alter the rayon APIs. They are meant to gather feedback
and collect constraints, as well as to bikeshed various API options.

### Warning: Not really open for business

**The text below is preliminary.** We've not really done any RFCs yet
and we don't quite know what process we want. The idea is to start off
with a few "seed RFCs" and gain experience before we start fielding
many RFCs.

### How to open an RFC

First, hop onto the Rayon Gitter channel and run the idea by somebody
in the [Rayon core team]. If there is general agreement that the idea
is worth pursuing, then you can go ahead and open a PR that adds a new
file to the `accepted` directory, starting from [the
template](template.md). Tag the PR with a subject line like "\[WIP] My
great idea". Opening early helps make the process more
collaborative. We can discuss on the PR itself and ultimately reach
some sort of consensus.

### Decision making process

As with Rust RFCs, the intention is for discussion to reach a "general
consensus" -- not necessarily about the *outcome* but about the
*contours of the solution space*. In other words, a general agreement
as to the pros and cons of each alternative, if not necessarily the
relative importance of those pros and cons. Ulimately, decisions
regarding Rayon APIs are decided via the [Rayon core team]. The
reasoning of those decisions will always be laid out in comments on
the RFC itself.

[Rayon core team]: https://github.com/orgs/rayon-rs/teams/core
