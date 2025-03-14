# Explainer for adding noise to canvas readbacks

This proposal is an early design sketch by Chrome to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

## Participate
- https://github.com/explainers-by-googlers/canvas-noise-on-readbacks/issues

## Introduction

The canvas APIs allow websites to draw shapes and forms on a canvas and read back the rendered image. However, the browser's rendering process leaks details about the GPU's properties. The goal of adding noise to canvas readbacks is to prevent scripts from easily obtaining identifying information that can be used to re-identify a browser across contexts.

## Goals:

* Prevent identifying information from being easily extracted from GPU-rendered canvases.
* Improve user's privacy by aligning the exposed information with other data partitioning practices in the Web Platform.
* Minimize the impact, both in terms of performance and breakage, for applications that intensively use `<canvas>` elements by only intervening when necessary.
* Ensure that the noise added to canvas readbacks is applied every time that the raw pixels of the canvas are exposed to the website.

## Non-goals:

* This document does not propose any changes to the way that operations on the canvas are executed, and only discusses noise that is added upon readback.
* Applications that never read back the contents of the canvas will not be intervened upon.

## Overview
Small changes to the RGBA pixel values of the canvas are made whenever the contents of a canvas are read out, and the GPU was used to render contents on that canvas. While CPU rendering can - at least in theory - be implemented by browser vendors in a deterministic way that doesn't leak much information about the user's device, GPU rendering will have slightly different behavior for many operations depending on drivers and the specifications of the device. As a result, GPU rendering is more likely to end up leaking information about the user’s device. Initially, noise might be added when one of the following functions is called:

* HTMLCanvasElement.toDataURL
* HTMLCanvasElement.toBlob
* OffscreenCanvas.convertToBlob
* CanvasRenderingContext2D.getImageData

Additionally, noise might be added when the contents of the canvas data are transferred to another host (e.g. a MediaStream) that would allow reading out those contents.

The noise that is added is fairly small, in the range of -3 to 3, making the noise visually unnoticeable in most cases. Furthermore, the noise that is added to a specific canvas is deterministic and depends on the contents of the canvas and a seed derived from a per-session random token and the context the canvas is included in. This context is based on the origin of the embedding document as well as that of the top-level document, and follows state partitioning properties.

Adding noise to canvas readbacks is most helpful in browsing modes where the user explicitly or implicitly signaled they care most about their privacy, for instance in Incognito mode.

To reduce any potential UX breakage caused by changing pixel values, we make use of heuristics to determine whether noise should be applied. These heuristics target canvas operations currently used to extract information and will likely change over time.

## Noising algorithm

At a high level, the algorithm for adding noise to a canvas readback is implemented according to the following pseudo-code:

```python
# For each partition a different session-bound random value is assigned
TOKENS = map(PARTITION -> TOKEN)

for (x, y) in canvas:
  r, g, b, a = get_pixel_at(x, y)
  pixel_hash = hash_init(TOKENS[PARTITION])

  # Each pixel is determined by their RGBA channels and their (x, y) coordinates
  r, g, b, a, x, y = pixel

  # The RGBA values of the current pixel are used as a new seed for the rolling hash
  rolling_hash.update(r, g, b, a)

  # Determine the (x,y)-offset for the other pixel that will influence the noise.
  # The location is determined based on the token, partition, and RGBA values of the
  # current pixel. These offsets are in the range of -10 to 10.
  otherX, otherY = determine_offset(pixel_hash)

  # Get the RGBA values for the other pixel, and use them to update the rolling hash.
  otherR, otherG, otherB, otherA = get_pixel_at(otherX, otherY)
  rolling_hash.update(otherR, otherG, otherB, otherA)

  # Add noise deltas to the RGBA channels of the current pixel, clamped to [0, 255].
  dR, dG, dB, dA = get_noise_deltas(pixel_hash)
  noised_canvas[x][y] = [clamp(r + dR), clamp(g + dG), clamp(b + dB), clamp(a + dA)]

return noised_canvas
```

For a single pixel, the noise that is added is determined by the following factors:

* The context the related canvas is included in. The same canvases within the same context will still get the exact same noise.
* Its own RGBA values. These might be unknown if it depends on the GPU-specific rendering.
* The RGBA values of a neighboring pixel. This preserves identical noise distribution for identical pixel regions. Furthermore, [scaling attacks](https://github.com/google/security-research/security/advisories/GHSA-24cm-69m9-fpw3) are rendered substantially more difficult and expensive due to the unpredictable nature of hashing and random neighbor selection.

An optimization is made to the algorithm to reduce the overhead on large surfaces with the same pixel value, which can be compressed well without noise but would compress badly when noise is added. If the pixel value in the original canvas is an exact match, the exact same noise is applied to that pixel. As a result, the large surface will still be of the same value and thus compress well.

## FAQs

**Canvas noise on readback is breaking my application, what should I do?**

Do you specifically need the GPU to render the contents? If not, you can consider switching using CPU rendering: set `willReadFrequently: true` when getting the context (see [MDN](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/getContext#willreadfrequently)). Note that for canvases with frequent readbacks, or for canvases where only few operations are performed, it is typically faster to use the CPU for rendering.
If it’s not possible to switch to CPU rendering, please [file a bug](https://issues.chromium.org/issues/new?component=1456351&title=%5BCanvas%20anti-FP%20breakage%5D&description=-%20Breaking%20Site:%20https:%2F%2Fwww.example.com%2F%0A-%20Platform:%20%0A-%20Version:%20%0A%0A-%20Screenshot%20%5Boptional%5D:%20%0A%0A-%20Observed%20behavior:%20%0A-%20Expected%20behavior:%20%0A&template=0) with Chromium; we are interested to understand your use-case.

## Security and privacy considerations
The goal of canvas noising is to reduce information available through the Canvas API that can be used to re-identify the user across sites.

The per-partition token should be considered site data, and thus should be removed whenever the user deletes data for a particular site. This prevents the token from being abused as a persistent identifier.
