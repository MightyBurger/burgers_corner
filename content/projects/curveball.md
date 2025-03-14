+++
title = "Curveball"
description = "Curve generator for Neverball"
weight = 1

[extra]
local_image = "/projects/curveball.png"
read_time = true
# link_to = "https://github.com/MightyBurger/curveball"
+++

Curveball is my curve generator tool for [Neverball].

![curveball logo](/projects/curveball.png)

You can use Curveball [on the web](https://curveball.mightyburger.net)!

The source code is available on [Github](https://github.com/MightyBurger/curveball). You can find some more screenshots there.

If you'd rather run Curveball as a desktop app, you can download a release from Github or compile it yourself. Don't worry - compiling Curveball is easy, especially if you already have Rust.

Oh yeah. This thing is written in Rust 🦀. That means it must be good, right?

# Curve madness

Why Curveball? I'm working on a level set for an open-source game called [Neverball], and I needed some fancy shapes.

Neverball already has [a curve generating tool](https://github.com/Neverball/neverball/blob/master/contrib/curve.c) called `curve.c`. It was good enough for all the existing Neverball level sets (except Nevermania - [fwp](https://github.com/fwp) must have written a couple of his own scripts). But `curve.c` only generates one kind of curve: a circular arc. Granted, it gives you a lot of knobs to turn.

I needed curves `curve.c` couldn't make, so I wrote some scripts to fill the gaps.

I later had an idea to combine these random scripts into one tool and have a visualizer for it, like [curve.js](https://play.neverball.org/curve.js/). I started working on it and got a little carried away. The final result is Curveball!

# How does it work?

My early scripts were very basic, but I gradually found neat abstractions to make it better and less cumbersome.

## Ask for the universe

Neverball levels are made up of many "brushes". Each brush is a little piece of geometry.

Brushes are defined in a strange way: each brush is an *intersection of halfspaces*. That sounds intimidating if you haven't heard of it before, so let me explain.

Imagine you form a brush like this: first, let your brush take up the entire universe. Then, cut the universe in half over and over again until you're left with a little piece.

It's bizarre, but there's technical reasons why it is done this way, such as making the collision code easier.

Now we need some way to describe these shapes in a text file. Neverball parses the [Quake map format](https://quakewiki.org/wiki/Quake_Map_Format), where each "cut" is defined by three points in the plane. These points have to be in a certain order; if you get it wrong, you might accidentally cut away the wrong half of the universe!

If you're curious, here's an example of the data for a cube:
```QUAKE
{
"classname" "worldspawn"
// brush 0
{
( 0 0 64 ) ( 0 64 64 ) ( 64 0 64 ) mtrl/invisible 0 0 0 0.5 0.5 0 0 0
( 0 0 0 ) ( 0 0 64 ) ( 64 0 64 ) mtrl/invisible 0 0 0 0.5 0.5 0 0 0
( 0 0 64 ) ( 0 0 0 ) ( 0 64 64 ) mtrl/invisible 0 0 0 0.5 0.5 0 0 0
( 64 0 64 ) ( 64 64 64 ) ( 64 64 0 ) mtrl/invisible 0 0 0 0.5 0.5 0 0 0
( 64 64 0 ) ( 64 64 64 ) ( 0 64 64 ) mtrl/invisible 0 0 0 0.5 0.5 0 0 0
( 0 0 0 ) ( 64 0 0 ) ( 64 64 0 ) mtrl/invisible 0 0 0 0.5 0.5 0 0 0
}
}
```

The points are in the parenthesis. Everything after that is related to textures.

## Convexity

Unfortunately, the way brushes work means every single brush has to be [convex](https://en.wikipedia.org/wiki/Convex_polytope). Yet, most curves are not convex. So, curves generally need to be made up of a bunch of smaller brushes.

## An easier way

`curve.c` and my early scripts just carefully chose the points manually. This is really error prone and a bit of a pain. It turns out our problem is the kind a [convex hull algorithm](https://en.wikipedia.org/wiki/Convex_hull_algorithms) can solve. I found the [chull crate](https://crates.io/crates/chull), which implements a convex hull algorithm in Rust. Now I can give `chull` a bunch of points and let it figure out how to make my shape out of them. Cool!

This made it easy to throw together new curve generators quickly. However, it also made assigning textures to each face painful. [Trenchbroom](https://trenchbroom.github.io/) makes it quick to paint a texture on lots of faces, so I opted not to worry about this.

Using `chull` made it easy to generate these curves just by defining the vertices. But there was still more improvement to be had.

## The key idea: extrusions

I discovered a really neat abstraction for generating these curves. It was inspired by mechanical CAD software like [Solidworks](https://www.solidworks.com/). The idea is you *define a 2D profile* that you *extrude into 3D space*.

In Curveball, the 2D profile is copied multiple times along the path. Curveball then takes all the vertices in two adjacent faces and runs it through `chull` to produce a brush. Repeat until you reach the end of the path, and the curve is made.

This feels really natural to generate shapes with, and it opens up a world of options. Every combination of *profile* and *path* makes a new curve. Some are really weird and probably aren't useful, but there's a lot of interesting, useful curves you can make with it. If I code up $m$ different profiles and $n$ different paths, now I can produce $m \cdot n$ different curves! Manually coding every permutation would have taken ages.

## Orienting

An interesting problem to solve is rotating the profile along the path. This is especially natural for the *revolve* path, but it could be useful for other paths.

I ran into trouble programming this. My initial attempts failed because they all had the same misunderstanding. It turns out the *point* of the path and the *direction* you are headed isn't enough information to know how you should rotate! There's a degree of freedom I was missing: torsion.

I eventually stumbled upon the [Frenet frame](https://en.wikipedia.org/wiki/Frenet%E2%80%93Serret_formulas). If I can define $\mathbf{T}$, $\mathbf{N}$, and $\mathbf{B}$ vectors at any point along the curve, then I simply multiply the matrix $\left[ \mathbf{T}, \mathbf{N}, \mathbf{B} \right]$ by the point to rotate that point.

You can use interesting techniques like integrating the [Frenet-Serret formulas](https://en.wikipedia.org/wiki/Frenet%E2%80%93Serret_formulas) or finding the [rotation-minimizing frames](https://medium.com/intuition/lockdown-geometry-rotation-minimizing-frames-ff373d2f355b) to find these vectors, but the paths in Curveball are all simple enough it's easy to just define the three vectors directly, so that's what I did.

Rotating the profile along the path isn't always the desired behavior, so I made it optional in Curveball.

This is the function signature:

```rust
pub fn extrude<PRF, PTH>(
    n: u32,
    profile: &PRF,
    path: &PTH,
    profile_orientation: ProfileOrientation,
) -> CurveResult<Vec<Brush>>
where
    PRF: Profile,
    PTH: Path,
```

Not too bad! A `Profile` is something that produces a `Vec<Brush>`, possibly varying along the path. A `Path` is something that produces a point and a Frenet Frame that varies to produce the path.

## Exceptions to the rule

In addition to an **Extrusion**, curveball can also generate three other curves I call **Curve Classic**, **Curve Slope**, and **Rayto**.

**Curve Classic** *can* in fact be generated as an extrusion. This is because `extrude()` can accept 2D profiles that vary along the path. I opted to implement it separately to make the interface similar to `curve.c`.

**Rayto** can also be generated as an extrusion. You just have to make really clever use of the functionality I describe above. That would be a pain, so instead I implement it directly.

**Curve Slope** is the only curve that cannot be generated by `extrude()`. The reason is it generates brushes differently. When you make this curve in Curveball, you'll notice the brushes look like triangular pieces.

# What's next?

It's time to use this tool! I have some levels I paused because I needed a certain curve shape. Now I can go forward and make some fun levels.

I'm excited to see what the Neverball community comes up with using this tool.

[Neverball]: https://neverball.org/
