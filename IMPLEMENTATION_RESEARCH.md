# Implementation Research: Preset CLI Updates (Filtered Export, Delete, and Cascade Import)

## Executive Summary

This document provides a deep technical analysis of what would be needed to implement the features described in the PRD for the Preset CLI (backend-sdk). The analysis covers:

1. **Feature 1**: Filtered Export (export-assets enhancements)
2. **Feature 2**: Delete Assets Command (new command)
3. **Feature 3**: Cascade Control for Import (import-assets enhancements)

---

## Current Codebase Architecture

### Project Structure
```
src/preset_cli/
├── api/
│   ├── clients/
│   │   ├── superset.py          # SupersetClient - main API client
│   │   ├── preset.py            # PresetClient - Preset Manager API
│   │   └── dbt.py               # DBT API client
│   └── operators.py             # Filter operators (Equal, OneToMany)
├── cli/
│   ├── superset/
│   │   ├── main.py              # CLI entry point & command registration
│   │   ├── export.py            # export-assets command
│   │   ├── lib.py               # Progress logging utilities
│   │   └── sync/
│   │       └── native/
│   │           └── command.py   # import-assets (native) command
│   └── main.py                  # Main preset-cli entry point
├── lib.py                       # Common utilities
└── exceptions.py                # Custom exceptions
```

### Key Components

1. **SupersetClient** (`api/clients/superset.py`): Main API client with methods for:
   - `get_resources()` - List resources with filtering
   - `export_zip()` - Export resources to ZIP
   - `import_zip()` - Import ZIP bundles
   - No delete methods currently exist

2. **export_assets** (`cli/superset/export.py`): Current implementation:
   - Accepts `--asset-type`, `--dashboard-ids`, etc.
   - Uses `export_resource()` to export each type
   - Writes YAML files to a directory with Jinja escaping

3. **import-assets/native** (`cli/superset/sync/native/command.py`): Current implementation:
   - Reads YAML files from directory
   - Supports Jinja2 templating
   - Split mode for individual imports with dependency ordering
   - No cascade control

---

## Feature 1: Filtered Export (export-assets)

### Current State Analysis

**File**: `src/preset_cli/cli/superset/export.py`

The current export flow:
1. `export_assets()` command accepts ID-based filters (`--dashboard-ids`, etc.)
2. Calls `export_resource()` for each resource type
3. `export_resource()` fetches all resources via `client.get_resources()`
4. Filters by requested IDs locally
5. Calls `client.export_zip()` with filtered IDs
6. Writes YAML files to directory

### Required Changes

#### 1. Add Filter Parsing Utility

**New file or addition to `lib.py`**:

```python
from typing import Any, Dict, Tuple

DASHBOARD_FILTER_KEYS = {
    "id": int,
    "slug": str,
    "dashboard_title": str,
    "certified_by": str,
    "is_managed_externally": bool,  # maps to managed_externally in PRD
}

def parse_filters(filters: Tuple[str, ...], allowed_keys: Dict[str, type]) -> Dict[str, Any]:
    """
    Parse repeatable key=value filter strings.

    Args:
        filters: Tuple of "key=value" strings
        allowed_keys: Dict mapping key names to their expected types

    Returns:
        Dict of parsed and type-coerced filters

    Raises:
        ValueError: If key is not allowed or type coercion fails
    """
    result = {}
    for filter_str in filters:
        if "=" not in filter_str:
            raise ValueError(f"Invalid filter format: {filter_str}. Expected key=value")

        key, value = filter_str.split("=", 1)
        key = key.strip()
        value = value.strip()

        if key not in allowed_keys:
            raise ValueError(f"Unknown filter key: {key}. Allowed: {list(allowed_keys.keys())}")

        expected_type = allowed_keys[key]
        if expected_type == bool:
            result[key] = value.lower() in ("true", "1", "yes")
        elif expected_type == int:
            result[key] = int(value)
        else:
            result[key] = value

    return result
```

#### 2. Modify export.py

**New CLI options**:

