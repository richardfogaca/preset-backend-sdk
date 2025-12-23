# Preset CLI Updates - Deliverables Breakdown

This document breaks down the implementation into discrete, shippable deliverables organized into phases.

---

## Phase 1: Foundation (Prerequisites)

### Deliverable 1.1: Filter Parsing Utilities
**Scope**: Shared utilities for parsing `key=value` filter strings

**Files to create/modify**:
- `src/preset_cli/lib.py` - Add `parse_filters()` function
- `tests/lib_test.py` - Add filter parsing tests

**Acceptance Criteria**:
- [ ] Parse `key=value` strings into typed dictionaries
- [ ] Support types: `int`, `str`, `bool`
- [ ] Validate against allowed keys
- [ ] Handle edge cases (empty values, missing `=`, invalid types)
- [ ] Unit tests with >90% coverage

**Estimated effort**: 0.5 days

---

### Deliverable 1.2: New API Filter Operators
**Scope**: Add filter operators needed for server-side filtering

**Files to create/modify**:
- `src/preset_cli/api/operators.py` - Add new operators
- `tests/api/operators_test.py` - Add operator tests

**New Operators**:
```python
class Contains(Operator):       # "ct" - partial match
    operator = "ct"

class NotEqual(Operator):       # "neq" - not equal
    operator = "neq"

class IsNull(Operator):         # "is_null" - null check
    operator = "is_null"
```

**Acceptance Criteria**:
- [ ] Operators work with prison encoding
- [ ] Operators integrate with `SupersetClient.get_resources()`
- [ ] Unit tests for each operator

**Estimated effort**: 0.5 days

---

## Phase 2: Filtered Export

### Deliverable 2.1: Basic Filter Support for export-assets
**Scope**: Add `--filter` option to export-assets command

**Files to modify**:
- `src/preset_cli/cli/superset/export.py`
- `tests/cli/superset/export_test.py`

**CLI Addition**:
```bash
preset-cli superset export-assets ./exports \
  --asset-type dashboard \
  --filter certified_by="Analytics Team" \
  --filter is_managed_externally=true
```

**Acceptance Criteria**:
- [ ] `--filter` option accepts multiple `key=value` pairs
- [ ] Filters are ANDed together
- [ ] Supports: `id`, `slug`, `dashboard_title`, `certified_by`, `is_managed_externally`
- [ ] Partial match for `dashboard_title`
- [ ] Works alongside existing `--dashboard-ids` option
- [ ] Integration tests with mocked API

**Estimated effort**: 1.5 days

**Dependencies**: 1.1, 1.2

---

### Deliverable 2.2: ZIP Export Mode
**Scope**: Add `--output-zip` option to export directly to ZIP file

**Files to modify**:
- `src/preset_cli/cli/superset/export.py`
- `tests/cli/superset/export_test.py`

**CLI Addition**:
```bash
preset-cli superset export-assets \
  --asset-type dashboard \
  --output-zip="export_$(date +%Y%m%d).zip"
```

**Acceptance Criteria**:
- [ ] `--output-zip` creates ZIP file instead of directory
- [ ] Mutually exclusive with directory argument
- [ ] Parent directories created automatically
- [ ] ZIP internal structure matches directory structure
- [ ] Error handling for write failures

**Estimated effort**: 1 day

**Dependencies**: None (can be done in parallel with 2.1)

---

### Deliverable 2.3: Simple File Names Option
**Scope**: Add `--simple-file-names` to remove numeric suffixes

**Files to modify**:
- `src/preset_cli/cli/superset/export.py`
- `tests/cli/superset/export_test.py`

**Behavior**:
- `quarterly_sales_12345.yaml` → `quarterly_sales.yaml`
- Handle collisions: `sales.yaml`, `sales_2.yaml`, `sales_3.yaml`

**Acceptance Criteria**:
- [ ] Remove `_\d+` suffix from file names
- [ ] Collision detection and resolution
- [ ] Works with both directory and ZIP modes
- [ ] Unit tests for name collision handling

**Estimated effort**: 0.5 days

**Dependencies**: 2.1 or 2.2

---

### Deliverable 2.4: Per-Asset Folder Structure
**Scope**: Add `--per-asset-folder` for organized exports

**Files to modify**:
- `src/preset_cli/cli/superset/export.py`
- `tests/cli/superset/export_test.py`

**Output Structure**:
```
exports/
└── dashboards/
    └── quarterly_sales/
        ├── dashboard.yaml
        ├── charts/
        │   └── revenue_chart.yaml
        └── datasets/
            └── sales_data.yaml
```

