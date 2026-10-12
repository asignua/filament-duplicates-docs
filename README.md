# Filament Duplicates

> **Documentation only.** Filament Duplicates is a commercial plugin: this public repository holds its
> documentation, changelog and licence terms. The package itself is installed from the private Composer
> repository you get with a licence — [buy one on Anystack](https://checkout.anystack.sh/filament-duplicates).
> Questions: support@asign.in.ua.

<img class="filament-hidden" src="https://raw.githubusercontent.com/asignua/filament-duplicates-docs/main/art/cover.jpg" alt="Filament Duplicates">

Find, review and merge duplicate records in any [Filament](https://filamentphp.com) 5 resource — and delete a record
*without losing what points at it*.

Every CRM, shop and member list collects the same person twice. This plugin gives you the whole loop:

- **Detect** — declare per model which attributes make two records "the same" (phone, e-mail, name + city, a tax id, any
  closure). Exact matches come from a fingerprint index and one grouped query, so it works on big tables.
- **Review** — a *Duplicates* page with a tab per model, group counts, a rule filter and "Not duplicates" (remembered).
- **Merge** — pick the survivor, pick field by field which value wins (with a preview), and everything that pointed at the
  others — `hasMany`, `morphMany`, pivots — moves to the survivor in one transaction.
- **Prevent** — a live "a record like this already exists" warning in forms, a "create anyway?" step on create pages and a
  blocking validation rule.
- **Delete safely** — *Delete with reassign* shows how many rows reference a record and moves them to another record first.
- **Authorized** — every operation asks your policies about every record involved; a forged Livewire call gets nowhere.

## Screenshots

![The Duplicates page: groups of matching records with the matching values highlighted](https://raw.githubusercontent.com/asignua/filament-duplicates-docs/main/art/duplicates.jpg)

Pick the survivor and, field by field, which value wins; combinable fields and related records are kept together:

![The merge dialog with field choices, combinable values and a preview](https://raw.githubusercontent.com/asignua/filament-duplicates-docs/main/art/merge.jpg)

Delete a record without losing what points at it:

![Delete with reassign: move the related records to another record](https://raw.githubusercontent.com/asignua/filament-duplicates-docs/main/art/delete-reassign.jpg)

A live warning while the form is being filled in:

![A "Similar records already exist" warning on a create page](https://raw.githubusercontent.com/asignua/filament-duplicates-docs/main/art/create-warning.jpg)

Dark mode:

![The Duplicates page in dark mode](https://raw.githubusercontent.com/asignua/filament-duplicates-docs/main/art/duplicates-dark.jpg)

## Purchase

Filament Duplicates is a commercial plugin. Both tiers include one year of updates; after that the
plugin keeps working on the last release you received, and you can renew for further updates.

| Tier | Projects | Activations | Price | Renewal |
| --- | --- | --- | --- | --- |
| Single Project | 1 | up to 3 (production, staging, local) | €69 | €35 / year |
| Unlimited | any number, SaaS included | unlimited | €169 | €85 / year |

Refunds are available within 14 days of purchase.

**[Buy a licence on Anystack](https://checkout.anystack.sh/filament-duplicates)** — after the purchase you receive a licence key and
access to the private Composer repository. See [LICENSE.md](https://github.com/asignua/filament-duplicates-docs/blob/main/LICENSE.md) for the licence terms.

## Requirements

- PHP 8.3+
- Laravel 12 or 13
- Filament 5

## Installation

1. **Add the private repository** (the same for every customer):

   ```bash
   composer config repositories.filament-duplicates composer https://filament-duplicates.composer.sh
   ```

2. **Add your credentials.** The username is the e-mail address you bought the licence with; the
   password is your licence key. A **Single Project** licence is bound to a fingerprint, so the password
   is the key followed by a colon and the fingerprint you activated — usually the production domain:

   ```bash
   # Single Project
   composer config --auth http-basic.filament-duplicates.composer.sh you@example.com "LICENCE-KEY:shop.example.com"

   # Unlimited
   composer config --auth http-basic.filament-duplicates.composer.sh you@example.com "LICENCE-KEY"
   ```

   This writes `auth.json` next to your `composer.json` — keep it out of git (add `auth.json` to `.gitignore`).
   On CI and servers pass the same JSON through the `COMPOSER_AUTH` environment variable instead.
   A Single Project licence allows three activations (for example production, staging and local); manage
   them in your Anystack account.

3. **Require the package:**

   ```bash
   composer require asignua/filament-duplicates
   ```

4. **Register the plugin** on your panel:

   ```php
   use Asignua\FilamentDuplicates\DuplicatesPlugin;

   $panel->plugin(DuplicatesPlugin::make());
   ```

## Usage

### 1. Run the migration

```bash
php artisan vendor:publish --tag=filament-duplicates-migrations   # optional, to edit it
php artisan migrate
```

The migration creates two tables: `duplicate_fingerprints` (the index) and `duplicate_dismissals` ("not duplicates").
Rename them in `config/filament-duplicates.php` **before** migrating.

### 2. Declare the rules

On the model (it then needs nothing else):

```php
use Asignua\FilamentDuplicates\Concerns\HasDuplicateRules;
use Asignua\FilamentDuplicates\Contracts\DeduplicatesRecords;
use Asignua\FilamentDuplicates\Enums\MergeStrategy;
use Asignua\FilamentDuplicates\Support\ModelConfig;
use Asignua\FilamentDuplicates\Support\Rule;

class Contact extends Model implements DeduplicatesRecords
{
    use HasDuplicateRules;

    public static function duplicateConfig(): ModelConfig
    {
        return ModelConfig::make()
            ->rule(
                Rule::make('phone')->on('phone', 'phone')->multiple(),          // "050 111 22 33; 067 …" are separate keys
                Rule::make('email')->on('email', 'email'),
                Rule::make('name_city')->attributes(['last_name', 'first_name', 'city']),
                Rule::make('similar_name')->attributes(['last_name', 'first_name'])->fuzzy(threshold: 88),
            )
            ->relations(['deals', 'tags', 'comments'])                           // what moves to the survivor
            ->strategies(['notes' => MergeStrategy::Concat, 'labels' => MergeStrategy::Union])
            ->summary(['email', 'phone', 'city']);                               // shown on the Duplicates page
    }
}
```

In the merge dialog every differing field lists its distinct values once each («Ivan Petrenko — in 3 records»), and
«Automatic» says what the field will get (the kept record's value or one borrowed from a named record). A combinable field
(`Concat` / `Union`, and every long-text column — TEXT, MEDIUMTEXT… — without its own strategy, see `text_strategy` in the
config) gets checkboxes instead: all values ticked, untick what should not stay. Many-to-many relations (BelongsToMany /
MorphToMany, e.g. tags) list what comes from the other records, ticked; untick what should not move (the kept record's own
items are shown locked). From code: `merge($survivor, $losers, ['notes' => [$a->id, $c->id]], relations: ['tags' => [3, 5]])`.
Records with the same title are told apart by key («Acme #12»). A foreign key of a typed `BelongsTo` relation is shown as
the title of its target instead of the number; anything else can be printed your way:

```php
->displayUsing([
    'country_code' => fn (?string $code): ?string => $code === null ? null : Countries::name($code), // null = default display
])
```

or on the plugin, for a model you do not own:

```php
$panel->plugin(
    DuplicatesPlugin::make()
        ->model(Company::class, ModelConfig::make()->rule(Rule::make('tax_id')->on('tax_id', 'alnum')))
        ->models([Contact::class]),
);
```

Then index what already exists (new and changed records are indexed automatically):

```bash
php artisan duplicates:index
```

#### Normalisers

A rule compares *normalised* values. Built-in, by name or as a factory: `text` (trim, collapse spaces, case-insensitive),
`digits`, `alnum`, `phone` and `email`.

```php
Normalizers::phone('380', 9)   // "0501234567", "+38 (050) 123-45-67", "00380501234567" => "+380501234567"
Normalizers::email(gmailDots: true, plusTags: true)   // "John.Doe+x@gmail.com" => "johndoe@gmail.com"
Rule::make('doc')->on('passport', fn ($v) => preg_replace('/\W/', '', strtoupper((string) $v)) ?: null)
Rule::make('any')->keysUsing(fn (Model $m) => [$m->tax_id, $m->vat_id])   // several keys, one fingerprint each
```

Set the defaults of `phone` once in `config/filament-duplicates.php` (`phone.country_code`, `phone.national_length`).

**National vs. international phones.** `"063 555 66 77"` and `"+380635556677"` match **only if `phone.country_code` is set**
(`'380'`): then a trunk `0`, a `00` prefix, a bare `380…` and `+38 (063)…` all normalise to the same `+380635556677`. With the
code left `null` nothing is guessed — only numbers written in the same form match. Fingerprints already stored keep the old
keys, so after changing the config run `php artisan duplicates:index` to rebuild them.

### 3. The Duplicates page

Registered with the plugin. Each tab is a model you may `viewAny`; each group is the records that share a key.

- **Why they are grouped.** The card header says `Matched on: <rule label> — «<key>»` and lists the matching fields. Every record
  shows those fields first (with the human labels from `ModelConfig::attributeLabels()`, never raw column names), then the
  `summary()` attributes. The values that make the record match (by the rule's normalised key — so `Радість` and `Radist` both match a
  transliteration rule) are shown as a highlighted badge; values that differ are plain text.
  A rule with custom keys (`keysUsing()`) has no attribute list: name what to show and highlight with `->highlight(['phone'])`.
- **Merge only some of them.** Every record has a checkbox (all ticked by default). **Merge (N)** merges the ticked records only
  and is disabled, with a hint, below two. The other records of the group are not touched.
- **Not a duplicate.** The per-record link marks that record as not a duplicate of the others in the group (stored as dismissed
  pairs with each member), so the group shrinks; **Not duplicates** on the card dismisses the whole group.
- The page never trusts the browser: the action arguments carry the group (`rule` + `hash`) and the ticked keys, and the server
  checks that every key belongs to that group and that the user may act on each record before it writes.

### 4. Merge actions

```php
use Asignua\FilamentDuplicates\Filament\Actions\MergeAction;
use Asignua\FilamentDuplicates\Filament\Actions\MergeBulkAction;

$table
    ->recordActions([MergeAction::make()])                              // "merge others into this one"
    ->toolbarActions([BulkActionGroup::make([MergeBulkAction::make()])]) // select 2+ rows
```

The dialog lists only the fields where the records differ. *Automatic* keeps the survivor's value and fills it from the
others when it is blank (or follows the field's strategy: `fill_blank`, `survivor`, `concat`, `union`). A live preview shows
what will change. On confirm, in one transaction: relations move, losers are deleted (soft-deleted when the model uses
`SoftDeletes`), the survivor is updated, the index and the dismissals are cleaned.

Models with mass assignment switched off or a write layer of their own: `->persistUsing(fn (Model $survivor, array $changed) => $repository->update($survivor, $changed))`.

From code: `app(MergeService::class)->merge($survivor, [$loser], choices: ['city' => (string) $loser->getKey()])`.

**Relations.** List them with `->relations([...])`. Supported: `HasOne`, `HasMany`, `MorphOne`, `MorphMany`, `BelongsToMany`,
`MorphToMany`. `HasOne`/`MorphOne` move only if the survivor has none. Pivot rows the survivor already has are kept as
they are (their pivot columns win); the rest move with their pivot columns. Soft-deleted children move too.
`->discoverRelations()` adds every relation with a supported *return type* — opt-in, check
`app(DuplicateRegistry::class)->relationNames(Contact::class)` first. Relations that come from vendor traits (roles, permissions,
media, tokens…) are never discovered, so a merge cannot hand the loser's roles to the survivor; `discoverRelations(except: [...])`
leaves out more. Pivot rows move with an `UPDATE` of the foreign key, so every pivot column travels with its row.

**Two HasOne children.** When the survivor already has a HasOne/MorphOne child, the loser's one is moved anyway by default
(nothing is lost; a unique index refuses the merge instead). `->onChildConflict(ChildConflict::Keep)` leaves it on the loser and
reports it in `MergeResult::$skipped`; `ChildConflict::Refuse` refuses the merge. Pass relation names to set it per relation.

**Hidden attributes** (`$hidden`, or anything outside `$visible`) are neither shown nor merged; `->includeHidden()` merges them
(they are still never printed in the dialog). Labels: `->attributeLabels([...])`, `->relationLabels([...])`.

**Events.** `BeforeMerge` (inside the transaction, before anything is written; throw to veto), `MergeCompleting` (inside the
transaction, after relations moved, losers deleted and the survivor saved, right before commit), `RecordsMerged` (after commit;
`$event->result->loserSnapshots` holds what the losers looked like, `->plan->changes` the field changes, `->moved` the rows
moved per relation).

**Only real relations can be listed.** Every name in `->relations([...])` must be an Eloquent `HasOne`/`HasMany`/`MorphOne`/
`MorphMany`/`BelongsToMany`/`MorphToMany` method. A name that is not one is refused with a message naming the model and the
relation: at registration when the method is missing or declares another return type, and at the latest when the merge dialog
opens or a merge starts (before anything is written) for methods without a declared return type.

**Moving data that is not an Eloquent relation** (audit logs, morph tables you read through a trait, custom storage). Do not
list it in `relations`; move it from a listener. `MergeCompleting` runs inside the merge transaction, so what the listener writes
commits together with the merge, and an exception it throws rolls the whole merge back (relations, deleted losers, survivor
fields). `RecordsMerged` runs after the commit: use it only for side effects that must not be rolled back (notifications, cache).

```php
use Asignua\FilamentDuplicates\Events\MergeCompleting;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Event;

Event::listen(MergeCompleting::class, function (MergeCompleting $event): void {
    if (!$event->result->survivor instanceof Contact) {
        return;
    }

    // activity_log has no Eloquent relation on Contact: re-point its rows by hand
    DB::table('activity_log')
        ->where('subject_type', (new Contact)->getMorphClass())
        ->whereIn('subject_id', collect($event->result->plan->losers)->map->getKey())
        ->update(['subject_id' => $event->result->survivor->getKey()]);
});
```

Use `BeforeMerge` instead when the data must be handled while the losers still exist, or to veto the merge.

### 5. Warn (or refuse) on create / edit

```php
// in the form schema; the watched fields should be ->live(onBlur: true)
DuplicateWarning::make(Contact::class),                               // fields named like the attributes
DuplicateWarning::make(Contact::class, ['email' => 'contact_email']), // or mapped
```

On a create page, ask before saving a lookalike:

```php
class CreateContact extends CreateRecord
{
    use RefusesDuplicates;     // modal with the matches and "Create anyway"; override findDuplicates() to narrow
}
```

A hard block, as a validation rule (only exact rules; the check sees *all* rows, also those the user may not view):

```php
TextInput::make('email')->rules([NotDuplicate::make(Contact::class, 'email')->ignoring($record)])
```

### 6. Delete with reassign

```php
->recordActions([DeleteWithReassignAction::make()->relations(['deals', 'tags'])])   // or the relations of the model's config
```

Unreferenced: the usual confirmation. Referenced: the modal says *"This record is in use: 3 × deals, 1 × tags"* and offers
**move them to another record** or **delete them together with this record** (`->onlyReassign()` removes the second
choice). Deleting children asks the policy for each one.

## Authorization

Every service call checks, on freshly locked rows:

| Operation | Checked |
| --- | --- |
| Merge | `update` on survivor and losers, `delete` on losers |
| Not duplicates | `update` on every record of the pair(s) |
| Delete with reassign | `delete` on the record, `update` on the replacement, `delete` on each child deleted along |
| Duplicates page, matches, suggestions | `view` — records you may not view are left out |

The **bulk "Merge" button** is shown by a class-level check, the first of these the policy (or a Gate definition) defines:
`mergeAny` → `deleteAny` → `updateAny`. A policy that defines none of them (only per-record `update`/`delete`, the common
case) shows the button; the per-record checks in the table above then decide at execution. With no policy at all the
button follows `authorize_when_no_policy`. Define `mergeAny` to control the button on its own.

Abilities are the Gate abilities in `filament-duplicates.abilities`. A model **without a policy** follows Filament's own
behaviour (allowed) — **merge and reassign delete records, so give every registered model a policy or set
`authorize_when_no_policy` to `false`**, which denies instead. Class names from a request are resolved
through the registry of registered models only.

## Styling

The views style themselves with a hand-written, plugin-scoped stylesheet (`resources/dist/filament-duplicates.css`,
`.fi-dup-*` classes, Filament's colour variables, dark mode via `.dark`; no build step). It is registered as a Filament
asset and linked by the page and the dialogs themselves, so it works in a panel whose theme does not scan the plugin
views. Run `php artisan filament:assets` on deploy as for any Filament plugin.

## Configuration

`php artisan vendor:publish --tag=filament-duplicates-config` — tables, `auto_index`, `store_values` (keep only hashes
if you do not want normalised e-mails/phones in a second table), `authorize_when_no_policy`, `abilities`, `max_group_size`,
`per_page`, `default_strategy`, `force_delete_losers`, `phone.*`, `email.*`, `fuzzy_max_candidates`.

Plugin: `->page(false)` (no page, actions only), `->slug('data-duplicates')` (URL of the page, default `duplicates` — use it
when your app already has a page with that slug), `->navigationGroup()`, `->navigationIcon()`, `->navigationSort()`,
`->navigationLabel()`, `->authorizeUsing(fn () => …)`.

## Multi-tenancy

The fingerprint index has no tenant column, so exact groups are found across tenants. Narrow every read with
`ModelConfig::scopeUsing(fn (Builder $query) => $query->where('team_id', Filament::getTenant()?->getKey()))`: groups,
warnings, suggestions, `NotDuplicate`, merge and reassign then only see records inside the scope (a hit outside it does not
count, so nothing leaks). The tab counters are computed on the index and may count groups that the scope later hides;
they are upper bounds. A per-tenant partition column in the index is not implemented.

## Dismissed pairs

"Not duplicates" is undone from the *Dismissed pairs* panel at the bottom of the Duplicates page (needs `update` on both records).

## Gotchas

- **The index is the source of truth for groups.** Records written with `insert()`/`update()` queries (no model events) are
  not indexed — run `php artisan duplicates:index` after bulk imports, or schedule it.
- **Counts are candidate groups before dismissals.** A group whose pairs were all dismissed disappears from the list, but the
  tab counter and the pager still count it, so a page can show fewer groups than the page size.
- **Unique columns.** The survivor is updated *after* the losers are deleted, but a soft-deleted loser still holds its unique
  e-mail, so merging a unique column from a loser into the survivor fails (and rolls back). Use `->forceDeleteLosers()` for such models.
- **Fuzzy rules are a hint.** They compare at most `limit` candidates that share the first characters of one attribute and
  never run over a whole table; `NotDuplicate` ignores them.
- **Children are moved with one `UPDATE`** (no model events, no timestamps). Delete mode loads and deletes children one by one.
- `RefusesDuplicates` defines `mutateFormDataBeforeCreate()`; if the page has its own, call `$this->refuseDuplicates($data)` in it.

## Translations

The interface is translated into English and Ukrainian. German, Spanish, French, Italian, Dutch, Polish, Brazilian Portuguese
and Turkish files exist with the English strings as placeholders (so a missing key never shows raw), until they are translated.
All live under the `filament-duplicates::filament-duplicates` namespace, and a test keeps every language in step with the English keys. Override a
string by publishing the translations (`--tag=filament-duplicates-translations`) and editing the copy in
`lang/vendor/filament-duplicates`.

## AI agents

The package ships [Laravel Boost](https://laravel.com/docs/boost) guidelines (`resources/boost/guidelines/core.blade.php`) so a
coding agent wires it up correctly.

## Testing

```bash
composer install
vendor/bin/phpunit
vendor/bin/phpstan analyse --memory-limit=1G
vendor/bin/pint --test
```

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## Licence

Proprietary. See [LICENSE.md](https://github.com/asignua/filament-duplicates-docs/blob/main/LICENSE.md).
