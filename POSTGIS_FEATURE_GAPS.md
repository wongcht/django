# PostGIS SQL functions missing from Django's `django.contrib.gis.db.models`

Date: 2026-06-19

Sources compared:
- `https://raw.githubusercontent.com/postgis/postgis/3.6.4/NEWS` (PostGIS 3.4.0–3.6.4 "New Features" sections)
- `https://raw.githubusercontent.com/postgis/postgis/3.6.4/postgis/postgis.sql.in` (canonical SQL function signatures)
- `django/contrib/gis/db/models/functions.py`, `aggregates.py`, and `django/contrib/gis/db/backends/postgis/operations.py` (everything Django currently exposes as a `GeoFunc`/`GeoAggregate`/lookup or backend function-name mapping)

Companion to [`GEOS_API_GAPS.md`](GEOS_API_GAPS.md), which covers the same exercise one layer down, for the C library Django's Python `GEOSGeometry` binds against. Several items below are explicitly cross-referenced from there: the same operation is missing on both the Python side and the SQL side.

## Methodology

PostGIS 3.6.4 itself is a pure bugfix release. The actual new-feature surface accumulated across **3.4.0 (2023/08), 3.5.0 (2024/09), and 3.6.0 (2025/09)** — the three feature releases since Django's currently-documented minimum (PostGIS 3.2). Every function name below was checked against the full Django source tree (`grep -r` across `*.py`/`*.txt`); none of them appear anywhere today.

Unlike the GEOS document, this is not an exhaustive diff against PostGIS's entire function surface (that's many hundreds of functions across raster, topology, and tiger_geocoder, most of which are deliberately out of scope — see "Not real gaps" below). It's scoped to what's new since 3.4.0, plus a few older functions that are direct SQL-side counterparts to gaps already flagged in `GEOS_API_GAPS.md`.

**Version is not a hard exclusion**, same precedent as the GEOS doc: Django already gates individual lookups/functions behind `connection.ops.spatial_version` checks (e.g. `BaseSpatialField`/backend `spatial_version` plumbing). A function below being PostGIS 3.5+ or 3.6+ only is useful context, not a reason to exclude it.

---

## Priority 1 — Plain scalar/binary `GeoFunc`, no version-gating complexity beyond what exists

