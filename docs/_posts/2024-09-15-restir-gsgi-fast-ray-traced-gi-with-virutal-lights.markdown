---
layout: post
title:  "ReSTIR GSGI - Fast Ray Traced GI with Virtual Lights"
date:   2024-09-15 16:46:12 +0100
categories: jekyll update
---

![GSGI arch comparison](/img/gsgi/gsgi_arch_comparison.jpg)

Today I'm going to talk about a ray traced global illumination technique which I'm calling ReSTIR Geometry Sampling Global Illumination, or GSGI for short. It's based around the ReSTIR direct lighting algorithm, and aims to achieve the following goals:

- Very high speed
- Diffuse (Lambertian) ray traced GI
- Infinite bounces
- No restrictions on geometry
- Capable of handling arbitrarily large scenes
- Can be tuned for very fast lighting response
- Good enough quality

The last point is obviously subjective. The authors of the original ReSTIR paper have also created a technique called ReSTIR GI, which extends ReSTIR's resampling approach to global illumination. ReSTIR GI produces very high quality results, and can do so faster than previous solutions which offer similar quality, but still has a high performance cost for a real-time use case like a videogame.

ReSTIR GSGI uses a very different approach to ReSTIR GI, but I'll be comparing it to ReSTIR GI in both performance and quality. The goal is to achieve around an order of magnitude faster performance, making it viable for real-time use on mid-range hardware, while still achieving quality which is subjectively good enough.

## ReSTIR

First, we need a brief introduction to ReSTIR. A full description of how ReSTIR functions is well beyond the scope of this article, but there are some key concepts we'll need to understand for context. I'll include links below for anyone interested in learning more about it.

ReSTIR stands for Reservoir Spatio-Temporal Importance Resampling. It's a direct lighting algorithm, and what makes it so appealing is that it can efficiently calculate lighting (and shadowing) for an extremely large number of light sources, into the millions. The algorithm itself is $O(0)$ with respect to the number of light sources, which is to say its performance is completely independent of the number of lights, but memory limits do come into play, as you have to actually store all those lights in GPU memory, and there are some pre-processing steps which scale with the number of lights. Still, the real-world performance impact of moving from a single-digit number of light sources to hundreds of thousands is very low.

A key part of the ReSTIR algorithm is resampling. Here it reuses information from neighbouring pixels (spatial resampling) and the same pixel in previous frames (temporal resampling) in order to significantly improve the sample quality for each pixel. How the spatial and temporal resampling interact with GSGI will be important later.

## Geometry Sampling Global Illumination

The core concept of GSGI is to randomly sample points on the geometry in the scene, and then create virtual lights representing indirect illumination reflected off those points, which are then fed into the ReSTIR algorithm to calculate global illumination.

Using virtual lights to represent global illumination not a new concept, and has been used in the past in techniques such as Instant Global Illumination. Those previous approaches have usually been limited to a very low number of virtual lights, though, for performance reasons. With ReSTIR being able to handle a very large number of light sources with minimal performance impact, it's now feasible to use hundreds of thousands, or even millions of virtual lights in a real-time environment.

This can achieve a very high level of performance because there isn't a separate per-pixel GI pass. Virtual lights are fed into the same ReSTIR pass as is used for direct illumination, so both direct and indirect illumination are calculated as part of a single pass. The creation of virtual lights is independent of resolution, and good quality can be achieved with only 16K new virtual lights per frame, so performance can be far higher than any technique which requires non-trivial per-pixel calculations.

The technique also allows for an effectively infinite number of light bounces, as the virtual lights from previous frames can be included in the light sources used to calculate the light reflected off new samples. Each sample represents a single bounce, but as samples feed into each other they can represent more complex lighting paths.

## Radial Geometry Sampling

There are many potential approaches to sampling scene geometry. The one which I've settled on, which I'll call radial sampling, was chosen as it generates a higher sample density close to the camera, and a lower density further away. Specifically, it produces a sample density proportional to $1/d^2$, which means the sample density is proportional the size of objects on screen. This is perfect for large scenes, such as in open world games, as we can represent relatively fine-grained GI on objects close to the camera, with coarse-grained GI further away, all exactly in proportion to the size they would be on screen. This bias can then be easily corrected when generating the virtual lights.

