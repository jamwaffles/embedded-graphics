# Migrating from embedded-graphics 0.6.x to 0.7.0

## Table of contents

- [Migrating from embedded-graphics 0.6.x to 0.7.0](#migrating-from-embedded-graphics-06x-to-070)
  - [Table of contents](#table-of-contents)
  - [New features](#new-features)
    - [Primitives](#primitives)
    - [Mock display](#mock-display)
    - [Geometry](#geometry)
    - [Color](#color)
    - [Sub draw targets](#sub-draw-targets)
    - [Text rendering](#text-rendering)
  - [For display driver authors](#for-display-driver-authors)
  - [For crates that handle images](#for-crates-that-handle-images)
  - [For text rendering crates](#for-text-rendering-crates)
    - [Method changes](#method-changes)
  - [General](#general)
    - [`Drawable`](#drawable)
    - [`IntoIterator` changes](#intoiterator-changes)
  - [Macros are removed](#macros-are-removed)
  - [Primitives](#primitives-1)
    - [Circle](#circle)
    - [Rectangle](#rectangle)
    - [Triangle](#triangle)
  - [Geometry](#geometry-1)
  - [Mock display](#mock-display-1)
  - [Style module](#style-module)

## New features

// TODO: This entire section will be moved into a GH release text body and/or blog post. Leaving here until that release is published.

### Primitives

New primitives have been added:

- `Arc`
- `Sector`
- `Polyline`
- `Ellipse`

The `Line` primitive now supports stroke widths greater than 1px.

Primitives now support stroke alignment using the `StrokeAlignment` enum.

### Mock display

`MockDisplay` now supports all RGB and grayscale color types in its patterns.

New methods:

- `swap_xy` - copies the current display with X and Y coordinates swapped.
- `map` - create a copy of the current display with a predicate applied to all pixels.
- `affected_area` - gets the bounding box of all changes made to the display.

### Geometry

Adds the `Point::x_axis`, `Point::y_axis`, `Size::x_axis` and `Size::y_axis` methods to get one axis of a point or size, with the other set to zero.

```rust
use embedded_graphics::geometry::Point;

let point = Point::new(15, 23);

assert_eq!(point.x_axis() = Point::new(15, 0));
assert_eq!(point.y_axis() = Point::new(0, 23));
```

`Point::new_equal` and `Size::new_equal` have also been added as convenience methods to create points or sizes with equal coordinates.

```diff
use embedded_graphics::geometry::Size;

- let size = Size::new(20, 20);
+ let size = Size::new_equal(20);
```

`Point::length_squared` was added to `Point`.

Other methods added to both `Point` and `Size` are:

- `component_min`
- `component_max`
- `component_mul`
- `component_div`

The `Rectangle::intersection` method was added to get the intersecting `Rectangle` between `self` and another given `Rectangle`.

### Color

The `ToBytes` trait has been added to support conversion of colors into byte arrays.

### Sub draw targets

The `DrawTargetExt` trait is introduced to allow a translated, cropped or clipped sub-area of a `DrawTarget` to be drawn to.

`DrawTargetExt` is implemented for `DrawTarget`.

Please search for `DrawTargetExt` on <https://docs.rs/embedded-graphics> for usage examples.

### Text rendering

TODO: Improve this section before release.

- `TextStyle` -> `MonoTextStyle`
- Added support for external renderers
- `MonoTextStyleBuilder::new(Font)` -> `MonoTextStyle::new().font(Font)`
- Added support for underline and strikethrough to the internal text renderer
- Added `TextStyle` to set the horizontal and vertical alignment for `Text` drawables
- New default vertical alignment is baseline
- New fonts with `ascii` and `latin1` glyph subsets

## For display driver authors

Driver authors should use `DrawTarget` exported by the [`embedded-graphics-core`](https://crates.io/crates/embedded-graphics-core) crate to integrate with embedded-graphics.

`DrawTarget` now uses an associated type for the target color instead of a type parameter.

`DrawTarget`s must also implement the `Dimensions` trait.

## For crates that handle images

Crates that handle images should now implement items exported by the [`embedded-graphics-core`](https://crates.io/crates/embedded-graphics-core) crate to integrate with embedded-graphics.

The `ImageDrawable` trait has moved there, as well as common use items like the `Dimensions` trait and `Rectangle` primitive.

TODO: Improve this section before release.

## For text rendering crates

Crates that handle text rendering should now implement items exported by the [`embedded-graphics-core`](https://crates.io/crates/embedded-graphics-core) crate to integrate with embedded-graphics.

TODO: Improve this section before release.

### Method changes

All `draw_*` methods to draw specific primitives (`draw_circle`, `draw_triangle`, etc) have been removed. The new methods provided by `DrawTarget` are as follows:

- `draw_iter`

  Draws individual pixels to the display without a defined order. This is the only required method in this trait, however will likely be the slowest pixel drawing implementation as it cannot take advantage of hardware accelerated features (e.g. filling a given area with a solid color with `fill_solid`).

- `fill_contiguous`

  Fills a given area with an iterator providing a contiguous stream of pixel colors. This may be used to efficiently draw an image or other non-transparent item to the display. The given pixel iterator can be assumed to be contiguous, iterating from top to bottom, each row left to right. This assumption potentially allows more efficient streaming of pixel data to a display.

- `fill_solid`

  Fills a given area with a solid color.

- `clear`

  Fill the entire display with a solid color.

These methods aim to be more compatible with hardware-accelerated drawing commands. Where possible, embedded-graphics primitives will use `fill_contiguous` and `fill_solid` to improve performance, however may fall back to `draw_iter` by default.

To reduce duplication, please search the `DrawTarget` documentation on <https://docs.rs/embedded-graphics> for more details on the usage and arguments of the above methods.

## General

### `Drawable`

The `Drawable` trait now uses an associated type for its pixel color instead of a type parameters.

An associated type, `Output`, has also been added which can be used to return values
from drawing operations. The unit type `()` can be used if the `draw` method doesn't need to return
anything, e.g. `type Output = ();`

```diff
- impl<'a, C: 'a> Drawable<C> for &Button<'a, C>
- where
-     C: PixelColor + From<BinaryColor>,
- {
-     fn draw<D>(self, display: &mut D) -> Result<(), D::Error> where D: DrawTarget<C> {
-         // ...
-     }
- }
+ impl<C> Drawable for Button<'_, C>
+ where
+     C: PixelColor + From<BinaryColor>,
+ {
+     type Color = C;
+
+     type Output = ();
+
+     fn draw<D>(&self, display: &mut D) -> Result<Self::Output, D::Error>
+     where
+         D: DrawTarget<Color = C>,
+     {
+         Rectangle::new(self.top_left, self.size)
+             .into_styled(PrimitiveStyle::with_fill(self.bg_color))
+             .draw(display)?;
+         Text::new(self.text, Point::new(6, 6))
+             .into_styled(TextStyle::new(Font6x8, self.fg_color))
+             .draw(display)
+     }
+ }
```

### `IntoIterator` changes

All `IntoIterator` impls are now replaced with the custom `IntoPixels` trait which exposes the `into_pixels()` method. This trait is included in the prelude, or can be included by adding `embedded_graphics::iterator::IntoPixels` to the imports list. Using the prelude is recommended.

For example, chaining two items together now requires explicit calls to `into_pixels()`:

```diff
+ use embedded_graphics::prelude::*;

let background = Rectangle::new(...);
let text = Text::new(...);

- background.into_iter().chain(&text)
+ background.into_pixels().chain(text.into_pixels())
```

## Macros are removed

All text, primitive and style macros have been removed. To create text, primitives and styles, use the appropriate struct methods instead.

For example, a styled rectangle is now built like this:

```diff
- let filled_rect: Styled<Rectangle, PrimitiveStyle<Rgb565>> = egrectangle!(
-     top_left = (10, 20),
-     bottom_right = (30, 40),
-     style = primitive_style!(stroke_color = Rgb565::RED, fill_color = Rgb565::GREEN)
- );
+ let filled_rect = Rectangle::with_corners(Point::new(10, 20), Point::new(30, 40))
+     .into_styled(
+         PrimitiveStyleBuilder::new()
+            .stroke_color(Rgb565::RED)
+            .fill_color(Rgb565::GREEN)
+            .build()
+     );
```

## Primitives

### Circle

A circle is now defined by it's top-left corner and diameter.

```diff
// Create a circle centered around (30, 30) with a diameter of 20px

use embedded_graphics::{geometry::Point, primitives::Circle};

- let circle = Circle::new(Point::new(30, 30), 10);
+ let circle = Circle::new(Point::new(20, 20), 20);
```

This allows circles with odd diameters to be created.

To retain old behaviour and create a circle from a center point and radius, use `Circle::with_center`.

```diff
use embedded_graphics::{geometry::Point, primitives::Circle};

- let circle = Circle::new(Point::new(20, 20), 5);
+ let circle = Circle::with_center(Point::new(20, 20), 5);
```

### Rectangle

Rectangles are now defined by their top-left corner and size instead of the top-left and bottom-right corner.

```diff
use embedded_graphics::{geometry::{Point, Size}, primitives::Rectangle};

- let rectangle = Rectangle::new(Point::new(20, 30), Point::new(40, 50));
+ let rectangle = Rectangle::new(Point::new(20, 30), Size::new(20, 30));
```

To retain the old behaviour, use `Rectangle::with_corners` instead:

```diff
use embedded_graphics::{geometry::{Point}, primitives::Rectangle};

- let rectangle = Rectangle::new(Point::new(20, 30), Point::new(40, 50));
+ let rectangle = Rectangle::with_corners(Point::new(20, 30), Point::new(40, 50));
```

### Triangle

The vertices of a triangle are now stored in a single `vertices` field with the type `[Point; 3]`. Previously, they were stored in three separate fields `p1`, `p2` and `p3`.

To access an individual vertex of a triangle, use `triangle.vertices[]`.

```diff
use embedded_graphics::{prelude::*, primitives::Triangle};

let triangle = Triangle::new(Point::new(20, 30), Point::new(40, 50), Point::new(60, 70));

- let p1 = triangle.p1;
- let p2 = triangle.p2;
- let p3 = triangle.p3;
+ let p1 = triangle.vertices[0];
+ let p2 = triangle.vertices[1];
+ let p3 = triangle.vertices[2];
```

To create a triangle from a slice, use the new `Triangle::from_slice` method:

```rust
use embedded_graphics::{geometry::{Point}, primitives::Triangle};

let points = [Point::new(20, 30), Point::new(40, 50), Point::new(60, 70)];

let triangle = Triangle::from_slice(&points);
```

It is no longer possible to create a triangle from an array of `Point`s. Instead, pass a reference to `Triangle::from_slice`.

## Geometry

Implementations of the `Dimensions` trait now only require the `bounding_box` method to be implemented. This should return a rectangle encompasing the entire shape.

## Mock display

The `MockDisplay`, used often for unit testing, now checks for pixel overdraw by default. To disable this behaviour, call `set_allow_overdraw(false)` on the `MockDisplay` instance.

It now also disallows out of bounds drawing by default. This behaviour can be changed by calling `display.set_allow_out_of_bounds_drawing(true)`.

## Style module

The `style` module has been removed. The items in it have been moved:

- `PrimitiveStyle` and `PrimitiveStyleBuilder` are now available under `embedded_graphics::primitives::{PrimitiveStyle, PrimitiveStyleBuilder}`.
- `TextStyle` and `TextStyleBuilder` are now available under `embedded_graphics::text::{TextStyle, TextStyleBuilder}`.

  Note that usage with `Text` has changed. Please see [the text changes section](#TODO) for more.

- `PrimitiveStyle` & `PrimitiveStyleBuilder` -> `primitives` module
- `TextStyle` & `TextStyleBuilder` -> `text` module + see text changes
- `Styled` is now exported from the crate root, e.g. `use embedded_graphics::Styled`.
