# Changelog

All notable changes to `asignua/filament-duplicates` are documented here.

## 1.0.2 - 2026-10-12

- Licence terms (`LICENSE.md`): the commercial licence that comes with an Anystack licence key; prices in the README.
- Docs: `scripts/sync-docs.sh` points the screenshots at the public docs repository.

## 1.0.1 - 2026-10-12

- Fix: the merge dialog shows array/JSON-cast fields as comma-separated values (`vip, fair`) instead of raw JSON.
- Fix: tab labels of models with a Filament resource start with a capital letter, like those of models without one.
- Fix: the create-form duplicate warning tells same-titled matches apart with `#id`.
- Docs: screenshots, cover and a workbench demo.

## 1.0.0 - 2026-10-09

- Duplicates page: record titles that clash inside a group (and in a dismissed pair) get their key appended, as in the
  merge dialog («Acme #12»).

- Merge dialog: combinable fields (Concat / Union, and long-text columns by default — config `text_strategy`) get checkboxes
  of the values to keep together, all ticked; many-to-many relations get a «Related records» section with what comes from
  the others (untick to leave it behind; the survivor's own items locked) and a «+ …» line in the preview.
  `MergeService::merge()` takes a list of keys per combinable field and `relations:` (relation => related keys).
  Option keys no longer follow the survivor, so switching it keeps the ticks; the record action fills the defaults when
  the «others» change.

- Merge dialog, per-field choices: one option per distinct value with the records that hold it (not one per record),
  «Automatic» states its result (kept value / borrowed from a named record / combined), blank values read «(empty)»,
  records with the same title get their key appended (also in the survivor select). Foreign keys of typed `BelongsTo`
  relations show the target's title (dialog, preview, Duplicates page); `ModelConfig::displayUsing()` for custom display;
  enums implementing Filament `HasLabel` show their label.

- Styling: the page, the merge preview and the "refuse duplicates" modal no longer rely on Tailwind utilities (a host theme
  does not scan plugin views, so spacing collapsed and attributes ran together). A hand-written `filament-duplicates.css`
  (`.fi-dup-*`, layer-pinned, dark mode) is registered as a plugin asset; matched values are highlighted, attribute pairs separated.
- The bulk merge button is visible when the policy allows `mergeAny` → `deleteAny` → `updateAny` (the first one defined),
  and for policies defining none of them; per-record `update`/`delete` checks still run at execution.

- Declarative duplicate rules per model (`Rule`, `ModelConfig`, `DeduplicatesRecords`): attributes + normalisers (`text`, `digits`,
  `alnum`, `phone`, `email`, closures), custom keys, multi-value cells, bounded fuzzy rules.
- Fingerprint index (publishable migration, configurable table names), kept in step by model events; `duplicates:index` command.
- Duplicates page: a tab per model, group counts, rule filter, paging, merge, "Not duplicates" (persisted per pair).
- Duplicates page, group cards: human attribute labels, the matching fields stated in the header, matched values highlighted, a `Rule::highlight()` names the shown/matched attributes of a custom-key rule.
  checkbox per record with a partial merge ("Merge (N)", server-side group membership check), per-record "Not a duplicate".
- Merge: record action, bulk action, page action; survivor and per-field value choice with preview; HasOne/HasMany/MorphOne/MorphMany
  and BelongsToMany/MorphToMany relations move (pivots deduplicated); one transaction; `BeforeMerge` / `MergeCompleting` (in-transaction) / `RecordsMerged` events.
- `DuplicatesPlugin::slug()` for the review page; a non-relation name in `ModelConfig::relations()` fails early with a clear message.
- Form-level `DuplicateWarning`, `RefusesDuplicates` create-page trait, `NotDuplicate` validation rule.
- `DeleteWithReassignAction`: reference counts, move to another record or delete together.
- Authorization of every involved record through the model policies, on freshly locked rows.
- Translations: English and Ukrainian (other locales fall back to English strings for now).
