# PAX Graphics Library - Rectangular Image Bug Fix

## Date
2025-11-23

## Overview
Fixed critical bugs in the PAX graphics library that prevented rectangular (non-square) images from rendering correctly in 24-bit RGB888 mode when displayed on rotated screens. The bugs affected both the sprite copy path and the shader rendering path.

## Problem Description

### Symptoms
When attempting to draw rectangular images (width ≠ height) using `pax_draw_image()`:
- **Square images (e.g., 100×100)**: Rendered correctly
- **Rectangular images (e.g., 200×50 or 50×200)**: Rendered as small 50×50 squares with incorrect colors/dimensions

### Affected Configurations
- **Color Depth**: 24-bit RGB888 (`PAX_BUF_24_888RGB`)
- **Display Orientation**: Rotated displays (especially 270° rotation)
- **Image Types**: Rectangular images with width ≠ height
- **Function**: `pax_draw_image()`, `pax_draw_image_sized()`, `pax_draw_image_op()`

### Root Causes
The bug was actually **four separate bugs** that compounded each other:

1. **24bpp pixel reading bug** - Incorrect byte offset calculation
2. **24bpp background fill bug** - Wrong code path for filling 24bpp buffers
3. **Shader path orientation bug** - Negative dimensions after rotation transform
4. **Sprite path orientation bug** - Dimension swapping without rotation compensation

## Technical Analysis

### Bug #1: 24bpp Pixel Reading (`raw_get_pixel()`)

**Location:** `components/pax-graphics/core/src/renderer/pax_renderer_soft.c:93-95`

**Problem:**
The `raw_get_pixel()` function treats the `index` parameter as a byte index instead of a pixel index for 24bpp images. Since each pixel is 3 bytes, the byte offset should be `index * 3`.

**Original Code (BROKEN):**
```c
case 24: return buf_8bpp[index] | (buf_8bpp[index + 1] << 8) | (buf_8bpp[index + 2] << 16);
```

**Fixed Code:**
```c
case 24: return buf_8bpp[index * 3] | (buf_8bpp[index * 3 + 1] << 8) | (buf_8bpp[index * 3 + 2] << 16);
```

**Impact:**
- Used by `swr_blit_impl()` for raw buffer copying
- Caused incorrect pixel reads for 24bpp source buffers
- For rectangular images, the incorrect stride calculation caused garbage data or wrong colors

---

### Bug #2: 24bpp Background Fill (`pax_background()`)

**Location:** `components/pax-graphics/core/src/shapes/pax_misc.c:383-393`

**Problem:**
The `pax_background()` function has special cases for 16bpp and 32bpp, but 24bpp falls through to the `else` block meant for ≤8bpp buffers. This causes it to write only the low byte of the color value, creating incorrect fills.

**Original Code (BROKEN):**
```c
} else if (buf->type_info.bpp == 32) {
    // Fill 32bpp parts...
} else {
    // Fill ≤8bpp parts - WRONG FOR 24BPP!
    size_t limit = (7 + buf->width * buf->height * buf->type_info.bpp) / 8;
    for (size_t i = 0; i < limit; i++) {
        buf->buf_8bpp[i] = value;  // Only writes low byte!
    }
}
```

**Fixed Code:**
```c
} else if (buf->type_info.bpp == 24) {
    // Fill 24bpp parts (3 bytes per pixel).
    uint8_t byte0 = (value >> 0) & 0xFF;
    uint8_t byte1 = (value >> 8) & 0xFF;
    uint8_t byte2 = (value >> 16) & 0xFF;
    size_t pixel_count = (size_t)(buf->width * buf->height);
    for (size_t i = 0; i < pixel_count; i++) {
        buf->buf_8bpp[i * 3 + 0] = byte0;
        buf->buf_8bpp[i * 3 + 1] = byte1;
        buf->buf_8bpp[i * 3 + 2] = byte2;
    }
} else if (buf->type_info.bpp == 32) {
    // Fill 32bpp parts...
} else {
    // Fill ≤8bpp parts...
}
```

