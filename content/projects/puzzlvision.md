---
title: "PuzzlVision"
tagline: "Point a webcam at a sudoku"
category: "ML/CV"
status: "complete"
type: "ai-tool"
description: "Reads a 9x9 sudoku off a live camera frame, recognises the digits, solves it, and draws the answer back onto the frame."
language: "Python"
stars: 1
tech:
  - "Python"
  - "OpenCV"
  - "TensorFlow"
  - "Algorithm X"
github: "https://github.com/AbuCTF/PuzzlVision"
images:
  - "/images/projects/puzzlvision/solver.jpg"
draft: false
---

Point a webcam at a sudoku. It finds the grid, reads the digits, solves it, and paints the answer back onto the frame you're looking at.

## The pipeline

Preprocess, find the grid, flatten it, split it, read it, solve it, put it back.

Preprocessing is grayscale, a 9x9 Gaussian blur, adaptive threshold, invert, then a morphological open and a single dilate. That last dilate matters more than it sounds, because it reconnects grid lines broken by camera noise.

Finding the grid is `findContours` with `RETR_EXTERNAL`, sorted by area, approximated with `approxPolyDP` at 1% of perimeter. The first four-corner contour with area over 1000 wins, but only if it's roughly square:

```python
if not (0.95 < ratio < 1.05):
    return [], None
```

A sudoku photographed at an angle isn't square in the frame, so that check rejects almost everything until the puzzle is reasonably face-on. It's the difference between a stable read and one that flickers.

Then `getPerspectiveTransform` flattens the quad into a square, Hough lines find the grid lines, and the lines get masked out so only digits remain.

## Reading the digits

The square splits into 81 tiles. A tile is called empty if 95% of it is near black, or if the central 80% band is 90% near black. The second test catches tiles where a grid line survived the mask and would otherwise read as a stroke.

The rest go to a small CNN: three Conv2D/MaxPool blocks into dense layers, dropout, and nine softmax outputs. Nine, not ten, because a sudoku has no zero:

```python
predictions = np.argmax(preds, axis=1) + 1
```

The board is carried as an 81-character string.

## Solving

Not backtracking. The solver is Knuth's Algorithm X over the exact-cover formulation, with the four constraint families: row-column, row-number, column-number, box-number. It's taken from [a McGill implementation](https://www.cs.mcgill.ca/~aassaf9/python/sudoku.txt) under the GPL, and it's fast enough that the solve isn't the bottleneck.

Solved and unsolvable boards get memoised, so once a puzzle is recognised the following frames skip straight to drawing.

## Putting it back

`findHomography` and `warpPerspective` map the solved grid back to the original quad, `fillConvexPoly` clears the region, and `cv2.add` composites it. The overlay lands on the real puzzle at the real angle, which is the part that makes the demo look better than it is.
