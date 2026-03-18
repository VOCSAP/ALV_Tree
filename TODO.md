# TODO — ALV_Tree Audit Findings

**Class:** `ZCL_GUI_ALV_TREE_UTIL`
**Date:** 2026-03-18
**Total:** 1 Critical | 4 Major | 7 Minor | 7 Suggestion = **19 findings**

---

## 🔴 Critical

### A-001 — Empty CATCH causes dump in EXPORT_TO_EXCEL
**Category:** Missing error handling
**Method:** `EXPORT_TO_EXCEL`
**Lines:** ~761–763 (CATCH) → ~828 (APPEND <lfs_s_outtab> TO <lfs_t_outtab>)

```abap
TRY.
    ...
    CREATE DATA lo_outtab_line TYPE HANDLE lo_struct_descr.
    ASSIGN lo_outtab_line->* TO <lfs_s_outtab>.
    ...
    CREATE DATA lot_outtab TYPE HANDLE lo_table_descr.
    ASSIGN lot_outtab->* TO <lfs_t_outtab>.
  CATCH cx_sy_struct_creation cx_sy_table_creation.
    " EMPTY — field-symbols remain unassigned
ENDTRY.
...
APPEND <lfs_s_outtab> TO <lfs_t_outtab>.  " DUMP if field-symbols unassigned
```

If the dynamic structure or table creation fails (e.g., a field has no rollname in the fieldcatalog), both `<lfs_s_outtab>` and `<lfs_t_outtab>` remain unassigned. The subsequent `APPEND` at the bottom of the loop causes a runtime error `GETWA_NOT_ASSIGNED`.

**Fix:** Add an error handler that sets `rv_subrc` and returns, or at minimum add an `IS ASSIGNED` guard before the APPEND:
```abap
CATCH cx_sy_struct_creation cx_sy_table_creation INTO DATA(lx_create).
    rv_subrc = 8.
    RETURN.
```

---

## 🟠 Major

### A-002 — add_node() result not checked in NODE_ADD_RECURSIVE
**Category:** Missing error handling
**Method:** `NODE_ADD_RECURSIVE`
**Lines:** ~1089–1103

```abap
me->mo_alv_tree->add_node(
  EXPORTING ...
  EXCEPTIONS
    relat_node_not_found = 1
    node_not_found       = 2
    OTHERS               = 3
).
" NO sy-subrc check
```

If `add_node()` fails (parent key not found), `lv_new_node_key` remains initial. All children will then be inserted as root-level nodes instead of as children. Structural corruption of the tree silently.

**Fix:**
```abap
IF sy-subrc NE 0.
  RETURN.
ENDIF.
```

---

### A-003 — READ TABLE without sy-subrc check → unassigned field-symbol dump
**Category:** Missing error handling
**Method:** `ALV_TREE_POPULATE_AUTO`
**Line:** ~464

```abap
READ TABLE it_hierarchy INDEX lines( it_hierarchy )
           ASSIGNING FIELD-SYMBOL(<lfs_s_hierarchy_lower>).
" NO sy-subrc check
...
lo_alv_tree_util->add_to_appropriate_node(
    is_hierarchy_lower = <lfs_s_hierarchy_lower>  " DUMP if not assigned
).
```

If `it_hierarchy` is empty, `lines( it_hierarchy )` returns 0, the READ fails, and `<lfs_s_hierarchy_lower>` is never assigned. The subsequent method call causes a `FIELD-SYMBOL_NOT_ASSIGNED` dump.

**Fix:**
```abap
READ TABLE it_hierarchy INDEX lines( it_hierarchy )
           ASSIGNING FIELD-SYMBOL(<lfs_s_hierarchy_lower>).
IF sy-subrc NE 0.
  RETURN.
ENDIF.
```

---

### A-004 — Missing sy-subrc check on ASSIGN COMPONENT → potential dump
**Category:** Missing error handling
**Method:** `DISPLAY_DATA_SET`
**Lines:** ~636–641

```abap
ASSIGN COMPONENT <lfs_s_component_display>
              OF STRUCTURE <lfs_data>
              TO <lfs_value_target>.
" NO sy-subrc check here
<lfs_value_target> = <lfs_value_source>.  " DUMP if not assigned
```

Unlike the `it_component_excluded` loop above (where sy-subrc IS checked), in the `it_component_display` loop the target ASSIGN is unchecked. If a field in `it_component_display` doesn't exist in the target structure, `<lfs_value_target>` is unassigned and the assignment causes a dump.

