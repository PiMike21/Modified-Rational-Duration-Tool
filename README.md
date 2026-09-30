# InfoDrainage Modified Rational Duration Tool

A standalone browser utility for making tightly scoped edits to **Modified Rational (Static)** catchment dynamic-sizing values in Autodesk InfoDrainage `.iddx` files.

The tool is intentionally conservative: it does **not** rebuild, reserialize, or broadly rewrite the IDDX XML. It keeps the original UTF-8 source text and patches only the exact numeric XML attribute values that the user explicitly changes.

---

## Contents

- [Purpose](#purpose)
- [What the tool can edit](#what-the-tool-can-edit)
- [What the tool will not edit](#what-the-tool-will-not-edit)
- [Duration of Peak and Dynamic ToC synchronization](#duration-of-peak-and-dynamic-toc-synchronization)
- [Supported catchments](#supported-catchments)
- [Requirements](#requirements)
- [Quick start](#quick-start)
- [Detailed workflow](#detailed-workflow)
- [Bulk editing](#bulk-editing)
- [Individual return-period editing](#individual-return-period-editing)
- [XML mapping](#xml-mapping)
- [Text-preserving export design](#text-preserving-export-design)
- [Validation and data-integrity safeguards](#validation-and-data-integrity-safeguards)
- [Security and privacy](#security-and-privacy)
- [Performance and file-size limits](#performance-and-file-size-limits)
- [Output file](#output-file)
- [Important behavior and limitations](#important-behavior-and-limitations)
- [Troubleshooting](#troubleshooting)
- [Technical architecture](#technical-architecture)
- [Recommended validation workflow](#recommended-validation-workflow)
- [Regression test checklist](#regression-test-checklist)

---

## Purpose

The **InfoDrainage Modified Rational Duration Tool** is designed for a specific InfoDrainage workflow:

1. Open an existing `.iddx` model in a web browser.
2. Find catchments that use **Modified Rational (Static)**.
3. Edit one or more of the following catchment dynamic-sizing properties:
   - Dynamic **Time of Concentration**.
   - **Duration of Peak**.
   - **Coefficient Factor**.
4. Export a new `.iddx` file while leaving all unrelated model XML unchanged.

The application runs entirely in the browser and does not require a server, installation process, build system, or external JavaScript library.

---

## What the tool can edit

The tool is intentionally restricted to the following existing XML attributes within supported catchments.

| InfoDrainage concept | IDDX XML location | Editable | Notes |
|---|---|---:|---|
| Dynamic Time of Concentration | `AreaInflowNode/RMDetails/@TC` | Yes | Displayed in minutes; stored in seconds. |
| Dynamic ToC calculator value | `AreaInflowNode/RMDetails/ToCCalculator/@TC` | Conditionally | Updated only when the direct calculator is manual, i.e. `FlagCalc` is not `True`. |
| Return Period | `DurAdFactorPair/@RMReturnPeriod` | No | Used to identify the event row. |
| Duration of Peak | `DurAdFactorPair/@DurationOfPeak` | Yes | Displayed in minutes; stored in seconds. |
| Coefficient Factor | `DurAdFactorPair/@AdjFactor` | Yes | Written as a dimensionless numeric value. |

The tool only patches attributes that already exist in the source IDDX. It does not create missing `TC`, `DurationOfPeak`, or `AdjFactor` attributes.

---

## What the tool will not edit

The application is specifically designed to leave unrelated IDDX content untouched.

Examples of values and model content that are **not** modified include:

- `AreaInflowNode/@TCPS` — Preliminary Sizing Time of Concentration.
- Catchment labels.
- Catchment GUIDs.
- Catchment geometry and coordinates.
- Catchment areas.
- Slopes and flow-path lengths.
- Runoff coefficients other than the return-period `AdjFactor` field.
- Destination/routing data.
- Structures and pipes.
- Rainfall definitions and rainfall GUIDs.
- Land-use data.
- Phase definitions.
- Analysis results.
- XML element order.
- XML attribute order.
- Whitespace outside edited attribute values.
- Line endings outside edited attribute values.
- Comments, processing instructions, and unrelated XML text.

The Preliminary Sizing value:

```xml
<AreaInflowNode ... TCPS="300.00000600000004" ...>
```

is deliberately separate from the Dynamic Sizing value:

```xml
<RMDetails ... TC="300.00000600000004" ...>
```

The tool edits the Dynamic Sizing `RMDetails/@TC`; it does **not** edit `AreaInflowNode/@TCPS`.

---

## Duration of Peak and Dynamic ToC synchronization

A core rule of the application is:

> **When Duration of Peak is changed for a catchment, that catchment's Dynamic Time of Concentration is automatically changed to the exact same value.**

Example:

- User changes Duration of Peak to **8.5 min**.
- The tool writes the selected `DurationOfPeak` value as **510 seconds**.
- Dynamic ToC is also set to **8.5 min / 510 seconds**.
- If the direct `RMDetails/ToCCalculator` is manual, its `TC` is also synchronized to **510 seconds**.

This applies whether Duration of Peak is changed through:

- the blanket bulk-edit workflow,
- the per-catchment bulk-edit workflow, or
- the individual return-period editor.

### One Dynamic ToC per catchment

A catchment has one Dynamic ToC but may have multiple return-period rows. Therefore, the tool prevents pending edits that would require one catchment to have more than one synchronized ToC value.

For example, in one catchment, this pending edit is blocked:

| Return Period | New Duration of Peak |
|---:|---:|
| 10 yr | 5 min |
| 100 yr | 10 min |

because a single Dynamic ToC cannot simultaneously equal both 5 and 10 minutes.

This is allowed:

| Return Period | New Duration of Peak |
|---:|---:|
| 10 yr | 7 min |
| 100 yr | 7 min |

because both edited durations synchronize to the same Dynamic ToC of 7 minutes.

Untouched return-period rows may retain their original values. The conflict rule applies to the **pending Duration of Peak edits** made by this tool.

### Standalone ToC editing

Dynamic ToC may also be edited directly when the catchment has a safely editable manual ToC.

If a Duration of Peak edit is already pending for that catchment, the Dynamic ToC is locked to the edited duration and cannot be changed to a different value.

---

## Supported catchments

A catchment must satisfy all of the tool's safety conditions before it appears as editable.

### 1. It must be an `AreaInflowNode`

The tool looks for catchment inflow nodes under the phase's `InflowNodes` collection.

### 2. It must use Modified Rational (Static)

The current implementation requires:

```xml
RunoffMethod="8"
```

and an `RMDetails` element.

Catchments using other runoff methods are not edited by this tool.

### 3. The DOM structure and source-text structure must align

The application independently:

- parses the IDDX as XML, and
- scans the original source text to locate the exact character spans of editable attributes.

A catchment is editable only if the source-text scanner can safely map the XML catchment and its return-period rows back to the original text.

If that mapping is ambiguous or inconsistent, the catchment is excluded rather than risking a broad or incorrect edit.

### 4. Duration changes require a safely editable Dynamic ToC

If the catchment's direct `RMDetails/ToCCalculator` has:

```xml
FlagCalc="True"
```

Dynamic ToC is treated as calculated/read-only by this application.

Because every Duration of Peak change must synchronize Dynamic ToC, Duration editing is also blocked for that catchment.

Coefficient Factor may remain independently editable when its source attribute can be safely located.

---

## Requirements

### Browser

Use a modern desktop browser with support for:

- `DOMParser`
- `TextDecoder`
- `TextEncoder`
- `Blob`
- `URL.createObjectURL`
- standard ES6 JavaScript collections such as `Map` and `Set`

A current Chromium-based browser such as Microsoft Edge or Google Chrome is a suitable target.

### Input file

The input must be:

- an InfoDrainage `.iddx` file,
- well-formed XML,
- UTF-8 encoded,
- no larger than **50 MB**.

UTF-8 files with or without a UTF-8 BOM are supported. If the source contains a UTF-8 BOM, the exported file preserves it.

### Unsupported encoding

UTF-16 IDDX files are rejected by design. This prevents an implicit re-encoding operation from altering unrelated file bytes.

---

## Quick start

1. Save a backup copy of the original `.iddx` model.
2. Open `InfoDrainage_Modified_Rational_Duration_Tool.html` in a modern browser.
3. Drag the `.iddx` file onto the drop zone or click **Choose .iddx**.
4. Select the desired phase.
5. Search or sort the catchment table as needed.
6. Edit values using either:
   - the Dynamic ToC field in the table,
   - **Bulk edit selected**, or
   - **Edit return periods** for an individual catchment.
7. Review the changed-count indicator.
8. Click **Export updated .iddx**.
9. Open the exported file in InfoDrainage and verify the edited catchments before continuing production work.

---

## Detailed workflow

### Loading a model

When a file is loaded, the app:

1. Confirms the extension is `.iddx`.
2. Enforces the 50 MB limit.
3. Reads the raw file bytes.
4. Detects and preserves a UTF-8 BOM if present.
5. Rejects UTF-16 input.
6. Decodes the file as strict UTF-8.
7. Rejects a `DOCTYPE` declaration.
8. Parses the XML using `DOMParser`.
9. Confirms the document root is `InfoDrainage`.
10. Extracts phases and catchments.
11. Identifies Modified Rational (Static) catchments.
12. Scans the original text to locate the exact source positions of editable attributes.
13. Excludes any catchment that cannot be safely mapped back to source text.
14. Selects an analyzed phase containing editable catchments when possible.

### Phase handling

Edits are stored independently of the current phase selection.

Changing the phase:

- clears the current catchment selection,
- clears per-catchment duration-entry drafts,
- does **not** discard already committed pending edits in other phases.

Use **Discard edits** to remove all pending changes and return the application's effective values to the original file state.

### Search and sort

The table can be filtered using catchment label, destination label, or runoff-method text.

Search is debounced to reduce unnecessary large-table rerendering.

Sorting supports ascending or descending catchment label order.

### Selection behavior

**Select visible** selects only the rows currently displayed in the table.

The table renders at most **2,500** matching catchments at once. If more than 2,500 rows match, Select visible affects only those first displayed rows.

Filtering after selecting catchments does not automatically deselect previously selected catchments in the same phase. Check the **Selected** count or use **Clear selection** before applying a bulk operation if necessary.

---

## Bulk editing

The Bulk Edit panel supports three independent fields:

- Dynamic Time of Concentration.
- Duration of Peak.
- Coefficient Factor.

A blank field means **no change** for that property.

### Return-period scope

Duration of Peak and Coefficient Factor can be applied to:

- **All return periods**, or
- one specific return period present in the active phase.

If a selected catchment does not contain the chosen return period, that catchment is skipped for the duration/factor portion of the bulk operation and the status message reports the skip.

### Same Duration of Peak for all selected catchments

Choose:

**Duration of Peak entry → Same value for all selected catchments**

Enter one value in minutes. Every eligible selected catchment receives that duration within the chosen return-period scope.

For each catchment receiving a Duration of Peak edit, Dynamic ToC is synchronized to the same value.

### Different Duration of Peak per catchment

Choose:

**Duration of Peak entry → Different value per catchment**

The tool displays the selected catchments and allows a separate duration value for each one.

Example:

| Catchment | Entered Duration | Synchronized Dynamic ToC |
|---|---:|---:|
| Catchment Area | 5 min | 5 min |
| Catchment Area (1) | 7 min | 7 min |
| Catchment Area (2) | 11 min | 11 min |
| Catchment Area (62) | 8.5 min | 8.5 min |

A blank per-catchment value means no Duration of Peak change for that catchment.

Per-catchment draft values are stored by return-period scope while the file remains loaded, so switching between scopes does not automatically destroy entries for another scope.

### Bulk ToC on calculated catchments

A standalone bulk ToC value is skipped for selected catchments whose Dynamic ToC is calculated or otherwise cannot be safely patched. The status message reports how many were skipped.

A Duration of Peak edit is stricter: because Duration must synchronize ToC, the operation is blocked if a targeted catchment cannot safely accept that synchronized ToC.

### Atomic bulk operation

Bulk edits are transactional at the in-memory edit-map level.

If validation fails while applying the batch, the tool restores the previous pending edits for the selected catchments instead of leaving a partially applied operation.

---

## Individual return-period editing

Click **Edit return periods** for a catchment to open its event table.

Each existing row displays:

- Return Period.
- Duration of Peak in minutes.
- Coefficient Factor.

### Save rules

Before committing dialog edits, the app validates the entire dialog.

It rejects:

- blank editable Duration of Peak values,
- blank editable Coefficient Factor values,
- non-numeric values,
- negative values,
- non-finite values,
- multiple different new Duration of Peak values that would require conflicting Dynamic ToCs,
- Duration changes when Dynamic ToC cannot be safely synchronized.

The dialog save is atomic: if any value fails validation, the prior pending-edit state for that catchment is restored.

---

## XML mapping

A representative Modified Rational catchment may contain data similar to:

```xml
<AreaInflowNode
    Label="Catchment Area (62)"
    RunoffMethod="8"
    TCPS="300.00000600000004"
    ...>

    <RMDetails
        TC="300.00000600000004"
        ...>

        <ToCCalculator
            FlagCalc="False"
            TC="300"
            ... />

        <DurAdFactorPairs>
            <DurAdFactorPair
                RMReturnPeriod="1"
                DurationOfPeak="300.00000600000004"
                AdjFactor="1" />
            <DurAdFactorPair
                RMReturnPeriod="2"
                DurationOfPeak="300.00000600000004"
                AdjFactor="1" />
            <!-- additional return periods -->
        </DurAdFactorPairs>
    </RMDetails>
</AreaInflowNode>
```

### UI-to-XML mapping

| UI value | XML attribute | Unit in UI | Unit in IDDX |
|---|---|---:|---:|
| Preliminary Sizing ToC | `AreaInflowNode/@TCPS` | min | sec | **Read only / untouched** |
| Dynamic ToC | `RMDetails/@TC` | min | sec | Editable |
| Manual calculator ToC | `RMDetails/ToCCalculator/@TC` | min | sec | Synchronized with Dynamic ToC when safe |
| Return Period | `DurAdFactorPair/@RMReturnPeriod` | yr | yr | Read only |
| Duration of Peak | `DurAdFactorPair/@DurationOfPeak` | min | sec | Editable |
| Coefficient Factor | `DurAdFactorPair/@AdjFactor` | numeric | numeric | Editable |

### Unit conversion

The application uses:

```text
seconds = minutes × 60
minutes = seconds ÷ 60
```

For example:

```text
5 min  = 300 sec
7.5 min = 450 sec
8.5 min = 510 sec
```

---

## Text-preserving export design

This is the most important architectural safeguard in the application.

### The application does not serialize the whole IDDX for export

Although the file is parsed with `DOMParser` for structure and validation, the final output is **not** generated with `XMLSerializer`.

Instead, the application stores the original decoded UTF-8 source text and records the exact start/end character positions of approved editable attribute values.

Example source:

```xml
<DurAdFactorPair Index="3" RMReturnPeriod="10" DurationOfPeak="300.00000600000004" AdjFactor="1" />
```

If the user changes Duration of Peak to 7 minutes, only the characters representing the existing value are replaced:

```xml
DurationOfPeak="420"
```

The rest of the source tag remains the original source text.

### Approved patch targets

The patch collector can emit replacements only for:

```text
RMDetails/@TC
RMDetails/ToCCalculator/@TC        (manual calculator only)
DurAdFactorPair/@DurationOfPeak
DurAdFactorPair/@AdjFactor
```

No generic XML write function is used to rebuild the output document.

### Existing attributes only

Before a patch is created, the app confirms that the exact original source substring still matches the attribute value recorded during file load.

If the source locator is missing or inconsistent, export stops instead of guessing.

### Patch-overlap detection

All patches are sorted by source position. If two patch spans overlap, export is stopped with an internal safety error.

### UTF-8 preservation

The exported file is encoded as UTF-8. If the original file contained a UTF-8 BOM, the output also contains a UTF-8 BOM.

Because valid UTF-8 input is decoded and encoded deterministically, untouched source text is preserved while only approved numeric value spans are replaced.

---

## Validation and data-integrity safeguards

The tool contains multiple validation layers.

### File-level checks

- `.iddx` filename requirement.
- 50 MB maximum file size.
- UTF-8 validation using a fatal decoder.
- UTF-16 rejection.
- `DOCTYPE` rejection.
- Well-formed XML requirement.
- `InfoDrainage` root-element requirement.

### Catchment eligibility checks

- Must be an `AreaInflowNode`.
- Must have `RunoffMethod="8"`.
- Must contain `RMDetails`.
- DOM and source-text return-period rows must align.
- Required source attribute positions must be known before editing.

### Numeric checks

The tool rejects:

- blank values where a complete row value is required,
- `NaN`,
- positive or negative infinity,
- negative Duration of Peak,
- negative Coefficient Factor,
- Dynamic ToC less than or equal to zero,
- minute values that overflow when converted to seconds.

### No-op detection

If a new value numerically matches the original value within the application's comparison tolerance, that property is removed from the pending patch list.

If all edits for a catchment are returned to their original values, the catchment is no longer counted as changed.

### ToC/Duration consistency validation

Before export:

- a catchment cannot contain multiple different pending Duration of Peak targets,
- an edited Duration of Peak requires a matching pending Dynamic ToC,
- the Dynamic ToC must numerically equal the edited Duration of Peak.

### Export verification

After applying patches to the original source text—but **before download**—the app reparses the generated XML and verifies every intended edited value.

It checks:

- `RMDetails/@TC`,
- manual `ToCCalculator/@TC`,
- each edited `DurationOfPeak`,
- each edited `AdjFactor`.

If verification fails, the file is not downloaded.

---

## Security and privacy

The current application is a standalone local HTML file.

### No network workflow

The source contains no server upload process and no application API calls for model processing. The selected IDDX is read by the browser from the user's local file selection and the result is generated as a local browser download.

### Content Security Policy

The page includes a restrictive Content Security Policy that blocks external network connections and common embedded-content vectors:

```text
default-src 'none'
connect-src 'none'
object-src 'none'
base-uri 'none'
form-action 'none'
```

Inline application JavaScript and CSS are allowed because the tool is intentionally distributed as one self-contained HTML file.

### File-derived text rendering

Catchment names, destinations, filenames, and similar source-derived values are placed into the UI using DOM text nodes / `textContent`, rather than interpreting the values as HTML.

### XML DTD handling

Files containing a `DOCTYPE` declaration are rejected. This reduces exposure to DTD/entity-style XML processing issues and keeps the supported input format intentionally narrow.

### Resource-exhaustion controls

The file-size limit is also a security control. Very large or pathological XML files can consume substantial browser memory and CPU even without executing script.

---

## Performance and file-size limits

### Maximum IDDX size

The current limit is:

```text
50 MB per .iddx file
```

This is intentional because the browser holds several in-memory representations during the workflow, including:

- original source text,
- parsed XML DOM,
- source locator metadata,
- extracted catchment records,
- pending edit maps,
- generated output during export and verification.

### Table rendering cap

The application renders at most:

```text
2,500 matching catchments
```

at one time.

If more rows match the current phase/search filter, the UI reports that the view is capped and recommends narrowing the search.

### Search debounce

Search rendering is delayed slightly after keystrokes to reduce unnecessary large-table redraws.

### Indexed row lookup

Catchments are stored in a `Map` keyed by their stable phase/catchment row identifier, avoiding repeated linear scans during editing operations.

---

## Output file

The current exported filename format is:

```text
<original-base-name>-dynamic-sizing-updated.iddx
```

Example:

```text
MC Test Project.iddx
```

becomes:

```text
MC Test Project-dynamic-sizing-updated.iddx
```

The original selected file is never overwritten by the browser application.

---

## Important behavior and limitations

### 1. This is a targeted editor, not a general IDDX editor

Only the approved Modified Rational fields documented above are intentionally supported.

### 2. Runoff Method 8 is required

The current build treats `RunoffMethod="8"` as Modified Rational (Static). Other runoff methods are excluded.

### 3. Calculated Dynamic ToC is read-only

If the direct `RMDetails/ToCCalculator` is marked with `FlagCalc="True"`, this application does not attempt to override the calculated ToC.

Because Duration of Peak changes must synchronize Dynamic ToC, Duration editing is blocked for those catchments.

### 4. Missing attributes are not created

If an expected editable attribute does not exist in the original source, the tool does not insert it. The catchment/property is treated as unsafe for that edit.

### 5. UTF-8 only

UTF-16 models are rejected. Re-save/convert them to UTF-8 before using the tool.

### 6. The app validates structure and intended patches, not hydraulic results

The app confirms that the generated XML is well formed and that the requested attributes were written correctly. It does **not** run an InfoDrainage hydraulic analysis or prove that the edited design is hydraulically appropriate.

### 7. Final InfoDrainage round-trip remains recommended

For production use, open the exported IDDX in InfoDrainage and confirm the edited catchments and analysis behavior before replacing a production model.

### 8. Search does not imply selection scope

Previously selected catchments may remain selected after the search filter changes. The selected-count indicator is the authoritative count for a pending bulk operation.

### 9. One synchronized duration target per catchment

Multiple return-period rows may be edited together only when their new Duration of Peak values are the same for a given catchment.

---

## Troubleshooting

### "Choose an .iddx file."

The selected file does not have an `.iddx` filename extension.

### "Text-preserving mode is limited to 50.0 MB per file."

The file is larger than the current browser safety limit.

Do not simply increase the limit without testing memory consumption and export behavior on representative machines.

### "UTF-16 IDDX files are not edited..."

The file has a UTF-16 byte-order mark. Re-save or convert the model to UTF-8 before using this tool.

### "The IDDX is not valid UTF-8."

The file cannot be decoded as strict UTF-8. The app refuses to perform a silent character-set conversion because doing so could alter unrelated file content.

### "contains a DOCTYPE"

The file includes a DTD/DOCTYPE declaration. This input form is intentionally rejected.

### Catchment is missing from the editable table

Check the following:

- Is it an `AreaInflowNode`?
- Does it use `RunoffMethod="8"`?
- Does it contain `RMDetails`?
- Could its XML be safely mapped to the original source text?

The load status reports how many logically supported catchments were excluded because they could not be safely source-mapped.

### Dynamic ToC field is disabled

The catchment may have a calculated ToC (`FlagCalc="True"`) or may lack a safely locatable manual ToC attribute.

### Duration field is disabled

Duration cannot be changed unless the same catchment's Dynamic ToC can be safely synchronized.

### "multiple Duration of Peak changes ... must use the same new duration"

You attempted to assign multiple different new Duration of Peak values to return-period rows in one catchment. Use one common edited duration for that catchment.

### "Dynamic ToC ... is locked to ... because Duration of Peak has been changed"

The catchment already has a pending Duration of Peak edit. Dynamic ToC must remain equal to that edited duration.

### Some catchments were skipped for a selected return period

Those catchments do not contain the selected `RMReturnPeriod` row. No synthetic event row is added.

### Export is disabled

Export becomes available only after at least one value differs from the original IDDX.

### "Nothing to export"

All pending values have returned to their original values, so there are no source-text patches to write.

### "Internal patch overlap detected"

Two proposed attribute replacements overlap in the original source. The app stops rather than risk corrupting the IDDX. Treat this as an unexpected file-structure or application error.

### Export verification failed

The generated output could not be reparsed or an intended value did not appear at the expected XML location. No file is downloaded.

---

## Technical architecture

The application is implemented as a single self-contained HTML document containing HTML, CSS, and vanilla JavaScript.

### High-level data flow

```text
Local .iddx file
      |
      v
Read raw bytes
      |
      +--> validate UTF-8 / BOM / size
      |
      v
Original source text -----------------------------+
      |                                           |
      +--> DOMParser --> structured model         |
      |                                           |
      +--> source scanner --> attribute locators  |
                                                  |
Structured model + locators                       |
      |                                           |
      v                                           |
User edits --> pending edit map                   |
      |                                           |
      v                                           |
Validation                                        |
      |                                           |
      v                                           |
Collect exact attribute-value patches             |
      |                                           |
      +-------------------------------------------+
      |
      v
Apply patches to original source text
      |
      v
Reparse and verify intended values
      |
      v
Local .iddx download
```

### Two representations are used deliberately

#### Parsed DOM

Used for:

- phase discovery,
- catchment discovery,
- runoff-method checks,
- reading current values,
- structural verification.

#### Original source text

Used for:

- locating exact editable value spans,
- preserving unrelated XML,
- final output construction.

The DOM is **not** the export source.

### Catchment identity

The application creates a stable internal row key from:

```text
phase GUID | catchment GUID | catchment index
```

This key is used to align:

- parsed XML records,
- source-text locators,
- selections,
- pending edits,
- export verification.

### Pending edits

Edits are stored separately from the parsed DOM in an in-memory map.

Conceptually:

```text
catchment
  ├─ optional Dynamic ToC override
  └─ return-period overrides
       ├─ optional Duration of Peak
       └─ optional Coefficient Factor
```

This keeps the original source and parsed source values immutable during normal editing.

### Source scanner

A lightweight XML-aware text scanner walks start tags while respecting:

- quoted attribute values,
- comments,
- CDATA,
- processing instructions,
- element nesting.

It records only the start/end character positions of the approved attributes.

The scanner does not attempt to replace an XML parser; it exists only to safely map approved parsed elements back to their original source-text spans.

---

## Recommended validation workflow

For production engineering use, use the following procedure after a new application build or whenever InfoDrainage's IDDX structure changes.

1. Create a small controlled InfoDrainage test model.
2. Include several Modified Rational catchments.
3. Use known values for:
   - Dynamic ToC,
   - Duration of Peak,
   - Coefficient Factor.
4. Save the source IDDX.
5. Use this tool to make obvious edits, for example:
   - Catchment A: 5 → 7 min,
   - Catchment B: 5 → 10 min,
   - Coefficient Factor: 1.0 → 1.2.
6. Export the modified IDDX.
7. Compare the original and exported files with a text/binary diff.
8. Confirm that differences are limited to the intended numeric attribute values.
9. Open the exported model in InfoDrainage.
10. Confirm the Dynamic Sizing UI displays the expected values.
11. Run the appropriate InfoDrainage analysis and confirm normal model behavior.

---

## Regression test checklist

Use this checklist whenever modifying the tool.

### File loading

- [ ] Valid UTF-8 IDDX loads successfully.
- [ ] UTF-8 BOM is accepted.
- [ ] Non-IDDX extension is rejected.
- [ ] UTF-16 file is rejected.
- [ ] Invalid UTF-8 is rejected.
- [ ] XML with `DOCTYPE` is rejected.
- [ ] Malformed XML is rejected.
- [ ] Non-`InfoDrainage` root is rejected.
- [ ] File over 50 MB is rejected.

### Catchment filtering

- [ ] `RunoffMethod="8"` + `RMDetails` is included when source mapping is safe.
- [ ] Other runoff methods are excluded.
- [ ] Source-mapping mismatch excludes the catchment.
- [ ] Phase counts are correct.
- [ ] An analyzed editable phase is selected by default when present.

### Dynamic ToC

- [ ] Manual Dynamic ToC can be edited.
- [ ] `RMDetails/@TC` is patched.
- [ ] Manual direct `ToCCalculator/@TC` is patched.
- [ ] `AreaInflowNode/@TCPS` remains unchanged.
- [ ] `FlagCalc="True"` ToC is read-only.
- [ ] Blank ToC is rejected.
- [ ] Zero/negative ToC is rejected.
- [ ] Non-finite ToC is rejected.
- [ ] Returning ToC to its original value removes the pending patch.

### Duration of Peak

- [ ] Uniform bulk duration works.
- [ ] Per-catchment duration works.
- [ ] Specific return-period scope works.
- [ ] All-return-period scope works.
- [ ] Missing selected return period is reported/skipped.
- [ ] Duration is converted from minutes to seconds correctly.
- [ ] Changing Duration automatically synchronizes Dynamic ToC.
- [ ] Manual calculator ToC synchronizes when applicable.
- [ ] Duration editing is blocked when Dynamic ToC cannot be safely synchronized.
- [ ] Different pending edited durations in one catchment are blocked.
- [ ] Same pending edited duration across several return periods is accepted.
- [ ] Untouched duration rows remain unchanged.

### Coefficient Factor

- [ ] Individual factor edit works.
- [ ] Bulk factor edit works.
- [ ] Specific return-period factor edit works.
- [ ] Negative factor is rejected.
- [ ] Missing `AdjFactor` is not created.
- [ ] Returning factor to original removes its pending patch.

### Atomicity

- [ ] Invalid individual-dialog input leaves all previous pending edits unchanged.
- [ ] Invalid bulk operation leaves all previous pending edits unchanged.

### Selection and filtering

- [ ] Select visible selects only displayed rows.
- [ ] Display cap does not cause hidden rows to be selected by Select visible.
- [ ] Clear selection works.
- [ ] Search works for catchment/destination.
- [ ] Sort ascending/descending works.
- [ ] Phase change clears selection but retains committed pending edits.

### Export preservation

- [ ] Output is well-formed XML.
- [ ] Export verification succeeds for intended edits.
- [ ] UTF-8 BOM state is preserved.
- [ ] No unrelated XML changes appear in a diff.
- [ ] Attribute order is unchanged outside target values.
- [ ] Element order is unchanged.
- [ ] Whitespace is unchanged outside target values.
- [ ] Comments are unchanged.
- [ ] `TCPS` is unchanged.
- [ ] Geometry/GUID/routing data are unchanged.
- [ ] No new editable attributes are inserted.
- [ ] Output opens successfully in InfoDrainage.

---

## Operational recommendation

Treat the application as a **surgical IDDX parameter editor**, not as a replacement for InfoDrainage model validation.

Always retain the original model, review the changed-count/status messages, and validate the exported file in InfoDrainage before using it as the authoritative engineering model.
