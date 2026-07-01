# Stage 2.5 — BRD/HLD → Domain Dictionary (concrete decision content)

**Why this stage exists.** The HLD is architecture and the FSD is functional design; both correctly defer
product specifics as "typed inputs / configuration". But `/implement-spec` is **forbidden to guess**
("every decision comes from the spec"). Without a home for the *actual values* — field types, DMN rows,
message field maps, accounting codes, checklists, SLA matrices — the implementer stalls or hallucinates at
the exact business core. **This stage is that home.** It mines the BRD (and only the BRD + cited public
standards) for concrete decision content and writes it as machine-readable data files under
`<domain_dir>` (default `docs/domain/`) that FSDs and specs then reference by path.

Runs after the FSD index/plan (stage 1) and **before** FSD authoring (stage 2), so authors reference it.

## Inputs (CONTEXT block appended below)
- `config` — apply `domain.*` decisions (do not re-ask). Skip this stage entirely if `domain.enabled: false`.
- The BRD (`<brd_glob>`) — **read it fully, including every appendix, table, field list, code list, and
  requirement-id catalogue.** This is the ONLY source of bank-specific values.
- The HLD (`<hld_dir>`) and the FSD catalog/`spec_plan` (stage-1 output) — tell you which products,
  sub-processes, DMN decisions, message types, and integrations need concrete data.
- `<rules_dir>` + `CLAUDE.md` — the `docs/domain/*` shapes must be expressible as `@Validate` Zod schemas,
  DMN tables, and adapter mappings the platform actually supports.

## The one rule that governs everything here
**Extract, cite, or flag — never invent.** Every value you write MUST be one of:
1. **BRD-sourced** — copied/derived from the BRD, tagged with its requirement id (`source: "BRD §x.y / HLR-N"`).
2. **Standard-sourced** — from a cited public standard (e.g. SWIFT MT700 field structure per `SR2021`,
   Incoterms 2020, ISO 4217), tagged `source: "SWIFT SR2021 MT700"`. The message *structure* is a
   standard; the *mapping from a domain field to a tag* is design and is allowed.
3. **TODO(domain-input)** — the value is bank-specific, the BRD does not supply it, and no public standard
   fixes it (GL/accounting codes, correspondent BICs, referral thresholds, SLA hours, the duplicate window
   `n`, charge/commission rates). Write a `TODO(domain-input)` entry naming EXACTLY what is missing and who
   must supply it — **do not fill it with a plausible guess.** A visible TODO is the correct output; an
   invented GL code is a defect.

## Output — write under `<domain_dir>`
Produce machine-readable YAML (or JSON) data files, one concern per file, plus an index. Cover every
concrete-content need the HLD/FSD catalog implies. Typical set (adapt to the actual domain):

| File (example) | Holds |
|---|---|
| `<domain_dir>/products/<product>.details.yml` | The product's `Transaction.details` field dictionary: `name`, `type`, `length/format`, `required`, `enum values`, `source`. This is the Zod schema, as data. |
| `<domain_dir>/dmn/<decision>.dmn.yml` | A real decision table: `inputs[]` (columns), `rows[]` (conditions → outputs, priority), `hitPolicy`. Every threshold/band a value or a TODO. |
| `<domain_dir>/messages/<MTxxx>.map.yml` | Message field map: `tag` → `source domain field` / fixed value, per cited standard; bank-specific tags (charges, correspondents) as TODO. |
| `<domain_dir>/accounting/<product>.gl.yml` | Posting steps → GL/accounting code ids (almost always TODO unless the BRD lists them). |
| `<domain_dir>/checklists/<product>.checklist.yml` | Document checklist items, mandatory flags, `source`. |
| `<domain_dir>/sla/<process>.sla.yml` | SLA/TAT + escalation matrix rows (hours, tier, action). |
| `<domain_dir>/README.md` | Index of the above **and the consolidated `TODO(domain-input)` register** (one table: item · file · what's missing · owner-role). |

Rules for the data files:
- Field/enum names must match what the FSD/spec will reference (keep the seam consistent).
- Types must be Zod-expressible (`string`, `number`, `enum[...]`, `iso-date`, `bic`, `currency`, `boolean`,
  nested object/array). No prose-only "TBD" — either a typed value or a `TODO(domain-input)` line.
- Cite `source:` on every leaf value. No uncited banking value may appear.

## Output contract
Return JSON only:
```json
{ "domain_dir": "docs/domain", "files": ["docs/domain/products/import-lc.details.yml", ...],
  "todo_count": <int>, "todos": [ { "item": "Import-LC charge GL code", "file": "...gl.yml", "owner": "TFO finance" } ] }
```
Do not paste file contents. A non-zero `todo_count` is expected and correct — it is the surfaced,
assignable gap, not a failure.