**Fix:**
```abap
ASSIGN COMPONENT <lfs_s_component_display>
              OF STRUCTURE <lfs_data>
              TO <lfs_value_target>.
IF sy-subrc NE 0.
  CONTINUE.
ENDIF.
<lfs_value_target> = <lfs_value_source>.
```

---

### A-005 — ls_hierarchy stale state causes missing node_level_key after loop
**Category:** Inconsistency / Logic bug
**Method:** `ADD_TO_APPROPRIATE_NODE`
**Lines:** ~296–302

```abap
LOOP AT it_component_key ASSIGNING FIELD-SYMBOL(<lfs_s_component_key>).
  TRY.
      " Node already exists — ls_hierarchy NOT updated
    CATCH cx_sy_itab_line_not_found.
      CLEAR : ls_hierarchy.
      READ TABLE it_hierarchy ... INTO ls_hierarchy.  " Only updated here
  ENDTRY.
ENDLOOP.

" After loop:
IF NOT ls_hierarchy-node_level_key_fieldname IS INITIAL.  " Uses stale ls_hierarchy!
    ASSIGN COMPONENT ls_hierarchy-node_level_key_fieldname ...
    lv_node_level_key = <lfs_node_level_key>.
ENDIF.
```

When all parent nodes already exist (TRY block succeeds for all iterations — no CATCH fired), `ls_hierarchy` remains empty (initialized at method start). The check `IS NOT INITIAL` evaluates to false, and `lv_node_level_key` is never set for the lower node. The lower node is created without its level key.

This bug manifests during tree re-population (second call with same data) or when the tree is partially pre-built.

**Fix:** The code after the loop should use `is_hierarchy_lower-node_level_key_fieldname` instead of `ls_hierarchy-node_level_key_fieldname`, since `is_hierarchy_lower` is passed as an explicit parameter and always reflects the lower level. `ls_hierarchy` was only needed inside the loop.

---

## 🟡 Minor

### A-006 — Unused variable lt_node_text_fieldname
**Category:** Dead code
**Method:** `ADD_TO_APPROPRIATE_NODE`
**Line:** ~113

```abap
DATA :
    lt_node_text_fieldname TYPE stringtab.
```

Declared but never used anywhere in the method. Leftover from an earlier development version.

**Fix:** Remove the declaration.

---

### A-007 — Non-standard method name prefix: _NODE_TEXT_SET
**Category:** Naming violation
**Method:** `_NODE_TEXT_SET`

Leading underscore is not a standard ABAP naming convention. SAP convention uses UPPERCASE + underscore separators without leading characters. The leading `_` suggests a pseudo-"private" marker but this is expressed via the EXPOSURE attribute, not the name.

**Fix:** Rename to `NODE_TEXT_SET` (the PRIVATE exposure already marks it as non-public).

---

### A-008 — Field-symbol prefix <lfs_> deviates from SAP standard
**Category:** Naming convention
**Scope:** All methods (used throughout)

All field-symbols use `<lfs_...>` (Local Field Symbol) where SAP standard (per project CLAUDE.md) requires:
- `<LS_>` for structure field-symbols
- `<LT_>` for table field-symbols
- `<LV_>` for variable field-symbols

The `<lfs_>` convention is applied consistently across the class but deviates from the project naming standard.

**Fix:** Rename field-symbols following SAP convention in SAP GUI:
- `<lfs_s_*>` → `<ls_*>`
- `<lfs_t_*>` → `<lt_*>`
- `<lfs_*>` (scalar) → `<lv_*>`

---

### A-009 — Duplicated description line in NODE_ADD_RECURSIVE header
**Category:** Dead code / Copy-paste artifact
**Method:** `NODE_ADD_RECURSIVE`
**Lines:** ~1039–1040

```abap
*& Description     : Création Noeud dans l'ALV Tree (récursif)         *
*& Description     : Création Noeud dans l'ALV Tree (récursif)         *
```

Identical line appears twice. Copy-paste error in the comment header.

**Fix:** Remove one of the two identical lines.

---

### A-010 — Hardcoded Windows path separator
**Category:** Hardcode
**Method:** `EXPORT_TO_EXCEL`
**Line:** ~840

```abap
lv_filename = |{ iv_filepath }\\{ iv_filename }|.
```

