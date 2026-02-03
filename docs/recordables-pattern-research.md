# The Recordables Pattern: Research & Analysis

## Purpose

This document summarizes research into the "recordables" pattern (Rails delegated types)
as described by 37signals/Basecamp, then analyzes how Fizzy (this repo) implements
related patterns. The goal is to understand the architecture deeply enough to port
its essence to other languages (TypeScript/Convex, Python/SQLModel, etc.).

---

## Part 1: Source Material Summaries

### Source 1: RECORDABLES Video (dev.37signals.com, Dec 19, 2025)

**URL:** https://dev.37signals.com/the-rails-delegated-type-pattern/

**What it is:** A 1-hour video conversation with Jeffrey Hardy (Principal Programmer,
18 years at 37signals) explaining how the recordables pattern powers Basecamp and HEY.

**Key learnings:**

1. **The core idea:** All content in Basecamp (messages, comments, documents, uploads,
   etc.) is modeled using a single `recordings` table that delegates its type to
   specific `recordable` tables via Rails `delegated_type`.

2. **The recordings table is lean:** It stores only metadata — foreign keys, timestamps,
   a creator_id, a color value, and the polymorphic reference (`recordable_type`,
   `recordable_id`). No text columns, no heavy content. This makes it cheap to index,
   query, and paginate.

3. **Recordables are immutable:** When content changes, a new recordable row is created
   rather than updating in place. The recording's pointer is updated to the new version.
   Old versions are preserved and linked through the Event system.

4. **Events track history:** An Event references both a recording and a recordable at
   a point in time. This enables version history, change logs, and "make this the
   current version" features — all by swapping which recordable a recording points to.

5. **Recordings form a tree:** Parent-child relationships exist between recordings
   (not recordables). A message board recording's children are message recordings.
   A message recording's children are comment recordings. Navigation always goes
   through recordings.

6. **Buckets contain recordings:** A Bucket is a container (project, template, ping)
   that controls access. Buckets also use delegated type (bucket → bucketable).
   Access control lives at the bucket level, completely decoupled from content.

7. **Recordables are "dumb":** A message recordable is literally just a title and
   content. No associations to the outside world. No timestamps of its own. No
   knowledge of who created it or where it lives. Some recordables are just an ID
   with no columns at all (placeholders).

8. **Copying is a pointer operation:** Copying a message means creating a new recording
   row pointing to the same message recordable. No content duplication. If a message
   is copied 100 times, there's still one recordable row. This is why demo projects
   across thousands of accounts are storage-efficient.

9. **Generic controllers:** One `RecordingsController` handles trash, archive, restore
   for all types. One copier controller handles all copy operations. One comments
   controller works with any commentable recording. Adding a new recordable type
   requires zero changes to these controllers.

10. **Capabilities are opt-in:** Each recordable type declares what it supports:
    `commentable?`, `subscribable?`, `exportable?`, `copyable?`, `movable?`, etc.
    Default is false/off. Types override to opt in. Generic controllers check these
    before acting.

11. **Caching is uniform:** Cache keys are based on recordings, not recordable types.
    `cache(recording) do...end` works identically for all content types. Russian doll
    caching propagates up the recording tree.

12. **API simplicity:** A single recordings JSON API serves the mobile apps. New
    recordable types don't require mobile app releases — the app handles unknown
    types generically (shows title, icon, timestamps).

13. **In HEY:** The same pattern is used but named differently. `Entry` delegates to
    `entryable` (messages, replies, notes representing emails). In HEY Calendar,
    the recordings table includes `starts_at`/`ends_at` because all calendar items
    need time boundaries.

14. **Drawbacks:**
    - Higher learning curve — new devs open the Message model and find almost nothing
    - Recordables need recording context passed in for operations that require it
    - Upfront cost of making things generic pays off later but feels unusual at first
    - Not suitable for every model — best for uniform content types

15. **10+ years proven:** Basecamp 3's architecture has lasted longer than any previous
    version without needing a rewrite. Basecamp 5 is being built on the same chassis.

### Source 2: Rails API Documentation — ActiveRecord::DelegatedType

