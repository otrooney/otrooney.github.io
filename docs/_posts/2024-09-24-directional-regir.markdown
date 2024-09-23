---
layout: post
title: "Directional ReGIR"
date: 2024-09-24 12:00:00s +0100
---

When rendering scenes with a large number of light sources, efficiently sampling those light sources is an important consideration. Uniform random sampling is unlikely to give good results, as for a given surface a large proportion of lights in the scene may provide little to no illumination. Therefore, there exist several techniques for light sampling which aim to efficiently sample light sources which provide meaningful lighting to a given point.

We'll be looking at an existing sampling algorithm called ReGIR, and a modfication of that algorithm I'm proposing called Directional ReGIR. The goal of Directional ReGIR is to incorporate the direction of the light source to the surface being shaded in the sampling algorithm. By taking into account the direction of the light sources, it would be possible to select light sources by sampling directions from the BRDF of the surface. In particular, this should achieve more efficient sampling of light sources contributing to specular lighting.

## ReGIR

ReGIR is a grid-based presampling algorithm. It divides the scene up into a grid of cells, and for each sell presamples light samples according to their light contribution across the volume of the cell. For each pixel, samples are then taken from those cells.

ReGIR doesn't depend on any particular grid arrangement, but for the purposes of these examples, I'm going to use the Onion structure implemented in the RTXDI sample app. This builds cells in layers centered around the camera location, with smaller cells closer to the camera, and larger cells further away.

Here's an example of the cells visualised in a sample scene:

![GSGI arch comparison](/docs/img/dirregir/regir_cells_visualisation.png)

Each ReGIR cell is divided into some number of reservoirs, or slots, and for each reservoir a number of light samples are taken and a sample is chosen using reservoir sampling. For example, if ReGIR is configured to have 256 slots per cell, and 8 samples per slot, then 2048 total samples required to build each cell.

When a pixel samples from those ReGIR cells, first a random jitter is applied to its position, to avoid discretization artifacts. Then, samples are taken from the ReGIR cell at that jittered position, choosing one of the reservoirs randomly per sample. If 8 initial samples are taken per pixel, and ReGIR is built using 8 samples per slot, then 64 total light samples are being considered per pixel, before spatial or temporal resampling.

## Directional ReGIR

### Motivation

One limitation of the standard ReGIR implementation is that the direction of light samples isn't taken into account. Light samples are weighted based on their illumination over the volume of the cell. For diffuse lighting, this will be a reasonably good approximation of the lighting on a given pixel within the cell. Even for specular illumination with a relatively low number of brigher light sources, the probability of a pixel picking up a sample for a light source providing meaningful specular illumination is quite high, particularly after resampling.

In my post on GSGI, though, I showed a case with a very large number of relatively dim virtual light sources being used for global illumination. Specular reflections of those light sources were a source of noise and boiling artifacts, though, in part becuase the probability of those light sources being sampled for pixels where they provide specular illumination is very low.

![GSGI pole comparison](/docs/img/gsgi/gsgi_counter_comparison.jpg)

In the rightmost image above, we can see the issues with specular reflections of a very large number of virtual light sources.

If we consider that the GSGI example involved approximately 500K virtual lights spread across the scene, with only a very small proportion of them providing meaningful specular lighting to a given pixel. With only 64 samples being used for initial sampling for each pixel via ReGIR, the probability of sampling one of these relevant virtual lights is very low. Even with spatial and temporal resampling increasing the effective number of samples, pixels are still very unlikely to sample one of these specular light sources.

A solution to this would be to consider the direction of light sources when sampling. For specular surfaces like the underneath of the counter in the example above, we can easily identify the direction in which we would want to find light sources by sampling the BRDF of the surface.

### Algorithm

Directional ReGIR uses the same cell structure as the existing ReGIR algorithm, but changes how samples are stored within those cells. While every reservoir in a cell in ReGIR is populated independently, and all reservoirs are interchangeable, in Directional ReGIR each slot in a cell represents a specific direction.

To do this, we organise reservoirs in the cell in a 2D grid, of 16x16 slots. Directions are mapped to positions in this grid using octahedral mapping, which is a technique which maps directions on the unit sphere to positions within a square by treating the sphere as an octahedron, and unfolding it into a square on a 2D plane.

(insert diagram)

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

Atomic functions are shader functions which are guaranteed to execute atomically as a single operation. For example, if a thread performed a read from a memory location, modified the contents, and then wrote the result back to the same memory location as a separate operation, then another thread may have changed the contents between the read and write operations. Atomic functions can perform a read and write in a single operation, which can be useful for implementing thread safe code.

Atomic functions can be very expensive, but the cost of using them on shared memory is much lower, due to the reduced latency compared to VRAM. This means that, if we can keep all relevant data in shared memory, we can use atomic functions as part of a thread-safe way of managing access between threads with a minimal performance impact.

In particular, my implementation for populating the grid involves five arrays of group shared memory:

```
groupshared float weightSumArray[16][16];
groupshared uint lightIndexArray[16][16];
groupshared uint nSamplesArray[16][16];
groupshared float selectedTargetPdfArray[16][16];
groupshared uint accessLockArray[16][16];
```

These arrays are all 16x16, matching the grid, and their size is part of the reason a 16x16 grid was chosen, as it allows them to fit in shared memory without limiting the number of active workgroups (at least on the GPU I'm using to test). The `accessLockArray` is used to manage access to the shared memory, and the others store the required data for each location on the grid.

When a sample has been generated, the atomic function `InterlockedCompareExchange` is used to check and set the lock for that grid location before updating the values in shared memory.

```
for (uint j = 0; j < MAX_LOCK_ATTEMPTS; j++)
{
    InterlockedCompareExchange(accessLockArray[arrayLoc.x][arrayLoc.y], LOCK_OPEN, threadIndex, lockValue);
    if (lockValue == LOCK_OPEN)
    {
        weightSumArray[arrayLoc.x][arrayLoc.y] += risWeight;

        if (risRnd * weightSumArray[arrayLoc.x][arrayLoc.y] < risWeight)
        {
            lightIndexArray[arrayLoc.x][arrayLoc.y] = rndLight;
            selectedTargetPdfArray[arrayLoc.x][arrayLoc.y] = targetPdf;
        }
        nSamplesArray[arrayLoc.x][arrayLoc.y] += 1;
        accessLockArray[arrayLoc.x][arrayLoc.y] = LOCK_OPEN;
        break;
    }
}
```

This implementation has a relatively minor performance impact compared to standard ReGIR, as we'll see later.