Hardcoded `\\` Windows path separator. While SAP GUI generally runs on Windows, this is a hardcode that should use a constant.

**Fix:**
```abap
CONSTANTS lc_path_sep TYPE string VALUE '\'.
lv_filename = |{ iv_filepath }{ lc_path_sep }{ iv_filename }|.
```

---

### A-011 — Hardcoded Excel starting position
**Category:** Hardcode
**Method:** `EXPORT_TO_EXCEL`
**Lines:** ~900–901

```abap
ls_table_settings-top_left_row    = 1.
ls_table_settings-top_left_column = 'A'.
```

These values should either be passed as optional parameters or defined as named constants to document intent.

**Fix:** Define as local constants:
```abap
CONSTANTS lc_excel_start_row    TYPE i      VALUE 1.
CONSTANTS lc_excel_start_column TYPE string VALUE 'A'.
```

---

### A-012 — Duplicated comment block label
**Category:** Dead code / Copy-paste artifact
**Method:** `ALV_TREE_POPULATE_AUTO`
**Lines:** ~447 and ~466

```abap
" -----------------------------------------------------------
" Constitution des champs clefs     ← line ~447 (correct)
" -----------------------------------------------------------

" -----------------------------------------------------------
" Constitution des champs clefs     ← line ~466 (wrong — this is the SORT step)
" -----------------------------------------------------------
SORT it_alv_tree_data BY (lt_sort_table).
```

The second occurrence should describe the SORT operation, not the key construction.

**Fix:** Replace second occurrence with: `" Tri des données par ordre hiérarchique`

---

## 🔵 Suggestion

### A-013 — ADD 1 TO → += 1
**Category:** Modernization
**Method:** `ADD_TO_APPROPRIATE_NODE`
**Line:** ~161

```abap
ADD 1 TO lv_node_level.
```
→
```abap
lv_node_level += 1.
```

---

### A-014 — CREATE OBJECT → NEW in ALV_TREE_POPULATE_AUTO
**Category:** Modernization
**Method:** `ALV_TREE_POPULATE_AUTO`
**Lines:** ~438–440

```abap
CREATE OBJECT lo_alv_tree_util
  EXPORTING
    io_alv_tree = co_alv_tree.
```
→
```abap
lo_alv_tree_util = NEW #( io_alv_tree = co_alv_tree ).
```

---

### A-015 — CREATE OBJECT (×2) → NEW in EXPORT_TO_EXCEL
**Category:** Modernization
**Method:** `EXPORT_TO_EXCEL`
**Lines:** ~882–883

```abap
CREATE OBJECT lo_excel.
CREATE OBJECT lo_excel_writer.
```
→
```abap
lo_excel        = NEW #( ).
lo_excel_writer = NEW #( ).
```

---

### A-016 — MOVE-CORRESPONDING → CORRESPONDING #(...)
**Category:** Modernization
**Method:** `DISPLAY_DATA_SET`
**Line:** ~592

```abap
MOVE-CORRESPONDING is_data TO <lfs_data>.
```
→
```abap
<lfs_data> = CORRESPONDING #( is_data ).
```

---

### A-017 — CONCATENATE → string template
**Category:** Modernization
**Method:** `ADD_TO_APPROPRIATE_NODE`
**Line:** ~180

```abap
lv_key_value = |{ <lfs_key_value> }|.
CONCATENATE lv_key lv_key_value INTO lv_key RESPECTING BLANKS.
```
→
```abap
lv_key = |{ lv_key }{ <lfs_key_value> }|.
```
(Removes the intermediate `lv_key_value` variable too.)

---

### A-018 — REFRESH → CLEAR
**Category:** Modernization
**Methods:** `ALV_TREE_POPULATE` (~line 346), `ALV_TREE_POPULATE_AUTO` (~line 432)

```abap
REFRESH : et_level_node_key.
```
→
```abap
CLEAR et_level_node_key.
```

---

### A-019 — Remove [] table body notation
**Category:** Modernization
**Method:** `_NODE_TEXT_SET`
**Line:** ~1289

```abap
lt_node_text_dynamic[] = is_hierarchy-node_text_dynamic[].
```
→
```abap
lt_node_text_dynamic = is_hierarchy-node_text_dynamic.
```
Also: `IF NOT is_hierarchy-node_text_dynamic[] IS INITIAL.` → `IF is_hierarchy-node_text_dynamic IS NOT INITIAL.`