```python
@click.option(
    "--filter", "-t",
    "filters",
    multiple=True,
    help="Repeatable key=value filter. Multiple filters are ANDed together.",
)
@click.option(
    "--output-zip", "-z",
    type=click.Path(resolve_path=True),
    help="Export to a ZIP file instead of a directory.",
)
@click.option(
    "--per-asset-folder",
    is_flag=True,
    default=False,
    help="Create a subfolder for each exported dashboard containing its YAML and dependencies.",
)
@click.option(
    "--simple-file-names", "-s",
    is_flag=True,
    default=False,
    help="Remove numeric suffixes from exported YAML file names.",
)
```

**Modified export_assets() function**:

```python
def export_assets(
    ctx: click.core.Context,
    directory: str,
    asset_type: Tuple[str, ...],
    database_ids: List[str],
    dataset_ids: List[str],
    chart_ids: List[str],
    dashboard_ids: List[str],
    filters: Tuple[str, ...],  # NEW
    output_zip: Optional[str],  # NEW
    per_asset_folder: bool,  # NEW
    simple_file_names: bool,  # NEW
    overwrite: bool = False,
    disable_jinja_escaping: bool = False,
    force_unix_eol: bool = False,
) -> None:
    # ... validation
    if directory and output_zip:
        raise click.UsageError("Cannot specify both directory and --output-zip")

    # Parse filters
    parsed_filters = {}
    if filters:
        parsed_filters = parse_filters(filters, DASHBOARD_FILTER_KEYS)

    # Apply filters to get matching dashboard IDs
    if parsed_filters and (not asset_type or "dashboard" in asset_type):
        matching_ids = get_filtered_resource_ids(client, "dashboard", parsed_filters)
        ids["dashboard"] = ids["dashboard"].union(matching_ids)

    # ... rest of export logic with per_asset_folder and simple_file_names support
```

**New helper function for filtered ID retrieval**:

```python
def get_filtered_resource_ids(
    client: SupersetClient,
    resource_name: str,
    filters: Dict[str, Any],
) -> Set[int]:
    """
    Fetch resources and filter by provided criteria.
    Returns set of matching resource IDs.
    """
    resources = client.get_resources(resource_name)
    matching_ids = set()

    for resource in resources:
        matches = True
        for key, value in filters.items():
            # Map filter keys to API response keys
            api_key = key
            if key == "is_managed_externally":
                api_key = "is_managed_externally"

            resource_value = resource.get(api_key)

            if key == "dashboard_title":
                # Partial match for title
                if value.lower() not in (resource_value or "").lower():
                    matches = False
                    break
            else:
                if resource_value != value:
                    matches = False
                    break

        if matches:
            matching_ids.add(resource["id"])

    return matching_ids
```

#### 3. ZIP Export Mode

**New function for ZIP creation**:

```python
from zipfile import ZipFile
from io import BytesIO

def export_to_zip(
    output_path: str,
    contents: Dict[str, str],
    per_asset_folder: bool,
    simple_file_names: bool,
) -> None:
    """
    Export contents to a ZIP file.
    """
    with ZipFile(output_path, "w") as zf:
        for file_name, file_content in contents.items():
            # Transform file names if needed
            if simple_file_names:
                file_name = remove_numeric_suffix(file_name)

            if per_asset_folder:
                file_name = reorganize_for_per_asset(file_name)

            zf.writestr(file_name, file_content)
```

#### 4. Simple File Names & Per-Asset Folder Logic

```python
import re
from collections import defaultdict

def remove_numeric_suffix(file_name: str) -> str:
    """
    Remove numeric suffix from file name.
    e.g., "sales_12345.yaml" -> "sales.yaml"
    """
    path = Path(file_name)
    stem = path.stem
    # Match pattern: name_<numbers> at end
    new_stem = re.sub(r"_\d+$", "", stem)
    return str(path.with_stem(new_stem))

def handle_name_collision(
    file_name: str,
    existing_names: Dict[str, int],
) -> str:
    """
    Handle collisions by appending _N suffix.
    """
    if file_name not in existing_names:
        existing_names[file_name] = 1
        return file_name

    existing_names[file_name] += 1
    path = Path(file_name)
    new_name = f"{path.stem}_{existing_names[file_name]}{path.suffix}"
    return str(path.with_name(new_name))
```

