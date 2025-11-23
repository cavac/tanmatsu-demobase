# PAX Graphics Bug Report - Issue #5

## Overview

**Issue:** pax_draw_image() Color Format Issue
**Upstream URL:** https://github.com/robotman2412/pax-graphics/issues/5
**Status:** Open (as of 2025-11-23)
**Reporter:** renzenicolai (2025-09-20)
**Severity:** High - Breaks image rendering in 16-bit and 24-bit color modes

## Local Context

This project uses a **local fork** of PAX Graphics installed in:
```
components/pax-graphics/
```

This is NOT the managed component from ESP-IDF component registry. All PAX code modifications and bug fixes must be applied to the local copy in the `components/` directory.

## Problem Description

The `pax_draw_image()` function fails to properly display images when the source image buffer uses 24-bit RGB888 or 16-bit RGB565 color formats. Only 32-bit ARGB8888 buffers render correctly.

### Symptoms

When calling `pax_draw_image()` with different buffer formats:
- **32-bit ARGB (PAX_BUF_32_8888ARGB)** - ✅ Works correctly, displays intended colors
- **24-bit RGB (PAX_BUF_24_888RGB)** - ❌ Displays only red channel or garbage
- **16-bit RGB (PAX_BUF_16_565RGB)** - ❌ Displays only red channel or garbage

### Test Case from Upstream

According to the issue report, the following test was performed:

```c
// Create three buffers with different color depths
pax_buf_t buf32, buf24, buf16;
pax_buf_init(&buf32, NULL, 320, 240, PAX_BUF_32_8888ARGB);
pax_buf_init(&buf24, NULL, 320, 240, PAX_BUF_24_888RGB);
pax_buf_init(&buf16, NULL, 320, 240, PAX_BUF_16_565RGB);

// Fill with different colors
pax_background(&buf32, 0xFFFF0000); // Red - should work
pax_background(&buf24, 0xFF00FF00); // Green - fails
pax_background(&buf16, 0xFF0000FF); // Blue - fails

// Display each buffer in sequence
while (1) {
    pax_draw_image(display_buf, &buf32, 0, 0);
    vTaskDelay(pdMS_TO_TICKS(200));

    pax_draw_image(display_buf, &buf24, 0, 0);
    vTaskDelay(pdMS_TO_TICKS(200));

    pax_draw_image(display_buf, &buf16, 0, 0);
    vTaskDelay(pdMS_TO_TICKS(200));
}
```

**Expected Result:** Red → Green → Blue screen cycling
**Actual Result:** Red → Red (or garbage) → Red (or garbage)

## Technical Analysis

### Code Location

**File:** `components/pax-graphics/core/src/shapes/pax_misc.c`
**Function:** `pax_draw_image()` (line 332) and `draw_image_impl()` (line 303)

### Implementation Details

The `pax_draw_image()` function uses a helper `draw_image_impl()` that has two rendering paths:

1. **Fast path** (line 321-323) - Direct sprite copy when:
   - No scaling (width/height match exactly)
   - No transparency
   - Identity matrix (no transformation)
   - Delegates to `pax_draw_sprite()`

2. **Shader path** (line 324-328) - Texture shader rendering:
   - Used when scaling, rotation, or alpha blending needed
   - Uses `PAX_SHADER_TEXTURE()` or `PAX_SHADER_TEXTURE_OP()`
   - Delegates to `pax_shade_rect()`

### Suspected Root Causes

Based on the behavior, the bug likely exists in one or more of these areas:

1. **Color Format Conversion in Shader Path**
   - The texture shader may not properly read/convert 16-bit or 24-bit pixel data
   - Color channel extraction/packing might be incorrect for non-32-bit formats
   - Related files: `core/src/pax_renderer.c`, shader implementations

2. **Sprite Copy Path (pax_draw_sprite)**
   - May not handle buffer-to-buffer color format conversion correctly
   - `buf->col2buf()` and `buf->buf2col()` function pointers might be misconfigured
   - Related files: `core/src/pax_setters.c`, `core/src/shapes/pax_misc.c:123-182`