| Function | Signature | Since | Why it matters |
|---|---|---|---|
| `ST_HasZ`, `ST_HasM` | `(geometry) → boolean` | 3.5.0 | Direct SQL-side counterpart to the GEOS-level gap already noted in `GEOS_API_GAPS.md` (`geometry.py:259-264` only exposes `hasm` via Python, gated to GEOS≥3.12, with no `hasz`/no SQL equivalent). Fits the existing `@BaseSpatialField.register_lookup` + `Transform` pattern used by `IsEmpty`/`IsValid` ([functions.py:450-466](django/contrib/gis/db/models/functions.py#L450-L466)). Fills a real gap: `NumDimensions` (`ST_NDims`) reports 2/3/4 but can't tell you whether the 3rd ordinate is Z or M. |
| `ST_RemoveSmallParts` | `(geometry, double precision, double precision) → geometry` | 3.5.0 | Drops rings/parts under given width/height thresholds — common dataset-cleanup step. Same shape as the existing multi-numeric-arg `GeomOutputGeoFunc` subclasses (`Scale`, `Rotate`). |
| `ST_RemoveIrrelevantPointsForView` | `(geometry, box2d, boolean DEFAULT false) → geometry` | 3.5.0 | Viewport-aware simplification for tile/map-rendering endpoints. Second arg is a literal `box2d`, not an SRID-bearing geometry, so it should bypass `geom_param_pos`'s auto-`Transform` (the box is already meant to be in the same frame as the rendering viewport, not reprojected). |
| `ST_Project` | `(geom, float8 distance, float8 azimuth) → geometry` *and* `(geom1, geom2, float8 distance) → geometry` | 3.4.0 | "Move point by distance+bearing" / "move point1 towards point2 by distance" — two overloads, same dual-signature pattern Django already handles for `SnapToGrid` (`nargs in (1,2,4)` dispatch in [functions.py:594-610](django/contrib/gis/db/models/functions.py#L594-L610)). |
| `ST_LineExtend` | `(geom, float8 distance_forward, float8 distance_backward DEFAULT 0.0) → geometry` | 3.4.0 | Extends a `LineString` at either end. Trivial `GeomOutputGeoFunc`, default-valued second arg same as `Scale`'s `z=0.0`. |
| `ST_ShortestLine` | `(geom1, geom2) → geometry` | pre-3.4, but 3.4.0 added **geography** support | Django already has `ClosestPoint` (`ST_ClosestPoint`, a point) but no `ShortestLine` (the connecting segment) — a natural sibling, same `arity=2`/`geom_param_pos=(0,1)` shape. 3.4.0 made it usable directly on `geography` columns without an explicit cast, removing the previous reason to skip it. |

## Priority 2 — Direct SQL-side counterparts to gaps already flagged in `GEOS_API_GAPS.md`

These aren't new in 3.4–3.6 (PostGIS has wrapped the underlying GEOS calls for years), but they're listed because `GEOS_API_GAPS.md` Priority 1 flags the *Python-level* `GEOSFunc` as missing, and the *SQL-level* `Func` is equally absent from `functions.py`. A user who gets one will want the other (querying in the DB vs. operating on an already-loaded `GEOSGeometry`).

| Function | Signature | Why it matters |
|---|---|---|
| `ST_Polygonize` | aggregate `(geometry) → geometry`, also `(geometry[]) → geometry` | SQL counterpart to `GEOSPolygonize` (GEOS gaps doc, Priority 1). Fits `GeoAggregate` exactly like `Collect`/`Union`/`MakeLine` in [aggregates.py](django/contrib/gis/db/models/aggregates.py) — PostGIS already ships `CREATE AGGREGATE ST_Polygonize (geometry) (...)`. |
| `ST_BuildArea` | `(geometry) → geometry` | SQL counterpart to the same GEOS gap; single-geometry `GeomOutputGeoFunc`, no aggregate needed. |
| `ST_Node` | `(geometry) → geometry` | SQL counterpart to `GEOSNode` (GEOS gaps doc, Priority 1) — noding line networks before polygonizing. |
| `ST_Snap` | `(geom1, geom2, float8 tolerance) → geometry` | SQL counterpart to `GEOSSnap`. Three-arg `GeomOutputGeoFunc`, `geom_param_pos=(0,1)`, same shape as `Difference`/`SymDifference`. |
| `ST_SharedPaths` | `(geom1, geom2) → geometry` | SQL counterpart to `GEOSSharedPaths`. |
| `ST_OrientedEnvelope` | `(geometry) → geometry` | SQL counterpart to `GEOSMinimumRotatedRectangle`. (Not to be confused with `BoundingCircle`, already mapped to `ST_MinimumBoundingCircle` in [operations.py:182](django/contrib/gis/db/backends/postgis/operations.py#L182) — this is the *rectangle*, still missing.) |
| `ST_MinimumClearance`, `ST_MinimumClearanceLine` | `(geometry) → double precision` / `(geometry) → geometry` | SQL counterparts to the same-named GEOS functions — robustness/validity diagnostics, no equivalent today. |
| `ST_HausdorffDistance`, `ST_FrechetDistance` | `(geom1, geom2[, float8]) → double precision` | SQL counterparts to `GEOSHausdorffDistance(Densify)`/`GEOSFrechetDistance(Densify)`. Standard similarity metrics; `output_field = FloatField()` like `LineLocatePoint`. |
| `ST_OffsetCurve` | `(geom, float8 distance, text params DEFAULT '') → geometry` | SQL counterpart to `GEOSOffsetCurve`; natural sibling of the existing offset-style ops. |
| `ST_ConcaveHull` | `(geom, float8 pctconvex, boolean allow_holes DEFAULT false) → geometry` | SQL counterpart to `GEOSConcaveHull`; alternative to the existing `convex_hull`-only coverage. |

## Priority 3 — Window functions: an entire category with zero Django support today

PostGIS has a family of functions declared `LANGUAGE 'c' ... WINDOW` — called as `fn(geom, ...) OVER (PARTITION BY ...)`, operating per-row but using the whole window partition as context. Django's `functions.py`/`aggregates.py` have no representative of this category at all: `GeoFunc` assumes a plain scalar call, `GeoAggregate` assumes a single collapsed output row. Both `Window(expression=...)` and `WindowFrame` already exist in core Django (`django.db.models`), so the missing piece is purely a `GeoFunc` subclass with correct `output_field`/SRID handling that's documented as window-only — not new ORM plumbing.

| Function | Signature | Since | Why it matters |
|---|---|---|---|
| `ST_CoverageClean` | `(geom, gapMaximumWidth float8 DEFAULT 0.0, snappingDistance float8 DEFAULT -1.0, overlapMergeStrategy text DEFAULT 'MERGE_LONGEST_BORDER')` | 3.6.0 | Edge-matches/fills gaps across a polygon coverage partition (parcels, admin boundaries) — the single most requested geometry-cleanup feature in this NEWS window, and currently impossible to express through the ORM at all (a user would have to drop to `RawSQL`). |
| `ST_CoverageSimplify` | `(geom, tolerance float8, simplifyBoundary boolean DEFAULT true)` | 3.4.0 | Topology-preserving simplification across an adjacent-polygon partition — unlike a plain `Simplify`, won't open gaps/overlaps between neighbors. |
| `ST_CoverageInvalidEdges` | `(geom, tolerance float8 DEFAULT 0.0)` | 3.4.0 | Diagnostic: flags edges that break the "this is a clean coverage" invariant, natural pairing with `CoverageClean`. |
| `ST_ClusterWithinWin` | `(geom, distance float8)` | 3.4.0 | Distance-based clustering as a window function — assigns a cluster id per row without a separate aggregate query. |
| `ST_ClusterIntersectingWin` | `(geom)` | 3.4.0 | Same category, intersecting-geometry clustering. |
| `ST_ClusterDBSCAN` | `(geom, eps float8, minpoints int)` | older (2.3.0) but listed for completeness — same gap | DBSCAN clustering; no Django binding exists despite being a long-standing, widely-used PostGIS feature. |

`ST_CoverageUnion` (3.4.0) is the one function in this family that **does** fit the existing `GeoAggregate` base cleanly — PostGIS ships both a `(geometry[])` form and a `CREATE AGGREGATE ST_CoverageUnion (geometry) (...)` form, exactly mirroring how `Union`/`Collect` are already implemented. It's an optimized union specifically for non-overlapping coverage polygons (faster than general `ST_Union` for that input shape) — listed here rather than Priority 1 only because it belongs conceptually with the rest of the coverage family above.

## Priority 4 — Real, but a bigger implementation lift

| Function(s) | Signature | Since | Why it matters |
|---|---|---|---|
| `ST_LargestEmptyCircle` | `(geom, tolerance float8 DEFAULT 0.0, boundary geometry DEFAULT 'POINT EMPTY', OUT center geometry, OUT nearest geometry, OUT radius double precision)` | 3.4.0 (requires GEOS 3.9+) | Facility-siting / "biggest gap" queries. Returns a **composite/row type** (three `OUT` params), which doesn't map onto Django's `Func.output_field` (single value) the way every other candidate here does — would need either three separate calls extracting `(...).center`/`(...).nearest`/`(...).radius` via raw SQL fragments, or a small composite-type wrapper. Worth doing, but not a one-class addition like the rest of this document. |
| `ST_ClusterKMeans` | `(geom, integer, max_radius float8 DEFAULT 0)` | older, window function | Same window-function category as Priority 3, grouped separately only because it takes an upfront cluster *count* rather than a distance/eps parameter — different enough calling convention (and less commonly requested) to sequence after the others. |

## Not real gaps — intentionally out of scope

| Area | Why excluded |
|---|---|
| SFCGAL `CG_*` functions (`CG_Simplify`, `CG_3DAlphaWrapping`, `CG_StraightSkeleton*`, `CG_*Partition`, `CG_Visibility`, `CG_Intersection`/`CG_Union`/`CG_Difference`/etc.) | Gated behind the optional `postgis_sfcgal` extension — PostGIS-only, no SpatiaLite/Oracle/MySQL equivalent. `django.contrib.gis.db.models.functions` is deliberately cross-backend; every existing `GeoFunc` has at least a plausible path on other backends. |
| `postgis_topology` additions (`TotalTopologySize`, `ValidateTopologyPrecision`, `MakeTopologyPrecise`, bigint topology support, `RenameTopology`, `TopoGeo_LoadGeometry`) | Django has never modeled `postgis_topology`'s `TopoGeometry` type at all. This is a new subsystem, not an incremental addition to existing `GeometryField` support. |
| Raster additions (`ST_AsRasterAgg`, `ST_ReclassExact`, `ST_IntersectionFractions`, `ST_Clip` touched-option, min/max resampling) | `functions.py` has **zero** raster `Func` classes today — raster support is field/lookup-level only (`RasterField`, `raster=` flag on `PostGISOperator`). Adding a raster ORM function surface is a separate, much larger effort than any single item above. |
| `address_standardizer` (`debug_standardize_address`) | Separate optional extension for US/Canada address parsing, unrelated to geometry handling. |
| `postgis.gdal_cpl_debug` GUC, `--without-tiger`/`--disable-extension-upgrades-install` configure switches, interrupt-handling/PG18 signal changes | Server configuration / build-time / C-extension-internal — nothing to bind at the Django ORM layer. |
| `ST_QuantizeCoordinates` speed-up, `ST_AsSVG` curve-type support, `postgis_proj_version`/`postgis_full_version` metadata changes | Internal performance/output improvements to functions Django already calls (`AsSVG` exists; version introspection happens in `operations.py`) — nothing new to expose. |

---

## Suggested order of work

1. `ST_HasZ`/`ST_HasM` — smallest possible PR, same `register_lookup`/`Transform` pattern as `IsEmpty`/`IsValid` (Priority 1).
2. `ST_ShortestLine`, `ST_Project`, `ST_LineExtend`, `ST_RemoveSmallParts`, `ST_RemoveIrrelevantPointsForView` — plain `GeoFunc`/`GeomOutputGeoFunc`, no new abstractions (Priority 1).
3. Round out the GEOS-gap SQL counterparts that are cheap two/three-arg functions: `ST_Snap`, `ST_BuildArea`, `ST_SharedPaths`, `ST_OrientedEnvelope`, `ST_OffsetCurve`, `ST_HausdorffDistance`/`ST_FrechetDistance`, `ST_ConcaveHull`, then `ST_Polygonize`/`ST_Node` together since polygonizing a network needs noding first (Priority 2).
4. Design the one new abstraction this document calls for — a window-function-aware `GeoFunc` variant — then land `ST_CoverageUnion` (fits `GeoAggregate` immediately, no new abstraction needed) followed by `ST_CoverageClean`/`ST_CoverageSimplify`/`ST_CoverageInvalidEdges`/`ST_ClusterWithinWin`/`ST_ClusterIntersectingWin`/`ST_ClusterDBSCAN` (Priority 3).
5. `ST_LargestEmptyCircle` (composite return type) and `ST_ClusterKMeans` as standalone follow-ups once the window-function base exists (Priority 4).
