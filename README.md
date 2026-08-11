cairo-sdl3-demo
===============

A live chart you can print — [**cairo**](https://github.com/sysl-lang/cairo) draws every frame,
[**sdl3**](https://github.com/sysl-lang/sdl3) shows it, and the same drawing code writes a PDF at the
press of a key.

![the demo](demo.png)

```
brew install cairo sdl3
sysl run . --include-path /opt/homebrew/include --link-path /opt/homebrew/lib
```

| | |
|---|---|
| `S` | write the current frame to `frame.pdf` and `frame.svg` |
| `space` | pause |
| `R` | re-seed the wave |
| `Q` | quit |

Move the mouse across the chart to read a value off it.

Why it takes two packages
-------------------------

**Neither library can do this alone**, which is the whole reason this is its own demo rather than a
third one for either package. SDL3 has a window, an event queue and a clock, and rasterizes nothing
but rectangles and textures. Cairo draws real vector graphics — gradients, Béziers, measured text,
antialiasing — and has no idea what a window is.

The join is **one buffer of ARGB32 pixels**. That is cairo's native format and one of SDL's texture
formats, and on a little-endian machine both mean the bytes B, G, R, A — so nothing is converted
anywhere and there is no per-pixel loop in this program at all:

```sysl
draw(cr, m)                                              -- cairo rasterizes the frame
frame.flush()
tex.update(as_ptr(frame.data()), i32(frame.stride()))    -- straight into the texture
r.copy(tex)
r.present()
```

The point: the window and the PDF are the same function
-------------------------------------------------------

`draw` takes a `Context`, not a window. It never asks what is underneath it. The loop hands it a
context over an image surface sixty times a second; pressing `S` hands the *same* function a context
over a PDF surface, with the *same* model:

```sysl
var pdf = pdf_surface("frame.pdf", WIDTH, HEIGHT)
var pcr = context(pdf)

draw(pcr, m)
```

So the file you get is not a screenshot. It is vector art at any zoom, with selectable text and real
curves in it — of exactly the frame you were looking at. `demo.png` above and `frame.pdf` beside it
came out of one run and are the same picture.

A chart in a canvas element cannot do that, and it is the honest reason to build one this way.

**Everything the drawing depends on is in one `Model` struct**, which is what makes that true rather
than nearly true. A frame is a pure function of a `Model`, so the export cannot drift from the
window: it is not re-derived, it is re-drawn. The hover readout is computed from the same `signal`
the curve was plotted from — never sampled off the pixels — so it survives into the PDF exact.

What it exercises
-----------------

- **cairo** — linear gradients for the background, the area fill and every bar; a rounded rectangle
  built from four arcs; dashed gridlines set and unset; text measured with `text_extents` and then
  centred or right-aligned on the measurement; a path built once and used twice with `fill_preserve`
- **sdl3** — a window, a streaming texture, vsync, the event queue for keys and mouse motion, and
  `ticks_ns` for the frame clock and the fps counter

It costs about 15% of one core at 60 fps on an M-series Mac, which is cairo software-rasterizing
900×560 every frame. **RSS is flat** — roughly ten thousand cairo objects are created and destroyed
per thousand frames, and a missing `destroy` would show up as a climb within seconds.

`--shot`
--------

```
sysl run . -- --shot
```

Draws one frame at a fixed `t` under SDL's dummy video driver and writes `demo.png`, `frame.pdf` and
`frame.svg`. Every line of the program still runs — only the display and the clock are stood in for
— so the picture in this README is one the program made and can make again.

No files
--------

No font, no image, no data file. The signal is three sines, the typeface is whatever the system calls
`"sans"`, and the only files touched are the three it writes.

License
-------

[ISC](LICENSE)