**Acceptance Criteria**:
- [ ] Each dashboard gets its own subfolder
- [ ] Dependencies organized within dashboard folder
- [ ] Works with `--simple-file-names`
- [ ] Works with both directory and ZIP modes

**Estimated effort**: 1 day

**Dependencies**: 2.1

---

## Phase 3: Delete Assets Command

### Deliverable 3.1: Delete Methods in SupersetClient
**Scope**: Add delete functionality to API client

**Files to modify**:
- `src/preset_cli/api/clients/superset.py`
- `tests/api/clients/superset_test.py`

**New Methods**:
```python
def delete_resource(self, resource_name: str, resource_id: int) -> bool
def delete_resources(self, resource_name: str, ids: List[int]) -> Dict[str, Any]
def get_resource(self, resource_name: str, resource_id: int) -> Dict[str, Any]
```

**Acceptance Criteria**:
- [ ] Single resource delete via `DELETE /api/v1/{resource}/{id}`
- [ ] Bulk delete support (with fallback to individual deletes)
- [ ] Proper error handling and response validation
- [ ] Integration tests with mocked API responses

**Estimated effort**: 1 day

**Dependencies**: None

---

### Deliverable 3.2: Delete Command - Dry Run Mode
**Scope**: Basic delete-assets command with dry-run functionality

**Files to create/modify**:
- `src/preset_cli/cli/superset/delete.py` (new)
- `src/preset_cli/cli/superset/main.py`
- `tests/cli/superset/delete_test.py` (new)

**CLI**:
```bash
preset-cli superset delete-assets \
  --asset-type dashboard \
  --filter slug="test-dashboard"
# Dry-run by default - shows what would be deleted
```

**Acceptance Criteria**:
- [ ] New `delete-assets` command registered
- [ ] Uses filter parsing from Phase 1
- [ ] Dry-run mode by default
- [ ] Clear output showing what would be deleted
- [ ] At least one filter required (safety)

**Estimated effort**: 1.5 days

**Dependencies**: 1.1, 3.1

---

### Deliverable 3.3: Delete Command - Cascade Options
**Scope**: Add cascade deletion for dependent resources

**Files to modify**:
- `src/preset_cli/cli/superset/delete.py`
- `tests/cli/superset/delete_test.py`

**CLI**:
```bash
preset-cli superset delete-assets \
  --asset-type dashboard \
  --filter slug="test-dashboard" \
  --cascade-charts \
  --cascade-datasets
```

**Acceptance Criteria**:
- [ ] `--cascade-charts` includes associated charts
- [ ] `--cascade-datasets` requires `--cascade-charts`
- [ ] `--cascade-databases` requires `--cascade-datasets`
- [ ] Dependency tracking (extract chart/dataset IDs from dashboards)
- [ ] Proper deletion order (dashboards → charts → datasets → databases)

**Estimated effort**: 1.5 days

**Dependencies**: 3.2

---

### Deliverable 3.4: Delete Command - Actual Deletion
**Scope**: Enable actual deletion with safety confirmation

**Files to modify**:
- `src/preset_cli/cli/superset/delete.py`
- `tests/cli/superset/delete_test.py`

**CLI**:
```bash
preset-cli superset delete-assets \
  --asset-type dashboard \
  --filter slug="test-dashboard" \
  --cascade-charts \
  --dry-run=false \
  --confirm=DELETE
```

**Acceptance Criteria**:
- [ ] `--dry-run=false` enables actual deletion
- [ ] `--confirm=DELETE` required (case-sensitive)
- [ ] Deletion results reported (success/failure counts)
- [ ] Failed deletions logged with error details
- [ ] Transaction-like behavior (continue on error, report all)

**Estimated effort**: 1 day

**Dependencies**: 3.3

---

## Phase 4: Cascade Import Control

### Deliverable 4.1: Cascade Flag for import-assets
**Scope**: Add `--cascade/--no-cascade` flag

**Files to modify**:
- `src/preset_cli/cli/superset/sync/native/command.py`
- `tests/cli/superset/sync/native/command_test.py`

**CLI**:
```bash
# Default behavior (cascade enabled)
preset-cli superset import-assets ./exports --overwrite

# Only update dashboards, don't touch existing charts/datasets
preset-cli superset import-assets ./exports --overwrite --no-cascade
```