### API Considerations

The Superset API's `GET /api/v1/dashboard/` endpoint returns:
- `id`, `dashboard_title`, `slug`, `certified_by`, `is_managed_externally`
- These can be filtered server-side using the prison-encoded query format

**Potential optimization**: Use server-side filtering instead of client-side:

```python
def get_filtered_resource_ids(
    client: SupersetClient,
    resource_name: str,
    filters: Dict[str, Any],
) -> Set[int]:
    """
    Fetch resources with server-side filtering.
    """
    # Build prison-encoded filters for API
    api_filters = {}
    for key, value in filters.items():
        if key == "dashboard_title":
            # Use 'ct' (contains) operator for partial match
            api_filters[key] = ContainsOperator(value)
        else:
            api_filters[key] = Equal(value)

    resources = client.get_resources(resource_name, **api_filters)
    return {r["id"] for r in resources}
```

This requires adding new operators to `api/operators.py`:

```python
class ContainsOperator(Operator):
    """Contains (case-insensitive partial match) operator."""
    operator = "ct"

class NotEqual(Operator):
    """Not equal operator."""
    operator = "neq"
```

### Estimated Changes

| File | Changes |
|------|---------|
| `src/preset_cli/lib.py` | Add `parse_filters()` function (~50 lines) |
| `src/preset_cli/api/operators.py` | Add new operators (~20 lines) |
| `src/preset_cli/cli/superset/export.py` | Significant modifications (~200 lines) |
| `tests/cli/superset/export_test.py` | New tests (~300 lines) |

---

## Feature 2: Delete Assets Command (delete-assets)

### Current State Analysis

There is **no delete functionality** in the current CLI. The `SupersetClient` doesn't have delete methods.

### Required Changes

#### 1. Add Delete Methods to SupersetClient

**File**: `src/preset_cli/api/clients/superset.py`

```python
def delete_resource(self, resource_name: str, resource_id: int) -> bool:
    """
    Delete a single resource by ID.

    Args:
        resource_name: Type of resource (dashboard, chart, dataset, database)
        resource_id: ID of the resource to delete

    Returns:
        True if deletion was successful

    Raises:
        SupersetError: If deletion fails
    """
    url = self.baseurl / "api/v1" / resource_name / str(resource_id)

    _logger.debug("DELETE %s", url)
    response = self.session.delete(url)
    validate_response(response)

    return True

def delete_resources(self, resource_name: str, ids: List[int]) -> Dict[str, Any]:
    """
    Bulk delete multiple resources.

    Uses the bulk delete endpoint if available, otherwise deletes one by one.

    Returns:
        Dict with 'deleted' and 'failed' lists
    """
    # Try bulk delete endpoint first
    url = self.baseurl / "api/v1" / resource_name / ""
    params = {"q": prison.dumps(ids)}

    _logger.debug("DELETE %s", url % params)
    response = self.session.delete(url, params=params)

    if response.status_code == 404:
        # Bulk delete not supported, fall back to individual deletes
        deleted = []
        failed = []
        for id_ in ids:
            try:
                self.delete_resource(resource_name, id_)
                deleted.append(id_)
            except SupersetError as e:
                failed.append({"id": id_, "error": str(e)})
        return {"deleted": deleted, "failed": failed}

    validate_response(response)
    return response.json()
```

#### 2. Create New delete_assets Command

**New file**: `src/preset_cli/cli/superset/delete.py`

