# ALV_Tree — Project Context

## Source file

| File | Lines | Format |
|------|-------|--------|
| `NUGG_ALV_TREE_V2.0.nugg` | 1365 | SAPLink NUGG (XML) |

## ABAP Objects contained

| Object | Type | Description |
|--------|------|-------------|
| `ZCL_GUI_ALV_TREE_UTIL` | Class (FINAL, PUBLIC) | Utility class for ALV Tree control |

## Class structure

### Types (10)
| Type | Kind | Description |
|------|------|-------------|
| `TS_NODE_DATA` | Structure | Node data (id, parent_id, data ref, layout, text, etc.) |
| `TT_NODE_DATA` | Sorted Table | Node data table, unique key: parent_node_id + node_id |
| `TS_KEY` | Structure | Node key-to-UUID mapping |
| `TT_KEY` | Sorted Table | Key table, unique key: key (string) |
| `TS_NODE_TEXT_DYNAMIC` | Structure (public) | Dynamic text descriptor for a field |
| `TT_NODE_TEXT_DYNAMIC` | Standard Table (public) | Table of dynamic text descriptors |
| `TS_HIERARCHY` | Structure (public) | ALV Tree level definition |
| `TT_HIERARCHY` | Sorted Table (public) | Hierarchy table, unique keys: node_level + fieldname |
| `TS_LEVEL_NODE_KEY` | Structure (public) | Node key with full text for a level |
| `TT_LEVEL_NODE_KEY` | Standard Table (public) | Table of level node keys |

### Attributes (3, all private)
| Attribute | Type | Description |
|-----------|------|-------------|
| `MO_ALV_TREE` | `CL_GUI_ALV_TREE` | ALV Tree control instance |
| `MT_KEY` | `TT_KEY` | Key → node_id mapping |
| `MT_NODE_DATA` | `TT_NODE_DATA` | Internal staging table for nodes |

### Methods (9)
| Method | Visibility | Static | Description |
|--------|-----------|--------|-------------|
| `CONSTRUCTOR` | Public | No | Assigns io_alv_tree → mo_alv_tree |
| `ADD_TO_APPROPRIATE_NODE` | Private | No | Routes a data row to the correct tree node (creates parent nodes as needed) |
| `ALV_TREE_POPULATE` | Public | No | Flushes mt_node_data to ALV tree (manual mode) |
| `ALV_TREE_POPULATE_AUTO` | Public | Yes | Full pipeline: sort data, build hierarchy, populate ALV tree |
| `DISPLAY_DATA_SET` | Private | Yes | Creates a filtered copy of a data structure for display |
| `EXPORT_TO_EXCEL` | Public | Yes | Exports ALV tree content to Excel via ZCL_EXCEL |
| `HIERARCHY_USAGE` | Public | Yes | Extracts display data + node text from a hierarchy entry |
| `NODE_ADD_RECURSIVE` | Private | No | Recursive: adds a node and all its children to the ALV tree |
| `NODE_DATA_ADD` | Public | No | Adds a node entry to the internal staging table mt_node_data |
| `_NODE_TEXT_SET` | Private | Yes | Computes node display text from hierarchy definition |

## External dependencies
- `CL_GUI_ALV_TREE` / `CL_GUI_COLUMN_TREE` — SAP standard ALV Tree control
- `CL_GUI_FRONTEND_SERVICES` — file save dialog + download
- `CL_ABAP_STRUCTDESCR` / `CL_ABAP_TABLEDESCR` / `CL_ABAP_DATADESCR` — RTTI
- `CL_SYSTEM_UUID` — UUID generation
- `CL_BCS_CONVERT` — xstring conversion
- `ZCL_EXCEL` / `ZCL_EXCEL_WORKSHEET` / `ZCL_EXCEL_WRITER_2007` / `ZCX_EXCEL` — custom Excel library
- `CGPL_FIELD_NAMES` — SAP standard type for field name tables

## Audit scope
- 1 NUGG file, 1 class, 9 methods
- No database access (UI utility class)
- No AUTHORITY-CHECK expected (no business data access)
- ABAP target: 7.40 / ECC6

## Audit status
✅ Complete — see TODO.md and MEMORY.md
