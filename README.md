# Elix RFCs

The RFC (Request for Comments) process is intended to provide a consistent and
controlled path for new features to enter the Elix project. Many project
changes, including bug fixes and documentation improvements, can be implemented
and reviewed via normal GitHub pull requests. For more substantial changes, we
ask that they be put through a design process to produce a consensus among the
Elix core team.

[Active RFC List](https://github.com/Elix/rfcs/pulls)


## When you need to follow this process

You need to follow this process if you intend to make substantial changes to
Elix. A substantial change may include the following:

* A new feature that creates new API surface area.
* The removal of features that already shipped as part of the release channel.
* The introduction of new idiomatic usage or conventions, even if they do not
  include code changes to Elix itself.

Changes that do not require an RFC:

* Rephrasing, reorganizing or refactoring.
* Addition or removal of warnings
* Additions that strictly improve objective, numerical quality criteria
  (performance, better browser support).
* Additions only likely to be _noticed by_ other Elix contributors and invisible
  to users of the Elix components.

If you submit a pull request to implement a new feature without going through
the RFC process, it may be closed with a polite request to submit an RFC first.


## Gathering feedback before submitting

It's often helpful to get feedback on your concept before diving into the level
of API design detail required for an RFC. **You may open an issue on this repo
to start a high-level discussion**, with the goal of eventually formulating an
RFC pull request with the specific implementation design.


## Making a proposal

To add a major feature to Elix, prepare an RFC and get it merged into this
repo as a Markdown file. At that point the RFC
is 'active' and may be implemented with the goal of eventual inclusion
into Elix.

* Fork this RFC repo (http://github.com/Elix/rfcs).
* Copy `0000-template.md` to `proposals/0000-my-feature.md` (where
  'my-feature' is descriptive. Do not assign an RFC number yet).
* Fill in the RFC template. Put care into the details: **RFCs that do not
  present convincing motivation, demonstrate understanding of the impact of the
  design, or are disingenuous about the drawbacks or alternatives tend to be
  poorly-received**.
* Submit a pull request. As a pull request the RFC will receive design feedback
  from the larger community, and the author should be prepared to revise it in
  response.
* Build consensus and integrate feedback. RFCs that have broad support are much
  more likely to make progress than those that don't receive any comments.
* Eventually, the Elix core team will decide whether the RFC is a candidate for
  inclusion in Elix.
* RFCs that are candidates for inclusion in Elix will enter a "final comment
  period" lasting 7 days. The beginning of this period will be signaled with a
  comment and tag on the RFC's pull request. Furthermore, the core team will
  announce the RFC on Twitter to attract the community's attention.
* An RFC can be modified based upon feedback from the [core team] and community.
  Significant modifications may trigger a new final comment period.
* An RFC may be rejected by the core team after public discussion has settled
  and comments have been made summarizing the rationale for rejection. A member
  of the core team will then close the RFC's associated pull request.
* An RFC may be accepted at the close of its final comment period. A core team
  member will merge the RFC's associated pull request, at which point the RFC
  will become active.


## RFC lifecycle

Once an RFC becomes active, the core team has agreed to it in principle and are
amenable to merging it. Contributors may then implement it and submit the
feature as a pull request to the Elix repo.

Becoming active is not a rubber stamp, and in particular still does not mean the
feature will ultimately be merged.  Furthermore, the fact that a given RFC has
been accepted and is active implies nothing about what priority is assigned to
its implementation, nor whether anybody is currently working on it.

Modifications to active RFCs can be done in followup PRs. We strive to write
each RFC in a manner that it will reflect the final design of the feature, but
the nature of the process means that we cannot expect every merged RFC to
actually reflect what the end result will be at the time of the next major
release. Therefore we try to keep each RFC document somewhat in sync with the
language feature as planned, tracking such changes via followup pull requests to
the document.


## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the RFC
author, like any other contributor, is welcome to post an implementation for
review after the RFC has been accepted.

If you are interested in working on the implementation for an active RFC, but
cannot determine if someone else is already working on it, feel free to ask
(e.g. by leaving a comment on the associated issue).


## Reviewing RFCs

The Elix core team reviews open RFC pull requests in its regular meetings, and
reports the results of those discussion in its meeting notes. Every accepted
feature should have a core team champion, who will represent the feature and its
progress.


Elix's RFC process was inspired by that used by Ember.js and Rust.