Radial geometry sampling leverages ray tracing hardware, and works by casting rays in random directions from the location of the camera. We use anyhit shaders, which are typically used for transparency calculations, as they trigger for every piece of geometry hit by the ray. This is beneficial to us, as we want to randomly sample one of those surfaces for each ray.

To do this, we use reservoir sampling. For each surface we hit, we calculate a weight $W$ based on the angle of the surface to the ray, and we accept this surface as our sample with a probability of $W/W_s$, where $W_s$ is the sum of all weights (including the new one).

The specific weight we use is $1/cos(θ)$, where $θ$ is the angle between the ray and the surface normal(?). To illustrate why we use this weight, consider the following diagram:

(insert diagram)

Here the red dot is the camera position, the red line is the ray we're using to sample geometry, and the black lines are the pieces of geometry we hit. The blue lines are some angle around our ray.

As you can see from the diagram, the surface area within the angle around the ray for each piece of geometry is proportional to two things; the distance of the geometry from the camera position, and the angle of the geometry with respect to the ray. As the blue angle around the ray goes towards zero, the surface area for each sample becomes proportional to $d^2/cos(θ)$.

If we wanted perfectly uniform sampling in world space, then we could use $d^2/cos(θ)$ as our weight. However, if we use $1/cos(θ)$ as our weight instead, then we bias the sampling to the tune of $1/d^2$, as described above.

Once caveat of this approach to geometry sampling is that triangles which are close to colinear with the camera position (ie have a very tight angle to the camera) have a low probability of being sampled. Using $1/cos(θ)$ as the weight makes it very likely that these triangles will be sampled if they're hit by a ray, but the probability of them being hit could be very low. This could present itself as noise on light reflected from these surfaces. To counteract this, we can randomly displace the origin of the geometry sampling rays around the camera location. This doesn't introduce bias, but should help reduce this noise.

For each ray, we store details about the chosen geometry sampling in a G buffer, including the sum of weights, which are used to generate the virtual light.

## Creating Virtual Lights

We then perform a separate pass to calculate the lighting for each of these geometry samples. This is based on the ReSTIR DI light sampling, and generates initial samples in largely the same way as ReSTIR DI does. Resampling can be performed here, and this will be discussed in more detail later, but for the purposes of my example code one resampling technique is used, called screen space resampling, which re-uses reservoirs from ReSTIR DI for any geometry samples which are visible on screen. Ray traced visibility testing is then used on the chosen sample.

Once a source light sample is chosen, diffuse lighting is calculated from that source light to the geometry sample, and a virtual light is created at that point to represent the diffuse reflected light. Although you could represent these as point lights, for my example code I've used disk lights, as it reduces noise, as we can increase the radius in proportion to the distance from the camera, which corrects for the $1/d^2$ bias introduced by the radial sampling.

The radiance of the virtual light is scaled according to the sampling density and the sum of the weights for the ray used to generate that sample.

(equation)

Unless instant response to lighting changes is required, each virtual light created would usually have a lifespan of multiple frames before being replaced. So, for example if the lifespan is 10 frames, only 10% of virtual lights would be replaced with newly sampled lights each frame. This increases performance, as fewer new virtual lights need to be generated each frame, at the expense of slower response to lighting changes. As will be discussed in more detail later, a longer lifespan also allows ReSTIR DI's temporal resampling to be more effective.

## Implementation

For my example implementation, I've built on a fork of Nvidia's RTXDI, which you can find on github here. This saved me from having to reimplement ReSTIR from scratch, and also allows me to perform direct comparisons to ReSTIR GI, which is also included in RTXDI.

All code was run on a system with a desktop RTX 3070 GPU, and were run at 1080p, with the default settings with the RTXDI sample app (except where otherwise noted). This includes NRD denoising, and DLSS (with both input and output resolutions of 1080p). The following GSGI examples are using 16,384 samples per frame, with a virtual light lifespan of 30 frames, resulting in just under 500k virtual lights being used. ReSTIR GI uses the default RTXDI settings.