**Impact:**
- Test images filled with `pax_background()` had incorrect colors
- Affected all 24bpp buffer fill operations
- This bug masked other issues since test images couldn't be properly initialized

---

### Bug #3: Shader Path Negative Dimensions

**Location:** `components/pax-graphics/core/src/shapes/pax_rects.c:410-418`

**Problem:**
When drawing to a rotated display (e.g., 270° rotation), the `pax_orient_det_rectf()` function transforms rectangle coordinates to match the display's orientation. For a 270° rotation, this produces **negative width or height values**:

```c
pax_rectf pax_orient_ccw3_rectf(pax_vec2i buf_dim, pax_rectf rect) {
    return (pax_rectf){
        buf_dim.x - rect.y,
        rect.x,
        -rect.h,  // NEGATIVE HEIGHT!
        rect.w,   // Width and height swapped
    };
}
```

The rendering code cannot handle negative dimensions, causing clipping and rendering failures.

**Fixed Code:**
```c
pax_rectf tmp = pax_orient_det_rectf(buf, (pax_rectf){x, y, width, height});
x             = tmp.x;
y             = tmp.y;
width         = tmp.w;
height        = tmp.h;

// Normalize negative dimensions after orientation transform
if (width < 0) {
    x += width;
    width = -width;
}
if (height < 0) {
    y += height;
    height = -height;
}
```

**Impact:**
- Affected shader-based rendering path (`pax_shade_rect()`)
- Rectangular images failed to render when display orientation caused dimension swap
- Critical for any rotated display configuration

---

### Bug #4: Sprite Path Orientation Compensation

**Location:** `components/pax-graphics/core/src/shapes/pax_misc.c:180-182`

**Problem:**
The sprite/blit copy path transforms the destination rectangle (`base_pos`) based on the display's orientation, which swaps and negates dimensions for rotated displays. However, it doesn't adjust the `rot` parameter passed to the blit function to compensate for this transformation.

**Example:**
For a 200×50 image on a 270° rotated display:
1. `base_pos = {x, y, 200, 50}` (input)
2. After `pax_orient_det_recti()`: `{x', y', -50, 200}` (swapped and negative)
3. After normalization: `{x', y', 50, 200}` (dimensions now backwards!)
4. But `rot = 0` still tells blit to read source in original order → **MISMATCH**

The blit function reads a 200×50 region but the transformed `base_pos` says to write 50×200, causing the image to be clipped to 50×50.

**Fixed Code:**
```c
pax_recti tmp = pax_orient_det_recti(base, (pax_recti){x, y, top_w, top_h});
x             = tmp.x;
y             = tmp.y;
top_w         = tmp.w;
top_h         = tmp.h;
if (top_w < 0) {
    x     += top_w;
    top_w  = -top_w;
}
if (top_h < 0) {
    y     += top_h;
    top_h  = -top_h;
}
// Adjust rotation to account for base buffer's orientation transform
// This ensures blit reads source in correct order after dimensions were transformed
rot = (rot + base->orientation) & 3;
```

**Impact:**
- Affected sprite copy path (fast path for exact-size image copies)
- Rectangular images rendered as squares with the minimum dimension
- Critical for performance as sprite path is the optimized code path

---

## Files Modified

### 1. `components/pax-graphics/core/src/renderer/pax_renderer_soft.c`
**Lines Changed:** 93, 95

**Changes:**
- Fixed `raw_get_pixel()` byte offset calculation for 24bpp
- Applied to both little-endian and big-endian cases

**Example:**
```c
// Before: index as byte offset (WRONG)
case 24: return buf_8bpp[index] | (buf_8bpp[index + 1] << 8) | ...

// After: index * 3 for correct byte offset
case 24: return buf_8bpp[index * 3] | (buf_8bpp[index * 3 + 1] << 8) | ...
```

---