**URL:** https://api.rubyonrails.org/classes/ActiveRecord/DelegatedType.html

**What it is:** Official Rails API reference for the `delegated_type` method.

**Key learnings:**

1. **Three approaches to hierarchy mapping:**
   - **Abstract classes / class-table inheritance:** Each subclass has its own table
     with all shared attributes duplicated. Can't paginate across types.
   - **Single-table inheritance (STI):** One mega table with all columns from all
     subtypes. Table bloats with divergent types.
   - **Delegated types:** Superclass has its own table with shared attributes.
     Subclasses have separate tables for specific attributes. Best of both worlds.

2. **The `delegated_type` declaration:**
   ```ruby
   class Entry < ApplicationRecord
     belongs_to :account
     belongs_to :creator
     delegated_type :entryable, types: %w[ Message Comment ]
   end
   ```

3. **What it generates automatically:**
   - `Entry.messages` — scope filtering to Message type
   - `entry.message?` — type predicate
   - `entry.message` — returns the message record (or nil)
   - `entry.message_id` — returns entryable_id when type matches
   - `entry.entryable_class` — returns Message or Comment class
   - `entry.entryable_name` — returns "message" or "comment"

4. **The Entryable module** (convention, not generated):
   ```ruby
   module Entryable
     extend ActiveSupport::Concern
     included do
       has_one :entry, as: :entryable, touch: true
     end
   end
   ```

5. **Polymorphism via delegation:**
   ```ruby
   class Entry < ApplicationRecord
     delegated_type :entryable, types: %w[ Message Comment ]
     delegate :title, to: :entryable
   end
   ```
   Each type implements `title` differently. `entry.title` dispatches polymorphically.

6. **Querying across types** is the killer feature:
   `Account.find(1).entries.order(created_at: :desc).limit(50)` — one query, all types.

7. **Rendering:** `render "entries/entryables/#{entry.entryable_name}", entry: entry`
   dispatches to type-specific partials.

8. **Nested attributes** are supported for creating delegator + delegatee together.

### Source 3: Rails Guides — Active Record Associations (Section 6: Delegated Types)

**URL:** https://guides.rubyonrails.org/association_basics.html#delegated-types

**What it is:** The official Rails guide section on delegated types with step-by-step
setup instructions.

**Key learnings:**

1. **Eliminates the STI table bloat problem** while keeping the ability to query
   across types from one table.

2. **Setup steps:**
   - Generate superclass model with `entryable_type:string entryable_id:integer`
   - Generate subclass models with their specific columns
   - Add `delegated_type :entryable, types: [...]` to superclass
   - Create `Entryable` module with `has_one :entry, as: :entryable, touch: true`
   - Include module in each subclass

3. **Object creation:** `Entry.create! entryable: Comment.new(content: "Hello!")`

4. **Further delegation** is the recommended pattern — don't just use delegated type
   for type checking. Use it to dispatch behavior polymorphically.

### Source 4: DHH's Original Pull Request (rails/rails#39341, May 2020)

**URL:** https://github.com/rails/rails/pull/39341

**What it is:** The PR that added `delegated_type` to Rails, authored by DHH.

**Key learnings:**

1. **Motivation was Basecamp-specific:** The pattern was extracted from Basecamp 3
   after years of production use. DHH: "The dash of sugar used to support the key
   architectural pattern in BC."

2. **Similar to Django's multi-table inheritance** but uses composition/delegation
   rather than actual class inheritance.

3. **The naming debate:** Many contributors suggested alternatives ("specialization",
   "subtype", etc.). DHH's rationale: "Entry delegates its type to the entryable
   role, of which we currently have Message and Comment." The delegation is conceptual
   — the entry delegates both type identity and behavior to the entryable.

4. **No referential integrity at DB level:** Uses polymorphic associations under the
   hood, so no foreign key constraints. This is a known tradeoff.

5. **Very small code change:** The entire implementation is ~100 lines of Ruby. DHH:
   "The value here is really the pattern, less the code. You could copy/paste this
   code into your ApplicationRecord and use it right now."

