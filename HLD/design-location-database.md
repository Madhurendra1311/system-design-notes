# Designing a Location Database: QuadTrees and Hilbert Curves (HLD Notes)

## 1. Why location databases matter

Apps like Google Maps, Uber, and Swiggy must answer one question millions of times a day: what is near this point? A location database stores places or moving objects by coordinates and answers spatial queries quickly.

Typical queries:

- **Nearest neighbours:** the closest 10 drivers to a rider, or restaurants near a customer.
- **Range query:** everything inside a rectangle or radius, such as the visible map area.
- **Point lookup:** which area or region does this coordinate belong to?

A normal database index (a B-tree on one column) handles one-dimensional data well. Location is two-dimensional (latitude and longitude), so we need structures that keep nearby points close together. These notes cover two: the quadtree and the Hilbert curve. Polygon intersections and R-trees are out of scope.

## 2. Pincodes, measurable distance, and proximity

A pincode is the easiest way to group locations: index by pincode, and a search for "restaurants in 560001" becomes a simple lookup. It breaks down in three ways.

- **Pincodes are not uniform:** one can be a few streets, another a whole district, so result sizes vary wildly.
- **No distance meaning:** pincode 560001 and 560002 may be adjacent or far apart. Numbers close in value are not guaranteed to be close on the ground.
- **Borders:** a user at the edge of one pincode is closer to places in the neighbouring one.

**Measurable distance.** Coordinates give us a real distance. For short ranges, treat the area as a flat plane and use straight-line distance; for long ranges, use the haversine formula on the sphere.

**Proximity.** "Near" means within some distance of a point. To answer it fast, we need a structure where points that are close in space are also close in storage, so we do not have to compare against every record.

## 3. Suitable data structures and 2D representation

We want fast insert, delete, and nearby-search. Candidate structures:

| Structure | Insert | Proximity search | Problem |
| --- | --- | --- | --- |
| Unsorted list | O(1) | O(n), scan everything | Too slow at scale |
| Sorted by one axis (say latitude) | O(log n) | Narrows one axis only | Still scans a long strip of the other axis |
| Grid of cells | O(1) | Look up a cell and its neighbours | Needs a sensible cell size |

**2D representation.** Model the world as a grid. Each location gets an (x, y) cell, where x comes from longitude and y from latitude. Nearby places land in the same or adjacent cells.

**Bits for the X and Y axes.** To make cells addressable, divide each axis into 2^k equal steps, so k bits describe a position on that axis. With k bits per axis, the grid has 2^k × 2^k cells. More bits mean smaller cells and finer precision. A cell can then be named by combining its x bits and y bits into one number, which can be stored and indexed like any key.

## 4. Searching in 2D and the drawback of a fixed grid

To find places near a point, compute its cell, then check that cell and the 8 surrounding cells. If the search radius is larger than one cell, widen the ring of cells checked. Each cell holds a short list of the places inside it, so a search touches a small, fixed number of lists instead of the whole dataset.

The drawback is that real data is not evenly spread:

- **Dense areas:** in a city centre one cell may hold thousands of places, so searching it is slow again.
- **Sparse areas:** in a desert or ocean, most cells are empty and waste memory.
- **One size fits none:** a cell size that suits Mumbai is far too fine for rural Rajasthan, and the reverse.

What we want is a grid whose cell size adapts to how crowded each region is. That is the quadtree.

## 5. Quad trees and range queries

A quadtree starts with one square covering the whole map and splits it into four equal quadrants only where there is too much data.

**How it is built**

1. Start with a single node covering the entire area.
2. Insert points. When a node holds more than a set capacity (say 100 points), split it into four children: north-west, north-east, south-west, south-east.
3. Move the node's points down into the matching child.
4. Repeat recursively. Crowded areas end up with deep, small cells; empty areas stay as large cells.

**Operations**

- **Insert or lookup a point:** walk from the root, choosing the quadrant that contains the point. Cost is the depth of the tree, roughly O(log n) for reasonably spread data.
- **Range query:** start at the root and visit only quadrants that overlap the search rectangle. Skip the rest. Collect the points that fall inside the rectangle at the leaves.
- **Nearest neighbours:** search the leaf containing the point, then expand to neighbouring quadrants until enough results are found.