3. **Buffer Getter/Setter Functions**
   - 16-bit and 24-bit getters may not correctly extract color values
   - Related functions: `pax_index_getter_16bpp()`, `pax_index_getter_24bpp()`
   - Related file: `core/src/pax_setters.c`

## Relationship to This Project

### Our Configuration

This project uses:
- **Display:** ST7701 MIPI DSI controller (480×800 pixels)
- **Display Format:** 24-bit RGB888 (recently converted from 16-bit)
- **Framebuffer Format:** 32-bit ARGB8888 (PAX_BUF_32_8888ARGB)
- **Graphics Library:** Local PAX fork in `components/pax-graphics/`

### Impact Assessment

**Current Status:** ✅ **NOT AFFECTED** (but should verify)

We are likely **not impacted** by this bug because:
1. Our main framebuffer uses 32-bit ARGB format (which works according to the bug report)
2. We primarily use drawing primitives (circles, rectangles) rather than `pax_draw_image()`
3. The bouncing balls demo doesn't use image rendering

**Potential Future Risk:**
- If we add image/sprite rendering features in the future
- If we need to composite multiple PAX buffers together
- If we implement off-screen rendering with 16/24-bit buffers for performance

### Previous PAX Bug Fix

We have already fixed **one critical bug** in the local PAX fork:

**Bug:** `pax_range_setter_24bpp()` index initialization error
**File:** `components/pax-graphics/core/src/pax_setters.c:408`
**Fix Date:** 2025-11-19
**Issue:** Filled circles had black gaps in 24-bit RGB888 mode
**Solution:** Changed loop initialization from `int i = 0` to `int i = index`
**Documentation:** See CLAUDE.md and 24BPP.md for complete details

This demonstrates that the local PAX fork **can and should be patched** for critical bugs.

## Investigation Checklist

If this bug affects future features, investigate in this order:

### Phase 1: Verify Bug Exists Locally
- [ ] Create test case with 16-bit and 24-bit source buffers
- [ ] Attempt to draw images using `pax_draw_image()`
- [ ] Confirm colors display incorrectly (red only)

### Phase 2: Identify Root Cause
- [ ] Enable PAX debug logging if available
- [ ] Test direct sprite copy path vs shader path separately
- [ ] Examine `pax_draw_sprite()` implementation for format conversion
- [ ] Check texture shader code for color channel extraction
- [ ] Review getter/setter functions for 16-bit and 24-bit formats

### Phase 3: Potential Code Locations
- [ ] `core/src/shapes/pax_misc.c:303-351` - draw_image_impl()
- [ ] `core/src/shapes/pax_misc.c:123-182` - pax_draw_sprite variants
- [ ] `core/src/pax_setters.c` - Getter/setter functions
- [ ] `core/src/pax_renderer.c` - Rendering and shader dispatch
- [ ] Shader implementations - Texture sampling code

### Phase 4: Fix and Test
- [ ] Implement fix in local `components/pax-graphics/` fork
- [ ] Test with all three color formats (16/24/32-bit)
- [ ] Verify no regressions in existing drawing code
- [ ] Update this document with fix details

## Upstream Status

**As of 2025-11-23:**
- Issue remains **open** with no comments
- No assigned maintainer
- No pull requests addressing this issue
- No community discussion or proposed fixes

**Recommendation:** If we discover a fix, consider contributing back to upstream repository at https://github.com/robotman2412/pax-graphics

## Related Documentation

- **24BPP.md** - Complete 24-bit RGB888 conversion documentation
- **CLAUDE.md** - Implementation log including previous PAX bug fix
- **components/pax-graphics/docs/** - PAX library documentation (if present)

## Notes

1. This bug does NOT affect the 24-bit display mode itself (ST7701 controller)
2. This bug is specifically about PAX buffer-to-buffer image operations
3. Our main rendering (shapes, text) works correctly in all color modes
4. The local PAX fork gives us control to fix bugs independently of upstream

---

**Last Updated:** 2025-11-23
**Next Review:** When implementing image/sprite features
