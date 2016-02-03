---
layout: post
title: "Volumetric lighting 1"
categories: graphics
comments: true
---

# Volumetric lighting 1 - A naive approach

About a year ago I experimented with implementing volumetric lighting[^volumetric_lighting] using ray marching[^ray_marching]. I had gotten the basic idea from a lecture and liked the simplicity of the algorithm. My implementation back then was not particularly clever though, and since then I have kept getting interesting ideas on how to improve upon it. This series of post will explore the basic algorithm and various improvements I have made to it.

![volumetric lighting example](/assets/posts/2016-01-31-volumetric-lighting-1/volumetric_lighting_example.png)


## Basic algorithm

To reiterate, I'm going to focus on a ray-marching based algorithm for approximating volumetric lighting. While there also exist (likely faster) methods for approximating it in [screen space](http://http.developer.nvidia.com/GPUGems3/gpugems3_ch13.html), this approach should be a bit more accurate. It also has the benefit of allowing for textured lights[^textured_light]. Most importantly I think it is neat because it is a very intuitive algorithm.

![basic algorithm](/assets/posts/2016-01-31-volumetric-lighting-1/basic_algo.jpg)

So, how does it work? Essentially it is an extension of the normal shadow mapping algorithm. Instead of only sampling[^shadow_map_sampling] the shadow map at the position of the object we also sample it a number of times in the area between the camera and the object. This is illustrated in the image above. If we take the average of all these samples we get a value in the range [0, 1] for how much of the air between the camera and object is in light. Some pseudocode below:

	sum = 0
	for each sample point p on the ray to the object
		s = sample shadow map with p
		// s = 0: in shadow
		// s = 1: in light
		sum += s
	sum /= number of samples


## Weight problems

My description of the basic algorithm above sounds simple enough and it was more or less what my old implementation was doing. In many cases it also looks perfectly fine, but if we take a step back and think about it it stops making sense. If we just average all the samples we assume that each sample contributes the same amount to the final image, i.e. a sample 1000 meters away is as important as a sample 1 meter away. It gets even worse when we consider the fact that the range we are looking at changes depending on how far the away the object on the screen is.

![wrong result](/assets/posts/2016-01-31-volumetric-lighting-1/wrong_result.jpg)

In the image above we consider two pixels, A and B. Pixel A should probably be brighter than B as it has more light on it (from two light sources). In the current model pixel B will actually be brighter than A as a larger percentage of the distance between the camera and the object is in light. In order to do this properly we clearly need a model to apply different weights to the samples somehow, which is the first improvement of my new implementation. I decided on a linear model with the following properties:

$$ f(x) = m + kx, ~ x \in [0, d_\max] $$

$$ \int_0^{d_\max} f(x) ~ dx = 1, ~  f(d_\max) = 0 $$

$$ m = \frac{2}{d_\max}, ~ k = \frac{-2}{(d_\max)^2} $$


This is not very intuitive to understand, so take a look at this graph instead:

![weight graph](/assets/posts/2016-01-31-volumetric-lighting-1/weight_graph.jpg)

The weight for a given sample is the area under the graph for the distance the sample covers. The total area for all samples (given that we only sample from 0 to d<sub>max</sub>) is 1. The weight decreases the further away from the camera a sample is, and after d<sub>max</sub> the light contributes nothing to the final image at all. In practice I'm going to approximate the weights somewhat:

![weighted graph approximation](/assets/posts/2016-01-31-volumetric-lighting-1/weight_graph_approx.jpg)

In the above example the weight for the first sample s<sub>1</sub> is going to be the range covered (d<sub>max</sub> / 4) times f(s<sub>1</sub>).


## Light model

For this series of posts I'm going to limit myself to spotlights, mainly because that's what I primarily use (for now). But most (if not all) of the optimizations and improvements should be easily transferable to omni-directional point lights as well.

My spotlight model is fairly simple. Each spotlight has a position, a direction, outer and inner angles (used for attenuation along the light edge), a range (used for quadratic light falloff) and a color. The following struct is the input to the shader:

~~~glsl
struct Spotlight {
	vec3 vsPos; // Position in view space
	vec3 vsDir; // Direction in view space
	vec3 color; // Color of the light
	float range; // How far the light reaches, can also be thought of as strength
	float softFovRad; // The sharp, outer edge of the spotlight
	float sharpFovRad; // The soft, inner edge of the spotlight
	float softAngleCos;  // cos(softFovRad / 2)
	float sharpAngleCos; // cos(sharpFovRad / 2)
	mat4 lightMatrix;  // Transforms point in view space to shadow map coordinate
};
~~~

## Naive implementation

~~~glsl
// User defined parameters
uniform int uNumSamples; // Number of samples per pixel
uniform float uMaxDist;
uniform float uScaleFactor; // Simply a factor to scale the result

