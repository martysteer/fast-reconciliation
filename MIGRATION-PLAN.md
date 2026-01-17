# FAST Reconciliation Service - Migration Plan

## Executive Summary

Port the clean architecture from `geonames-reconciliation` to create a production-ready FAST reconciliation service, incorporating the best elements from the first FAST attempt.

---

## Repository Comparison

### GeoNames Reconciliation (Template)

**Strengths:**
- Elegant, minimal design
- Clean separation of concerns
- Native + Docker dual-mode support
- Environment variable configuration for Docker
- Excellent documentation
- Simple pipeline: Download → Extract → SQLite/FTS5
- Single data source, single table
- Comprehensive test target

**Architecture:**
```
Source (TSV) → SQLite → FTS5 → Datasette → Reconciliation API
```

**Key Files:**
- `Makefile` (single, clean)
- `geonames.metadata.json`
- `Dockerfile`
- `docker-compose.yml`
- `README.md`
- `test-data.csv`

### FAST First Attempt (Source Material)

**Strengths:**
- Working XSLT transforms (MARC → SKOS → CSV)
- Rich metadata with 9 facets
- Good understanding of FAST data structure
- LCSH cross-references preserved
- Hierarchical relationships (broader/narrower/related)
- Multiple reconciliation endpoints (per-facet + combined)

**Weaknesses:**
- No Docker support
- More complex Makefile (harder to maintain)
- Requires Saxon (Java dependency)
- No test target
- No health checks

**Architecture:**
```
Source (MARC XML) → SKOS/RDF → CSV → SQLite → FTS5 → Datasette
```

**Valuable Assets:**
- `xslt/fast2skos.xsl` - MARC to SKOS transformation
- `xslt/skos2csv-reconcile.xsl` - SKOS to flat CSV
- `fast.metadata.json` - Datasette reconciliation config
- Pipeline knowledge

---

## Migration Strategy

### Phase 1: Core Structure
Copy the clean architecture from GeoNames:
- Dual-mode Makefile (native + Docker)
- Environment variable configuration
- Consistent target naming
- Help system

### Phase 2: FAST-Specific Pipeline
Integrate the MARC → SKOS → CSV pipeline:
- Keep XSLT transforms
- Add Saxon as Docker dependency
- Make transformation stage explicit and optional

### Phase 3: Docker Support
Add containerization:
- Multi-stage Dockerfile (Saxon + Python)
- Docker Compose with volumes
- Health checks

### Phase 4: Documentation & Testing
- README mirroring GeoNames style
- Test data with sample headings
- Usage examples for OpenRefine

---

## Detailed File Structure

```
fast-reconciliation/
├── README.md                    # Main documentation
├── Makefile                     # Build automation
├── fast.metadata.json           # Datasette reconciliation config
├── Dockerfile                   # Container definition
├── docker-compose.yml           # Container orchestration
├── .gitignore                   # Ignore data/generated files
├── .python-version              # pyenv version (optional)
├── test-data.csv                # Sample reconciliation data
├── xslt/
│   ├── fast2skos.xsl           # MARC XML → SKOS/RDF
│   └── skos2csv-reconcile.xsl  # SKOS → Flat CSV
└── docs/                        # Additional documentation
    └── ...
```

---

## Key Design Decisions

### 1. Pipeline Strategy

**Option A: Full Pipeline (MARC → SKOS → CSV → SQLite)**
- Pros: SKOS intermediate is reusable, standard format
- Cons: Longer build, larger storage, Saxon dependency

**Option B: Direct Pipeline (MARC → CSV → SQLite)**
- Pros: Simpler, faster
- Cons: Loses SKOS intermediate, harder XSLT

**Recommendation: Option A (Full Pipeline)**
The SKOS intermediate provides:
- Reusability for other tools
- Standards compliance
- Cleaner separation of concerns
- Already implemented and working

### 2. Database Structure

**Option A: Single Combined Table**
- All facets in one `FAST` table
- Filter by `type` field
- Simpler for users

**Option B: Separate Tables + Combined View**
- `FASTTopical`, `FASTPersonal`, etc.
- Plus combined `FAST` table
- Multiple endpoints

**Recommendation: Option B (from first attempt)**
- Allows facet-specific reconciliation when needed
- Combined table still available as primary endpoint
- More flexible for advanced users

### 3. Docker Base Image

**Option A: Python-slim + Saxon via apt/manual**
**Option B: Eclipse Temurin (Java) + Python installed**
**Option C: Multi-stage build**

**Recommendation: Multi-stage build**
- Stage 1: Java/Saxon for transformations
- Stage 2: Python-slim for runtime
- Keeps final image small

### 4. Saxon in Docker

Since Saxon requires Java and adds complexity:
- Make XSLT transformation an optional build stage
- Allow pre-transformed CSV to be mounted
- Default: full pipeline in Docker

---

## Makefile Target Mapping

| GeoNames Target | FAST Equivalent | Notes |
|-----------------|-----------------|-------|
| `build` | `build` | Full pipeline |
| `serve` | `serve` | Start datasette |
| `test` | `test` | FTS + endpoint tests |
| `status` | `status` | Show stats |
| `update` | `update` | Re-download and rebuild |
| `clean` | `clean` | Remove DB only |
| `clean-all` | `clean-all` | Remove everything |
| `venv` | `venv` | Python environment |
| `download` | `download` | Get source data |
| (n/a) | `extract` | Unzip MARC XML |
| (n/a) | `skos` | MARC → SKOS |
| (n/a) | `csv` | SKOS → CSV |

---

## Implementation Checklist

### Phase 1: Foundation
- [ ] Create clean Makefile with GeoNames structure
- [ ] Add configuration variables section
- [ ] Add Docker mode detection
- [ ] Implement help target

### Phase 2: Pipeline
- [ ] Copy and adapt XSLT files
- [ ] Implement download target
- [ ] Implement extract target
- [ ] Implement skos target
- [ ] Implement csv target
- [ ] Implement database build

### Phase 3: Service
- [ ] Copy and adapt fast.metadata.json
- [ ] Implement serve target
- [ ] Implement test target
- [ ] Implement status target

### Phase 4: Docker
- [ ] Create Dockerfile (multi-stage)
- [ ] Create docker-compose.yml
- [ ] Test full Docker workflow

### Phase 5: Documentation
- [ ] Write README.md
- [ ] Create test-data.csv
- [ ] Add inline Makefile comments
- [ ] Update .gitignore

---

## Next Steps

1. **Approve this plan** or suggest modifications
2. **Start implementation** with the Makefile
3. **Copy XSLT files** from first attempt
4. **Create Docker files**
5. **Write documentation**
6. **Test full workflow**

---

## Estimated Effort

| Phase | Time |
|-------|------|
| Foundation (Makefile) | 1-2 hours |
| Pipeline integration | 1 hour |
| Docker support | 1-2 hours |
| Documentation | 1 hour |
| Testing | 1 hour |
| **Total** | **5-8 hours** |

---

## Questions for Review

1. Should we keep the intermediate SKOS files, or treat them as build artifacts?
2. Do you want the `docs/` folder content migrated?
3. Any preference on test data (sample headings to include)?
4. Should Docker be the primary method, or keep native as default?
