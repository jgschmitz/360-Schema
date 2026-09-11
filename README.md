# Database Schema & Collection Guide

This repository contains native MongoDB `$jsonSchema` validation rules for 11 collections divided into three primary functional groups:

- **CONTENT**: `templates`, `rules`, `keyrefs`, `template_maps`, `template_types`
- **VERSION HISTORY**: `template_versions`, `rules_version`, `keyref_versions`
- **ACTIVITY / AUDIT**: `audit_template`, `audit_rule`, `audit_keyref`

---

## Architecture Overview & Schema Standards

All schemas in this repository follow standard MongoDB `$jsonSchema` validation practices:

* **Document-Level Validation:** Evaluates individual documents (roots are `"bsonType": "object"`), rather than collection arrays.
* **Native BSON Types:** Utilizes native `objectId`, `date`, `int`, `bool`, and `binData` types directly, eliminating legacy `$oid`, `$date`, and `$binary` Extended JSON wrapper objects.
* **Strict Key Validation:** Explicitly declares `_id` (`objectId`) alongside `"additionalProperties": false` to prevent insert/update errors during standard Mongo operations.

---

## 1. Content Collections

### `templates`
* **File:** `schema_templates.json` / `templates.sample.json`
* **Main Fields:**
  * **Identity/Routing Metadata:** `key`, `fileName`, `title`, `templateTypeId`, `lob`, `state`, `subTenancy`, `parentId`, `xmlTemplatePath`, `resourceMetadataId`
  * **Content Trees:** `blocks[]` (`DocBlock` tree containing headings, sections, fields, groups), `conditionalVariants[]` (whole-topic rule-gated alternates), `costShares[]` (tier/bucket cost-share values referencing keyrefs or manual metadata)
  * **Derived Arrays:** `ruleRefs[]`, `keyrefRefs[]`, `ruleBindings[]`
  * **Housekeeping:** `updatedAt` (`date`), `updatedBy` (`string`), `version` (`int`)
* **Population Strategy:** Authors edit `blocks`, `conditionalVariants`, and `costShares` via the UI. Recomputation pipelines automatically recalculate `ruleRefs`, `keyrefRefs`, and `ruleBindings` on save to match markup.
* **Rule & Keyref Resolution:** Scans inline `data-rule` and `data-var` HTML attributes, rule-gated variants, and cost-share references to map exact dependencies (`RuleBinding`).
* **Versioning & Audit Trigger:** Save events store a binary snapshot in `template_versions` and append a event line to `audit_template`.

---

### `rules`
* **File:** `schema_rules.json` / `rules.sample.json`
* **Main Fields:**
  * **Identity:** `key` (e.g., `"rs700"`), `ruleId` (e.g., `"uhcRule_DesignatedNetwork_Applies"`)
  * **Expression Tree:** `root` (recursive `RuleNode` using `RuleGroupNode` and `RuleConditionNode`)
  * **Metadata:** `valid` (`bool`), `description`, `appName`, `subTenancy`, `updatedAt` (`date`), `updatedBy`, `version` (`int`)
* **Population Strategy:** Built in the Rule Registry UI as a condition tree evaluating plan-config conditions (`field`, `operator`, `value`).
* **Binding Mechanism:** Templates embed rules via `data-rule="key(true|false)"`. Rule logic is evaluated dynamically at generation/render time against active plan data.
* **Versioning & Audit Trigger:** Saves push an inline snapshot to `rules_version` (incrementing `version`) and append to `audit_rule`.

---

### `keyrefs`
* **File:** `schema_keyrefs.json` / `keyrefs.sample.json`
* **Main Fields:**
  * **Identity:** `key`, `variableName`
  * **Source Info:** `appName`, `subTenancy`, `sourceSystemService`, `sourceSystem` (`sourceSystemName`: `"Plan Library"` | `"Nimbus"` | `"Cirrus"`, `sourceSystemInfo`: generic lookup metadata bag)
  * **Metadata:** `updatedAt` (`date`), `updatedBy`, `version` (`int`)
* **Population Strategy:** Imported and mapped from upstream sheets (Plan Library, Nimbus, Cirrus). Sub-fields vary depending on the `sourceSubSection` originating tab.
* **Binding Mechanism:** Referenced in template HTML via `data-var` or cost-share bucket pointers. Values resolve at runtime via `sourceSystemInfo`.
* **Versioning & Audit Trigger:** Saves insert snapshots into `keyref_versions` and append to `audit_keyref`.

---

### `template_maps`
* **File:** `schema_template_maps.json` / `template_maps.sample.json`
* **Main Fields:**
  * **Map Identity:** `mapName`, `key`
  * **Relationships:** `templateRefs[]` (ordered array of template keys defining a ditamap)
  * **Routing Config:** `templateConfig` (`planId`, `lob`, `planTypes`, `configInfo`, `searchConfigMap[]` lookup rules)
  * **Metadata:** `updatedAt` (`date`), `updatedBy`, `version` (`int`)
* **Population Strategy:** Assembled during document generation (DBD). `templateRefs` controls document ordering, while `templateConfig` maps plan routing criteria.
* **Versioning & Audit Trigger:** Membership changes log to `audit_template` via `AddedToDitamap` and `RemovedFromDitamap` actions.

---

### `template_types`
* **File:** `schema_template_types.json` / `template_types.sample.json`
* **Main Fields:**
  * **Identity:** `typeName`, `documentType` (`"DBD"` | `"SBC"`)
  * **Structure:** `sections[]` (`sectionKey`, `sectionName`, `sectionType`, `sequenceNo`, `required`, `fields[]`)
  * **Metadata:** `updatedAt` (`date`), `updatedBy`, `version` (`int`)
* **Population Strategy:** Admin-configured structural schema defining required document sections and field shapes for UI editor skeletons.
* **Versioning & Audit Trigger:** Overwritten directly on edit; does not utilize version-history or audit collections.

---

## 2. Version History Collections

All version history collections are **insert-only** and store historical snapshots created on document updates.

| Collection | Target Document Reference | Storage Model | Snapshot Field |
| :--- | :--- | :--- | :--- |
| **`template_versions`** | `templateId` (`string`) | GridFS-style `binData` payload | `docBytes` (`binData`) |
| **`rules_version`** | `ruleId` (`objectId`) | Full prior inline document | `previousDoc` (`object`) |
| **`keyref_versions`** | `keyrefId` (`objectId`) | Full prior inline document | `previousDoc` (`object`) |

*Note: `template_versions` utilizes raw binary storage (`binData`) to bypass the 16MB BSON document size limit and avoid heavy whole-document rewrites on large templates.*

---

## 3. Activity / Audit Collections

Activity logs record discrete lifecycle events. All three collections share an identical minimal schema structure:

- **`audit_template`** (`templateId`, `action`, `notes`, `user`, `lastUpdateTime`)
- **`audit_rule`** (`ruleId`, `action`, `notes`, `user`, `lastUpdateTime`)
- **`audit_keyref`** (`keyrefId`, `action`, `notes`, `user`, `lastUpdateTime`)

### Characteristics
* **Insert-Only:** Logged asynchronously via non-blocking write operations during save/update pipelines.
* **Event Scoping:** Logs human-readable activity descriptions (`notes`) across lifecycle operations (`Created`, `Updated`, `Uploaded`, `Checkout`, `AddedToDitamap`, `Referenced`, etc.).
* **Data Separation:** Detailed before/after field diffs are omitted from audit entries—content state is tracked inside the respective `*_version(s)` collections.