```python
"""
A command to delete Superset assets with filtering and cascade support.
"""

import logging
from typing import Any, Dict, List, Optional, Set, Tuple

import click
from yarl import URL

from preset_cli.api.clients.superset import SupersetClient
from preset_cli.lib import parse_filters, DASHBOARD_FILTER_KEYS

_logger = logging.getLogger(__name__)


class DeletionPlan:
    """Tracks what will be deleted."""

    def __init__(self):
        self.dashboards: List[Dict[str, Any]] = []
        self.charts: List[Dict[str, Any]] = []
        self.datasets: List[Dict[str, Any]] = []
        self.databases: List[Dict[str, Any]] = []

    def is_empty(self) -> bool:
        return not any([self.dashboards, self.charts, self.datasets, self.databases])

    def format_output(self, cascade_charts: bool, cascade_datasets: bool, cascade_databases: bool) -> str:
        """Format the deletion plan for display."""
        lines = []

        lines.append(f"  Dashboards ({len(self.dashboards)}):")
        for d in self.dashboards:
            lines.append(f"    - [ID: {d['id']}] {d['dashboard_title']} (slug: {d.get('slug', 'N/A')})")

        lines.append(f"\n  Charts ({len(self.charts)}):")
        if cascade_charts:
            for c in self.charts:
                lines.append(f"    - [ID: {c['id']}] {c['slice_name']} (dashboard: {c.get('dashboard_name', 'N/A')})")
        else:
            lines.append("    (not cascading)")

        lines.append(f"\n  Datasets ({len(self.datasets)}):")
        if cascade_datasets:
            for ds in self.datasets:
                lines.append(f"    - [ID: {ds['id']}] {ds['table_name']}")
        else:
            lines.append("    (not cascading)")

        lines.append(f"\n  Databases ({len(self.databases)}):")
        if cascade_databases:
            for db in self.databases:
                lines.append(f"    - [ID: {db['id']}] {db['database_name']}")
        else:
            lines.append("    (not cascading)")

        return "\n".join(lines)


def build_deletion_plan(
    client: SupersetClient,
    asset_type: str,
    filters: Dict[str, Any],
    cascade_charts: bool,
    cascade_datasets: bool,
    cascade_databases: bool,
) -> DeletionPlan:
    """
    Build a plan of what would be deleted.
    """
    plan = DeletionPlan()

    if asset_type == "dashboard":
        # Get matching dashboards
        dashboards = get_filtered_resources(client, "dashboard", filters)
        plan.dashboards = dashboards

        if cascade_charts:
            # Get charts used by these dashboards
            for dashboard in dashboards:
                dashboard_detail = client.get_resource("dashboard", dashboard["id"])
                chart_ids = extract_chart_ids_from_dashboard(dashboard_detail)
                for chart_id in chart_ids:
                    try:
                        chart = client.get_resource("chart", chart_id)
                        chart["dashboard_name"] = dashboard["dashboard_title"]
                        plan.charts.append(chart)
                    except Exception:
                        pass

            if cascade_datasets:
                # Get datasets used by these charts
                dataset_ids = set()
                for chart in plan.charts:
                    if ds_id := chart.get("datasource_id"):
                        dataset_ids.add(ds_id)

                for ds_id in dataset_ids:
                    try:
                        dataset = client.get_resource("dataset", ds_id)
                        plan.datasets.append(dataset)
                    except Exception:
                        pass

                if cascade_databases:
                    # Get databases used by these datasets
                    db_ids = set()
                    for dataset in plan.datasets:
                        if db_id := dataset.get("database", {}).get("id"):
                            db_ids.add(db_id)

                    for db_id in db_ids:
                        try:
                            database = client.get_resource("database", db_id)
                            plan.databases.append(database)
                        except Exception:
                            pass

    return plan


def execute_deletion(
    client: SupersetClient,
    plan: DeletionPlan,
    cascade_charts: bool,
    cascade_datasets: bool,
    cascade_databases: bool,
) -> Dict[str, Any]:
    """
    Execute the deletion plan.
    Returns summary of what was deleted.
    """
    results = {
        "dashboards": {"deleted": [], "failed": []},
        "charts": {"deleted": [], "failed": []},
        "datasets": {"deleted": [], "failed": []},
        "databases": {"deleted": [], "failed": []},
    }

    # Delete in reverse dependency order
    # 1. Dashboards first
    for dashboard in plan.dashboards:
        try:
            client.delete_resource("dashboard", dashboard["id"])
            results["dashboards"]["deleted"].append(dashboard["id"])
        except Exception as e:
            results["dashboards"]["failed"].append({"id": dashboard["id"], "error": str(e)})

    # 2. Charts (if cascading)
    if cascade_charts:
        for chart in plan.charts:
            try:
                client.delete_resource("chart", chart["id"])
                results["charts"]["deleted"].append(chart["id"])
            except Exception as e:
                results["charts"]["failed"].append({"id": chart["id"], "error": str(e)})

    # 3. Datasets (if cascading)
    if cascade_datasets:
        for dataset in plan.datasets:
            try:
                client.delete_resource("dataset", dataset["id"])
                results["datasets"]["deleted"].append(dataset["id"])
            except Exception as e:
                results["datasets"]["failed"].append({"id": dataset["id"], "error": str(e)})

    # 4. Databases (if cascading)
    if cascade_databases:
        for database in plan.databases:
            try:
                client.delete_resource("database", database["id"])
                results["databases"]["deleted"].append(database["id"])
            except Exception as e:
                results["databases"]["failed"].append({"id": database["id"], "error": str(e)})

    return results


@click.command()
@click.option(
    "--asset-type",
    type=click.Choice(["dashboard", "chart", "dataset", "database"], case_sensitive=False),
    required=True,
    help="Type of asset to delete.",
)
@click.option(
    "--filter", "-t",
    "filters",
    multiple=True,
    required=True,
    help="Repeatable key=value filter. At least one filter is required.",
)
@click.option(
    "--cascade-charts", "-c",
    is_flag=True,
    default=False,
    help="Also delete charts associated with the dashboards.",
)
@click.option(
    "--cascade-datasets", "-d",
    is_flag=True,
    default=False,
    help="Also delete datasets associated with cascaded charts (requires --cascade-charts).",
)
@click.option(
    "--cascade-databases", "-b",
    is_flag=True,
    default=False,
    help="Also delete databases associated with cascaded datasets (requires --cascade-datasets).",
)
@click.option(
    "--dry-run", "-r",
    is_flag=True,
    default=True,
    help="Preview what would be deleted without making changes (default: True).",
)
@click.option(
    "--confirm",
    type=str,
    default=None,
    help="Must be set to 'DELETE' to proceed with actual deletion.",
)
@click.pass_context
def delete_assets(
    ctx: click.core.Context,
    asset_type: str,
    filters: Tuple[str, ...],
    cascade_charts: bool,
    cascade_datasets: bool,
    cascade_databases: bool,
    dry_run: bool,
    confirm: Optional[str],
) -> None:
    """
    Delete assets matching the specified filters.

    Safety measures:
    - Dry-run by default
    - At least one filter required
    - Must pass --confirm=DELETE to execute
    """
    # Validate cascade hierarchy
    if cascade_datasets and not cascade_charts:
        raise click.UsageError("--cascade-datasets requires --cascade-charts")
    if cascade_databases and not cascade_datasets:
        raise click.UsageError("--cascade-databases requires --cascade-datasets")

    # Validate confirmation for actual deletion
    if not dry_run and confirm != "DELETE":
        raise click.UsageError(
            "Actual deletion requires --confirm=DELETE (case-sensitive)"
        )

    auth = ctx.obj["AUTH"]
    url = URL(ctx.obj["INSTANCE"])
    client = SupersetClient(url, auth)

    # Parse filters
    parsed_filters = parse_filters(filters, DASHBOARD_FILTER_KEYS)

    # Build deletion plan
    plan = build_deletion_plan(
        client,
        asset_type,
        parsed_filters,
        cascade_charts,
        cascade_datasets,
        cascade_databases,
    )

    if plan.is_empty():
        click.echo("No assets match the specified filters.")
        return

    # Display plan
    if dry_run:
        click.echo("Dry-run mode: No changes will be made.\n")

    click.echo("Assets to be deleted:")
    click.echo(plan.format_output(cascade_charts, cascade_datasets, cascade_databases))

    if dry_run:
        click.echo("\nTo proceed with deletion, run with: --dry-run=false --confirm=DELETE")
        return

    # Execute deletion
    click.echo("\nExecuting deletion...")
    results = execute_deletion(
        client,
        plan,
        cascade_charts,
        cascade_datasets,
        cascade_databases,
    )

    # Report results
    total_deleted = sum(len(r["deleted"]) for r in results.values())
    total_failed = sum(len(r["failed"]) for r in results.values())

    click.echo(f"\nDeletion complete: {total_deleted} deleted, {total_failed} failed")

    if total_failed > 0:
        click.echo("\nFailed deletions:")
        for resource_type, result in results.items():
            for failure in result["failed"]:
                click.echo(f"  - {resource_type} ID {failure['id']}: {failure['error']}")
```

