---
categories:
- process
- questions
date: '2023-08-25'
layout: post
title: Rethinking the RDKit versioning model
description: Do we need to be so conservative about rolling out new features?
toc: true

---

# Background

The current RDKit versioning scheme is pretty simple: we do two major releases a
year, one in March/April and one in September/October, and a patch release about
once a month. So version `2023.03.3` is the second patch release of the
`2023.03` major release cycle. In the patch releases we (try to) only include
bug fixes which don't change the API or introduce major backwards incompatible
changes. Everything else - new features, deprecations, code refactoring, API
changes, larger backwards incompatibilities, etc. - goes in the major releases.

I think aspects of this make a lot of sense: if you install a patch release of
the RDKit you can be comfortable that whatever code you've built on top of it
will continue to run. Your code may generate different results if it's using
RDKit code where a bug fix was applied (after all, fixing bugs can change
results), but everything should still run. You're probably safe just installing
a patch release without being too concerned about reading the release notes.
However, you *definitely* want to read the release notes, particularly the
"backwards incompatibilities" part, before installing a new major version of the
RDKit.

> Aside: if you are working on a research project and you are concerned about
> reproducibility, or if you've deployed a model which uses fingerprints or
> descriptors as input, then you should be *very* careful about changing
> versions of the RDKit or any other dependency. Bug fixes may well change the
> results you get in unpredictable ways. It's always safest to either stick with
> whatever software version(s) you are using or to regenerate all of your
> results.
>
> My general practice here is to create a conda environment at the beginning of
> a project which pins the versions of the RDKit and other sensitive libraries
> (like scikit-learn) and then to use that environment throughout the project.
> If I write things up, I will include the `.yml` file for the environment as
> part of the SI/code for the paper.


# A possible change

Lately I've been thinking about the fact that there are many circumstances where
adding new features is as safe as (or even safer than) doing bug fixes. For
example, us doing things like adding a new fingerprint type, supporting a new
file format, or adding a new feature to the conformer generation algorithm will
have zero impact on existing code. So we could theoretically roll changes like
this out in a point release without worrying about breaking anyone's code. It
would be nice to be able to get new stuff out there to the community more than
twice a year.

If we went down this road, we'd have to think carefully about which new features
got rolled out in the point releases. Obviously anything where major refactoring
work was done or which makes backwards incompatible changes wouldn't qualify for
a point release. We probably wouldn't want to add new descriptors - that will
change the length of the descriptor list, which might break code - but that
would be something to think about. It would mean that we (the maintainers) would
have to think a bit more carefully when assigning milestones to new features and
it would also create a small amount of extra work when creating releases (more
stuff to backport), but I suspect that both of those would be fairly minor.

I do think it would be cool and useful to be able to make new stuff available
more quickly.

Any thoughts or concerns? Something I missed? As always, feedback is appreciated.


> Final note: this post is not an invitation to debate the merits of different
> version numbering schemes. We aren't going to make any changes there.