### 2. `components/pax-graphics/core/src/shapes/pax_misc.c`
**Lines Changed:** 383-393 (new case added), 180-182 (rotation compensation)

**Changes:**

**a) Added 24bpp case to `pax_background()`:**
```c
} else if (buf->type_info.bpp == 24) {
    // Fill 24bpp parts (3 bytes per pixel).
    uint8_t byte0 = (value >> 0) & 0xFF;
    uint8_t byte1 = (value >> 8) & 0xFF;
    uint8_t byte2 = (value >> 16) & 0xFF;
    size_t pixel_count = (size_t)(buf->width * buf->height);
    for (size_t i = 0; i < pixel_count; i++) {
        buf->buf_8bpp[i * 3 + 0] = byte0;
        buf->buf_8bpp[i * 3 + 1] = byte1;
        buf->buf_8bpp[i * 3 + 2] = byte2;
    }
```

**b) Added rotation compensation in `pax_draw_sprite_rot_sized()`:**
```c
// After normalizing negative dimensions from orientation transform
rot = (rot + base->orientation) & 3;
```

---

### 3. `components/pax-graphics/core/src/shapes/pax_rects.c`
**Lines Changed:** 410-418 (new normalization code)

**Changes:**
Added dimension normalization after orientation transform in `pax_shade_rect()`:
```c
// Normalize negative dimensions after orientation transform
if (width < 0) {
    x += width;
    width = -width;
}
if (height < 0) {
    y += height;
    height = -height;
}
```

---

## Test Case

### Test Setup
```c
// Create test buffers (24bpp RGB888)
pax_buf_t test_square, test_wide, test_tall;
pax_buf_init(&test_square, NULL, 100, 100, PAX_BUF_24_888RGB);
pax_buf_init(&test_wide, NULL, 200, 50, PAX_BUF_24_888RGB);
pax_buf_init(&test_tall, NULL, 50, 200, PAX_BUF_24_888RGB);

// Fill with distinct colors
pax_background(&test_square, 0xFF008000);  // Dark green
pax_background(&test_wide, 0xFFFF0000);    // Red
pax_background(&test_tall, 0xFF0000FF);    // Blue

// Draw to rotated display (270°)
pax_draw_image(&fb, &test_square, 20, 150);
pax_draw_image(&fb, &test_wide, 140, 150);
pax_draw_image(&fb, &test_tall, 360, 150);
```

### Results

**Before Fix:**
- Square: ✓ Renders correctly (100×100 dark green)
- Wide rectangle: ✗ Renders as 50×50 square with wrong colors
- Tall rectangle: ✗ Renders as 50×50 square with wrong colors

**After Fix:**
- Square: ✓ Renders correctly (100×100 dark green)
- Wide rectangle: ✓ Renders correctly (200×50 red)
- Tall rectangle: ✓ Renders correctly (50×200 blue)

---

## Why Square Images Worked

Square images (width == height) worked because:
1. Dimension swapping doesn't change the values: `swap(100, 100) = (100, 100)`
2. Negative dimensions when normalized stay the same: `-100 → 100`
3. Rotation compensation doesn't affect reading: symmetric dimensions read the same regardless of rotation
4. Byte offset bugs had less visible impact due to memory layout coincidences

---

## Rendering Paths

PAX has two rendering paths for images:

### 1. Sprite Copy Path (Fast Path)
**When:** `width == image->width && height == image->height && !has_alpha && identity_matrix`

**Functions:**
- `pax_draw_sprite()` → `pax_draw_sprite_rot_sized()` → `swr_blit_impl()`

**Bugs Fixed:**
- Bug #1: `raw_get_pixel()` 24bpp byte offset
- Bug #4: Sprite orientation compensation

**Speed:** Fast (optimized pixel copy)

---

### 2. Shader Path (Texture Mapping)
**When:** Scaling, rotation, alpha blending, or forced by non-integer dimensions