**Strengths and weaknesses**

- Adapts to uneven density, unlike the fixed grid.
- Needs rebalancing or re-splitting as points move, which hurts for fast-moving objects such as drivers.
- It is a tree in memory, which is harder to store and index in a standard database table.

## 6. Fractals from 2D to 1D

A Hilbert curve is a single continuous line that visits every cell of a grid exactly once. Numbering the cells in the order the line visits them turns a 2D position into one number, which we can store in an ordinary sorted index such as a B-tree.

The goal is locality: cells close together in 2D should get numbers close together in 1D, so a range scan on the number returns mostly nearby places.

**Why not just join the bits?** Simple schemes such as row by row, or interleaving the x and y bits (a Z-order curve), jump across the map: the last cell of one row sits far from the first cell of the next. Hilbert's curve has no such long jumps, since consecutive numbers are always adjacent cells.

**How it is built (a fractal)**

1. Order 1: a U-shape through a 2 × 2 grid.
2. Order 2: divide each quadrant into four. Place a copy of the order-1 shape in each quadrant, rotating or flipping the ones in the lower two quadrants so the ends connect into one line.
3. Repeat for higher orders. Each step quadruples the cell count and keeps the same overall shape, which is what makes it a fractal.

With k bits per axis, the curve passes through 2^k × 2^k cells and each location maps to a number up to 2^(2k) − 1.

## 7. Hilbert curve examples and using it in a database

| Order | Grid | Cells | Curve indexes |
| --- | --- | --- | --- |
| 1 | 2 × 2 | 4 | 0 to 3 |
| 2 | 4 × 4 | 16 | 0 to 15 |
| 3 | 8 × 8 | 64 | 0 to 63 |
| k | 2^k × 2^k | 4^k | 0 to 4^k − 1 |

In the order-1 curve the path goes bottom-left, top-left, top-right, bottom-right. In the order-2 curve, cells 0 to 3 fill the bottom-left quadrant, 4 to 7 the top-left, 8 to 11 the top-right, and 12 to 15 the bottom-right. Each quadrant is a smaller U, and every quadrant owns one contiguous block of numbers.

That last point is the useful one: **every quadrant of the map, at any zoom level, maps to a contiguous range of Hilbert numbers. A search area that is not aligned to quadrants becomes a handful of such ranges.**

**Using it in a database**

1. Convert each (latitude, longitude) to a grid cell at the chosen precision.
2. Convert the cell to its Hilbert index and store it as an indexed column.
3. To search an area, work out which cells it covers and turn them into a small set of index ranges.
4. Run a normal range scan (for example, WHERE hilbert\_id BETWEEN a AND b) for each range, then filter by exact distance.

**Limits.** The curve preserves locality well but not perfectly: two cells that touch across a major quadrant boundary can have very different numbers. So searches near boundaries need several ranges, and results still need an exact distance check. Google's S2 library follows this same idea of turning space into one-dimensional keys using a Hilbert curve.

## 8. Quick revision

|  | Fixed grid | Quadtree | Hilbert curve |
| --- | --- | --- | --- |
| Idea | Equal square cells | Square split into 4 only where crowded | Number grid cells along one continuous curve |
| Handles uneven density | No | Yes | Partly, by choosing precision |
| Storage | Hash map or table | In-memory tree | Plain sorted index (B-tree) |
| Range query | Check cell and neighbours | Visit overlapping quadrants | Scan a few index ranges |
| Weak spot | Dense or empty cells | Rebalancing when objects move | Boundary jumps, needs exact distance check |
| Good for | Simple, even data | In-memory search, game maps | Database-backed location search |

**Interview takeaways**

- Start by saying why pincodes and one-axis indexes fail for proximity.
- Explain the grid first, then show how the quadtree fixes uneven density.
- To store location in a normal database, map 2D to 1D with a Hilbert curve and use a range scan.
- Always finish with an exact distance filter, since cells only narrow the candidates.