void main()
{
	vec3 vsPos = ... ; // Reconstruct from depth buffer
	vec3 rayDir = ... ; // For example normalize(vsPos)
	float distToPos = length(vsPos);

	// Camera distance weight function
	// f(x) = m + k * x
	// m = 2 / uMaxDist, k = -2 / uMaxDist²
	// f(uMaxDist) = 0, integral f(x) under [0, uMaxDist] = 1
	float eyeWeightM = 2.0 / uMaxDist;
	float eyeWeightK = -2.0 / (uMaxDist * uMaxDist);
	float sampleStep = (min(distToPos, uMaxDist) / float(uNumSamples - 1));

	float factor = 0.0;
	for (int i = 0; i < uNumSamples; ++i) {
		float sampleT = float(i) * sampleStep;
		vec3 samplePos = sampleT * rayDir;

		float shadowSample = sampleShadowMap(samplePos);
		float falloff = calcLightFalloff(samplePos);
		float attenuation = calcLightAttenuation(samplePos);
		float eyeDistWeight = eyeWeightM + eyeWeightK * sampleT;
		float weight = eyeDistWeight * sampleStep;

		factor += shadowSample * falloff * attenuation * weight;
	}

	outFragColor = vec4(uScaleFactor * factor * uSpotlight.color, 1.0);
}
~~~

As you can see some details have been removed for brevity, but the intent should be clear. I introduced two additional user defined parameters, one for setting the number of samples per pixel and one for scaling the result. The latter is needed in order to fine tune the results for a given scene. Below is an image of the result:

![naive result](/assets/posts/2016-01-31-volumetric-lighting-1/naive_result.png)

We have some noticeable banding artifacts on the volumetric light. The reason for this is that most of the samples actually miss the spotlight itself in this particular image. We could fix this by increasing the number of samples per pixel (currently at 128), or perhaps by doing something smarter...

## Performance

I'm going to test on 3 different configurations, a desktop with Nvidia GTX 980Ti, a laptop with Nvidia GTX 970M and the same laptop with Intel HD Graphics 4600. The rendering resolution is always 2560x1440, but the volumetric lighting is going to be rendered both at full resolution (2560x1440) and at quarter resolution (1280x720). Number of samples per pixel is 128 unless otherwise noted. The resolution of the shadow map (for the volumetric lighting, not the standard shading) is 256x256, which I have found to give quite nice results will still being relatively cheap to sample from multiple times.

The test program will be [this commit of snakium-cubed](https://github.com/SkipIfZero/snakium-cubed/commit/1c7a700c03b596042ee2954fbf2c565707d2b01c) (with one spotlight). The test will be done by starting a new game and letting it run for a while without touching anything. This particular test includes the camera being in most relevant positions relative to the spotlight.

Listed below is the result of the naive implementation. All times are in milliseconds. Average (avg), standard deviation (std) and maximum frametime are from a sample of the 5000 latest frametimes. Baseline is when the shader simply returns black color without doing any calculations.

**2560x1440** | **980Ti** | Avg | SD | Max | **970M** | Avg | SD | Max | **HD 4600** | Avg | SD
-|-|-|-|-|-|-|-|-|-|-|-
**Baseline**       | **#** | 1.5 | 0.1 | 1.9 | **#**| 7.9  | 5.3 | 20   | **#** | 27.1 | 2
**Naive Marching** | **#** | 5.6 | 1.8 | 9.6 | **#**| 18.5 | 5.6 | 30.8 | **#** | 102  | 35

**1280x720** | **980Ti** | Avg | SD | Max | **970M** | Avg | SD | Max | **HD 4600** | Avg | SD
-|-|-|-|-|-|-|-|-|-|-|-
**Baseline**       | **#** | 1.5 | 0.1 | 1.9 | **#**| 7.9  | 5.4 | 20.4 | **#** | 24.5 | 2.4
**Naive Marching** | **#** | 2.6 | 0.6 | 4.3 | **#**| 10.5 | 5.1 | 41.2 | **#** | 43.7 | 9.8

![performance graph 2560x1440](/assets/posts/2016-01-31-volumetric-lighting-1/perf_graph_2560x1440.png)

![performance graph 1280x720](/assets/posts/2016-01-31-volumetric-lighting-1/perf_graph_1280x720.png)

From this it should be quite clear that the effect is quite expensive, even when rendered in a lower resolution. And if we look at the maximum frametimes during the run the result is even worse. So the question is, can we do better? If we think about it for a bit, we will realize we are doing something very wasteful. We know that all the samples outside the spotlight cone will be in shadow and not contribute to the final image, so any samples taken there will be totally wasted. Wouldn’t it be better if we only sampled inside the spotlight cone where we don’t know the values? Stay tuned for part 2 in which we will resolve this issue.

## Footnotes

[^volumetric_lighting]: [Volumetric lighting](https://en.wikipedia.org/wiki/Volumetric_lighting) (also commonly known under the names god rays, light shafts, air shafts, etc) is a term used for the phenomena where you can see a shaft of light in the air. An example would be light streaming into a dusty room through a window.

[^ray_marching]: [Ray marching](https://en.wikipedia.org/wiki/Volume_ray_casting) in this context refers to a variant of raytracing where you sample multiple points along the ray against a distance field (shadow map).

[^textured_light]: With textured light I mean using a texture to decide the color and intensity for each "ray" of the light source. For example a projector projecting an image onto a wall could be accomplished using a spotlight with the image projected as the texture.

[^shadow_map_sampling]: Whenever I talk about sampling the shadow map I actually mean getting the depth value, comparing it and getting a value between 0 and 1 back.
