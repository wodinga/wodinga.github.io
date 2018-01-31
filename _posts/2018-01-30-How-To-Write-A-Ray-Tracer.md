---
title: Ray Tracing In Swift
categories: [iOS, Ray Tracing]
tags: [math, first post, swift]
---

When learning new topics I like to dive right in and attempt to learn the hardest thing I possibly can. That's why one of the first projects I attempted in Swift was ray tracing. Ray tracing is the technique of rendering an image by calculating how light rays bounce around. The reason this is such an awesome method is because it allows ones to realistically model lights, shadows, mirrors, and material effects.

Normally, when graphics are rendered on a computer, they are rendered by a GPU using a process called rasterization. This is a fancy word for converting 3D graphics into a 2D image. This generally involves many tricks that may render lights, shadows, and reflections in a way that is good enough but not necessarily “realistic” (although looking at the latest video games that are coming out, I’d say we’re getting pretty close!). This process is fairly easy to parallelize, which is another reason it is done on your GPU.

Ray tracing, on the other hand, is the process of rendering an image by calculating the way that light rays bounce around a scene. At a basic level, this is a mostly serial method of rendering, which is why it is done by the CPU, which can perform serial calculations much faster and with greater accuracy than a GPU.

To create my ray tracer, I followed this [tutorial][swift ray tracer] which was really just a swift translation of the code and techniques found in this book by Peter Shirley, [Ray Tracing in One Weekend][minibook].

How I created my ray tracer
----------------------------

One of the main differences between my code and the code from Marius (the author of the [Metal Kit blog][swift ray tracer]) was his project was created as a Swift Playground and mine was a full Mac OS X application. The reason I did this was because I found it easier to debug my ray tracer. In Playgrounds you can see the results of your lines of code as they execute but you can't step through your code line by line.

The first thing I did was create a `struct` that contained the data for the RGB values of the pixels that were going to be displayed on the screen. Then I created a function that would write the pixels out to an array that stored their values.

Next, I wrote a function to actually draw this array to a `CIImage`, which is a surface that OS X uses to draw raster images to.

In this ray tracer we called represented the light rays with a struct called `ray` consisting of an **origin** vector and a **direction** vector which is a standard way of representing light.

To represent the vectors, I created a vector **struct**. I overloaded some operators to do vector math. I created this vector **struct** and the vector math functions by following the tutorial, but it turns out that there is a built-in vector type class called `double3` in the `simd` framework, which is a highly optimized vector math framework so I switched to that.

For messing around with ray tracers, the math is often a little easier use spheres because the hit equation works the same given a center. If the sphere is at the origin, the equation is $$x^2 + y^2 + z^2 = R^2$$ or $$(x-cx)^2 + (y-cy)^2 + (z-cz)^2 = R^2$$ when the center is at $$(cx, cy, cz)$$. 

```swift
public struct ray {
    public var origin: double3
    public var direction: double3
    public func point_at_parameter(t: Double) -> double3{
        return origin + t * direction
    }

    public init(origin: double3, direction: double3)
    {
        self.origin = origin
        self.direction = direction
    }
}
```

I represented the spheres in a class called `sphere`. Here's an "interface" look at this class.

```swift

public class sphere: hitable {
    var center = double3(x: 0.0, y: 0.0, z: 0.0)
    var radius = Double(0.0)
    let mat_ptr: material
    public init(c: double3, r: Double, m: material) {
        center = c
        radius = r
        mat_ptr = m
    }

    public func hit(r: ray, _ tmin: Double, _ tmax: Double, _ rec: inout hit_record) -> Bool
}
```

Basically the `hit` function returned true if the ray touched the sphere. Mathematically, we're just looking to see if the function of the ray and the function of the sphere has a root or two.

Our `world` consisted of several spheres so we put them all in a list called `hitable_list`. This had a `hit` function that went through the list of spheres in our world and checked if anything was hit. Whichever sphere was closest to our viewport is the one we drew and everything else was discarded.

Next we needed a function to compute the **color** based whether the `ray` hits a `sphere`. This could be determined with the following function. The `var scattered` and `var attenuation` were properties used by whatever the materials ended up being and changed the spheres from being matte or reflective depending.

```swift
public func color(r: ray, world: hitable, _ depth: Int) -> double3 {
    var rec = hit_record()
    if world.hit(r: r, 0.001, Double.infinity, &rec) {
        var scattered = r
        var attenuation = double3()
        //We either attentuate the ray and scatter it or we absorb it
        if depth < 50 && rec.mat_ptr.scatter(ray_in: r, rec, &attenuation, &scattered)
        {
            return attenuation * color(r: scattered, world: world, depth + 1)
        } else {
            return double3(x: 0, y: 0, z: 0)
        }
    }
    else {
        let unit_direction = normalize(r.direction)
        let t = 0.5 * (unit_direction.y + 1)
        return (1.0 - t) * double3(x: 1, y: 1, z: 1) + t * double3(x: 0.5, y: 0.7, z: 1.0)
    }
}
```

Also you'll a `var t` throughout the ray tracer. This represents the *time* in the simulation in which the ray meets the surface of the sphere.

In `public func imageFromPixels(_ width: Int, _ height: Int) -> CIImage` I looped through each pixel in the window to determine if its color by calling `color` and then performing some sampling to anti-alias the image and smooth out the colors a little bit.

Last, I drew this image into an `NSView` by overriding its `draw` function with

```swift
override func draw(_ dirtyRect: NSRect) {
        super.draw(dirtyRect)

        let context = NSGraphicsContext.current?.cgContext

        if let cgImg = image.cgImage {
            context?.draw(cgImg, in: rect)
        }
    }
```
Hope you enjoyed my very first real blog post! This was a really interesting project and I was so glad to have finished it because I had a project like this in college that I wasn't able to finish.

You can find the project [here][project].

[swift ray tracer]: http://metalkit.org/2016/03/21/ray-tracing-in-a-swift-playground.html
[minibook]: https://www.amazon.com/Ray-Tracing-Weekend-Peter-Shirley-ebook/dp/B01B5AODD8
[project]: http://github.com/wodinga/wodinga.github.io