6. **Key insight from DHH:** "It's an inversion of the polymorphic type. In a
   polymorphic pattern, a child record can belong to any kind of thing. In delegated
   type, the parent record can have any kind of thing. You start at the top."

### Source 5: Jorge Manrubia — "Globals, callbacks and other sacrileges" (Jul 2023)

**URL:** https://dev.37signals.com/globals-callbacks-and-other-sacrileges/

**What it is:** A blog post by Principal Programmer Jorge Manrubia showing how Basecamp
uses Rails callbacks, `CurrentAttributes`, and `.suppress` — with concrete code examples
from the Bucket/Recording/Event system.

**Key learnings:**

1. **Buckets use `delegated_type` too:** The post confirms the Bucket architecture with
   a class diagram. A `Bucket` has a delegated type `Bucketable` (e.g. `Project`).
   A `Bucket` contains many `Events`. An `Event` aggregates an `Event::Request`
   (HTTP request metadata) and an `Event::Detail` (action-specific data).

2. **Bucketable auto-creates its Bucket via callback:** The `Bucketable` concern uses
   `after_create` to automatically create the companion Bucket record. This means the
   controller can just do `Current.account.projects.create!` without knowing about
   Buckets — the callback handles it. This is justified because the operation is
   simple and secondary to the Project's primary responsibility.

   ```ruby
   module Bucketable
     extend ActiveSupport::Concern
     included do
       after_create { create_bucket! account: account unless bucket.present? }
     end
   end
   ```

3. **Recordings use a factory (Recorder) instead of callbacks:** Unlike Bucketables,
   creating recordings is more complex, so a `Bucket::Recorder` factory is used.
   All recordings are created through `bucket.record(recordable, ...)`. This is the
   canonical way to create content in Basecamp:

   ```ruby
   class Bucket < ApplicationRecord
     def record(...)
       Recorder.new(self).record(...)
     end
   end

   # In a controller:
   @recording = @bucket.record new_message,
     parent: @parent_recording,
     category: find_category,
     subscribers: find_subscribers,
     status: status_param
   ```

   This reveals that recording creation takes many parameters (parent, category,
   subscribers, status, visibility, scheduled posting time) — too complex for a
   simple callback, hence the factory.

4. **`CurrentAttributes` for implicit creator tracking:** `Project` declares
   `belongs_to :creator, default: -> { Current.person }`. The creator is automatically
   set from the authenticated user without the controller needing to pass it. This
   is used throughout Basecamp and HEY for audit tracking.

5. **Event tracking via callbacks + CurrentAttributes:** The `Bucket::Eventable` concern
   combines both patterns. An `after_create` callback calls `track_event(:created)`,
   which creates an `Event` record. The event's creator defaults to `Current.person`.
   The event then auto-builds an `Event::Request` that captures HTTP details
   (`request_id`, `user_agent`, `ip_address`) from `Current`. The result: creating
   a project automatically produces a fully-audited event with request metadata,
   with zero explicit wiring in the controller.

   ```ruby
   module Bucket::Eventable
     included do
       has_many :events, dependent: :destroy
       after_create :track_created
     end

     def track_event(action, creator: Current.person, **particulars)
       Event.create! bucket: self, creator: creator, action: action,
         detail: Event::Detail.new(particulars)
     end
   end
   ```

6. **`Event.suppress` for copying:** When copying recordings, the `Recording::Copier`
   wraps the operation in `Event.suppress { ... }` to prevent the normal event
   tracking from firing. This confirms that copying goes through
   `destination_bucket.record(source_recording.recordable, ...)` — the copied
   recording points to the same recordable instance. The `.suppress` mechanism
   lets exceptional flows bypass normally-correct default behavior without adding
   conditional logic to the event system itself.

   ```ruby
   class Recording::Copier
     def copy_recording
       Event.suppress do
         @destination_recording = destination_bucket.record(
           source_recording.recordable,
           parent: destination_parent,
           **copyable_attributes
         )
       end
     end
   end
   ```

