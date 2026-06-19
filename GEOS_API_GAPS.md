# GEOS C API functions missing from Django's `django.contrib.gis.geos`

Date: 2026-06-19
Sources compared:
- `/Users/tommywong/GitHub/geos/capi/geos_c.h.in` (canonical GEOS C API declarations, GEOS 3.15.0 dev tree)
- `/Users/tommywong/GitHub/django/django/contrib/gis/geos/prototypes/*.py` (everything Django currently binds via `GEOSFuncFactory`)

## Methodology

GEOS exposes ~290 distinct functions (after collapsing the `_r` reentrant variants, which is what Django always calls — see `GEOSFunc.__init__` in
[`prototypes/threadsafe.py`](django/contrib/gis/geos/prototypes/threadsafe.py#L34-L37), which appends `_r` to every name Django requests).
Diffing that list against every string literal passed to `GEOSFuncFactory(...)` across `prototypes/*.py` plus the raw bindings in
[`libgeos.py`](django/contrib/gis/geos/libgeos.py) leaves **187 unbound functions**. After excluding functions that are intentionally
unneeded (legacy one-shot WKT/WKB convenience functions superseded by Django's Reader/Writer objects, legacy global byte-order setters), this
document ranks what's left.

**Version is not treated as a hard exclusion.** Django already has a precedent for gating individual features behind a runtime GEOS version
check and raising if the installed library is too old:

```python
# geometry.py:259-264
@property
def hasm(self):
    "Return whether the geometry has a M dimension."
    if geos_version_tuple() < (3, 12):
        raise GEOSException("GEOSGeometry.hasm requires GEOS >= 3.12.0.")
    return capi.geos_hasm(self.ptr)
```

Any function below — regardless of its `@since` version — can be added using this same pattern. The version noted next to each function is
just useful context for how much real-world deployment risk/benefit it carries, not a reason to exclude it.

---

## Priority 0 — Fix a real bug first

**`GEOSCoordSeq_destroy` is not bound, and Django currently leaks memory because of it.** Fixed; see commit on this branch.

- [`GEOSCoordSeq`](django/contrib/gis/geos/coordseq.py#L16-L19) sets no `destructor`, so `CPointerBase.__del__` ([ptr.py:32-37](django/contrib/gis/ptr.py#L32-L37)) never frees the underlying `GEOSCoordSequence`.
- [`geometry.py:190-193`](django/contrib/gis/geos/geometry.py#L190-L193) (`coord_seq` property) and [`coordseq.py:240-242`](django/contrib/gis/geos/coordseq.py#L240-L242) (`.clone()`) both call `GEOSCoordSeq_clone`, which allocates a brand-new, independently-owned sequence.
- Every single call to `geom.coord_seq` therefore leaks one `GEOSCoordSequence`. Coordinate sequences created in [`linestring.py:83,119`](django/contrib/gis/geos/linestring.py#L83) are fine — they're immediately consumed by `GEOSGeom_createLineString`/`createPolygon`, which takes ownership.

Proven via RSS measurement (`resource.getrusage`, not `tracemalloc` — this memory lives in libgeos' C heap):
200,000 calls to `geom.coord_seq` grew process RSS by ~12 MiB with the bug present, ~0 MiB once each clone is freed.

**The naive fix is unsafe.** Just binding `destructor = capi.cs_destroy` on the whole `GEOSCoordSeq` class crashes (SIGABRT) immediately, because not every `GEOSCoordSeq` instance owns its pointer:
- `.clone()` / `coord_seq` (`GEOSCoordSeq_clone`) → an independently-owned copy. Must be freed.
- [`geometry.py:66-67`](django/contrib/gis/geos/geometry.py#L66-L67) (`GEOSGeometry._cs`, via `GEOSGeom_getCoordSeq`) → a pointer **borrowed** from the parent geometry, freed when the geometry itself is destroyed. Freeing it independently double-frees it.

Fix actually applied: `GEOSCoordSeq.__init__` takes an `owned=False` keyword; only `owned=True` (set by `.clone()`) binds the instance's `destructor` to the new `capi.cs_destroy` binding (`GEOSCoordSeq_destroy`). The borrowed case in `geometry.py` keeps the default `owned=False` and is never destroyed independently. Regression tests: `tests/gis_tests/geos_tests/test_coordseq.py::GEOSCoordSeqDestructorTests`.

---

## Priority 1 — High value, stable since GEOS 3.2–3.10 (no version gate needed beyond what already exists)

| Function | Since | Why it matters |
|---|---|---|
| `GEOSCoveredBy` | 3.3 | Django's DB lookups already support `__coveredby` on **every** backend (postgis/spatialite/oracle/mysql `operations.py`), but `GEOSGeometry` has `covers()` with no `covered_by()` — a glaring symmetry gap with no DB-only workaround for in-Python use. |
| `GEOSPolygonize`, `_full`, `_valid`, `GEOSPolygonizer_getCutEdges` | pre-3.2 | One of GEOS's oldest, most-used operations (PostGIS `ST_BuildArea`/`ST_Polygonize`). Django has zero support for building polygons out of a line network. |
| `GEOSReverse` | 3.7 | Trivial wrap, commonly requested (Shapely has `reverse()`); currently no workaround at all. |
| `GEOSSnap` | 3.3 | Standard fix for near-miss topology before overlay ops (PostGIS `ST_Snap`). |
| `GEOSMinimumRotatedRectangle` | 3.6 | Common bounding-shape op (PostGIS `ST_OrientedEnvelope`, Shapely `minimum_rotated_rectangle`); Django only has axis-aligned `envelope`. |
| `GEOSMinimumBoundingCircle` | 3.8 | Same category as above (PostGIS `ST_MinimumBoundingCircle`). |
| `GEOSMakeValidWithParams` + `GEOSMakeValidParams_create/destroy/setMethod/setKeepCollapsed` | 3.10 | `make_valid()` ([geometry.py:238-243](django/contrib/gis/geos/geometry.py#L238-L243)) hardcodes the default strategy; exposing params lets callers pick the structure vs. linework repair method. |
| `GEOSHausdorffDistance(Densify)`, `GEOSFrechetDistance(Densify)` | 3.2 / 3.7 | Standard similarity metrics (PostGIS `ST_HausdorffDistance`/`ST_FrechetDistance`); no equivalent exists today. |
| `GEOSOffsetCurve` | 3.3 | PostGIS `ST_OffsetCurve`; natural sibling of existing `buffer`. |
| `GEOSNode` | 3.4 | PostGIS `ST_Node`; needed before `GEOSPolygonize` is useful on arbitrary line networks. |
| `GEOSMinimumClearance`, `GEOSMinimumClearanceLine` | 3.6 | Useful validity/robustness diagnostics, no equivalent today. |
| `GEOSSharedPaths` | 3.3 | PostGIS `ST_SharedPaths`. |

## Priority 2 — High value, but require a version gate (GEOS 3.11–3.12, same pattern as `hasm`/`equals_identical`)

| Function | Since | Why it matters |
|---|---|---|
| `GEOSLineSubstring` | 3.12 | PostGIS `ST_LineSubstring`; direct sibling of the already-wrapped `interpolate`/`project` on `LinearGeometryMixin` ([geometry.py:698-731](django/contrib/gis/geos/geometry.py#L698-L731)). |
| `GEOSDensify` | 3.10 | PostGIS `ST_Densify`. |
| `GEOSRemoveRepeatedPoints` | 3.11 | PostGIS `ST_RemoveRepeatedPoints`; common cleanup step. |
| `GEOSConcaveHull`, `GEOSConcaveHullOfPolygons`, `GEOSConcaveHullByLength` | 3.11 | Increasingly requested alternative to the existing `convex_hull`. |
| `GEOSGeom_getExtent`, `getXMin/getXMax/getYMin/getYMax` | 3.11 | **Performance fix, not just a new feature.** `extent` ([geometry.py:668-683](django/contrib/gis/geos/geometry.py#L668-L683)) currently builds a full envelope *geometry* and unpacks it in Python to get 4 floats; this is a single, much cheaper C call doing the same thing. |
| `GEOSWKBReader_setFixStructure`, `GEOSWKTReader_setFixStructure` | 3.11 | Lets readers auto-repair malformed WKB/WKT (bad ring orientation, mismatched dims) instead of raising — directly useful for ingesting messy user-supplied geometry through forms/serializers. |
| `GEOSPreparedContainsXY`, `GEOSPreparedIntersectsXY` | 3.12 | Skip constructing a throwaway `Point` geometry for hot-loop point-in-polygon tests against a prepared geometry. |
| `GEOSDisjointSubsetUnion` | 3.12 | Faster `unary_union` alternative when inputs are known non-overlapping. |
| `GEOSGeom_createRectangle` | 3.11 | Convenience constructor, low effort. |

## Priority 3 — Performance-only additions to features Django already has

| Function | Since | Why it matters |
|---|---|---|
| `GEOSCoordSeq_copyToArrays/Buffer`, `copyFromArrays/Buffer`, `getXY/getXYZ`, `setXY/setXYZ` | 3.10 | Bulk coordinate access. Today `GEOSCoordSeq` ([coordseq.py](django/contrib/gis/geos/coordseq.py)) reads/writes **one ordinate at a time** via individual ctypes calls. This would meaningfully speed up numpy interop and construction of large `LineString`/`LinearRing` objects. |
| `GEOSPreparedDistance`, `GEOSPreparedDistanceWithin`, `GEOSPreparedNearestPoints` | 3.9 / 3.10 / 3.9 | Django wraps 9 prepared predicates already ([prototypes/prepared.py](django/contrib/gis/geos/prototypes/prepared.py)) but stops short of distance ops — exactly what matters most for repeated-query / spatial-join workloads where preparing a geometry pays off. |
| `GEOSDistanceIndexed` | 3.7 | Faster `distance()` for large line/polygon inputs. |

## Priority 4 — Real, but bigger implementation lift

| Function(s) | Since | Why it matters |
|---|---|---|
| `GEOSSTRtree_create/destroy/build/insert/query/nearest/nearest_generic/iterate/remove` | varies (old) | GEOS's R-tree spatial index has **no** Django binding. Valuable for in-memory nearest-neighbor/bbox queries over Python-side `GEOSGeometry` collections — the exact use case Shapely's `STRtree` serves. Bigger lift: needs a new class plus callback/array marshalling, not a one-line wrap. |
| `GEOSGeoJSONReader_create/destroy/readGeometry`, `GEOSGeoJSONWriter_*` | old | GEOS has native GeoJSON I/O. Could replace Django's hand-rolled `.json` property ([geometry.py:425-433](django/contrib/gis/geos/geometry.py#L425-L433)) and add a GeoJSON *reader*, which doesn't exist today. |
| `GEOSVoronoiDiagram` | 3.5 | PostGIS `ST_VoronoiPolygons`; real analysis feature, niche usage. |
| `GEOSDelaunayTriangulation`, `GEOSConstrainedDelaunayTriangulation` | 3.4 / 3.10 | PostGIS `ST_DelaunayTriangles`; same category. |
| `GEOS_interruptRegisterCallback/Request/Cancel`, `GEOS_interruptThread` | old / 3.14 | Lets a runaway `buffer()`/`union()` triggered by adversarial user-submitted geometry be cancelled cooperatively — a DoS-hardening option for public-facing GIS endpoints. Not commonly needed directly by app code, but valuable as internal plumbing Django could use to bound worst-case request time. |
| `GEOSCoverageUnion`, `GEOSCoverageIsValid`, `GEOSCoverageSimplifyVW`, `GEOSCoverageClean(WithParams)`, `GEOSCoverageCleanParams_*` | 3.8 / 3.12 / 3.12 / 3.15-ish | "Coverage" operations for polygon mosaics (adjacent, non-overlapping polygons — parcels, admin boundaries). Growing feature area in PostGIS too; useful but a distinct sub-domain from typical single/pairwise geometry ops. |
| `GEOSWKBWriter_setFlavor/getFlavor` | 3.10 | Controls ISO vs. extended (EWKB) output flavor; moderate value for interop with non-PostGIS WKB consumers. |

## Priority 5 — Bleeding edge (GEOS 3.13–3.15, not yet in any released GEOS Django users widely run)

These are legitimate functions and not excluded on principle, but they ship in GEOS 3.15 — currently the dev tip of the checked-out GEOS tree, not yet a stable release — so near-zero installed base today. Good candidates to revisit once 3.15 actually ships and gains adoption:

| Function(s) | Since |
|---|---|
| `GEOSClusterDBSCAN`, `GEOSClusterGeometryDistance/Intersects`, `GEOSClusterEnvelopeDistance/Intersects`, `GEOSClusterInfo_*` | 3.15 (no `@since` tag yet — newest, undocumented) |
| `GEOSSubdivideByGrid` | 3.15.0 |
| `GEOSCoverageEdges` | 3.15 |
| `GEOSGridIntersectionFractions` | 3.14.0 |
| `GEOSHilbertCode` | 3.11 (older, but niche — spatial-sort key generation, rarely needed at the ORM/framework level) |

## Not real gaps — intentionally unbound by design

| Function(s) | Why excluded |
|---|---|
| `GEOSGeomFromWKT/ToWKT`, `GEOSGeomFromWKB_buf/ToWKB_buf`, `GEOSGeomFromHEX_buf/ToHEX_buf` | Legacy one-shot convenience functions; Django already uses the equivalent `GEOSWKTReader`/`GEOSWKTWriter`/`GEOSWKBReader`/`GEOSWKBWriter` object API everywhere ([prototypes/io.py](django/contrib/gis/geos/prototypes/io.py)). |
| `GEOS_getWKBByteOrder/setWKBByteOrder`, `GEOS_getWKBOutputDims/setWKBOutputDims`, `GEOS_printDouble` | Globals tied to the deprecated non-reentrant API Django never uses (Django always calls the `_r` variants via `GEOSFunc`). |
| `GEOSContext_setErrorMessageHandler/setNoticeMessageHandler` | Alternate-signature variants of handlers Django already sets via `GEOSContext_setErrorHandler`/`setNoticeHandler` ([libgeos.py:68-71](django/contrib/gis/geos/libgeos.py#L68-L71)). |
| `GEOSGeom_getUserData/setUserData` | Django manages geometry metadata (SRID, etc.) itself in Python; no use for GEOS's opaque user-data slot. |

---

## Suggested order of work

1. Fix the `GEOSCoordSeq_destroy` leak (Priority 0).
2. `covered_by()`, `reverse()`, polygonize family, `snap()` — small, self-contained, immediately useful, no version gate complexity (Priority 1).
3. `extent` via `GEOSGeom_getExtent` — pure perf win on an existing property, plus `GEOSLineSubstring`/`densify`/`remove_repeated_points`/`concave_hull` (Priority 2, light version gating).
4. Bulk `GEOSCoordSeq` array access and `GEOSPrepared*Distance*` — perf-focused, more invasive to `coordseq.py` (Priority 3).
5. `GEOSSTRtree` binding and GeoJSON reader/writer as standalone follow-up features (Priority 4).
6. Revisit clustering/coverage-edges/subdivide-by-grid once GEOS 3.15 actually ships (Priority 5).
