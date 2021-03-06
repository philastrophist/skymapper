[![PyPI](https://img.shields.io/pypi/v/skymapper.svg)](https://pypi.python.org/pypi/skymapper/)
[![License](https://img.shields.io/github/license/pmelchior/skymapper.svg)](https://github.com/pmelchior/skymapper/blob/master/LICENSE.md)

# Skymapper

*A collection of matplotlib instructions to map astronomical survey data from the celestial sphere onto 2D.*

The purpose of this package is to facilitate interactive work as well as the the creation of publication-quality plots with a python-based workflow many astronomers are accustomed to. The primary motivation is a truthful representation of samples and fields from the curved sky in planar figures, which becomes relevant when sizable portions of the sky are observed.

What can it do? For instance, find the optimal projection for a given list of RA/Dec coordinations and [creating a density map](examples/example1.py) from a catalog in a few lines:

```python
import skymapper as skm

# define the best Albers projection for the footprint
# minimizing the variation in distortion
crit = skm.stdDistortion
proj = skm.Albers.optimize(ra, dec, crit=crit)

# construct map: will hold figure and projection
# the outline of the sphere can be styled with kwargs for matplotlib Polygon
map = skm.Map(proj)

# add graticules, separated by 15 deg
# the lines can be styled with kwargs for matplotlib Line2D
# additional arguments for formatting the graticule labels
sep = 15
map.grid(sep=sep)

# make density plot
nside = 32
mappable = map.density(ra, dec, nside=nside)
cb = map.colorbar(mappable, cb_label="$n_g$ [arcmin$^{-2}$]")

# add footprint
map.footprint("DES", zorder=20, edgecolor='#2222B2', facecolor='None', lw=1)
```

![Random density in DES footprint](https://github.com/pmelchior/skymapper/raw/master/examples/example1.png)

For exploratory work, you can zoom and pan, also scroll in/out (google-maps style). The `map` will automatically update the location of the graticule labels, which are not regularly spaced.

The syntax mimics `matplotlib` as closely as possible. Currently supported are canonical plotting functions

* `plot`
* `scatter`
* `hexbin`
* `text` (with an optional `direction in ['parallel','meridian']` argument to align along either graticule)

as well as special functions

* `footprint` to load a RA/Dec list and turn it into a closed polygon
* `vertex` to plot a list of simple convex polygons
* `healpix` to plot a healpix map as a list of polygons
* `density` to create a density map (see example above)
* `interpolate` to interpolate given samples over their convex hull
* `extrapolate` to generate a field from samples over the entire sky or a subregion 

## Installation and Prerequisites

You can either clone the repo and install by `python setup.py install` or get the latest release with

```
pip install skymapper
```

Dependencies:

* numpy
* scipy
* matplotlib

For the healpix methods, you'll need `healpy`.

## Background

The essential parts of the workflow are

1. Creating the `Projection`, e.g. `Albers`
2. Setting up a `Map` to hold the projection and matplotlib figure, ax, ...
3. Add data to the map

Several map projections are available, the full list is stored in the dictionary `projection_register`. If the projection you want isn't included, open an issue, or better: create it yourself (see below) and submit a pull request.

Since most ground-based surveys have predominant East-West coverage, we suggest using conic projections, in particular the equal-area `Albers` conic (an discussion why exactly that one is [here](http://pmelchior.net/blog/map-projections-for-surveys.html)).

Map projections can preserve sky area, angles, or distances, but never all three. That means defining a suitable projection must be a compromise. For most applications, sizes should exactly be preserved, which means that angles and distances may not be. The optimal projection for a given list of `ra`, `dec` can be found by calling:


```python
crit = skm.projection.stdDistortion
proj = skm.Albers.optimize(ra, dec, crit=crit)
```

This optimizes the `Albers` projection parameters to minimize the variance of the map distortion (i.e. the apparent ellipticity of a true circle on the sky). Alternative criteria are e.g. `maxDistortion` or `stdScale` (for projections that are not equal-area).

### Defining a survey

To make it easy to share survey footprints and optimal projections, we can hold them in a common place. To create one you only need to derive a class from [`Survey`](skymapper/survey/__init__.py), which only needs to implement two methods:

* `getFootprint` to return a RA, Dec list of the footprint
* `getConfigfile` to return the full path of a pickled map configuration, created by `Map.save`. 

### Defining a projection

For constructing your own projection, derive from [`Projection`](skymapper/projection.py). You'll see that every projection needs to implement three methods: 

* `transform` to map from RA/Dec to map coordinates x/y
* `invert` to map from x/y to RA/Dec
* `contains` to test whether a given x/y is a valid map coordinate

If the projection has several parameters, you will want to create a special `@classmethod optimize` because the default one only determines the best RA reference. An example for that is given in e.g. `ConicProjection.optimize`.

### Limitation(s)

The combination of `Map` and `Projection` is *not* a [matplotlib transformation](http://matplotlib.org/users/transforms_tutorial.html). Among several reasons, it is very difficult (maybe impossible) to work with the `matplotlib.Axes` that are not rectangles or ellipses. So, we decided to split the problem: making use of matplotlib for lower-level graphics primitive and layering the map-making on top of it. This way, we can control e.g. the interpolation method on the sphere or the location of the tick labels in a way consistent with visual expectations from hundreds of years of cartography. While `skymapper` tries to follow matplotlib conventions very closely, some methods may not work as expected. Open an issue if you think you found such a case.

In particular, we'd appreciate help to make sure that the interactive features work well on all matplotlib backends.