7. **Architectural philosophy:** Callbacks + CurrentAttributes work well for
   *orthogonal concerns* — things like auditing, event tracking, and creator
   assignment that are secondary to the primary domain operation. The indirection
   is a feature, not a bug. AOP (Aspect Oriented Programming) ideas, pragmatically
   applied. The alternative (explicit factories/services wiring everything together)
   couples unrelated concerns and bloats controllers.

**Why this matters for porting:** When implementing recordables in another language,
you need equivalents for:
- **Lifecycle hooks** (after_create) to auto-create companion records
- **Request-scoped globals** (CurrentAttributes) for implicit audit context
- **A factory/builder** for recording creation (too complex for simple hooks)
- **A suppression mechanism** for exceptional flows like copying

### Source 6: Steven Buccini's Blog Post (stevenbuccini.com, Jun 2020)

**URL:** https://www.stevenbuccini.com/how-to-use-delegate-types-in-rails-6-1

**What it is:** Early adopter's experience implementing delegated types for user profiles.

**Key learnings:**

1. **Practical use case:** User profiles that vary by role (Author, Editor, Reader).
   Profile is the superclass, role-specific data lives in subclass tables.

2. **Nested attributes gotcha:** Counterintuitively, the subclass
   `accepts_nested_attributes_for :profile`, not the other way around. Users fill
   out the subclass form, which creates the superclass record.

3. **Pundit integration:** Since Profile is the real superclass backed by its own table,
   you write one `ProfilePolicy` and tell subclasses to use it. Permissions and
   associations tie to the superclass.

4. **Documentation bug caught:** The PR originally had incorrect creation syntax.
   Correct: `Entry.create! entryable: Message.new(subject: "hello!")`

---

## Part 2: The Pattern Distilled (Language-Agnostic)

### Core Data Model

```
┌─────────────────────────────────────────┐
│ recordings (the "superclass" table)     │
│─────────────────────────────────────────│
│ id                                      │
│ recordable_type   (string)              │
│ recordable_id     (foreign key)         │
│ bucket_id         (container reference) │
│ parent_id         (self-referential)    │
│ creator_id                              │
│ created_at                              │
│ updated_at                              │
│ position          (ordering)            │
│ ... (other shared metadata)             │
└─────────────────────────────────────────┘
          │ points to one of:
          ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ messages     │  │ comments     │  │ documents    │
│──────────────│  │──────────────│  │──────────────│
│ id           │  │ id           │  │ id           │
│ title        │  │ body         │  │ title        │
│ content      │  │              │  │ content      │
└──────────────┘  └──────────────┘  └──────────────┘
  (recordable)      (recordable)      (recordable)
```

### The Seven Pillars

1. **Delegated Type:** One "superclass" table delegates its concrete type to separate
   "subclass" tables. Query the superclass for cross-type operations.

2. **Immutable Recordables:** Content rows are never updated in place. New versions
   are new rows. The recording pointer is updated.

3. **Recording Tree:** Parent-child relationships between recordings (not recordables)
   form a navigable hierarchy.

4. **Event Log:** Events snapshot the recording-to-recordable relationship at a point
   in time, enabling version history and audit trails. Events auto-capture HTTP
   request metadata (IP, user agent, request ID) via CurrentAttributes.

5. **Opt-in Capabilities:** Recordable types declare which operations they support
   via boolean methods. Generic code checks these before acting.

6. **Recorder Factory:** Recordings are created through `bucket.record(recordable, ...)`
   — a factory that handles the complex wiring (parent, subscribers, status, etc.).
   Simpler delegated types (like Bucketables) use lifecycle callbacks instead.

7. **Implicit Context via CurrentAttributes:** Creator tracking, request metadata,
   and account scoping flow implicitly through `Current.*` rather than being
   threaded explicitly through every call site. Orthogonal concerns (auditing,
   event tracking) attach via callbacks that read from this context.

### Why It Works

| Problem | How Recordables Solve It |
|---------|------------------------|
| Paginating mixed types | Query one table (recordings) with LIMIT/OFFSET |
| Adding new content types | New table + new recordable class. No migrations to recordings. No controller changes. |
| Copying content | New recording row, same recordable. O(1) storage. |
| Version history | Immutable recordables + event log. Compare any two versions. |
| Uniform caching | Cache key = recording. Same pattern for all types. |
| Access control | Lives on bucket, decoupled from content type. |
| API stability | New types don't break existing clients. Generic rendering. |
| Moving content | Update recording's bucket/parent. Recordable unchanged. |
| Auditing / tracking | Callbacks + CurrentAttributes auto-capture creator and request metadata |
| Copying without side effects | `Event.suppress { }` bypasses event tracking during copy |