**Acceptance Criteria**:
- [ ] `--cascade` is default (maintains backward compatibility)
- [ ] `--no-cascade` skips updates to existing dependencies
- [ ] Dependencies still created if missing
- [ ] Works with split mode (`--split`)
- [ ] Clear documentation of behavior

**Estimated effort**: 2 days

**Dependencies**: None

---

### Deliverable 4.2: UUID Pre-check for No-Cascade Mode
**Scope**: Optimize no-cascade mode with UUID checking

**Files to modify**:
- `src/preset_cli/cli/superset/sync/native/command.py`
- `src/preset_cli/api/clients/superset.py` (if needed)

**Behavior**:
- Pre-fetch existing UUIDs before import
- Filter bundle contents to exclude existing dependencies
- Only include dependencies that don't exist yet

**Acceptance Criteria**:
- [ ] Efficient UUID pre-fetching
- [ ] Correct filtering of bundle contents
- [ ] Performance acceptable for large imports
- [ ] Edge case handling (API errors, missing UUIDs)

**Estimated effort**: 1 day

**Dependencies**: 4.1

---

## Phase 5: Documentation & Polish

### Deliverable 5.1: CLI Help Text Updates
**Scope**: Update all help text and usage examples

**Files to modify**:
- All command files with new options
- `README.rst`

**Acceptance Criteria**:
- [ ] All new options have clear help text
- [ ] Examples in help output
- [ ] README updated with new features

**Estimated effort**: 0.5 days

---

### Deliverable 5.2: Integration Testing
**Scope**: End-to-end testing against real/mock Superset

**Files to create**:
- Integration test suite

**Acceptance Criteria**:
- [ ] Export → Delete → Import workflow tested
- [ ] Filter combinations tested
- [ ] Cascade scenarios tested
- [ ] Error conditions tested

**Estimated effort**: 1 day

---

## Delivery Summary

| Phase | Deliverables | Total Effort |
|-------|-------------|--------------|
| Phase 1: Foundation | 1.1, 1.2 | 1 day |
| Phase 2: Filtered Export | 2.1, 2.2, 2.3, 2.4 | 4 days |
| Phase 3: Delete Assets | 3.1, 3.2, 3.3, 3.4 | 5 days |
| Phase 4: Cascade Import | 4.1, 4.2 | 3 days |
| Phase 5: Documentation | 5.1, 5.2 | 1.5 days |
| **Total** | **14 deliverables** | **14.5 days** |

---

## Recommended Implementation Order

```
Week 1:
├── 1.1 Filter Parsing Utilities ─────────┐
├── 1.2 New API Filter Operators ─────────┤
├── 2.2 ZIP Export Mode (parallel) ───────┤
└── 3.1 Delete Methods (parallel) ────────┘
                                          │
Week 2:                                   ▼
├── 2.1 Basic Filter Support ─────────────┐
├── 2.3 Simple File Names ────────────────┤
├── 2.4 Per-Asset Folder ─────────────────┤
└── 3.2 Delete Dry-Run ───────────────────┘
                                          │
Week 3:                                   ▼
├── 3.3 Delete Cascade Options ───────────┐
├── 3.4 Delete Actual Deletion ───────────┤
└── 4.1 Cascade Flag ─────────────────────┘
                                          │
Week 4:                                   ▼
├── 4.2 UUID Pre-check ───────────────────┐
├── 5.1 Documentation ────────────────────┤
└── 5.2 Integration Testing ──────────────┘
```

---

## Milestone Releases

### v0.4.0 - Filtered Export
- Deliverables: 1.1, 1.2, 2.1, 2.2, 2.3, 2.4
- Features: `--filter`, `--output-zip`, `--simple-file-names`, `--per-asset-folder`

### v0.5.0 - Delete Assets
- Deliverables: 3.1, 3.2, 3.3, 3.4
- Features: `delete-assets` command with cascade and dry-run

### v0.6.0 - Cascade Import Control
- Deliverables: 4.1, 4.2, 5.1, 5.2
- Features: `--cascade/--no-cascade` for import-assets

---

## Risk Mitigation per Deliverable

| Deliverable | Risk | Mitigation |
|-------------|------|------------|
| 2.1 Filter Support | API filter behavior varies | Test against multiple Superset versions |
| 2.4 Per-Asset Folder | Complex dependency tracking | Reuse existing split-mode logic |
| 3.3 Cascade Delete | Wrong resources deleted | Extensive dry-run testing, confirmation |
| 4.1 Cascade Flag | API doesn't support selective overwrite | Pre-filter bundle contents client-side |