#### 3. Register Command

**File**: `src/preset_cli/cli/superset/main.py`

```python
from preset_cli.cli.superset.delete import delete_assets

# Add to command registration
superset_cli.add_command(delete_assets)
```

### Safety Considerations

1. **Dry-run default**: The command defaults to `--dry-run=true`
2. **Confirmation required**: Actual deletion needs `--confirm=DELETE`
3. **Filter required**: At least one filter must be provided
4. **Cascade hierarchy**: Enforced cascade order (charts → datasets → databases)
5. **Error handling**: Failed deletions are logged but don't stop the process

### API Endpoints Used

| Operation | Endpoint | Method |
|-----------|----------|--------|
| List resources | `/api/v1/{resource}/` | GET |
| Get single resource | `/api/v1/{resource}/{id}` | GET |
| Delete single | `/api/v1/{resource}/{id}` | DELETE |
| Bulk delete | `/api/v1/{resource}/` | DELETE (with q param) |

### Estimated Changes

| File | Changes |
|------|---------|
| `src/preset_cli/api/clients/superset.py` | Add delete methods (~80 lines) |
| `src/preset_cli/cli/superset/delete.py` | New file (~350 lines) |
| `src/preset_cli/cli/superset/main.py` | Register command (~3 lines) |
| `tests/cli/superset/delete_test.py` | New test file (~400 lines) |
| `tests/api/clients/superset_test.py` | Add delete tests (~100 lines) |