### Mapping to Other Languages

| Rails Concept | TypeScript/Convex Equivalent | Python/SQLModel Equivalent |
|--------------|------------------------------|---------------------------|
| `recordings` table | `recordings` table with `recordableType` discriminator + `recordableId` | `Recording` model with `recordable_type` + `recordable_id` |
| `delegated_type` | Union type + runtime dispatch | Polymorphic pattern via SQLAlchemy/SQLModel |
| Concerns/mixins | Interfaces + composition functions | Mixins or Protocol classes |
| `has_one :recording, as: :recordable` | Back-reference query by recordableId | Relationship with `back_populates` |
| `recording.message?` | `recording.recordableType === "message"` | `isinstance()` or discriminator check |
| `Recording.messages` scope | Query filter: `where("recordableType", "==", "message")` | `.filter(Recording.recordable_type == "message")` |
| `delegate :title, to: :recordable` | Interface with `title` property, dispatch by type | Abstract method or protocol |
| `after_create_commit` | Mutation + triggered function | Signal/event handler or post-commit hook |
| `CurrentAttributes` | Request-scoped context (React context, AsyncLocalStorage, middleware) | `contextvars.ContextVar` or Flask `g` / FastAPI middleware |
| `Bucket::Recorder` factory | Builder function: `createRecording(bucket, recordable, opts)` | Factory class or service method |
| `Event.suppress { }` | Feature flag or context param: `{ suppressEvents: true }` | Context manager: `with suppress_events():` |

---

## Part 3: How Fizzy Implements These Patterns

### Critical Finding: Fizzy Does NOT Use `delegated_type`

Fizzy is a simpler, more focused app than Basecamp. It's a kanban board tool with
primarily two content types: **Cards** and **Comments**. Because of this narrower
domain, Fizzy uses **standard Rails patterns** rather than the full recordables
architecture:

- **No `Recording` model** — there is no central superclass table
- **No `delegated_type` declaration** anywhere in the codebase
- **No `Bucket` model** — boards directly own cards
- **No immutable recordables** — cards and comments are updated in place
- **No recording tree** — hierarchy is direct: Board → Card → Comment

### What Fizzy DOES Share With Basecamp

Despite not using delegated types, Fizzy implements several of the same architectural
principles through different mechanisms:

#### 1. Composable Capabilities via Concerns (Like Recordable Opt-in)

Card declares its capabilities through included concerns:

```ruby
# app/models/card.rb:2-4
class Card < ApplicationRecord
  include Accessible, Assignable, Attachments, Broadcastable, Closeable, Colored,
    Commentable, Entropic, Eventable, Exportable, Golden, Mentions, Multistep,
    Pinnable, Postponable, Promptable, Readable, Searchable, Stallable, Statuses,
    Storage::Tracked, Taggable, Triageable, Watchable
```

Comment has a smaller set:

```ruby
# app/models/comment.rb:2
class Comment < ApplicationRecord
  include Attachments, Eventable, Mentions, Promptable, Searchable, Storage::Tracked
```

This mirrors Basecamp's pattern where recordables opt into capabilities like
`commentable?`, `subscribable?`, etc. In Fizzy, including `Card::Commentable`
gives the card a `commentable?` method that returns true.

#### 2. Polymorphic Event System (Like Basecamp's Events)

```ruby
# app/models/event.rb:7
belongs_to :eventable, polymorphic: true  # Can be Card or Comment
```

```ruby
# app/models/concerns/eventable.rb:8
def track_event(action, creator: Current.user, board: self.board, **particulars)
```

Events store `eventable_type` + `eventable_id` and a JSON `particulars` column.
This is structurally identical to Basecamp's event system, just without the
immutable recordable snapshots for version history.