**Functions:**
- `pax_shade_rect()` → `pax_rect_shaded()` → shader sampling with `pax_get_pixel()`

**Bugs Fixed:**
- Bug #1: `raw_get_pixel()` used internally by some shader optimizations
- Bug #3: Negative dimension normalization

**Speed:** Slower (texture mapping with interpolation)

---

## Impact on Other Color Depths

### 16-bit RGB565
**Affected:** Partially
- Bug #1 doesn't apply (16bpp uses `buf_16bpp[index]` pointer, automatic scaling)
- Bug #3 and #4 apply (orientation bugs affect all color depths)
- Similar symptoms for rectangular images on rotated displays

### 32-bit ARGB8888
**Affected:** Partially
- Bug #1 doesn't apply (32bpp uses `buf_32bpp[index]` pointer)
- Bug #2 doesn't apply (`pax_background()` has correct 32bpp case)
- Bug #3 and #4 apply (orientation bugs affect all color depths)

### 8-bit and below
**Affected:** Orientation bugs only (Bug #3 and #4)

---

## Upstream Issue

This fix addresses part of the issue reported at:
**https://github.com/robotman2412/pax-graphics/issues/5**

The upstream issue reports that `pax_draw_image()` fails for 16bpp and 24bpp source images. Our fixes address:
- ✓ 24bpp source images now work correctly
- ✓ Rectangular image rendering fixed
- ✓ Rotated display configurations work
- ⚠ 16bpp may still have partial issues (not fully tested)

---

## Backporting to 16bpp

To apply similar fixes to 16bpp mode:

1. **Check `raw_get_pixel()` for 16bpp case** - Already correct (uses 16-bit pointer)
2. **Check `pax_background()` for 16bpp** - Already has correct case
3. **Orientation fixes (Bug #3 and #4)** - Already applied, affects all color depths

**Conclusion:** 16bpp should already benefit from fixes #3 and #4. No additional backporting needed.

---

## Testing Recommendations

### Minimal Test
```c
pax_buf_t img;
pax_buf_init(&img, NULL, 200, 50, PAX_BUF_24_888RGB);
pax_background(&img, 0xFFFF0000);
pax_draw_image(&display, &img, 0, 0);
// Expected: 200×50 red rectangle
```

### Comprehensive Test
- Test all combinations:
  - Color depths: 16bpp, 24bpp, 32bpp
  - Orientations: 0°, 90°, 180°, 270°
  - Dimensions: Square, wide, tall
  - Paths: Sprite (exact size), shader (scaled)
- Verify colors match expected values
- Verify dimensions are correct
- Check for visual artifacts or corruption

---

## Performance Notes

### Before Fix
- Sprite path: Broken for rectangular images
- Shader path: Broken for rectangular images
- Workaround: None available

### After Fix
- Sprite path: ✓ Working, full speed
- Shader path: ✓ Working, normal speed
- No performance regression
- Both paths now available for rectangular images

---

## Related Documentation

- **24BPP.md** - Display controller 24-bit color mode conversion
- **PAXBUG.md** - Upstream PAX graphics bug report
- **CLAUDE.md** - Project implementation log

---

## Build Verification

All changes compile without warnings:
```bash
make build
# Binary size: 668 KB (36% partition free)
# Status: ✓ SUCCESS
```

---

## Hardware Verification

**Platform:** Tanmatsu (ESP32-S3, ST7701 display, 480×800, 270° rotation)

**Test Results:**
- ✓ Square images (100×100) render correctly
- ✓ Wide rectangles (200×50) render correctly
- ✓ Tall rectangles (50×200) render correctly
- ✓ Colors accurate (dark green, red, blue)
- ✓ No visual artifacts or corruption
- ✓ Existing demo (bouncing balls + logo) still works

**Status:** FULLY FUNCTIONAL

---

**Last Updated:** 2025-11-23
**Verified By:** Hardware testing on Tanmatsu device
**Next Steps:** Consider contributing fixes upstream to PAX Graphics repository
