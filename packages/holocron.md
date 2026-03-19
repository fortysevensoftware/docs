## Contents

- [Introduction](#introduction)
- [When To Use This Package](#when-to-use-this-package)
- [Installation](#installation)
- [Configuration](#configuration)
- [Manual Recording](#manual-recording)
  - [Recording an Event](#recording-an-event)
  - [Recording Field Changes](#recording-field-changes)
  - [Attaching Metadata](#attaching-metadata)
  - [Categorising Events](#categorising-events)
- [Model Integration](#model-integration)
  - [Preparing Your Model](#preparing-your-model)
  - [Automatic Model Events](#automatic-model-events)
  - [Controlling Tracked Fields](#controlling-tracked-fields)
  - [Controlling Tracked Events](#controlling-tracked-events)
  - [Recording History from the Model](#recording-history-from-the-model)
  - [Querying History from the Model](#querying-history-from-the-model)
- [Querying](#querying)
  - [Filtering by Subject](#filtering-by-subject)
  - [Filtering by Actor](#filtering-by-actor)
  - [Filtering by Event and Category](#filtering-by-event-and-category)
- [The HolocronEntry Model](#the-holocronentry-model)
- [Testing](#testing)
- [Requirements](#requirements)

---

<a name="introduction"></a>
## Introduction

Holocron is a Laravel package for recording and querying historical events across your application. It combines audit-style field diffing with human-readable event logging, so you can track both low-level attribute changes and meaningful business events in a single, consistent history record.

Rather than choosing between a rigid audit log and a custom activity feed, Holocron gives you both in one entry — the _what changed_ and the _what happened_ side by side.

```php
// Record a manual event with a human-readable message
Holocron::record('status_changed')
    ->on($order)
    ->by(auth()->user())
    ->message('Order marked as paid')
    ->withMeta(['from' => 'pending', 'to' => 'paid'])
    ->save();

// Automatically track field changes via a model trait
class Order extends Model
{
    use HasHistory;

    protected array $holocronTrack = ['status', 'total'];
}

// Query the timeline
$order->holocronTimeline();
```

---

<a name="when-to-use-this-package"></a>
## When To Use This Package

Use Holocron when your application needs a persistent, queryable record of what happened, who caused it, and what changed.

**Common use cases:**

- Order and payment lifecycle tracking
- Content moderation and editorial history
- Support ticket and case notes
- User account activity logs
- Audit trails for compliance or debugging

**When not to use this package:**

Holocron is not an error-tracking or performance monitoring tool. For exceptions and stack traces, a tool like Flare or Sentry is more appropriate.

If you only need lightweight model diffing without human-readable events or actor tracking, a simpler audit package may be sufficient. Holocron is designed for applications where both the audit trail and the activity timeline matter.

---

<a name="installation"></a>
## Installation

Install the package via Composer:

```bash
composer require egough/holocron
```

### Publishing Assets

Publish the configuration file:

```bash
php artisan vendor:publish --tag=holocron-config
```

Publish the migrations:

```bash
php artisan vendor:publish --tag=holocron-migrations
```

Run your migrations:

```bash
php artisan migrate
```

This creates the `holocron_entries` table (or whichever table name you configure) used to store all history records.

---

<a name="configuration"></a>
## Configuration

After publishing, the configuration file will be available at `config/holocron.php`.

```php
return [

    'table_name' => 'holocron_entries',

    'auto_record' => [
        'created'  => true,
        'updated'  => true,
        'deleted'  => true,
        'restored' => true,
    ],

    'exclude' => [
        'created_at',
        'updated_at',
    ],

    'actor_resolver' => static fn () => auth()->user(),

];
```

### Options

**`table_name`**

The database table used to store history entries. Defaults to `holocron_entries`. Change this before running migrations if you need a different table name.

**`auto_record`**

Controls which Eloquent model events are automatically recorded when using the `HasHistory` trait. Each event can be toggled independently. These defaults apply globally and can be overridden per model.

**`exclude`**

A list of attribute names that should be excluded from automatic field diffing. `created_at` and `updated_at` are excluded by default, as they change on almost every write and are rarely meaningful in a history record.

**`actor_resolver`**

A callable used to resolve the actor when one is not provided explicitly. Defaults to `auth()->user()`. You can replace this with any closure that returns a model or null — useful for applications using custom auth guards or tenant-scoped user resolution.

---

<a name="manual-recording"></a>
## Manual Recording

Use the `Holocron` facade to record events anywhere in your application — controllers, jobs, listeners, services, or commands.

<a name="recording-an-event"></a>
### Recording an Event

```php
use Egough\Holocron\Facades\Holocron;

Holocron::record('status_changed')
    ->on($order)
    ->by(auth()->user())
    ->message('Order marked as paid')
    ->save();
```

Each call to `record()` starts a fluent builder. The only required call is `save()` — all other methods are optional and additive.

| Method | Description |
| --- | --- |
| `->on($model)` | The subject model this history entry belongs to (polymorphic). |
| `->by($actor)` | The actor who caused the event. Falls back to `actor_resolver` if omitted. |
| `->message(string)` | A human-readable description of what happened. |
| `->category(string)` | An optional category for grouping and filtering entries. |
| `->withMeta(array)` | Arbitrary key-value metadata stored as JSON. |
| `->withChanges(array)` | Explicit attribute diffs in `['field' => ['old' => ..., 'new' => ...]]` format. |
| `->save()` | Persists the entry to the database. |

<a name="recording-field-changes"></a>
### Recording Field Changes

You can record explicit attribute diffs alongside or instead of a message. This is useful when you want to capture the before and after state of specific fields without relying on automatic model tracking.

```php
Holocron::record('updated')
    ->on($post)
    ->by($user)
    ->withChanges([
        'title' => [
            'old' => 'Old Title',
            'new' => 'New Title',
        ],
        'status' => [
            'old' => 'draft',
            'new' => 'published',
        ],
    ])
    ->message('Post published with updated title')
    ->save();
```

Changes are stored as JSON and can be retrieved and rendered for an audit-style diff view.

<a name="attaching-metadata"></a>
### Attaching Metadata

Metadata allows you to store any additional context that does not fit into the standard fields. It is stored as a JSON column and is freely structured.

```php
Holocron::record('invoice_resent')
    ->on($invoice)
    ->by($user)
    ->withMeta([
        'recipient'    => 'billing@example.com',
        'attempt'      => 3,
        'triggered_by' => 'admin_panel',
    ])
    ->save();
```

Metadata is separate from field changes. You may use both `withMeta()` and `withChanges()` on the same entry.

<a name="categorising-events"></a>
### Categorising Events

Categories allow you to group related events for filtering and display purposes.

```php
Holocron::record('note_added')
    ->on($ticket)
    ->by(auth()->user())
    ->category('communication')
    ->message('Support note added by agent')
    ->save();

Holocron::record('status_changed')
    ->on($ticket)
    ->by(auth()->user())
    ->category('lifecycle')
    ->message('Ticket escalated to tier 2')
    ->save();
```

Categories are stored as plain strings. There is no predefined list — you define the taxonomy that makes sense for your application.

---

<a name="model-integration"></a>
## Model Integration

The `HasHistory` trait can be added to any Eloquent model to enable both automatic event recording and convenient history querying directly from the model.

<a name="preparing-your-model"></a>
### Preparing Your Model

```php
use Egough\Holocron\Concerns\HasHistory;
use Illuminate\Database\Eloquent\Model;

class Order extends Model
{
    use HasHistory;
}
```

You can add the trait to as many models as you need. Each model's history is stored polymorphically, so all entries live in the same table regardless of which model they belong to.

```php
class Post extends Model
{
    use HasHistory;
}

class Project extends Model
{
    use HasHistory;
}
```

<a name="automatic-model-events"></a>
### Automatic Model Events

Once the trait is applied, Holocron will automatically record entries for the following Eloquent events:

| Event | Recorded by default |
| --- | --- |
| `created` | Yes |
| `updated` | Yes |
| `deleted` | Yes |
| `restored` | Yes |

For `updated` events, Holocron compares the model's dirty attributes against their original values and stores the diff. Only changed attributes are recorded.

```json
{
    "status": {
        "old": "draft",
        "new": "active"
    },
    "published_at": {
        "old": null,
        "new": "2025-06-01 09:00:00"
    }
}
```

Attributes listed in `config/holocron.php` under `exclude` are ignored during diffing, as are attributes not present in the model's `$holocronTrack` list if one is defined.

<a name="controlling-tracked-fields"></a>
### Controlling Tracked Fields

By default, all changed attributes are diffed on update (minus the global exclusions). You can narrow this to a specific list of fields per model using the `$holocronTrack` property.

```php
class Order extends Model
{
    use HasHistory;

    protected array $holocronTrack = ['status', 'total', 'notes'];
}
```

When `$holocronTrack` is set, only changes to those fields will be included in the diff. Changes to other attributes will be silently ignored.

<a name="controlling-tracked-events"></a>
### Controlling Tracked Events

You can override which Eloquent events are recorded for a specific model using `$holocronEvents`. This takes precedence over the global `auto_record` configuration for that model.

```php
class Order extends Model
{
    use HasHistory;

    protected array $holocronEvents = ['created', 'deleted'];
}
```

In this example, `updated` and `restored` events will not be automatically recorded for `Order`, even if they are enabled globally in the config.

<a name="recording-history-from-the-model"></a>
### Recording History from the Model

You can record history entries directly from a model instance, which is convenient when the subject is implicit:

```php
$project->recordHistory(
    event: 'archived',
    message: 'Project archived by admin',
    actor: auth()->user(),
);
```

You may also pass `meta` and `changes` as named arguments:

```php
$project->recordHistory(
    event: 'settings_updated',
    message: 'Project visibility changed to private',
    actor: $user,
    meta: ['visibility' => 'private'],
);
```

<a name="querying-history-from-the-model"></a>
### Querying History from the Model

The `HasHistory` trait provides several ways to query a model's history:

```php
// All history entries as a collection
$order->history;

// Query builder for full Eloquent control
$order->history()->latest('recorded_at')->get();

// Convenience scope returning the most recent entries first
$order->latestHistory()->get();

// Timeline-ready output
$order->holocronTimeline();
```

The `holocronTimeline()` method returns entries in a format suitable for rendering a human-readable activity feed, with messages and metadata pre-formatted for display.

---

<a name="querying"></a>
## Querying

The `HolocronEntry` model provides a set of query scopes for retrieving history records across your entire application, independent of a specific model instance.

<a name="filtering-by-subject"></a>
### Filtering by Subject

```php
use Egough\Holocron\Models\HolocronEntry;

HolocronEntry::query()
    ->forSubject($order)
    ->latest('recorded_at')
    ->get();
```

`forSubject()` accepts any Eloquent model and scopes results to entries with a matching polymorphic relationship.

<a name="filtering-by-actor"></a>
### Filtering by Actor

```php
HolocronEntry::query()
    ->causedBy($user)
    ->latest('recorded_at')
    ->get();
```

`causedBy()` accepts any Eloquent model and scopes results to entries where that model is the recorded actor.

<a name="filtering-by-event-and-category"></a>
### Filtering by Event and Category

```php
HolocronEntry::query()
    ->forSubject($order)
    ->event('status_changed')
    ->get();

HolocronEntry::query()
    ->causedBy($user)
    ->category('communication')
    ->get();
```

Scopes can be chained freely:

```php
HolocronEntry::query()
    ->forSubject($ticket)
    ->causedBy($agent)
    ->category('communication')
    ->latest('recorded_at')
    ->get();
```

---

<a name="the-holocronentry-model"></a>
## The HolocronEntry Model

Each history entry is stored as a `HolocronEntry` Eloquent model. The following attributes are available:

| Attribute | Type | Description |
| --- | --- | --- |
| `id` | int | Primary key. |
| `subject_type` | string | The class name of the subject model. |
| `subject_id` | int | The ID of the subject model. |
| `actor_type` | string\|null | The class name of the actor model. |
| `actor_id` | int\|null | The ID of the actor model. |
| `event` | string | The event name (e.g. `status_changed`, `created`). |
| `message` | string\|null | A human-readable description of the event. |
| `category` | string\|null | An optional category for grouping. |
| `metadata` | array\|null | Arbitrary JSON metadata, cast to array. |
| `changes` | array\|null | Field-level diffs, cast to array. Each entry follows `['old' => ..., 'new' => ...]`. |
| `recorded_at` | Carbon | The timestamp the event was recorded. |

---

<a name="testing"></a>
## Testing

The package ships with a full test suite backed by an in-memory SQLite database, so no external database service is required.

<a name="running-the-tests"></a>
### Running the Tests

Install development dependencies:

```bash
composer install
```

Run the test suite:

```bash
composer test
```

Or run PHPUnit directly:

```bash
vendor/bin/phpunit
```

<a name="testing-your-own-application"></a>
### Testing Your Own Application

When writing tests for code that records or queries history, you can interact with the `Holocron` facade or `HolocronEntry` model directly to assert state.

**Asserting that an event was recorded:**

```php
use Egough\Holocron\Models\HolocronEntry;

it('records a status_changed event when an order is paid', function () {
    $order = Order::factory()->create(['status' => 'pending']);

    $order->markAsPaid();

    expect(
        HolocronEntry::query()
            ->forSubject($order)
            ->event('status_changed')
            ->exists()
    )->toBeTrue();
});
```

**Asserting the recorded actor:**

```php
it('records the correct actor when an admin marks an order as paid', function () {
    $admin = User::factory()->admin()->create();
    $order = Order::factory()->create();

    $this->actingAs($admin);

    $order->markAsPaid();

    $entry = HolocronEntry::query()
        ->forSubject($order)
        ->event('status_changed')
        ->first();

    expect($entry->actor_id)->toBe($admin->id);
    expect($entry->message)->toBe('Order marked as paid');
});
```

**Asserting field-level diffs:**

```php
it('records the old and new status when an order is updated', function () {
    $order = Order::factory()->create(['status' => 'pending']);

    $order->update(['status' => 'paid']);

    $entry = $order->history()
        ->event('updated')
        ->latest('recorded_at')
        ->first();

    expect($entry->changes['status']['old'])->toBe('pending');
    expect($entry->changes['status']['new'])->toBe('paid');
});
```

Because entries are written to the database, they will be reset between tests as long as you use the `RefreshDatabase` trait in your test cases.

---

<a name="requirements"></a>
## Requirements

| Requirement | Version |
| --- | --- |
| PHP | 8.2+ |
| Laravel | 11, 12, or 13 |