#### 3. Polymorphic Cross-Cutting Concerns

Multiple subsystems use polymorphic associations to work with both Cards and Comments:

| Fizzy Subsystem | Polymorphic Column | Types | Basecamp Equivalent |
|----------------|-------------------|-------|-------------------|
| Events | `eventable_type/id` | Card, Comment | Events on recordings |
| Mentions | `source_type/id` | Card, Comment | Mentions on recordings |
| Reactions | `reactable_type/id` | Card, Comment | Boosts on recordings |
| Search Records | `searchable_type/id` | Card, Comment | Search on recordings |
| Storage Entries | `recordable_type/id` | Card, Comment, Board | Storage tracking |
| Notifications | `source_type/id` | Event, Mention | Notifications |

#### 4. Template Method Pattern

Concerns define interfaces that models must implement:

```ruby
# app/models/concerns/searchable.rb:56-62
# Models must implement these methods:
# - search_title: returns title string or nil
# - search_content: returns content string
# - search_card_id: returns the card id
# - search_board_id: returns the board id
# - searchable?: returns whether this record should be indexed
```

Card and Comment each implement these differently — the same polymorphic dispatch
that Basecamp achieves through recordables.

#### 5. State Modeled as Separate Tables

Like Basecamp's approach, boolean states are modeled as presence/absence of
associated records rather than columns:

| State | Table | Pattern |
|-------|-------|---------|
| Golden | `card_goldnesses` | `card.golden?` = `goldness.present?` |
| Closed | `closures` | `card.closed?` = `closure.present?` |
| Postponed | `card_not_nows` | `card.postponed?` = `not_now.present?` |
| Activity Spike | `card_activity_spikes` | Presence-based detection |

### Fizzy's Architecture Diagram

```
Account
  ├── Board
  │     ├── Column (workflow stage)
  │     ├── Card (the main content type)
  │     │     ├── Comment
  │     │     │     ├── Reaction (polymorphic: reactable)
  │     │     │     └── Mention (polymorphic: source)
  │     │     ├── Reaction (polymorphic: reactable)
  │     │     ├── Mention (polymorphic: source)
  │     │     ├── Assignment
  │     │     ├── Watch
  │     │     ├── Pin
  │     │     ├── Tagging → Tag
  │     │     ├── Step (checklist items)
  │     │     ├── Closure (state record)
  │     │     ├── Goldness (state record)
  │     │     └── NotNow (state record)
  │     ├── Event (polymorphic: eventable → Card or Comment)
  │     ├── Entropy (polymorphic: container → Account or Board)
  │     └── Webhook
  ├── Tag
  ├── User → Identity
  └── Storage::Entry (polymorphic: recordable → Card, Comment, or Board)
        └── Storage::Total (polymorphic: owner → Account or Board)
```

### Key Differences: Fizzy vs. Full Recordables

| Aspect | Basecamp (Full Recordables) | Fizzy |
|--------|---------------------------|-------|
| Central table | `recordings` with `delegated_type` | No central table; Card and Comment are standalone |
| Content types | 30+ recordable types | 2 (Card, Comment) |
| Type querying | `recordings.messages.limit(50)` | `board.cards.limit(50)` (direct association) |
| Mixed-type pagination | One query on recordings table | Not needed (only one primary content type per view) |
| Copying | New recording row → same recordable | Not implemented (no copy feature) |
| Version history | Immutable recordables + events | No version history; in-place updates |
| Tree structure | Recording parent/child | Board → Card → Comment (fixed 3-level hierarchy) |
| Access control | Bucket-level, decoupled | Board-level with `Accessible` concern |
| Containers | Buckets (delegated type) | Boards (concrete model) |
| Generic controllers | One RecordingsController for all | Separate CardsController, CommentsController |
| Caching | `cache(recording)` uniform | Standard Rails caching per model |

### Why Fizzy Doesn't Need Full Recordables

1. **Only 2 content types:** The recordables pattern shines when you have 10-30+
   content types that need uniform treatment. Fizzy has Cards and Comments.

2. **No mixed-type feeds:** Fizzy never needs to paginate a timeline of different
   content types in one query. Each view shows cards or comments, not both mixed.

