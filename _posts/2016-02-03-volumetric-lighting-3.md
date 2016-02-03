---
layout: post
title: "Volumetric lighting 3"
categories: graphics
comments: true
---

# Volumetric lighting 3 - Optimization and result

In the first post we created a basic volumetric lighting shader using ray marching. In the second post we improved the quality and performance by performing an intersection test against the spotlight before sampling. At this point we are pretty happy with the effect, but we want it to run way faster. Time for optimization!

## Resolutions

A very easy idea would be to simply lower the resolution of the effect. We can actually change the resolution of both the effect itself (number of rays) and of the shadow map. Below is a comparison image with some variations of resolutions (0.25<sup>2</sup>x means that the width and height is 0.25 times the full values):

![resolution comparison](/assets/posts/2016-02-03-volumetric-lighting-3/resolution_comparison.png)

This is very interesting. Besides some aliasing on the objects it seems we can turn the rendering resolution way down without making the result look worse. If we look back at the performance numbers in the previous posts we learn that lowering the resolution is a very significant performance boost, so this is a very good result. When rendering at 0.5<sup>2</sup> times the resolution (not shown in the comparison) I have a hard time telling the difference, so this is what I'm going to recommend as the default.

When it comes to the shadow map resolution having higher than 256<sup>2</sup> actually seems to make the quality worse. This makes sense considering a single texel on the shadow map is no longer as close to being representative of the whole range covered by a sample as before (think [mipmaps](https://en.wikipedia.org/wiki/Mipmap)). Since it's cheaper to take multiple samples from a smaller shadow map there really is no reason to use a high resolution one for volumetric lighting.

## Code optimization

Right now the code for the intersection test is mostly designed for readability. But from the performance numbers we can tell that it is still fairly fast. So it is probably a better idea to spend some time on improving the marching loop. It currently looks like this:

~~~glsl
for (int i = 0; i < uNumSamples; ++i) {
	float sampleT = startT + float(i) * sampleStep;
	vec3 samplePos = sampleT * rayDir;

	float shadowSample = sampleShadowMap(samplePos);
	float falloff = calcLightFalloff(samplePos);
	float attenuation = calcLightAttenuation(samplePos);
	float eyeDistWeight = eyeWeightM + eyeWeightK * sampleT;

	factor += shadowSample * falloff * attenuation * eyeDistWeight * sampleStep;
}
~~~

Hmm... Those function calls looks very suspect. Let us take a peak:

~~~glsl
float calcLightAttenuation(vec3 samplePos)
{
	vec3 lightToSampleDir = normalize(samplePos - uSpotlight.vsPos);
	return smoothstep(uSpotlight.softAngleCos, uSpotlight.sharpAngleCos, dot(lightToSampleDir, uSpotlight.vsDir));
}
~~~

**smoothstep()** looks unavoidable in this case, unless we replace it with some other function. :/

~~~glsl
float calcLightFalloff(vec3 samplePos)
{
	vec3 lightToSample = samplePos - uSpotlight.vsPos;
	return clamp(1.0 - (dot(lightToSample, lightToSample) / (uSpotlight.range * uSpotlight.range)), 0.0, 1.0);
}
~~~

Um...

~~~glsl
float sampleShadowMap(vec3 vsSamplePos)
{
	return textureProj(uShadowMap, uSpotlight.lightMatrix * vec4(vsSamplePos, 1.0));
}
~~~

Oh no

The worst offender looks to be **sampleShadowMap()**. In it we perform a 4x4 matrix transform each iteration. What we could do instead is to calculate the shadow map coordinate for the end and start positions and then simply linearly interpolate between them. This is probably a faster operation. We can also try to do the same for other linear things, such as the sample weight. In the code below we have done this and in the process also inlined the functions and precomputed as much as possible.

~~~glsl
float sampleStep = (endT - startT) / float(uNumSamples - 1);
float interpStep = 1.0 / float(uNumSamples - 1);

// Precompute shadow coord
vec4 startShadowCoord = uSpotlight.lightMatrix * vec4(startPos, 1.0);
vec4 endShadowCoord = uSpotlight.lightMatrix * vec4(endPos, 1.0);

// Precompute light falloff variables
vec3 startLightToSample = startPos - uSpotlight.vsPos;
vec3 endLightToSample = endPos - uSpotlight.vsPos;
float invSquaredLightRange = 1.0 / (uSpotlight.range * uSpotlight.range);

// Precompute eye dist weight and monte carlo weight and combine
float startWeight = (eyeWeightM + eyeWeightK * startT) * sampleStep;
float endWeight = (eyeWeightM + eyeWeightK * endT) * sampleStep;

float factor = 0.0;
for (int i = 0; i < uNumSamples; ++i) {
	// Interpolation factor
	float interp = interpStep * float(i);

	// Shadow map coord and sample
	vec4 sampleShadowCoord = mix(startShadowCoord, endShadowCoord, interp);
	float shadowSample = textureProj(uShadowMap, sampleShadowCoord);

	// Light falloff
	vec3 lightToSample = mix(startLightToSample, endLightToSample, interp);
	float falloff = 1.0 - dot(lightToSample, lightToSample) * invSquaredLightRange;

	// Light attenuation
	vec3 lightToSampleDir = normalize(lightToSample);
	float attenuation = smoothstep(uSpotlight.softAngleCos, uSpotlight.sharpAngleCos, dot(lightToSampleDir, uSpotlight.vsDir));

	// Calculate weight and update factor
	float weight = mix(startWeight, endWeight, interp);
	factor += shadowSample * falloff * attenuation * weight;
}
~~~

This looks promising. Time for some new performance numbers!

**2560x1440** | **980Ti** | Avg | SD | Max | **970M** | Avg | SD | Max | **HD 4600** | Avg | SD
-|-|-|-|-|-|-|-|-|-|-|-
**Baseline**                  | **#** | 1.5 | 0.1 | 1.9  | **#** | 7.9   | 5.3 | 20   | **#** | 27.1  | 2
**Naive marching**            | **#** | 5.6 | 1.8 | 9.6  | **#** | 18.5  | 5.6 | 30.8 | **#** | 102   | 35
**Test - No sampling**        | **#** | 1.9 | 0.3 | 2.7  | **#** | 8.7   | 5.3 | 20.6 | **#** | 33.3  | 5
**Test - Naive marching**     | **#** | 5.3 | 2   | 11.4 | **#** | 17.5  | 6   | 49   | **#** | 101   | 41.5
**Test - Optimized marching** | **#** | 4.4 | 1.5 | 10.3 | **#** | 14.4  | 5.5 | 44.5 | **#** | 86    | 33.3

**1280x720** | **980Ti** | Avg | SD | Max | **970M** | Avg | SD | Max | **HD 4600** | Avg | SD
-|-|-|-|-|-|-|-|-|-|-|-
**Baseline**                  | **#** | 1.5 | 0.1 | 1.9  | **#** | 7.9  | 5.4 | 20.4 | **#** | 24.5  | 2.4
**Naive marching**            | **#** | 2.6 | 0.6 | 4.3  | **#** | 10.5 | 5.1 | 41.2 | **#** | 43.7  | 9.8
**Test - No sampling**        | **#** | 1.6 | 0.1 | 2.3  | **#** | 8.1  | 5.4 | 34.5 | **#** | 26.2  | 2.5
**Test - Naive marching**     | **#** | 2.5 | 0.6 | 11.2 | **#** | 10.3 | 5.2 | 21.2 | **#** | 44.2  | 11.6
**Test - Optimized marching** | **#** | 2.2 | 0.4 | 3.3  | **#** | 9.5  | 5.3 | 41.2 | **#** | 40.5  | 9.6

This is a quite nice result. On a Nvidia 970M we have a **~22%** improvement at full resolution and a **~8%** improvement at a quarter resolution. And all thanks to some trivial code rearrangement! So, are we content?

No.

## Better sampling

Let us take a step back and consider the whole thing again. One of the original problems we had was with inefficient sampling, in particular wasted samples outside the spotlight. But are we really using our samples in the most efficient way right now? If we could decrease the number of samples we take without decreasing the quality we could probably improve performance even further.

Right now we are sampling with even intervals, in each iteration we calculate the weight of that particular sample. This might be stupid. Right now we have 128 samples per ray, if we were to sample all the way from 0 to the maximum visible distance d<sub>max</sub> we can observe something interesting. The first sample would have a very huge weight, and the last one would have a very small one. So we are basically saying that the first samples we take are more important than the later ones, and the last ones might contribute next to nothing to the final image! So instead of sampling at even intervals we could try to sample so that each sample have the same weight.

![equal weight sampling](/assets/posts/2016-02-03-volumetric-lighting-3/equal_weight_sampling.jpg)

This would look something like the image above. We would have a high density of samples close to the camera and then less the further away we get. Unlike earlier each sample will contribute exactly the same amount to the final image. This also means that the amount of samples we take for a given ray will depend entirely on how close we are to the spotlight in question.

~~~glsl
// Factor used in calculation of next sample position
float toNextScale = uMaxDist * uMaxDist / (2.0 * float(uNumSamples));

float intervalLength = endT - startT;
float factor = 0.0;
float currT = startT;
while (currT < endT) {
	// Interpolation factor
	float interp = (currT - startT) / intervalLength;

	// shadowSample, falloff and attenuation
	// ...

	// Calculate next sample position and update factor
	currT += toNextScale / (uMaxDist - currT);
	factor += shadowSample * dissipation * attenuation;
}

// Scale factor by weight (weight = 1 / numSamples)
factor /= float(uNumSamples);
~~~

No a very big change. Before we look at the performance we should take a look at the quality.

![equal weight comparison](/assets/posts/2016-02-03-volumetric-lighting-3/equal_weight_comparison.png)

It seems the equal weight version gives slightly worse quality in some situations, but mostly I have a hard time telling any difference. Which is a good thing. Let us look at the performance numbers again:

**2560x1440** | **980Ti** | Avg | SD | Max | **970M** | Avg | SD | Max | **HD 4600** | Avg | SD
-|-|-|-|-|-|-|-|-|-|-|-
**Baseline**                  | **#** | 1.5 | 0.1 | 1.9  | **#** | 7.9   | 5.3 | 20   | **#** | 27.1  | 2
**Naive marching**            | **#** | 5.6 | 1.8 | 9.6  | **#** | 18.5  | 5.6 | 30.8 | **#** | 102   | 35
**Test - No sampling**        | **#** | 1.9 | 0.3 | 2.7  | **#** | 8.7   | 5.3 | 20.6 | **#** | 33.3  | 5
**Test - Naive marching**     | **#** | 5.3 | 2   | 11.4 | **#** | 17.5  | 6   | 49   | **#** | 101   | 41.5
**Test - Optimized marching** | **#** | 4.4 | 1.5 | 10.3 | **#** | 14.4  | 5.5 | 44.5 | **#** | 86    | 33.3
**Test - Equal weight**       | **#** | 2.8 | 0.9 | 5.2  | **#** | 11.1  | 5.4 | 26.5 | **#** | 56.5  | 21.9

**1280x720** | **980Ti** | Avg | SD | Max | **970M** | Avg | SD | Max | **HD 4600** | Avg | SD
-|-|-|-|-|-|-|-|-|-|-|-
**Baseline**                  | **#** | 1.5 | 0.1 | 1.9  | **#** | 7.9  | 5.4 | 20.4 | **#** | 24.5  | 2.4
**Naive marching**            | **#** | 2.6 | 0.6 | 4.3  | **#** | 10.5 | 5.1 | 41.2 | **#** | 43.7  | 9.8
**Test - No sampling**        | **#** | 1.6 | 0.1 | 2.3  | **#** | 8.1  | 5.4 | 34.5 | **#** | 26.2  | 2.5
**Test - Naive marching**     | **#** | 2.5 | 0.6 | 11.2 | **#** | 10.3 | 5.2 | 21.2 | **#** | 44.2  | 11.6
**Test - Optimized marching** | **#** | 2.2 | 0.4 | 3.3  | **#** | 9.5  | 5.3 | 41.2 | **#** | 40.5  | 9.6
**Test - Equal weight**       | **#** | 1.9 | 0.3 | 3.2  | **#** | 8.6  | 5.2 | 20.9 | **#** | 32.8  | 6.2

![perf graph 2560x1440](/assets/posts/2016-02-03-volumetric-lighting-3/perf_graph_1440.png)

![perf graph 1280x720](/assets/posts/2016-02-03-volumetric-lighting-3/perf_graph_720.png)

Wow, this is a very good result. Compared to optimized marching we have (on an 970m) a **~30%** improvement at full resolution and a **~10%** improvement at a quarter resolution. Compared to where we were when we started this optimization process we have a **~58%** improvement at full resolution and a **20%** improvement at a quarter resolution. The improvement is even more impressive on the integrated intel card. At a quarter resolution we have almost reached playable speeds (30fps).

## Limitations

So, what's the catch with the equal weight version? The problem with it is that it has uneven performance. It performs optimally when you are looking at spotlights from the side while not being too close to them. Worst case is when you are inside them looking at the light source, in that scenario the performance might be as bad as naive marching.

Another limitation that is slightly harder to spot has to do with the user parameters. The weight function is defined by setting a max distance for visibility. You might want to be able to view the effect from very far away while still having smaller (relative to the max distance) light sources. In order to accomplish this you must increase both the scale factor and the number of samples to keep the quality, if you don't increase the number of samples a single sample might cover too much distance by itself. Depending on the scene this might work fine, but the problem is that increasing the number of samples effectively means increasing the worst case frametime.

In conclusion things are not really as bad as I might have made it sound here. It basically means that the parameters need to be tuned for the scene. And the light placement itself might need to be done with some care to how the algorithm works, but for most scenes I suspect it will work fine. But the only way to truly know for sure is to test it out in practice. If the scene in question is problematic there are always other algorithms.

## Conclusion

So, are we content?

For now, yes. In the end we managed to improve the quality and performance of the basic algorithm by quite a bit. I'm sure there are still many potential improvements to be made. One of the most obvious ones is to adapt it for textured lights. It would be interesting to see how expensive sampling a light texture is compared to the **smoothstep()** call each iteration.

If you have any questions or comments feel free to post a comment and I will try to answer is as best as I can.