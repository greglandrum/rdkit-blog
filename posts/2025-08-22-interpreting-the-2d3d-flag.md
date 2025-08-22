---
categories:
- documentation
- technical
date: '2025-08-22'
layout: post
title: How the 2D/3D flag in Mol blocks is used
description: Specifications meet the real world
toc: true

---

[This is an expanded version of a section which will be added to the RDKit
documentation to clarify an important detail about the way Mol and SD files are
parsed.]

# Background

Mol blocks, Mol files, and SD files can describe 2D or 3D molecules and include
a flag (called "dimensional code" in the spec) to indicate whether the
coordinates are 2D or 3D. The flag shows up in the second line of the Mol block,
and is present in both V2000:
```
nitrogen
     RDKit          2D

  2  1  0  0  0  0  0  0  0  0999 V2000
    0.0000    0.0000    0.0000 N   0  0  0  0  0  0  0  0  0  0  0  0
   -0.0000   -1.5000    0.0000 N   0  0  0  0  0  0  0  0  0  0  0  0
  1  2  3  0
M  END
```
and V3000 mol blocks:
```
nitrogen
     RDKit          2D

  0  0  0  0  0  0  0  0  0  0999 V3000
M  V30 BEGIN CTAB
M  V30 COUNTS 2 1 0 0 0
M  V30 BEGIN ATOM
M  V30 1 N 0.000000 0.000000 0.000000 0
M  V30 2 N -0.000000 -1.500000 0.000000 0
M  V30 END ATOM
M  V30 BEGIN BOND
M  V30 1 3 1 2
M  V30 END BOND
M  V30 END CTAB
M  END
```

The CTFile specification says the following about this flag:
> The “dimensional code” is maintained explicitly. Thus “3D” really means 3D,
> although “2D” will be interpreted as 3D if any non-zero Z-coordinates are
> found

That's simple (and logical) enough, but of course in the real world things are
more complicated. Files found "in the wild" include every possible combination
of 2D/3D flag and 2D/3D coordinates, so we need to decide how to interpret these
combinations. Things are made more complicated by the possible presence of
wedged bonds in the Mol block. Wedged bonds are typically a signal that
something is known about the stereochemistry of the molecule; how do we combine
this information with the coordinates?

> *Aside on 2D vs 3D coordinates*: in Mol blocks X, Y, and Z coordinates are always provided (if any of them are missing, they are set to zero). It's also possible to have 3D coordinates for planar molecules. For the purposes of this discussion, "2D" coordinates means that all Z coordinates are zero, and a "2D" RDKit molecule is one where the (default) conformer is marked as being 2D (i.e. the conformer's `Is3D()` method returns `False`).

# What the RDKit does

The following table describes how the RDKit interprets all possible combinations
of dimensionality flag, coordinate dimensionality, and the presence or absence
of wedged bonds:

|  flag |  coords  | wedging | result | notes |
|-------|----------|---------|--------|-------|
| 2D | 2D | no  | 2D | no chirality |
| 3D | 2D | no  | 3D | no chirality |
| 3D | 3D | no  | 3D | chirality from coords |
| 2D | 3D | no  | 3D | chirality from coords |
| 2D | 2D | yes | 2D | chirality from wedging |
| 3D | 2D | yes | 2D | chirality from wedging |
| 3D | 3D | yes | 3D | chirality from coords |
| 2D | 3D | yes | 3D | chirality from coords |

This is consistent with what the specification says except for the case where
the 3D flag is set for 2D coordinates and a wedge is present, in which case we
ignore the 3D flag, mark the conformer as 2D, and set the stereochemistry based
on the wedging.

In cases where no 2D/3D flag is provided, the default value of the flag is 2D. 

In a 2D structure, wedging is interpreted as a signal to indicate that
stereochemistry is present and to indicate what the stereochemistry is (i.e. the
stereochemistry is determined based on the 2D coordinates and the direction of
the wedge).

When 3D coordinates are provided, stereochemistry is perceived from the
coordinates themselves. If wedging is also present it will be ignored; the 3D
signal is "stronger" than the wedging signal. The exception to this rule is
[atropisomeric
bonds](https://www.rdkit.org/docs/RDKit_Book.html#atropisomeric-bonds), where
the wedged bond indicates that atropisomerism should be perceived, but the
direction of the wedge is ignored.
