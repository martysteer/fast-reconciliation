# Architecture Comparison

## Side-by-Side: GeoNames vs FAST

| Aspect | GeoNames | FAST (First Attempt) | FAST (Proposed) |
|--------|----------|---------------------|-----------------|
| **Data Source** | TSV (direct) | MARC XML | MARC XML |
| **Pipeline** | Simple | Complex | Complex |
| **Intermediate** | None | SKOS | SKOS |
| **Records** | ~13M | ~2.3M | ~2.3M |
| **Tables** | 1 | 10 (9+combined) | 10 |
| **Types** | 9 feature classes | 9 facets | 9 facets |
| **Docker** | ✓ Full support | ✗ | ✓ |
| **Native** | ✓ macOS/Linux | ✓ macOS only | ✓ macOS/Linux |
| **Dependencies** | Python only | Python + Java/Saxon | Python + Java/Saxon |
| **Build Time** | ~10 min | ~30-60 min | ~30-60 min |
| **DB Size** | ~3GB | ~1.5GB | ~1.5GB |

## Pipeline Comparison

### GeoNames (Simple)
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  allCountries   │────▶│    SQLite +     │────▶│   Datasette +   │
│    .zip/.txt    │     │    FTS5 Index   │     │   Reconcile     │
└─────────────────┘     └─────────────────┘     └─────────────────┘
     Download              sqlite-utils           datasette serve
```

### FAST (Complex)
```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  FASTAll.marc   │────▶│   FASTAll.skos  │────▶│   FASTAll.csv   │
│    xml.zip      │     │      (RDF)      │     │    (9 files)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
     Download           fast2skos.xsl         skos2csv.xsl
                          (Saxon)               (Saxon)
                              │
                              ▼
                    ┌─────────────────┐     ┌─────────────────┐
                    │    SQLite +     │────▶│   Datasette +   │
                    │    FTS5 Index   │     │   Reconcile     │
                    └─────────────────┘     └─────────────────┘
                       sqlite-utils           datasette serve
```

## Makefile Structure Comparison

### GeoNames (Clean)
```makefile
# Configuration
SHELL := /bin/bash
.DEFAULT_GOAL := help

# Tool paths (venv or system)
ifdef DOCKER
  PYTHON := python3
else
  PYTHON := $(VENV_DIR)/bin/python3
endif

# Main targets
build: $(VENV_DONE) $(SQLITE_DB)
serve: $(VENV_DONE) $(SQLITE_DB)
test: ...
status: ...
clean: ...

# Download
$(GEONAMES_TXT): $(GEONAMES_ZIP)
    unzip ...

# Database
$(SQLITE_DB): $(GEONAMES_TXT)
    sqlite-utils insert ...
    sqlite-utils enable-fts ...
```

### FAST First Attempt (Messy)
```makefile
# Same structure, but:
# - More intermediate targets
# - Saxon integration
# - Multiple file patterns
# - Less consistent formatting
```

### FAST Proposed (Clean + Complete)
```makefile
# Configuration (with Docker detection)
ifdef DOCKER
  PYTHON := python3
  SAXON := java -jar /opt/saxon/saxon.jar
else
  PYTHON := $(VENV_DIR)/bin/python3
  SAXON := java -jar $(SAXON_JAR)
endif

# Main targets (matching GeoNames)
build: $(VENV_DONE) $(SQLITE_DB)
serve: $(VENV_DONE) $(SQLITE_DB)

# Pipeline stages (FAST-specific)
skos: $(SKOS_FILES)
csv: $(CSV_FILES)

# Pattern rules
$(SKOS_DIR)/%.skosxml: $(MARCXML_DIR)/%.marcxml
    $(SAXON) -s:$< -xsl:fast2skos.xsl -o:$@
```

## Docker Comparison

### GeoNames Dockerfile
```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y \
    curl unzip sqlite3 make

WORKDIR /app
RUN pip install datasette datasette-reconcile sqlite-utils

COPY Makefile metadata.json ./
CMD ["make", "serve", "PUBLIC=1"]
```

### FAST Proposed Dockerfile
```dockerfile
# Stage 1: Build with Saxon
FROM eclipse-temurin:17-jdk AS builder
RUN apt-get update && apt-get install -y make curl unzip python3 python3-pip
RUN pip install sqlite-utils
COPY . /app
WORKDIR /app
RUN make csv DOCKER=1

# Stage 2: Runtime with Python
FROM python:3.12-slim
RUN apt-get update && apt-get install -y sqlite3
WORKDIR /app
RUN pip install datasette datasette-reconcile sqlite-utils httpx
COPY --from=builder /app/FASTAll.csv /app/
COPY --from=builder /app/fast.metadata.json /app/
CMD ["make", "serve", "PUBLIC=1"]
```

## Metadata Comparison

### GeoNames Types
```json
"type_default": [
  {"id": "P", "name": "P: Populated places"},
  {"id": "A", "name": "A: Administrative"},
  {"id": "H", "name": "H: Hydrographic"},
  ...
]
```

### FAST Types
```json
"type_default": [
  {"id": "Topical", "name": "Topical (subjects)"},
  {"id": "Personal", "name": "Personal (people)"},
  {"id": "Geographic", "name": "Geographic (places)"},
  ...
]
```

Both use the same plugin configuration structure - just different type codes.

## What We Keep from Each

### From GeoNames:
- Clean Makefile organization
- Docker/docker-compose setup
- Environment variable pattern for Docker
- README structure
- Test target pattern
- Status target pattern
- Help target formatting

### From FAST First Attempt:
- XSLT transforms (both files)
- Metadata JSON (all facet configs)
- Pipeline understanding
- Database schema (multi-table + combined)
- Saxon integration knowledge
- Facet type mappings

### New Additions:
- Multi-stage Dockerfile
- Health checks
- Test data CSV
- Improved documentation
- Linux native support