---

## Feature 3: Cascade Control for Import (import-assets)

### Current State Analysis

**File**: `src/preset_cli/cli/superset/sync/native/command.py`

The current import flow:
1. Reads YAML files from directory
2. Renders Jinja2 templates
3. In split mode, imports in dependency order: databases → datasets → charts → dashboards
4. Each asset import includes its dependencies in the bundle
5. **No explicit cascade control** - dependencies are always included

### Required Changes

#### 1. Add Cascade Options

**File**: `src/preset_cli/cli/superset/sync/native/command.py`

```python
@click.option(
    "--cascade/--no-cascade",
    default=True,
    help="Control whether to force update dependent assets. Default: cascade enabled.",
)
```

#### 2. Modify Import Logic

The key insight is that the current implementation always bundles dependencies when importing. With `--no-cascade`:
- Dependencies should still be included for creation (if they don't exist)
- But they should NOT be overwritten if they already exist

**Implementation approach**:

```python
def import_resources_individually(
    configs: Dict[Path, AssetConfig],
    client: SupersetClient,
    overwrite: bool,
    asset_type: ResourceType,
    continue_on_error: bool = False,
    cascade: bool = True,  # NEW PARAMETER
) -> None:
    """
    Import contents individually with cascade control.
    """
    # Get existing UUIDs to check what already exists
    existing_uuids = {}
    if not cascade:
        for resource_type in ["database", "dataset", "chart", "dashboard"]:
            try:
                existing_uuids[resource_type] = set(
                    str(uuid) for uuid in client.get_uuids(resource_type).values()
                )
            except Exception:
                existing_uuids[resource_type] = set()

    # ... existing import logic ...

    for path, config in configs.items():
        if path.parts[1] != resource_name:
            continue

        asset_configs = {path: config}

        for uuid in get_related_uuids(config):
            related = related_configs.get(uuid, {})

            if not cascade:
                # Filter out dependencies that already exist
                related = {
                    p: c for p, c in related.items()
                    if c.get("uuid") not in existing_uuids.get(get_resource_type(p), set())
                }

            asset_configs.update(related)

        # ... rest of import logic ...
```

#### 3. Alternative Implementation: Selective Overwrite

A simpler approach that doesn't require pre-fetching UUIDs:

```python
def import_resources(
    contents: Dict[str, str],
    client: SupersetClient,
    overwrite: bool,
    asset_type: ResourceType,
    cascade: bool = True,  # NEW
) -> None:
    """
    Import a bundle of assets with cascade control.
    """
    if not cascade and asset_type != ResourceType.ASSET:
        # When not cascading, only mark the primary asset type for overwrite
        # Dependencies are included but won't overwrite existing
        # This requires modifying the metadata or using a different import endpoint
        pass

    # ... existing implementation ...
```

**Note**: The Superset import API doesn't natively support selective overwrite per-asset-type. This feature may require:
1. Pre-checking which UUIDs exist
2. Filtering the bundle contents
3. Or using separate import calls for dependencies vs primary assets

### Behavioral Matrix

| Mode | Asset Type | Dependency Exists? | Action |
|------|------------|-------------------|--------|
| `--cascade` (default) | Dashboard | Yes | Update dashboard AND dependencies |
| `--cascade` | Dashboard | No | Create all |
| `--no-cascade` | Dashboard | Yes | Update dashboard only, skip dep updates |
| `--no-cascade` | Dashboard | No (dep missing) | Create dependency, update dashboard |

### Estimated Changes

| File | Changes |
|------|---------|
| `src/preset_cli/cli/superset/sync/native/command.py` | Add cascade logic (~100 lines) |
| `tests/cli/superset/sync/native/command_test.py` | Add cascade tests (~200 lines) |

---

## Implementation Priority & Effort Estimates

### Priority Order (Recommended)

1. **Feature 1: Filtered Export** - Foundation for Feature 2
   - Effort: Medium (2-3 days)
   - Dependencies: None
   - Risk: Low

2. **Feature 3: Cascade Control** - Builds on existing import
   - Effort: Medium (2-3 days)
   - Dependencies: None
   - Risk: Medium (API behavior uncertainty)

3. **Feature 2: Delete Assets** - New functionality
   - Effort: High (3-4 days)
   - Dependencies: Feature 1 (filter parsing)
   - Risk: Medium (destructive operation)

### Total Estimated Lines of Code

| Component | New Code | Test Code |
|-----------|----------|-----------|
| Filter parsing utilities | ~100 | ~80 |
| Export enhancements | ~200 | ~300 |
| Delete command | ~350 | ~400 |
| Import cascade control | ~100 | ~200 |
| API client additions | ~80 | ~100 |
| **Total** | **~830** | **~1,080** |

---

## API Compatibility Notes

### Superset API Endpoints Required

| Feature | Endpoint | Notes |
|---------|----------|-------|
| Filter export | `GET /api/v1/dashboard/` | Supports prison-encoded filters |
| Delete | `DELETE /api/v1/{resource}/{id}` | Standard REST delete |
| Bulk delete | `DELETE /api/v1/{resource}/` | May not be available in all versions |
| Cascade import | `POST /api/v1/{resource}/import/` | Existing endpoint |

### Version Compatibility

- Filter operators (`eq`, `ct`, etc.) should work with Superset 1.0+
- Delete endpoints are standard REST, available in all versions
- Bulk delete may require Superset 2.0+

---

## Testing Strategy

### Unit Tests

1. Filter parsing edge cases
2. Collision handling for simple file names
3. Deletion plan building
4. Cascade dependency tracking

### Integration Tests

1. Export with filters against mock API
2. ZIP export mode
3. Delete dry-run and actual deletion
4. Import with cascade/no-cascade

### Safety Tests

1. Confirm deletion safeguards work
2. Filter requirement enforcement
3. Cascade hierarchy validation

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Delete operation is destructive | Dry-run default, confirmation required |
| API version differences | Test against multiple Superset versions |
| Filter fields may vary | Document supported fields, validate inputs |
| Cascade delete order matters | Strict dependency-aware deletion order |
| Import cascade complexity | Pre-check existing UUIDs, clear documentation |

---

## Conclusion

The implementation is feasible and aligns well with the existing codebase architecture. The main challenges are:

1. **Filter parsing**: Needs careful type coercion and validation
2. **Delete safety**: Multiple safeguards required
3. **Cascade import**: May need workarounds for API limitations

Recommended approach: Implement in phases, with comprehensive testing at each stage.