3. **No copying/moving between contexts:** Cards move between boards but the operation
   is a simple `update!(board: new_board)`, not a pointer swap.

4. **No version history needed:** Cards are edited in place. There's no "see what
   changed" or "revert to previous version" feature.

5. **Fixed hierarchy:** The 3-level Board → Card → Comment structure is static.
   There's no need for a flexible tree.

---

## Part 4: Porting Guide — Minimum Viable Recordables

If porting the recordables pattern to another stack, here are the essential
pieces ranked by importance:

### Must Have (Core Pattern)

1. **Recordings table** with discriminator column + foreign key to recordable
2. **Separate recordable tables** with type-specific columns
3. **Query recordables through recordings** (the whole point — one table to paginate)
4. **Type-safe dispatch** — given a recording, get the right recordable and call
   type-specific methods

### Should Have (Major Benefits)

5. **Recording tree** (parent_id self-reference) for hierarchical content
6. **Bucket/container** abstraction for access control
7. **Opt-in capabilities** interface (commentable?, exportable?, etc.)
8. **Generic operations** that work on any recording regardless of type

### Nice to Have (Advanced)

9. **Immutable recordables** with pointer swapping for version history
10. **Event log** tying recording + recordable at a point in time
11. **Copy as pointer** (new recording → existing recordable)
12. **Uniform caching** keyed on recording

### TypeScript/Convex Sketch

```typescript
// recordings table
const recordings = defineTable({
  recordableType: v.string(),     // "message" | "comment" | "document"
  recordableId: v.id("messages"), // union of all recordable table IDs
  bucketId: v.id("buckets"),
  parentId: v.optional(v.id("recordings")),
  creatorId: v.id("users"),
  position: v.optional(v.number()),
})

// Each recordable is its own table
const messages = defineTable({
  title: v.string(),
  content: v.string(),
})

const comments = defineTable({
  body: v.string(),
})

// Query: all recordings in a bucket, paginated
const feed = await ctx.db
  .query("recordings")
  .withIndex("by_bucket", q => q.eq("bucketId", bucketId))
  .order("desc")
  .paginate(paginationOpts)

// Dispatch: load the right recordable
async function loadRecordable(recording) {
  switch (recording.recordableType) {
    case "message": return ctx.db.get(recording.recordableId)
    case "comment": return ctx.db.get(recording.recordableId)
    // ...
  }
}

// Capabilities (can be a map or interface)
const capabilities = {
  message: { commentable: true, exportable: true, subscribable: true },
  comment: { commentable: false, exportable: true, subscribable: false },
}
```

### Python/SQLModel Sketch

```python
class Recording(SQLModel, table=True):
    id: uuid.UUID = Field(default_factory=uuid4, primary_key=True)
    recordable_type: str  # "message", "comment", etc.
    recordable_id: uuid.UUID
    bucket_id: uuid.UUID = Field(foreign_key="bucket.id")
    parent_id: Optional[uuid.UUID] = Field(foreign_key="recording.id")
    creator_id: uuid.UUID = Field(foreign_key="user.id")
    created_at: datetime
    updated_at: datetime

class Message(SQLModel, table=True):
    id: uuid.UUID = Field(default_factory=uuid4, primary_key=True)
    title: str
    content: str

    # Capability declarations
    commentable: ClassVar[bool] = True
    exportable: ClassVar[bool] = True

class Comment(SQLModel, table=True):
    id: uuid.UUID = Field(default_factory=uuid4, primary_key=True)
    body: str

    commentable: ClassVar[bool] = False
    exportable: ClassVar[bool] = True

# Query all recordings in a bucket, paginated
recordings = session.exec(
    select(Recording)
    .where(Recording.bucket_id == bucket_id)
    .order_by(Recording.created_at.desc())
    .limit(50).offset(page * 50)
).all()

# Dispatch to load recordable
RECORDABLE_MODELS = {"message": Message, "comment": Comment}

def load_recordable(recording: Recording):
    model = RECORDABLE_MODELS[recording.recordable_type]
    return session.get(model, recording.recordable_id)
```
