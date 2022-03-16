# Circles in Snowflake

This repository contains two methods for generate circle objects in Snowflake from a given center location and radius. Snowflake's understanding of spatial data is based on geographical longitude and latitude instead of a standard 2-dimensional coordinate system. This is to support geospatial mapping and has many advantages, including being able to directly display any spatial objects in software such as Tableau on a geographical map.

The blog post describing the process behind this code is available [here](...) <TODO: Add link>

## General Approach

Regardless of which of the two methods you wish to you leverage (described below), the overall approach is the same. Our goal is to generate a number of points along the boundary of our circle, then connect these points to create an approximation of the circle itself. The image below demonstrates how our circle approximation becomes more accurate as the number of points is increased.

![Circle Approximation Accuracy](assets\circle-accuracy.png)

## Generating Circles on a Flat Two-Dimensional Plane

If you are not mapping your circles onto a geospatial map and instead wish to treat your canvas as a simple two-dimensional plane, you can calculate points on circles with a relatively simple formula.

### Mathematical Approach for Flat Planes

Consider a circle with center `(a, b)` and radius `r`, as in the image below.

![Flat Circle](assets\flat-circle.png)

Within these contraints, point `(x, y)` can be calculated with the following formula:

```math
x = a + rcos(θ)
```

```math
y = b + rsin(θ)
```

If we ensure all of our values are small enough to fit on a relatively flat section of the Earth, for example by dividing all of our coordinates and distances by 10,000, then we can reasonably assume that our canvas is a flat two-dimensional plane and plot our circles against it.

## Generating Circles on a Geospatial Surface (such as Earth)

If you wish to map your circles onto a geospatial surface such as the surface of Earth, the formula becomes more complex. We must now consider the curviture of the sphere whilst calculating the points on our circle, and can expect some odd-looking oval-shaped results when viewing our circles on a [Mercator projection](https://en.wikipedia.org/wiki/Mercator_projection), which is how we typically view the world on a map.

### Mathematical Approach for Geospatial Mapping

Consider a starting point with latitude `φ1` and longitude `λ1`, both calculated in radians. We wish to travel a distance `d` in a specific direction. In geospatial environments, the direction of travel is referred to as the bearing and is measured as the angle `θ` in radians travelling clockwise from the northward meridian to the direction of travel. We must also know the Earth's radius `R`.

First, we calculate the angular distance `δ`, which is the internal angle at the planet's core between our origin and desired destination.

```math
δ = d/R
```

Now that we know the angular distance, we can calculate the destination location with latitude `φ2` and longitude `λ2` in radians, using the following formulae:

```math
φ_2 = asin(sin(φ_1)cos(δ) + cos(φ_1)sin(δ)cos(θ))
```

```math
λ_2 = λ_1 + atan2(sin(θ)sin(δ)cos(φ_1), cos(δ) - sin(φ_1)sin(φ_2))
```

### Challenges with Geospatial Mapping

This gets us most of the way there, however we still face two issues.

#### Boundary for Longitude

The longitude must fit within the boundaries of `-2π <= λ2 <= 2π`. To resolve this, we can apply the following formula:

```math
((λ_2 + 3π) \% 2π) - π
```

In degrees instead of radians, our goal is to stay between -180° and 180°. This can be achieved with the following converted formula:

```math
((lat_2 + 540) \% 360) - 180
```

#### Rendering Across the Antimeridian

Like many tools, Snowflake leverages the GeoJSON data format to store geospatial data. GeoJSON faces a challenge when shapes cross the antimeridian, where East becomes West. When this occurs, rendering will loop all the way around the world instead of crossing the antimeridian.

The image below demonstrates how this can fail to render in Tableau (orange), compared to the desired intention (blue).

![GeoJSON Rendering Across the Antimeridian Without Cutting](assets\GeoJSON-antimeridian-crossing-example.png)

To correct this, we must cut our shape exactly at the antimerdian boundary, adding a new point to our circle each time it crosses -180° or 180° longitude. As our circle is an approximation based on a number of points, it is acceptable for us to simply apply a standard `y = mx + c` approach here to find the equation for the line that crosses the antimeridian and add a point where `x = 180` or `x = -180`.

Unfortunately, I have not been able to find a way to achieve this successfully in Snowflake at this time. I do not believe Snowflake's current GeoJSON functionality supports constructing polygons that are formed of MultiLineString object types. For now, my recommendation is to avoid using Snowflake for geospatial mapping when you know you will cross the antimeridian.

#### Credit

The variant of the Haversine formula that we have used to calculate these points was sourced from [Movable Type Scripts](https://www.movable-type.co.uk/scripts/latlong.html).
