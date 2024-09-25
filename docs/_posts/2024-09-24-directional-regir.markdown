---
layout: post
title: "Directional ReGIR"
date: 2024-09-24 12:00:00s +0100
---

When rendering scenes with a large number of lights, efficiently sampling those light sources is an important consideration. Uniform random sampling is unlikely to give good results, as for a given surface a large proportion of lights in the scene may provide little to no illumination. Therefore, there exist several techniques for light sampling which aim to efficiently sample light sources which provide meaningful lighting to a given point.

We'll be looking at an existing sampling algorithm called ReGIR, and a modification of that algorithm I'm proposing called Directional ReGIR. The goal of Directional ReGIR is to incorporate the direction of the light source to the surface being shaded in the sampling algorithm. By taking into account the direction of the light sources, it would be possible to select light sources by sampling directions from the BRDF of the surface. In particular, this should achieve more efficient sampling of light sources contributing to specular lighting.

## ReGIR

ReGIR is a grid-based presampling algorithm. It divides the scene up into a grid of cells, and for each cell presamples lights according to their light contribution across the volume of the cell. For each pixel, samples are then taken from those cells.

ReGIR doesn't depend on any particular grid arrangement, but for the purposes of these examples, I'm going to use the Onion structure implemented in the RTXDI sample app. This builds cells in layers centred around the camera location, with smaller cells closer to the camera, and larger cells further away.

Here's an example of the cells visualised in a sample scene:

![ReGIR cell visualisation](/img/dirregir/regir_cells_visualisation.jpeg)

Each ReGIR cell is divided into some number of reservoirs, or slots, and for each reservoir a number of light samples are taken and a sample is chosen using reservoir sampling. For example, if ReGIR is configured to have 256 slots per cell, and 8 samples per slot, then 2048 total samples are required to build each cell.

When a pixel samples from those ReGIR cells, first a random jitter is applied to its position, to avoid discretisation artifacts. Then, samples are taken from the ReGIR cell at that jittered position, choosing one of the reservoirs randomly per sample. If 8 initial samples are taken per pixel, and ReGIR is built using 8 samples per slot, then 64 total light samples are being considered per pixel, before spatial or temporal resampling.

## Directional ReGIR

### Motivation

One limitation of the standard ReGIR implementation is that the direction of light samples isn't taken into account. Light samples are weighted based on their illumination over the volume of the cell. For diffuse lighting, this will be a reasonably good approximation of the lighting on a given pixel within the cell. Even for specular illumination with a relatively low number of brighter light sources, the probability of a pixel picking up a sample for a light source providing meaningful specular illumination is quite high, particularly after resampling.

In my post on GSGI, though, I showed a case with a very large number of relatively dim virtual light sources being used for global illumination. Specular reflections of those light sources were a source of noise and boiling artifacts, in part because the probability of those light sources being sampled for pixels where they provide specular illumination is very low.

![GSGI specular comparison](/img/gsgi/gsgi_counter_comparison.jpg)

In the rightmost image above, we can see the issues with specular reflections of a very large number of virtual light sources.

If we consider that the GSGI example involved approximately 500K virtual lights spread across the scene, with only a very small proportion of them providing meaningful specular lighting to a given pixel. With only 64 samples being used for initial sampling for each pixel via ReGIR, the probability of sampling one of these relevant virtual lights is very low. Even with spatial and temporal resampling increasing the effective number of samples, pixels are still very unlikely to sample one of these specular light sources.

A solution to this would be to consider the direction of light sources when sampling. For specular surfaces like the underneath of the counter in the example above, we can easily identify the direction in which we would want to find light sources by sampling the BRDF of the surface.

### Algorithm

Directional ReGIR uses the same cell structure as the existing ReGIR algorithm, but changes how samples are stored within those cells. While every reservoir in a cell in ReGIR is populated independently, and all reservoirs are interchangeable, in Directional ReGIR each slot in a cell represents a specific direction.

To do this, we organise reservoirs in the cell in a 2D grid, of 16x16 slots. Directions are mapped to positions in this grid using octahedral mapping, which is a technique which maps directions on the unit sphere to positions within a square by treating the sphere as an octahedron, and unfolding it into a square on a 2D plane.

![Octahedral mapping](/img/dirregir/octahedral_mapping.png)

Octahedral mapping is commonly used for efficient storage of surface normals (for example in G buffers), and should be both quick and accurate enough for our purposes.

When populating the grid, we use the following algorithm:

- Select random location in the cell $L_c$
- Generate sample from random light source
- Calculate light contribution for that light sample to a hypothetical surface at $L_c$
- Convert direction from $L_c$ to the light sample to a location in the grid using octahedral mapping
- Use reservoir sampling to add that light sample to the reservoir at that grid position

Optionally, some jitter can be applied to the grid position before adding to a reservoir.

When sampling for a given pixel, a direction can be sampled from the BRDF for the surface, and then a sample can be taken from the location on the grid corresponding to that location.

### Implementation

I implemented Directional ReGIR on a fork of Nvidia's RTXDI repo, in which I've also implemented some other work, such as GSGI. Code for my implementation can be found [here](https://github.com/otrooney/RTXDI).

The major challenge involved in implementing this algorithm is that it involves multiple threads potentially writing to the same location in memory. GPUs are generally not well optimised for handling contention between multiple threads over a single location in memory, so some thought must be put into an implementation which is thread safe without a significant performance impact.

My approach to implementing Directional ReGIR involves group shared memory and atomic functions. Group shared memory is memory shared by all threads in a workgroup, located close to the execution units (ie. within the SM on Nvidia GPUs, or within the CU on AMD GPUs). Group shared memory can be accessed with much lower latency than VRAM.

Atomic functions are shader functions which are guaranteed to execute atomically as a single operation. For example, without atomic functions, if a thread performed a read from a memory location, modified the contents, and then wrote the result back to the same memory location as a separate operation, then another thread may have changed the contents between the read and write operations. Atomic functions can perform a read and write in a single operation, which can be useful for implementing thread safe code.

Atomic functions can be very expensive, but the cost of using them on shared memory is much lower, due to the reduced latency compared to VRAM. This means that, if we can keep all relevant data in shared memory, we can use atomic functions as part of a thread-safe way of managing access between threads with minimal performance impact.

In particular, my implementation for populating the grid involves five arrays of group shared memory:

```
groupshared float weightSumArray[16][16];
groupshared uint lightIndexArray[16][16];
groupshared uint nSamplesArray[16][16];
groupshared float selectedTargetPdfArray[16][16];
groupshared uint accessLockArray[16][16];
```

These arrays are all 16x16, matching the grid, and their size is part of the reason a 16x16 grid was chosen, as it allows them to fit in shared memory without limiting the number of active workgroups (at least on the GPU I'm using to test). The `accessLockArray` is used to manage access to the shared memory, and the others store the required data for each location on the grid.

When a sample has been generated, the atomic function `InterlockedCompareExchange` is used to check and set the lock for that grid location before updating the other values in shared memory.

This implementation has a relatively minor performance impact compared to standard ReGIR, as we'll see later.

When sampling from a cell using Directional ReGIR, I have implemented several options, including uniform hemisphere sampling and BRDF sampling. The BRDF sampling option can also mix in a proportion of uniform hemisphere samples.

### Results

As our first comparison point, let's look at a screenshot taken using a standard ReGIR implementation, with 256 slots per cell and 8 samples per slot. Here we're using ReSTIR with both temporal and spatial resampling, and denoising and DLSS (with input and output resolutions both set to 1080p). The examples are all running on an RTX 3070 desktop graphics card.

![ReGIR standard settings](/img/dirregir/regir_256_denoised_jitter.jpeg)

Next, we'll look at a screenshot with identical settings, but switching to Directional ReGIR:

![Directional ReGIR standard settings](/img/dirregir/directional_regir_denoised_jitter.jpeg)

Something obviously isn't quite right here. The floor at the bottom of the image is properly illuminated, but other areas of the image, including the back wall, are very dark.

For a more direct comparison, let's remove ReSTIR's resampling, and the denoising and DLSS steps, so we're just looking at the initial samples generated by ReGIR and Directional ReGIR. Here's ReGIR:

![ReGIR initial samples](/img/dirregir/regir_256_initial_samples_jitter.png)

And here's Directional ReGIR:

![Directional ReGIR initial samples](/img/dirregir/directional_regir_initial_samples_jitter.png)

We can see that Directional ReGIR seems to be producing good initial samples pretty consistently on the floor at the bottom of the image, but fewer good samples on other areas. This is particularly noticeable on the back wall, where ReGIR is able to generate samples about as well as anywhere else on the image, but Directional ReGIR generates almost no good samples.

As far as I can tell, what's happening here is relating to the jitter discussed earlier. When taking an initial sample for a pixel, ReGIR doesn't necessarily use the cell it's in, but instead applies a random jitter to the pixel's world space location and chooses a cell based on that jittered position. This reduces discretisation artifacts (ie a visible grid structure on the screen), and works well for ReGIR, as a light which provides meaningful illumination to one cell probably provides meaningful illumination to neighbouring cells.

Directional ReGIR complicates this, though, because we rely on the direction of lights with respect to points in the cell, and moving the point changes the direction of the light relative to that point.

![Directional light sampling with sample location jitter](/img/dirregir/directional_light_sampling_jitter.png)

For a relatively close light, applying jitter to the position we're sampling from can cause us to miss relevant lights, as the direction of the light relative to the new position can change substantially. For distant lights, though, jitter has less of an effect, as the direction only changes slightly.

This is the reason Directional ReGIR produces good samples for the floor in the image above, as the light sources are distant enough that the jitter being applied doesn't prevent the directional sampling from picking up relevant light samples. For other surfaces, though, they are mainly illuminated by close light sources, which are much less likely to be sampled for pixels after jitter is applied.

To illustrate, let's set the jitter to zero, and look at the same scene with just the initial samples again. First, here's standard ReGIR:

![ReGIR initial samples without jitter](/img/dirregir/regir_256_initial_samples_no_jitter.png)

And here's Directional ReGIR:

![Directional ReGIR initial samples without jitter](/img/dirregir/directional_regir_initial_samples_no_jitter.png)

The first thing we see is clear artifacts of the cell structure in both images, illustrating why jitter is applied in the first place.

Secondly, we see in the second image that by removing jitter, Directional ReGIR can produce much better samples for areas that were dark before, like the back wall. In fact, if we define a good sample as one which isn't occluded from its light source, then Directional ReGIR is producing more good samples than standard ReGIR here.

The hit percentage for rays during initial sampling (which are occlusion test rays) is 92% for ReGIR and 89% for Directional ReGIR, so Directional ReGIR is resulting in about 37% more good samples which aren't occluded. The opposite is true with the default amount of jitter applied, where standard ReGIR produces more good samples than Directional ReGIR.

Subjectively, we also seem to be getting better samples for lights contributing to specular lighting with Directional ReGIR.

The challenge here is that, without jitter, discretisation artifacts are clearly visible on screen, but with jitter, Directional ReGIR struggles to produce good results for the reasons described above.

One option, which has an obvious cost associated with it, is to use smaller cells (and therefore a larger number of them), potentially with an increased sample count. Let's look what happens when we use about 2x as many cells, and significantly increase the sample count, to 32 samples per thread during grid building and 32 initial samples per pixel (up from 8 in each case). First, here's standard ReGIR:

![ReGIR with small cells and increased sample counts](/img/dirregir/regir_small_cells_many_samples.png)

And here's Directional ReGIR:

![Directional ReGIR with small cells and increased sample counts](/img/dirregir/directional_regir_small_cells_many_samples.png)

Directional ReGIR benefits more from smaller cells and increased sample counts than standard ReGIR does. We can see an 89% hit rate during initial sampling for ReGIR, compared to an 81% hit rate for Directional ReGIR, indicating that Directional ReGIR produces about 73% more good (ie non-occluded) samples than standard ReGIR with these settings.

However, the performance impact is significant, with the Directional ReGIR grid building pass taking 5ms alone, along with a more expensive initial sampling pass. Furthermore, discretisation artifacts are still visible, and introducing jitter will still reintroduce the issues we saw above. It's possible that with sufficiently many small cells, and a very small amount of jitter, that Directional ReGIR may provide good results without discretisation artifacts, but it seems that the performance hit of doing so would likely to be too costly.

### Performance

The overall performance impact of Directional ReGIR, compared to standard ReGIR at similar settings, is relatively small. Building the Directional ReGIR grid only costs around an extra 0.1ms over standard ReGIR using default settings, despite the need to use atomic functions to manage memory contention.

There is some additional cost during the initial sample pass, which is partly due to the need to sample surface BRDFs during sampling. There is some scope for optimisation in the current implementation, though.

## Future Work

### Randomised Grid Building

The ReGIR implementation provided in the RTXDI code that I'm using for this example builds the physical structure of the grid in a deterministic way. That is, the cells will always occupy the same positions relative to the camera from one frame to the next.

It could be beneficial to apply some randomisation to this process. If cell locations varied in a random manner from one frame to the next, then discretisation artifacts (boundaries between cells) should appear in different locations every frame, and ReSTIR's temporal resampling should significantly reduce their visibility due to sample reuse across multiple frames. It's possible that jitter could then be reduced significantly, without artifacts being noticeable after spatial and temporal resampling.

### Bias Correction

One topic I didn't touch on above is that, in my current implementation, Directional ReGIR likely introduces some amount of bias, as we're no longer uniformly sampling light samples, but instead preferentially sampling them based on direction.

This shouldn't be too difficult to correct for, as we can calculate the PDF of each of the different sampling techniques, and apply the appropriate correction when sampling from Directional ReGIR. However, due to the issues shown above I didn't get around to implementing bias correction, so that's something to add in the future.

## Closing Comments

Although the current implementation of Directional ReGIR isn't quite capable of replacing standard ReGIR, I do feel that, with further work, it could be a good solution for light sampling. In particular, randomising cell locations during grid building seems like a good avenue for future work, as it could significantly reduce, or eliminate, discretisation artifacts with little to no jitter required, which could remove the major limitation of Directional ReGIR.


## Useful Links

- [Github repo containing my Directional ReGIR implementation](https://github.com/otrooney/RTXDI)
- [Rendering Many Lights with Grid-Based Reservoirs (ReGIR paper)](https://research.nvidia.com/labs/rtr/publication/boksansky2021rendering/)
- [Survey of Efficient Representations for Independent Unit Vectors (octahedral mapping paper)](https://jcgt.org/published/0003/02/01/)