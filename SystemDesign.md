# Mobile System Design — Interview Guide

## Approach Template
Every system design question follows these 4 phases in order:

### Phase 1: Requirement Clarification
Split into **functional** and **non-functional**:

**Functional** — what the app does (group by theme, bold the category, keep to 3-5 bullets):
- What content/data is displayed?
- What user actions are supported?
- What screens/flows are in scope?
- What's explicitly out of scope?

**Non-Functional** — how well it does it (include concrete targets, not vague adjectives):
- Scale (users, data volume, request rate)
- Latency / performance targets
- Offline support
- Reliability / error tolerance
- Accessibility
- Analytics / observability

> **Senior signal**: You drive this exhaustively. The interviewer should not need to volunteer requirements you missed.

### Phase 2: Data Model Design
- List core entities with a one-line description each — exact fields vary per question, define them during the interview based on requirements
- Call out client state modes if the screen has distinct visual states (e.g., IDLE / LOADING / RESULTS)
- Note key design choices: denormalization, what's local-only vs server-sourced, money representation, etc.

### Phase 3: API Design
Define endpoints, then reason about the protocol choices.

**API Types — When to Use What:**

| Type | Use When | Pros | Cons |
|------|----------|------|------|
| **GET** | Read data, idempotent | Cacheable, safe, bookmarkable | Query string length limits, no body |
| **POST** | Create resource, non-idempotent actions | Body for complex payloads, no size limit | Not cacheable, not idempotent by default |
| **PUT** | Full replace of a resource | Idempotent (safe to retry) | Must send entire object, not partial |
| **PATCH** | Partial update of a resource | Send only changed fields | Not all backends support it, less standard |
| **DELETE** | Remove a resource | Idempotent | Some proxies/firewalls block it; soft vs hard delete is a design choice |
| **WebSocket** | Real-time bidirectional (chat, live updates) | Low latency, server-initiated pushes | Connection management, reconnection logic, battery drain |
| **SSE (Server-Sent Events)** | Server-to-client streaming (AI responses, live feeds) | Simpler than WebSocket, auto-reconnect built in, HTTP-based | One-directional only (server → client), no binary data |
| **gRPC** | High-performance, typed contracts, streaming | Efficient binary format (protobuf), bidirectional streaming | Complex setup on mobile, not browser-friendly, harder to debug |

**Pagination Patterns:**

| Pattern | How It Works | Pros | Cons |
|---------|-------------|------|------|
| **Cursor-based** | Server returns `nextCursor`, client sends it back | Stable under inserts/deletes, efficient | Can't jump to page N |
| **Offset-based** | `?offset=20&limit=20` | Simple, supports jump-to-page | Breaks when items are inserted/deleted (duplicates/skips) |
| **Keyset** | `?after_id=abc&limit=20` | Like cursor but transparent, efficient with indexed columns | Requires sortable unique key |

> **Gotcha**: Always prefer cursor for feeds/timelines. Offset is only safe for static/rarely-changing lists.

**Cursor Implementation — Two Approaches:**

**Approach 1: Encoded cursor object (opaque to client)**
```
Client sends:  GET /feed?cursor=eyJ0cyI6MTcxMTU0MDgwMCwiaWQiOiJwXzEyMyJ9&limit=20
                         ^^^^ base64-encoded JSON

Server decodes: base64 → { "ts": 1711540800, "id": "p_123" }
Server queries:
  SELECT * FROM posts
  WHERE (created_at < 1711540800)
     OR (created_at = 1711540800 AND id < 'p_123')
  ORDER BY created_at DESC, id DESC
  LIMIT 21  -- fetch 1 extra: if 21 returned → more pages; if ≤20 → nextCursor=null

Server responds:
  { posts: [20 items], nextCursor: base64({ ts: last.created_at, id: last.id }) }
```
- Pros: client is fully decoupled — server can change cursor format without client update
- Pros: can encode complex state (filters, sort order) inside the cursor
- Cons: harder to debug (opaque string)

**Approach 2: Explicit fields (transparent to client)**
```
Client sends:  GET /feed?after_timestamp=1711540800&after_id=p_123&limit=20

Server queries: (same WHERE clause as above)

Server responds:
  { posts: [20 items], lastTimestamp: 1711540799, lastId: "p_456" }
  -- client stores these and sends them in the next request
```
- Pros: simple, easy to debug, client can inspect values
- Cons: client is coupled to pagination implementation — changing from timestamp to score-based ranking requires client update
- Cons: client could tamper with values

> **Recommendation**: Use Approach 1 (encoded cursor) for production APIs. Mention Approach 2 exists as a simpler alternative. Both use the same server-side query logic — the only difference is how the cursor is packaged.

**Common API Design Gotchas:**
- **Idempotency keys**: POST requests (create order, send message) should accept an idempotency key so retries don't create duplicates
- **Partial responses**: For large objects, support field selection (`?fields=id,name,avatar`) to reduce payload
- **Error contract**: Define a consistent error shape (`{ code, message, details }`) — don't mix HTTP status codes with ad-hoc error formats
- **Versioning**: URL path (`/v2/feed`) vs header (`Accept: application/vnd.api+json;version=2`) — mention you'd version the API
- **Rate limiting**: Backend should return `429 Too Many Requests` with `Retry-After` header — client should respect it
- **Optimistic vs pessimistic**:
  - **Optimistic**: update UI immediately (assume success), fire API in background, rollback if it fails. Use for low-risk reversible actions (like, add to cart)
  - **Pessimistic**: show spinner, wait for server response, then update UI. Use for high-risk irreversible actions (payment, order placement)

### Phase 4: Mobile Architecture Design
Layer the mobile client:

```
UI Layer (Compose)
  └─ Screens, components, navigation
Presentation Layer (ViewModel)
  └─ UiState, user action handling, business logic coordination
Domain Layer (optional)
  └─ Use cases / interactors (only if logic is complex enough to warrant it)
Data Layer (Repository)
  ├─ Remote data source (Retrofit / WebSocket / SSE client)
  ├─ Local data source (Room / DataStore)
  └─ Caching strategy (in-memory, disk, or both)
```

**Key decisions to call out:**
- Where does the single source of truth live? (Room? In-memory? Server?)
- How does data flow? (One-shot fetch vs reactive Flow/StateFlow)
- How are errors propagated? (Sealed class Result, exceptions, UiState flags)
- What survives process death? (ViewModel survives config change only; SavedStateHandle or Room for process death)

---

## 1. Feed / Timeline
Social feed, news feed, activity feed

### Phase 1: Requirement Clarification

**Functional Requirements — Ask About:**
- Feed content: Text / Image / Video / like / comments / views ..
- Actions on posts: like, comment, share, bookmark, report, edit, delete?
- Author interaction: tap avatar/name → profile screen?
- Feed algorithm: chronological or ranked? (affects caching & invalidation)

**Non-Functional Requirements — Ask About:**
- Scale: how many posts? How many followers per user?
- offline post caching: cache size, what to be cache E.g. just the post, or other metaData
- Cache limits: max cache size? Eviction policy? Prevent app storage from growing unbounded (e.g., cap Room DB or image cache at N MB, LRU evict oldest)

**Things Interviewers Leave Ambiguous on Purpose:**
- Whether likes are optimistic or wait-for-server
- What happens on network failure mid-scroll
- Whether feed persists across app restarts
- How stale data is handled when returning from background

### Phase 2: Data Model

```
Post {
  id: String
  auther: { autherId: String, autherName: String. autherAvatorUrl: String },
  content: String,
  media: List<MediaItem>
  likeCount: Int
  commentCount: Int,
  viewCount: Int,
  isLiked: Boolean
  updateddAt: Instant          // kotlinx-datetime, timezone-neutral point in time
}

MediaItem {
  id: String,
  type: MediaType            // IMAGE, VIDEO, GIF
  url: String
  thumbnailUrl: String?      // for video preview
}
```

### Phase 3: API Design

```
// Limit set by mobiles based on their screen sides
GET  /feed?cursor={cursor}&limit=20    // load more — returns next page after cursor
// Response: { posts: [Post], nextCursor: "abc123" }  // null when no more

// If likes are a separate table (creating/deleting a resource):
POST   /feed/{postId}/like      // insert into Likes table
DELETE /feed/{postId}/like      // delete from Likes table

GET  /feed/{postId}/comments?cursor={cursor}&limit=20
```

### Phase 4: Mobile Architecture

**Presentation Layer**
 * LazyColumn
 * PullToRefresh
 * auto load more with threshold -> (detect last 3-5 items visible → prefetch)

```kotlin
@Composable FeedListScreen

PullToRefreshBox {

    LazyColumn {
        // Trigger load more when near the end
        val shouldLoadMore by remember {
            derivedStateOf {
                val lastVisibleItem = listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
                val totalItems = listState.layoutInfo.totalItemsCount
                lastVisibleItem >= totalItems - 3
            }
        }

        LaunchedEffect(shouldLoadMore) {
            if (shouldLoadMore) {
                viewModel.loadMore()
            }
        }

        items { // feed rendering }
        }
    }
}

FeedListViewModel {
    val feeds = StateFlow<List<Feeds>> // collect with life cycle awareness
    
    fun load(cursor: String?)  
    fun refresh()
    fun onLikeClicked(isLiked: Boolean)
}
```
**Domain Layer**

```kotlin
FeedLikeUseCase // handle like and unlike

FeedsRepository {
    val feeds: Flow<Feeds>
        get = FeedDao.FeedFlow()
  
  suspend fun loadFeeds(cursor: String?) {
      // load from remote by cursor
      // insert feeds to Room with replaced conflict strategy
  }

  suspend fun likeOrUnLike(isLike: String) {
      // update local cache
      // enqueue Worker to update the backend
      // remove like action in local cache
  }
    
}
```
**Data Access Layer**

```kotlin

// Room
data class LikeAction(
    val postId: String,
    val isLike: Boolean,
)

interface FeedsDao {
  fun feedsFlow(): Flow<List<Feeds>>
}

interdace LikeActionDao {
    @Insert
   suspend fun insertLikeAction()
   
    @Delete
    suspend fun deleteLikeActionByPostId()
}

```

**Key Trade-offs:**

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Pagination | Cursor-based | Offset-based | Cursor — feed items shift |
| Caching | In-memory only | Room DB | Room if offline required or cold start <1s target |
| Image loading | Coil with disk cache | Manual | Always use Coil/Glide |
| Like action | Optimistic | Pessimistic | Optimistic — rollback on failure |
| New posts | Pull-to-refresh | WebSocket/polling | Pull-to-refresh for MVP |
| Feed source | Fan-out on write | Fan-out on read | Backend concern — mention both, write for small graphs |

**Gotchas:**
- Infinite scroll trigger: prefetch at last 3-5 items, not at the very bottom
- Stale on resume: returning from background → feed may be stale. Auto-refresh risks losing scroll position
- Scroll position: config change handled by `rememberLazyListState`, process death needs `rememberSaveable` or persisted index
- Like spam: debounce user input. Not a big deal if it is back by local cache

**Caching Strategy:**

Cache has one purpose: **store a limited number of latest posts for fast cold start before the initial fetch completes.**

- Source of truth is always **remote**
- Cache is a **read-only snapshot** — never mutated by user actions
- Data flow is one-directional: server → cache → UI
- On pull-to-refresh: clear cache → re-fetch → re-populate
- On app restart: show cache immediately → fetch in background → replace cache
- Cap cache size (e.g., last 100-200 posts) to prevent unbounded storage growth
- Eviction: delete oldest rows when exceeding cap
- Image/media cache handled separately by Coil/Glide disk cache (set max size, e.g., 250MB)

```
Server (source of truth) ──fetch──→ Room cache (read-only) ──observe──→ UI
```

Edge cases:
- Like then unlike same post while offline → **collapse the queue** (cancel out, remove both)
- Post deleted server-side while action is pending → replay fails, remove from queue, revert UI
- App killed while offline → queue is in Room, survives process death. WorkManager picks up on next launch

---

## 2. Messaging / Chat
1:1 chat, chat list + chat detail. Group chat as follow-up.

### Phase 1: Requirement Clarification

**Functional Requirements — Ask About:**
- Chat types: 1:1 only? Group? (scoped to 1:1, group as follow-up)
- Message types: text, attachment, image, video
- Message states: SENDING → SENT → DELIVERED → READ
- Chat list: how chat is sorted
- Message actions: delete? Edit? Reply? React?
- Typing indicator?
- Online/offline presence?
- Search and filtering? searchable properties

**Non-Functional Requirements — Ask About:**
- Latency: real-time delivery target? (WebSocket vs polling)
- Offline: can users compose and send messages offline?
- Media upload: max file size? Compression? Thumbnail generation?
- Push notifications: when app is backgrounded?

**Scope Boundaries — Ask These:**
- Media upload flow in scope or just text?
- Voice/video call?
- Group chat management (create, add/remove members)?
- Message forwarding?
- Per-conversation notification settings?

**Things Interviewers Leave Ambiguous on Purpose:**
- What happens when you send a message offline
- How to handle message ordering with unreliable network
- Whether history is fetched from server or local-first
- How read receipts are triggered (on screen visible? On conversation open?)

### Phase 2: Data Model

```
Participant {
  userId: String
  name: String
  avatarUrl: String
}

// Stored in Room — each row is one chat in the Chat List screen
// Query: SELECT * FROM Conversation ORDER BY lastMessageAt DESC
Conversation {
  id: String
  participants: List<Participant>  // 1 for 1:1, N for group chat (future)
  lastMessage: String?             // denormalized — preview text for chat list
  lastMessageAt: Instant?
  unreadCount: Int                 // badge on chat list row — how many messages user hasn't read
                                   // increment on new received message, reset to 0 when user opens + reads
}

// Value classes for type-safe IDs — prevents mixing up ID types, zero runtime overhead
@JvmInline value class MessageId(val value: String)
@JvmInline value class ConversationId(val value: String)
@JvmInline value class UserId(val value: String)

Message {
  id: MessageId                // client-generated UUID (enables offline send + dedup)
  conversationId: ConversationId
  senderId: UserId
  content: String
  attachments: List<Attachment>
  status: MessageStatus        // every message has a status
  createdAt: Instant
  updatedAt: Instant           // set on create, edit, or soft delete — used for delta sync
  isDeleted: Boolean           // soft delete flag — client removes from UI when true
}

// Every message has a status. UI decides what to show:
//   Sent messages: show status indicator (✓, ✓✓, blue ✓✓)
//   Received messages: status is DELIVERED or READ — UI can ignore it
MessageStatus: SENDING → SENT → DELIVERED → READ
  SENDING   → saved locally, not confirmed by server
  SENT      → server acknowledged receipt
  DELIVERED → recipient's device received it
  READ      → recipient opened the conversation

// Sealed interface — each type has different properties and caching strategies
sealed interface Attachment {

  // Image: single url field
  //   Sending (before upload): url = local URI
  //   After upload / received: url = remote URL
  //   Caching: Coil/Glide handles download + disk cache + eviction automatically
  data class Image(
    val url: String,
  ) : Attachment

  // Video: no local caching of full video
  //   Chat bubble: show thumbnail + duration + play button
  //   User taps play: stream from remoteUrl (no full download)
  //   Sending: local file exists briefly during upload only
  data class Video(
    val remoteUrl: String?,    // null while uploading, stream URL after
    val thumbnailUrl: String?, // shown in chat bubble, Coil handles caching
    val durationMs: Long,
  ) : Attachment

  // File: remoteUrl + localPath
  //   Needs real local path to open with other apps (Intent)
  //   Same caching logic as Video
  data class File(
    val remoteUrl: String?,    // null while uploading
    val localPath: String?,    // null if not downloaded / not yet picked
    val fileName: String,
    val fileSize: Long,        // bytes — for display ("2.4 MB")
    val mimeType: String,      // for icon / open-with intent
  ) : Attachment
}
```

> **Key**: Message `id` is generated **client-side** (UUID). This allows:
> - Persisting locally before server confirms
> - Sending offline (queue and replay)
> - Server-side deduplication on retry (same UUID = same message)

### Phase 3: API Design

```
// Conversation list — sorted by lastMessageAt DESC
GET  /conversations?cursor={cursor}&limit=20

// Messages in a conversation
GET  /conversations/{id}/messages?cursor={cursor}&limit=30    // older messages

// Send messages
WS  /ws?token={authToken}
// Server → Client:
{ type: "new_message", message: Message }
{ type: "status_update", messageId: String, status: MessageStatus }
{ type: "typing", conversationId: String, userId: String }

// Client → Server:
{ type: "send_message", clientId: UUID, conversationId: String, content: String }
{ type: "typing", conversationId: String }
{ type: "ack", messageId: String }     // client confirms receipt → triggers DELIVERED

// REST — used as fallback or primary if no WebSocket
POST /conversations/{id}/messages

// Upload media (separate from message send)
// Uses multipart/form-data, NOT JSON body
//   - JSON would require base64 encoding → 33% size overhead, entire file in memory
//   - Multipart sends raw binary → no overhead, supports streaming, progress tracking
//
// Multipart request structure:
//   Part 1: file (raw binary bytes, e.g., image/jpeg)
//   Part 2: metadata (JSON, e.g., conversationId)
//
// Retrofit:
//   @Multipart
//   @POST("upload")
//   suspend fun upload(
//       @Part file: MultipartBody.Part,
//       @Part("metadata") metadata: RequestBody
//   ): UploadResponse
//
// Progress tracking: wrap RequestBody to count bytes written → emit to Flow/callback
POST /upload
Body: multipart/form-data (file + metadata)
Response: { url: String, thumbnailUrl: String? }
```

**Real-time: WebSocket vs Polling vs Long Polling**

| | WebSocket | Polling | Long Polling |
|--|-----------|---------|-------------|
| **How** | Persistent connection, server pushes events | Client requests on interval | Client requests, server holds until data available |
| **Latency** | Near-instant (<100ms) | Depends on interval (3-5s) | Near-instant (server responds when data arrives) |
| **Battery** | One connection, efficient | Repeated requests, worst | Better than polling, worse than WebSocket |
| **Complexity** | Connection lifecycle, heartbeat, reconnection | Simple HTTP calls | Moderate — timeout handling, immediate re-request |
| **Direction** | Bidirectional | Client → server only | Server → client only (still need REST to send) |
| **Offline/background** | Dies when app is backgrounded | Can stop polling, resume on foreground | Same as polling |
| **Scaling** | Server holds open connections | Stateless, easiest | Server holds open connections (like WebSocket) |

**When to use which:**
- **WebSocket**: best for chat — bidirectional, lowest latency, most efficient when active
- **Polling**: simplest to implement, good enough for non-critical real-time (e.g., chat list updates)
- **Long Polling**: near-real-time without WebSocket complexity, useful when infra doesn't support WebSocket (some proxies/firewalls block it). Downside: each message needs a new HTTP connection, server must handle timeout (e.g., 30s no data → respond empty → client re-requests)

**WebSocket connection management:**

```
Lifecycle — WHEN to connect/disconnect:
  App foreground  → open WebSocket
  App background  → close WebSocket (saves battery, OS may kill it anyway)
  App returns     → reopen WebSocket + gap sync via REST
  App killed      → connection dies, rely on FCM push for notifications

Reconnection — WHAT to do when connection drops:
  Connection drops → wait 1s → retry
    fails → 2s → retry
    fails → 4s → retry
    fails → 8s → retry
    ... cap at 30s
  On success → reset backoff to 1s
  Without backoff: hammers server during outages
  Exponential backoff: backs off gracefully, recovers quickly

Heartbeat — HOW to detect silent connection death:
  Problem: connection looks open but network silently dropped it
  Client sends ping every 30s → server responds pong
  No pong within 10s → connection is dead → close and reconnect
  Without heartbeat: client sends messages into the void,
    user sees no error, could be stuck for minutes until TCP timeout
```

**Polling approach:**
```
// Poll for new messages every 3-5 seconds while chat is open
GET /conversations/{id}/messages?after={lastKnownMessageId}&limit=50

// Poll conversation list for updates (unread counts, last message)
GET /conversations?updatedAfter={lastSyncTimestamp}&limit=50
```

**Recommendation:**
- Use **WebSocket** when chat is open (real-time feel)
- Use **FCM push notification** when app is backgrounded
- On app reopen: reconnect WebSocket + sync gap via REST

**Disconnection recovery flow:**
```
Heartbeat ping → no pong → connection is dead
  → close WebSocket
  → two things happen in parallel:
      1. Reconnect WebSocket with exponential backoff (background)
      2. Any user sends → fall back to REST POST (user isn't blocked)
  → WebSocket reconnects → switch back to WebSocket + gap sync via REST
  → Server deduplicates by clientId (no double sends between REST and WebSocket)
```

### Phase 4: Mobile Architecture

**Presentation Layer**

```kotlin
@Composable ConversationListScreen // show a list of conversation

ConversationListViewModel

// show a list of message
@Composable MessagesScreen {
    Column {
      val listState = rememberLazyListState()
      val isAtBottom by remember {
        derivedStateOf {
          val lastVisible = listState.layoutInfo.visibleItemsInfo.lastOrNull()
          lastVisible != null && lastVisible.index >= messages.size - 1
        }
      }

      LaunchedEffect(messages.size) {
        if (!isAtBottom) {
          // trigger scroll to bottom action
        }
      }
      
      LazyColumn {
          items {
            Row(
              modifier = Modifier.fillMaxWidth(),
              horizontalArrangement = if (isOwner) Arrangement.End else Arrangement.Start,
            )
          }
      }
        
      OutlineEditTextField() // message input
    }
}

MessagesViewModel
```
**Domain layer**

```agsl
ConversationRepository // sync conversation list, update unread counts
MessageRepository // local-first writes, sync via WebSocket/REST
```
**Data access layer**
```agsl
Remote: WebSocket + Retrofit (fallback)
Local: Room (SOURCE OF TRUTH)
```

**Why Room is source of truth (not remote):**
- User creates data locally (sends messages)
- Messages must display immediately before server confirms
- Chat history is too large to re-fetch every time
- Offline support requires local persistence

**Data flow — Sending a message:**
```
1. User taps send
2. Generate client-side UUID
3. Insert into Room: status = SENDING
4. UI updates immediately (observing Room Flow)
5. If media attached: upload file first → get URL
6. Send via WebSocket (or REST fallback if disconnected)
7. Server acknowledges → update Room: status = SENT
8. Recipient receives → server pushes DELIVERED → update Room
9. Recipient reads → server pushes READ → update Room
```

**Data flow — Receiving a message:**
```
1. WebSocket receives new_message event
2. Insert into Room (INSERT OR IGNORE — dedup by id)
3. UI updates (observing Room Flow)
4. Send ack to server → triggers DELIVERED on sender's side
5. Update conversation: lastMessage, lastMessageAt, unreadCount
```

**Data flow — Gap sync (reconnecting after offline):**
```
1. WebSocket reconnects
2. Query: GET /conversations/{id}/messages?after={lastKnownId}
3. Insert all into Room (dedup by id)
4. UI catches up automatically via Room Flow
```

**Key Trade-offs:**

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Real-time | WebSocket | Polling | WebSocket when chat open, FCM when backgrounded |
| Source of truth | Room | Remote | Room — user creates data locally |
| Message ID | Server-generated | Client UUID | Client UUID — offline support + dedup |
| Message ordering | By createdAt | By server sequence number | Sequence number — client clocks drift |
| Media upload | Upload first, then send | Send with localUri, upload async | Upload first — simpler, message arrives complete |
| History storage | All in Room | Last N, fetch older on demand | Last N per conversation (e.g., 500), fetch older via API |

**Caching Strategy:**

Room IS the primary storage, not just a cache:
- All messages persisted locally on send and receive
- Cap per conversation (e.g., 500 messages), older available via API on scroll-up
- Conversations list always in Room, updated by WebSocket/polling
- Media: store URLs only in Room, Coil/Glide handles media caching with disk cache limit

**Delta Sync (optimization):**

Only fetch what changed since last sync — not everything:
```
// Store lastSyncTimestamp locally (DataStore or Room metadata table)

// Conversations — fetch updated since last sync
GET /conversations?updatedAfter={lastSyncTimestamp}

// Messages — fetch created, edited, or soft-deleted since last sync
GET /conversations/{id}/messages?updatedAfter={lastSyncTimestamp}

// Timestamp covers all cases:
//   New message at 10:05      → updatedAt > lastSync ✓
//   Message edited at 10:03   → updatedAt > lastSync ✓
//   Message deleted at 10:02  → updatedAt > lastSync ✓ (isDeleted=true)
//
// Client upserts into Room (or removes from UI if isDeleted=true)
// Update lastSyncTimestamp after successful sync
```

Trade-offs:

| | Full Fetch | Delta Sync |
|--|-----------|------------|
| **Bandwidth** | Re-fetches everything, wasteful | Only changed data, efficient |
| **Server complexity** | Simple query | Must track `updatedAt` on every mutation, must soft-delete |
| **Client complexity** | Replace local data | Upsert/merge logic, track `lastSyncTimestamp`, handle `isDeleted` |
| **Data consistency** | Always correct — full snapshot | Can drift if `lastSyncTimestamp` is lost or clock skews |
| **Recovery** | Just re-fetch | Need a "full sync" fallback for edge cases |
| **DB bloat** | N/A | Soft deletes accumulate — server needs periodic tombstone cleanup |

The real cost: soft deletes ripple through the entire backend — every query must filter `WHERE isDeleted = false`, every table needs `updatedAt` maintained on every write. It's a backend architectural commitment, not just a client optimization.

Worth it: high-frequency sync (chat, collaborative tools) with lots of data synced often.
Not worth it: feed apps, low-frequency data — just re-fetch.

Why timestamp, not lastKnownMessageId:
- Message ID only catches **new** messages
- Timestamp catches new + edited + deleted (anything with updatedAt > lastSync)

When to use:
- Active user with many conversations → delta sync saves significant bandwidth
- Fresh install or long offline → fall back to full fetch (too much delta to replay)

Gotcha — soft deletes required:
- Hard deletes disappear from the DB — delta sync can't discover them
- Server must soft-delete (set `isDeleted=true`, update `updatedAt`)
- Client picks up the soft delete and removes from UI

**Gotchas:**
- WebSocket reconnection: exponential backoff (1s, 2s, 4s... cap 30s). Don't hammer the server
- Background delivery: WebSocket dies when backgrounded. FCM push wakes the app / notifies user. On next open, reconnect + gap sync
- Duplicate messages: server deduplicates by `clientId`. Client deduplicates by `id` on Room insert (INSERT OR IGNORE)
- Message gap: after reconnecting, fetch `?after={lastKnownId}` to fill missed messages
- Read receipts: trigger when conversation is visible AND scrolled to bottom, not just on open. Don't send read receipts for every message — batch with the latest `lastReadMessageId`
- Scroll behavior: if user is reading old messages and new message arrives, do NOT auto-scroll. Show "New messages ↓" floating button
- Failed sends: messages stuck in SENDING after timeout → update to FAILED, show retry button in UI
- Upload failure: if media upload fails, message stays in SENDING. Don't send the text without the attachment — user expects them together
- Conversation ordering: chat list sorted by `lastMessageAt`. When a new message arrives (sent or received), that conversation moves to top

**Follow-up scope (group chat):**
- Conversation gains `type: GROUP`, `name`, `participants: List<Participant>`
- Read receipts become "read by N of M" — batch, don't send per-participant
- Typing indicator: "Alice is typing..." or "Alice and Bob are typing..."
- Member management: add/remove participants, admin roles
- Larger message volume — may need more aggressive local eviction

---

## 3. Media Player
Video player, music player with background playback

### Phase 1: Requirement Clarification
### Phase 2: Data Model
### Phase 3: API Design
### Phase 4: Mobile Architecture

---

## 4. Image-heavy App
Photo gallery, camera + upload, image editing

### Phase 1: Requirement Clarification
### Phase 2: Data Model
### Phase 3: API Design
### Phase 4: Mobile Architecture

---

## 5. Search
Search with autocomplete, filters, recent/trending

### Phase 1: Requirement Clarification

**Functional Requirements — Ask About:**
- **What's searchable**: entity types (products, users, posts, places)? Filters (category, price, rating)? Sort options?
- **As-you-type experience**: autocomplete suggestions? Recent searches? Trending when query is empty?
- **Results display**: list vs grid? What metadata per result? Highlight matched terms? Tap → where?

**Non-Functional Requirements — Ask About:**
- **Latency**: autocomplete target (<200ms), debounce to avoid per-keystroke server calls
- **Scale & search quality**: how many items indexed? Typo tolerance / fuzzy matching? Multi-language?
- **Offline & analytics**: search cached data offline? Track queries, clicks, conversion?

**Scope Boundaries — Ask These:**
- Is the search backend (Elasticsearch, Algolia) in scope, or just the mobile client?
- Filter UI: bottom sheet, separate screen, inline chips?
- Search within a specific context (e.g., search within a category) or global?
- Pagination of results: infinite scroll or "load more"?
- Deep linking to search results (share a search URL)?

**Things Interviewers Leave Ambiguous on Purpose:**
- Whether autocomplete hits the server or uses local data
- How to handle zero results (show suggestions? "Did you mean…"?)
- Whether filters are AND or OR within the same dimension
- When to persist a search to history (on submit? on result tap?)
- How debounce interacts with autocomplete UX

### Phase 2: Data Model

Search has 3 core models. Exact fields vary per question — define them based on requirements.

```agsl
// what the autocomplete dropdown shows (text + type: RECENT/TRENDING/AUTOCOMPLETE)
SearchSuggestion {
    id: String,
    suggestion: String,
    type: { RECENT/TRENDING/AUTOCOMPLETE }
}

SearchResult        — a single result item (id, title, image, type-specific metadata)
RecentSearch        — persisted locally in Room (query + timestamp)
```

**Client state has 3 modes:**
```
IDLE        → search bar empty → show trending/recent
SUGGESTING  → user is typing → show autocomplete dropdown
RESULTS     → user submitted query → show result list + filters
```

**Key design choices:**
- Filters come from the **server**, not hardcoded — backend can add/remove filter dimensions without client update
- Recent searches stored in **Room** — easy to query, sort by recency, cap at N rows

### Phase 3: API Design

```
// Autocomplete — called on every keystroke (after debounce)
// Lightweight endpoint: returns only suggestion strings, no full results
GET /search/autocomplete?q={query}&limit=8

// Full search — called on submit or filter change
// Returns results + available filters + total count
// Filters as individual query params — readable, debuggable, CDN-cacheable
GET /search?q={query}&cursor={cursor}&limit=20&category=running&sort=relevance
// Cursor is opaque (base64-encoded JSON) — client passes it back as-is
// What's inside depends on the sort:
//   sort=relevance → { score: 0.87, id: "r_123" }
//   sort=price_asc → { price: 49.99, id: "r_123" }
//   sort=newest    → { createdAt: 1711540800, id: "r_123" }
// Always includes id as tiebreaker — same pattern as feed cursor (Section 1)

// Trending searches — called when search bar is focused with empty query
GET /search/trending?limit=10
```

**Why separate autocomplete and search endpoints:**
- Autocomplete: called frequently (every keystroke after debounce), must be <100ms, returns lightweight data (just strings)
- Search: called on explicit submit, can be heavier, returns full results + filters + counts
- Different backend optimizations: autocomplete uses prefix matching / trie / completion index; search uses full-text ranking

**Caching autocomplete responses:**
```
Client-side LRU cache: Map<String, AutocompleteResponse>
  "sho"  → [shoes, shorts, shopping, ...]
  "shoe" → [shoes, shoe rack, shoe store, ...]

Cache hit: user types "shoe", deletes "e", types "e" again → "sho" is cached, no network call
Cache size: last 50-100 queries is enough (small payloads)
Invalidation: clear on session end or after 5-10 minutes (suggestions shouldn't be stale)

Server-side: autocomplete responses are highly cacheable (CDN/edge cache)
  Cache-Control: public, max-age=60 (suggestions don't change per-second)
```

**Filter interaction with search:**
```
User flow:
  1. Search "shoes" → results + available filters appear
  2. Tap "Category: Running" filter → re-fetch with filter applied
  3. Available filters UPDATE (counts change, some options may disappear)
     e.g., "Brand: Nike (45), Adidas (32)" → reflects filtered results

// Each filter change triggers a new search request:
GET /search?q=shoes&filters={"category":["running"]}&limit=20

// Server returns UPDATED filter groups — counts reflect current filtered set
// This is called "faceted search" — filters are dynamic, not static
```

### Phase 4: Mobile Architecture

**Presentation Layer**

```kotlin
@Composable
fun SearchBar() {
    // Compose Material 3 provides SearchBar and DockedSearchBar which have built-in support 
    // for showing search suggestions.

  SearchBar(
    inputField = {
      SearchBarDefaults.InputField(
        query = query,
        onQueryChange = { query = it },
        onSearch = { expanded = false },
        expanded = expanded, // whether the suggest panel is available
        onExpandedChange = { expanded = it },
        placeholder = { Text("Search") },
        leadingIcon = { Icon(Icons.Default.Search, contentDescription = null) },
      )
    },
    expanded = expanded,
    onExpandedChange = { expanded = it },
  ) {
    // Suggestion content goes here
    suggestions.forEach { suggestion ->
      ListItem(
        headlineContent = { Text(suggestion) },
        modifier = Modifier
          .clickable { query = suggestion; expanded = false }
          .fillMaxWidth()
      )
    }
  }
}

@Composable
fun SearchResultScreen()

clas SearchBarViewModel {
    // debounce on refreshing suggestions
    val suggestions = queryFlow
      .debounce(300)
      .filter { it.isNotBlank() }
      .mapLatest { query ->
        // When new query arrives, previous coroutine is CANCELLED
        // Retrofit cancels the underlying OkHttp Call too (best-effort)
        searchRepository.autocomplete(query)
      }
      .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())
}
```

**Domain Layer**
```kotlin
interface SearchRepository {
    val suggestions: Flow<List<Suggestion>>
    suspend fun refreshSuggestion(query: String)
    suspend fun query(query: String)
    suspend fun getRecentTrends()
}

@Dao
interface SuggestionsDao {
  @Query("SELECT * FROM SuggestionEntity WHERE suggestion LIKE '%' || :query || '%'")
  suspend fun getSuggestion(query: String): List<SuggestionEntity>
  
  @Insert
  suspend fun insertSuggestions(suggestions: List<Suggestion>)
}

// REST API calls
interface SearchClient
```

**Key Trade-offs:**

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Autocomplete source | Server | Local index | Server — keeps suggestions fresh, avoids syncing index to device |
| Debounce timing | 150ms (responsive) | 300ms (fewer requests) | 300ms default, 150ms if backend is fast and cheap |
| Recent searches | Room | DataStore | Room — easier to query, sort, cap at N rows |
| Result caching | Cache search results | Always re-fetch | Re-fetch — results change frequently, stale results confuse users |
| Filtering approach | Smart query parsing (backend extracts intent from query) | Explicit filter UI (chips + bottom sheet) | Smart parsing for most apps; explicit filters only for e-commerce/catalog where users need precise narrowing |
| Suggestion tap | Fill search bar + auto-submit | Fill search bar only | Auto-submit — user tapped a suggestion because they want those results |
| Search cancellation | Cancel on new query | Let all complete | Cancel (flatMapLatest) — only latest query matters |

**Recent searches — Room schema:**
```
@Entity
data class RecentSearchEntity(
  @PrimaryKey val query: String,       // deduplicated by query text
  val timestamp: Long                  // for ORDER BY DESC, cap at 20-30 rows
)

// On save: INSERT OR REPLACE — if "shoes" already exists, just update timestamp
// On display: SELECT * FROM recent_searches ORDER BY timestamp DESC LIMIT 20
// Cleanup: DELETE WHERE query NOT IN (SELECT query ORDER BY timestamp DESC LIMIT 30)
```

**Keyboard & focus management:**
```
SearchBar focus states:
  1. Screen opens → auto-focus SearchBar, show keyboard
  2. User types → show suggestions (SUGGESTING mode)
  3. User taps suggestion or presses search → hide keyboard, show results (RESULTS mode)
  4. User taps SearchBar again while in RESULTS → show suggestions over results
  5. User presses back:
       In SUGGESTING mode → clear focus, hide keyboard, return to IDLE
       In RESULTS mode → if SearchBar focused, unfocus; else navigate back

Implementation:
  val focusRequester = remember { FocusRequester() }
  LaunchedEffect(Unit) { focusRequester.requestFocus() }
  // On search submit:
  focusManager.clearFocus()    // hides keyboard
  keyboardController?.hide()   // explicit keyboard dismiss
```

**Gotchas:**
- Empty query autocomplete: don't send empty string to autocomplete endpoint — show recent/trending instead
- Filter counts: showing counts like "Nike (45)" requires the backend to compute facets — expensive. Mention this as a trade-off; some apps skip counts
- Recent search privacy: offer "clear all" and per-item delete. Consider not persisting sensitive-looking queries (though this is hard to detect)
- Screen rotation: search results and filters should survive config change via ViewModel. Active query and filters should survive process death via SavedStateHandle

---

## 6. E-commerce
Product catalog, cart, checkout flow

### Phase 1: Requirement Clarification

**Functional Requirements — Ask About:**
- **Browsing & discovery**: category tree, search with filters (price, brand, rating, size/color), sort (price, popularity, newest)? Recommendations ("also bought", "recently viewed")?
- **Product detail**: image carousel, variant selector (size × color matrix), specs, delivery estimate? Reviews — read-only or user-generated with photo upload?
- **Cart & checkout**: add/remove/update quantity, save for later, coupon/promo code? Checkout flow: address → payment → review → confirm? Guest checkout allowed?
- **Post-purchase**: order history, real-time tracking, cancel/return/refund flow?

**Non-Functional Requirements — Ask About:**
- **Latency & scale**: product page <500ms, search autocomplete <200ms; how many SKUs? Concurrent users during flash sales (10×–100× normal)?
- **Offline & caching**: browse previously viewed products offline? Offline cart editable? Cache budget for product images + catalog?
- **Inventory consistency**: real-time stock count, or "in stock / out of stock" boolean sufficient? Acceptable staleness window?
- **Payment security**: PCI compliance — tokenized payments via Stripe/Braintree SDK, no raw card data on device or your server
- **Analytics**: add-to-cart rate, checkout funnel drop-off, search-to-purchase conversion, impression tracking on product cards?

**Scope Boundaries — Ask These:**
- Search and filtering in scope, or just category browsing?
- Payment integration in scope, or just the UI flow?
- Seller/admin side in scope?
- Push notifications for order updates / price drops?
- Multi-currency / multi-language?
- Guest checkout or login required?

**Things Interviewers Leave Ambiguous on Purpose:**
- Whether cart lives on server, device, or both (and merge strategy for logged-out → logged-in)
- How to handle "item out of stock" after user added it to cart
- What happens if price changes between add-to-cart and checkout
- Flash sale / high-concurrency inventory race conditions

### Phase 2: Data Model

E-commerce has 6 core models. Exact fields vary per question — define them based on requirements.

```
Product         — catalog item (name, brand, images, base price, rating, variants, stock status)
Variant         — a purchasable SKU within a product (size × color combo, its own price + stock)
Price/Money      — money object: amount (Long in cents) + currencyCode (never a bare Long or Float/Double)
CartItem        — product + variant + quantity; denormalize name/image/price snapshot for display
Cart            — server-authoritative: items + subtotal/tax/discount/total + applied coupon
Order           — placed order: items snapshot, status (PLACED → SHIPPED → DELIVERED), tracking, address
```

**Client state has 3 screens with distinct modes:**
```
ProductList   → LOADING / CONTENT (with filters + pagination) / ERROR
Cart          → LOADING / CONTENT / UPDATING (quantity change in progress) / EMPTY
Checkout      → linear stepper: ADDRESS → PAYMENT → REVIEW → PLACING_ORDER → CONFIRMED
```

**Key design choices:**
- All money as a **Money object** `{ amount: Long, currencyCode: CurrencyCode }` — `$19.99 = Money(1999, USD)`. Never bare Long or Float/Double
- `CartItem` **denormalizes** product name, image, price — cart screen doesn't refetch product details
- Cart price is a **snapshot at add-time** — must be re-verified server-side at checkout
- `Cart.total` is **server-computed** — client never calculates totals (tax rules, coupons are server logic)

### Phase 3: API Design

```
// Catalog
GET  /products?category={cat}&cursor={cursor}&limit=20
GET  /products/search?q={query}&minPrice=1000&maxPrice=5000&cursor={cursor}&limit=20
GET  /products/{productId}                // full detail with variants, reviews summary

// Cart (server-authoritative)
GET    /cart                              // current user's cart
POST   /cart/items { productId, variantId, quantity }   // add item
PUT   /cart/items { productId, variantId, quantity }   // add item
DELETE /cart/items/{itemId}               // remove item
POST   /cart/coupon { code: "SAVE20" }   // apply coupon
DELETE /cart/coupon                       // remove coupon

// Checkout
Post /checkout/order/create { carts } 
GET  /orders/{orderId}                 // order detail + tracking
```

- **Cart is server-authoritative**: prevents price/inventory tampering. Client caches cart for display but server re-validates on every mutation
- **Idempotency key on order placement**: critical — retrying a failed checkout must not double-charge. Server dedupes by idempotency key
- **Checkout validation step**: separate call before `POST /orders` — catches out-of-stock or price-changed items and lets UI show specific errors before charging
- **Price in cents (Long)**: all price fields are integers in smallest currency unit — avoids floating point rounding issues across client and server
- **Optimistic vs pessimistic**: add-to-cart → optimistic (update UI, rollback on failure). Place order → pessimistic (show spinner, wait for server confirmation, never optimistic)

### Phase 4: Mobile Architecture

```
UI Layer
  └─ Screens:
       - ProductListScreen (LazyVerticalGrid + search bar + filter chips)
       - ProductDetailScreen (image carousel + variant selector + "Add to Cart")
       - CartScreen (item list + quantity stepper + coupon + checkout button)
       - CheckoutScreen (address, payment, order summary, "Place Order")
       - OrderConfirmationScreen
  └─ Shared: CartBadge (item count on bottom nav / toolbar)
ViewModel
  └─ ProductListViewModel — search, filter, paginate catalog
  └─ ProductDetailViewModel — load product, select variant, add to cart
  └─ CartViewModel — cart state, update quantities, apply coupon
  └─ CheckoutViewModel — validate → place order flow
Repository
  └─ ProductRepository — catalog search + detail, caches recent products in Room
  └─ CartRepository — wraps cart API, mirrors server cart locally for offline viewing
  └─ OrderRepository — order history + placement
Data Sources
  ├─ Remote: Retrofit (all endpoints above)
  ├─ Local: Room (product cache, cart mirror, order history)
  └─ DataStore: recently viewed products, saved search queries
```

**Key Trade-offs:**

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Cart storage | Server-only | Server + local mirror | Server authoritative + local cache for offline display and fast load |
| Price validation | Trust cart snapshot | Re-validate at checkout | Always re-validate — prices/stock can change |
| Product images | Single image | Multi-image LazyRow | LazyRow + BoxWithConstraints — each image fills screen width, swipe for next angle |
| Search | Client-side filter from Room | Server-side search | Depends on catalog size: small (hundreds–low thousands) → client-side from cached Room DB works + offline; large (tens of thousands+) → server-side only |
| Add to cart | Pessimistic (button shows "Adding...") | Optimistic (instant badge update, rollback on failure) | Depends — optimistic if stock is validated on product page (99% success, rollback is rare); pessimistic if stock is uncertain or flash sale (avoid confusing rollback UX) |
| Place order | Pessimistic | Optimistic | Pessimistic — never optimistic for payments |
| Variant selection | Flat list | Matrix (color × size) | Depends on variant count — matrix for 2 dimensions, flat for 1 |

**Gotchas:**
- **Money as Long (cents)**: never use `Float`/`Double` for currency. Display formatting: `amount / 100` with locale-aware `NumberFormat.getCurrencyInstance()`
- **Cart merge on login**: guest adds items to local cart → logs in → server has a different cart. Best approach: prompt user with a dialog showing both carts, let them choose to merge, keep server cart, or keep local cart — avoids silently losing items
- **Stale prices**: user leaves product in cart for days — price/stock may change. Two layers: (1) sync cart with server periodically whenever cart is non-empty (e.g., on app foreground or every N minutes) to catch changes early; (2) final validation at checkout as a safety net. Show "price updated" or "item out of stock" banner — never silently charge a different amount
- **Inventory race condition**: two users both see "1 left in stock", both add to cart. Depends on context: (1) flash sale / bidding — aggressively poll or push stock updates via WebSocket, expose as a shared Flow consumed by product detail, catalog, cart, and pre-checkout review screens; (2) normal shopping — rely on checkout validation to catch it, show "out of stock" error + offer alternatives. Don't over-engineer for the common case
- **Idempotent order placement**: network times out after "Place Order" tap — user taps again. Create a draft Order with server-generated ID at the preview/review step — checkout just confirms that order ID, naturally idempotent. Avoid client-generated UUIDs for idempotency — only the server can guarantee uniqueness
- **Deep linking**: `app://product/{id}` — must handle product not found (deleted, region-locked). Show error state, not crash
- **Image carousel performance**: preload adjacent images, show placeholder with correct aspect ratio to prevent layout shift
- **Checkout flow as linear stepper**: address → payment → review → confirm. Use navigation with backstack so user can go back and edit. Disable "Place Order" until all steps valid. Once "Place Order" is tapped: disable the button immediately (prevent double-tap), show a non-dismissable loading overlay that blocks back button, navigation, and all touch input until server responds — the payment processing state must be uninterruptible

---

## 7. Maps / Location
Ride-sharing, store locator, delivery tracking

### Phase 1: Requirement Clarification
### Phase 2: Data Model
### Phase 3: API Design
### Phase 4: Mobile Architecture

---

## 8. Offline-first / Sync
Notes app, to-do list, document editor

### Phase 1: Requirement Clarification

**Functional Requirements — Ask About:**
- **Content types**: plain text only? Rich text / markdown? Checklists? Attachments (images, files)?
- **Organization**: folders/notebooks? Tags/labels? Pinning/favorites? Archive/trash with restore?
- **Editing**: single user or collaborative (real-time co-editing)? Version history / undo? Auto-save frequency?
- **Sharing**: share a note/list with another user? Read-only vs editable? Share via link?
- **Search**: full-text search across notes? Search within a note?

**Non-Functional Requirements — Ask About:**
- **Offline**: full CRUD offline? How long can user stay offline (hours, days, weeks)?
- **Sync**: what happens when the same note is edited on two devices? Conflict resolution strategy?
- **Latency**: local-first means UI is always instant — but how fast should sync be? (<1s on good network?)
- **Data volume**: how many notes/items per user? Max size per note? Affects sync strategy (full vs delta)
- **Durability**: can user tolerate data loss? (No — notes are user-created data, not replaceable like a feed cache)
- **Multi-device**: phone + tablet + web? All must converge to the same state
- **Storage**: local DB size limit? Sync entire corpus or on-demand?

**Scope Boundaries — Ask These:**
- Real-time collaboration (Google Docs-style) in scope, or single-user multi-device sync?
- Rich text editor in scope, or plain text / markdown?
- Attachment sync (images/files) in scope, or text only?
- Share/permissions in scope?
- Version history / undo in scope?
- Web client in scope, or mobile only?

**Things Interviewers Leave Ambiguous on Purpose:**
- What happens when the same note is edited offline on two devices (conflict resolution)
- Whether sync is push-based (WebSocket) or pull-based (polling/on-demand)
- Whether deleted items sync (soft delete vs hard delete propagation)
- How large the local DB can grow and when to evict

### Phase 2: Data Model

**Screens:** Note List (shows notes + folders in current folder) → Note Detail (block-based content)

```
// Content block — a note is a List<NoteContent> (block-based, like Notion)
// Each block is one of: text, checklist, or attachment
// Sealed type — stored as JSON array in Room via TypeConverter
sealed interface NoteContent {

  data class Text(
    val words: String,                // plain text or markdown
  ) : NoteContent

  data class TodoList(
    val items: List<TodoItem>,        // checklist
  ) : NoteContent

  data class Attachment(
    val url: String,                  // remote URL (or local URI while uploading)
    val mimeType: String,             // image/jpeg, application/pdf, etc.
  ) : NoteContent
}

TodoItem {
  id: String                          // unique within the list — for reordering
  text: String
  isCompleted: Boolean
  position: Int                       // ordering (or fractional indexing)
}

// A note item is either a Note or a Folder — sealed type
// Both stored in same Room table
// Navigation: tap folder → get childrenIds from FolderType → SELECT * WHERE id IN (:childrenIds)
sealed interface NoteItem {

  data class Note(
    val localId: String,              // client-generated, always present
    val remoteId: String?,            // server-assigned, null until first sync
    val parentId: String,             // which folder contains this note
    val title: String,
    val content: List<NoteContent>,   // ordered list of content blocks
    val createdAt: Instant,
    val updatedAt: Instant,           // bumped on every edit — drives sync
    val syncStatus: SyncStatus,       // local-only, not sent to server
  ) : NoteItem

  data class Folder(
    val localId: String,
    val remoteId: String?,
    val parentId: String?,            // null = root folder
    val title: String,
    val childrenIds: List<String>,    // IDs of direct children — no duplicate data
    val createdAt: Instant,
    val updatedAt: Instant,
    val syncStatus: SyncStatus,
  ) : NoteItem
}

// Type-safe payload — sealed type, not nullable raw JSON
// Each NoteItem has exactly one: either content blocks (Note) or child IDs (Folder)
sealed interface NoteItemType {
  data class NoteType(
    val content: List<NoteContent>,   // ordered content blocks
  ) : NoteItemType

  data class FolderType(
    val childrenIds: List<String>,    // IDs only — no duplicate data
  ) : NoteItemType
}

SyncStatus: SYNCED, PENDING, DELETED
  SYNCED   → matches server, no action needed
  PENDING  → created or edited locally, not yet pushed to server
  DELETED  → soft-deleted locally, needs to sync deletion to server
             (after server confirms, hard-delete from Room)
```

**Block-based content (List\<NoteContent\>):**
- A note body is an **ordered list of blocks**, not a single string
- User can mix text, checklists, and attachments in one note (like Notion, Apple Notes)
- Example: `[Text("Meeting notes..."), TodoList([...]), Attachment("photo.jpg")]`
- Stored as a JSON array in a single Room column — always loaded/saved together as one unit

**Why sealed types (not flat fields with nulls):**
- `NoteContent` — each block is exactly one type, not a bag of nullable fields. No `words: String?, todos: List<Todo>?, url: String?` where you have to guess which is populated
- `NoteItem` — a Folder has `children`, a Note has `content`. No `content: List<NoteContent>?` that's null for folders

**Navigation — `childrenIds` drives folder traversal:**
- Tap a folder → read its `childrenIds` → `SELECT * FROM NoteItemEntity WHERE id IN (:childrenIds)`
- The list order in `childrenIds` preserves **display ordering**
- Tap a note → display its content blocks
- Reverse lookup (which folder is this note in?) → `parentId` or navigation backstack
- Moving a note = update note's `parentId` + remove from old folder's `childrenIds` + add to new folder's `childrenIds`

**Room entity — mirrors NoteItem structure:**
```
@Entity
data class NoteItemEntity(
  @PrimaryKey val localId: String,     // client-generated, always present
  val remoteId: String?,               // server-assigned, null until first sync
  val parentId: String?,               // which folder (null = root)
  val title: String,
  val typePayload: String,             // JSON of NoteItemType (sealed type)
                                       // NoteType → {"type":"note","content":[...]}
                                       // FolderType → {"type":"folder","childrenIds":["n1","n2"]}
  val createdAt: Long,
  val updatedAt: Long,
  val syncStatus: String,              // "SYNCED", "PENDING", "DELETED"
)
// TypeConverter: typePayload ↔ NoteItemType
// PrimaryKey is localId — stable, never changes
// remoteId filled in after first sync with server
```

**TypeConverter example — kotlinx.serialization polymorphism:**
```kotlin
// @SerialName sets the "type" discriminator in JSON
// encodeToString → {"type":"note","content":[...]}
// decodeFromString → reads "type" field → deserializes correct subtype

class NoteItemTypeConverter {
    private val json = Json { ignoreUnknownKeys = true }

    @TypeConverter
    fun fromNoteItemType(type: NoteItemType): String =
        json.encodeToString(type)

    @TypeConverter
    fun toNoteItemType(value: String): NoteItemType =
        json.decodeFromString(value)
}

// Register on Database
@TypeConverters(NoteItemTypeConverter::class)
@Database(entities = [NoteItemEntity::class], version = 1)
abstract class AppDatabase : RoomDatabase() { ... }

// What Room stores in the typePayload column:
// NoteType  → {"type":"note","content":[{"type":"text","words":"Hello"},{"type":"todo_list","items":[...]}]}
// FolderType → {"type":"folder","childrenIds":["n1","n2","f3"]}
//
// Nested polymorphism works automatically — NoteContent is also a sealed interface
// with @SerialName on each subtype (Text, TodoList, Attachment)
```

**Why `NoteItemType` instead of two nullable columns:**
- No nullable `contentJson: String?` / `childrenIds: String?` where you guess which is populated
- Type-safe: deserialize `typePayload` → get either `NoteType` or `FolderType`, never an invalid state
- Single column, single TypeConverter — simpler schema

**Why `childrenIds: List<String>` (IDs, not full objects):**
- Stores only IDs (`["n1", "n2", "f3"]`), not full objects — no duplicate data in Room
- List order = display order — no separate `position` column needed
- No `parentId` needed — folder membership is expressed by which folder's `childrenIds` contains the ID
- Reverse lookup via navigation backstack, not DB query

> **Key**: Room is the **source of truth**, not the server. UI always reads from Room. Server is the sync target, not the authority. This is the fundamental difference from feed/search (where server is source of truth and Room is just a cache).

**ID generation — two approaches:**

| | Client-generated UUID | Server-assigned ID |
|--|----------------------|-------------------|
| **How** | Client generates UUID locally on create | `POST /create { type }` → server returns unique ID |
| **Offline create** | Works immediately — UUID is the permanent ID | Create with local ID, get remote ID after sync |
| **Collision risk** | UUID v4: ~1 in 2¹²² (practically zero, not guaranteed) | Zero — server guarantees uniqueness |
| **ID format** | Long UUIDs (`"550e8400-e29b-41d4-a716-446655440000"`) | Can be shorter, sequential, human-readable (`"n_12345"`) |

**Server-assigned ID — benefits:**
- **Guaranteed uniqueness** — hard guarantee, not probabilistic. Matters for systems where collision is catastrophic (financial, medical)
- **Shorter, sequential IDs** — `n_12345` vs UUID. Sequential IDs are B-tree friendly (less page splitting), easier to debug/log
- **Server controls the namespace** — can enforce format, embed metadata (e.g., shard key), use for ordering. Useful at massive scale

**Server-assigned ID — ID pair approach (`localId` + `remoteId`):**
- `localId` (PrimaryKey) is stable — never changes, all local references use it
- `remoteId` is null until first sync, then filled with server-assigned ID
- Server communicates using `remoteId`, client uses `localId` internally
- ID swap on first sync is scoped:
  - Note gets remote ID → update Note entity + parent Folder's `childrenIds`  (2 writes)
  - Folder gets remote ID → update Folder entity + parent's `childrenIds` + children's `parentId` (3 writes)
- Common pattern — many production apps keep both IDs (e.g., local DB key vs server key)

### Phase 3: API Design

**Two-tier sync: foreground (scoped) + background (full)**

```
// ═══════════════════════════════════════════════════
// PUSH — always the same, regardless of foreground/background
// Query Room: SELECT * WHERE syncStatus IN ('PENDING', 'DELETED')
// ═══════════════════════════════════════════════════

POST /sync/push
Body: {
  items: List<NoteItem>,               // all PENDING + DELETED items
}
Response: {
  accepted: List<IdMapping>,           // server confirms + assigns remoteId for new items
}
// Server always accepts — last write wins
// Conflicts are rare in a single-user notes app (multiple devices, rarely same note)
// 200 → client updates syncStatus = SYNCED in Room
// If conflicts become a real problem later, add ConflictResult to response

IdMapping {
  localId: String                      // what client sent
  remoteId: String                     // server-assigned (new for first sync, existing for updates)
}

// ═══════════════════════════════════════════════════
// FOREGROUND PULL — scoped to what user sees on screen
// One endpoint, server returns the right shape based on item type
// ═══════════════════════════════════════════════════

GET /sync/item/{remoteId}
Response: {
  item: NoteItem,                      // the item itself (folder metadata or note content)
  children: List<NoteItem>?,           // non-null only if item is a Folder
                                       // → direct children's latest versions (eager load)
}
// Note  → item = note content, children = null
// Folder → item = folder metadata (title, updatedAt, etc.) + children = direct children

// ═══════════════════════════════════════════════════
// BACKGROUND PULL — full database sync (WorkManager, every few days)
// Client sends everything it has, server diffs and returns what's different
// ═══════════════════════════════════════════════════

POST /sync/pull
Body: {
  items: List<ItemSnapshot>,           // all SYNCED items in Room
}
Response: {
  items: List<NoteItem>,               // new + updated items only
  deletedIds: List<String>,            // remoteIds client has but server doesn't
}

ItemSnapshot {
  remoteId: String
  updatedAt: Instant
}
// Server compares client's list against its own:
//   - Server has newer version → include in "items"
//   - Server has item client doesn't → include in "items"
//   - Client has item server doesn't → include in "deletedIds"
//   - Same version → skip
// Client processing:
//   1. @Insert(onConflict = REPLACE) for items — handles new + updated
//   2. DELETE FROM NoteItemEntity WHERE remoteId IN (:deletedIds)

// ═══════════════════════════════════════════════════
// FRESH INSTALL — no separate API needed
// ═══════════════════════════════════════════════════
// Foreground: GET /sync/item/{rootId} → user sees root folder immediately
// Background: POST /sync/pull with empty list → server returns everything as "new"
// User starts using the app right away while background sync fills in the rest

// CREATE — online mode, get server-assigned remoteId immediately
POST /items { type: "NOTE" | "FOLDER", ... }
Response: { remoteId: String, ... }
```

**Why two-tier sync:**
- Foreground sync must be fast — user is waiting. Scoped GETs are lightweight
- Background sync can be heavy — user doesn't notice. Full diff catches everything
- Push is always full (all pending items) — never delay pushing local changes

**Conflict Resolution Strategies:**

| Strategy | How | Pros | Cons | When |
|----------|-----|------|------|------|
| **Last-write-wins (LWW)** | Higher `updatedAt` wins | Simplest, no user intervention | Silently drops the losing edit | Low-value data, settings, status fields |
| **Server-wins** | Always keep server version | Simple, consistent | User loses offline work | Admin-controlled data |
| **Client-wins** | Always keep client version | User never loses work | Other devices' edits lost | Single-user apps |
| **Manual merge** | Show both versions, user picks | No data loss, user decides | Interrupts workflow, confusing UX | Notes, documents — user-created content |
| **Field-level merge** | Merge non-conflicting fields | Preserves most changes automatically | Complex, still needs fallback for same-field conflicts | Structured data (to-do items, forms) |
| **CRDT / OT** | Operational transform / conflict-free replicated data types | True real-time collaboration, no conflicts | Very complex, large metadata overhead | Google Docs, Figma — real-time co-editing |

> **Recommendation for notes app**: **Field-level merge with manual fallback**. If different fields changed (e.g., Device A changed title, Device B changed content), auto-merge. If same field changed, show both versions and let user pick. This covers 90% of cases automatically and only interrupts user for true conflicts.

**Deep Dive: OT vs CRDT — Real-time Collaborative Editing**

These are solutions for the hardest version of the conflict problem: **two users editing the same document at the same time, character by character**. Field-level merge can't help here — both users are editing the same field (`content`).

**Operational Transform (OT) — Google Docs approach:**

Core idea: represent every edit as an **operation** (insert/delete at position), and **transform** concurrent operations so they produce the correct result regardless of arrival order.

```
Starting document: "ABCD"

User A (at position 1): insert "X"  →  Operation: Insert('X', pos=1)
User B (at position 3): delete 1    →  Operation: Delete(pos=3, count=1)

Without OT — apply naively:
  Server gets A first:  "ABCD" → "AXBCD"
  Then applies B's Delete(pos=3): "AXBCD" → "AXBD"  ← deleted 'C', correct ✓

  But if Server gets B first: "ABCD" → "ABC"
  Then applies A's Insert(pos=1): "ABC" → "AXBC"    ← wrong! 'D' was deleted, not 'C'

With OT — transform B's operation against A's:
  A inserted at pos=1, which shifts everything after pos=1 right by 1
  B's Delete(pos=3) must become Delete(pos=4) to account for the shift

  Server applies A: "ABCD" → "AXBCD"
  Server applies transformed B: Delete(pos=4) → "AXBD" ✓

  Both orderings now produce the same result.
```

**How OT works in practice:**
```
1. User types → generate operation (insert/delete with position + content)
2. Send operation to server immediately (low latency feel)
3. Server is the SINGLE AUTHORITY — processes ops in arrival order
4. For each incoming op, server transforms it against all ops
   that arrived since the client last synced
5. Server broadcasts transformed op to all other clients
6. Clients apply transformed ops to their local document

Key property: server is centralized coordinator
  → single point of serialization
  → operations are ordered by the server
  → clients converge because they all apply the same transformed sequence
```

**OT trade-offs:**
- **Pro**: battle-tested (Google Docs since 2010), well understood
- **Pro**: server can enforce access control, validation per operation
- **Con**: requires a central server — can't work peer-to-peer or offline for extended periods
- **Con**: transform functions are complex — N operation types need N×N transform pairs
- **Con**: server is a bottleneck and single point of failure

---

**CRDT (Conflict-free Replicated Data Type) — Figma approach:**

Core idea: design the data structure itself so that **any two replicas can merge automatically without conflicts**, regardless of operation order. No central server needed.

```
Starting document: "ABCD"
Each character has a unique ID (fractional position between neighbors):

  A(0.1)  B(0.3)  C(0.5)  D(0.7)

User A: insert "X" between A and B
  → X gets ID 0.2 (between 0.1 and 0.3)
  → A(0.1) X(0.2) B(0.3) C(0.5) D(0.7)

User B: delete C
  → Mark C(0.5) as tombstone (deleted but ID preserved)
  → A(0.1) B(0.3) [C(0.5)🪦] D(0.7)

Merge — just combine both sets of changes:
  A(0.1) X(0.2) B(0.3) [C(0.5)🪦] D(0.7)
  → Display: "AXBD" ✓

No transformation needed! The fractional IDs ensure characters
always sort into the correct order regardless of merge order.
```

**How CRDTs work in practice (for text):**
```
Two main CRDT text algorithms:

1. RGA (Replicated Growable Array) — used by Yjs, Automerge
   - Each character: { id: (timestamp, replicaId), value: char, tombstone: bool }
   - Insert: generate ID between neighbors using timestamp + replica ID
   - Delete: mark as tombstone (never physically remove — needed for merge)
   - Merge: union of all operations, sorted by ID → deterministic order
   - Tie-breaking: if two inserts land at same position,
     use (timestamp, replicaId) as tiebreaker → total order

2. Tree-based (e.g., Fugue, LSEQ)
   - Characters form a tree structure
   - Position is a path in the tree (avoids interleaving issues of flat lists)
   - More complex but better worst-case behavior

Both guarantee: any two replicas that have seen the same set of
operations will converge to the same document state — eventually consistent.
```

**CRDT sync flow:**
```
1. User types → operation applied to local CRDT immediately (instant)
2. Operation stored in local log
3. When online:
   a. Push local operations to server (or directly to peers)
   b. Pull remote operations from server (or peers)
   c. CRDT merge is automatic — just apply received ops to local state
   d. No conflicts possible by design — CRDT guarantees convergence
4. When offline:
   a. All edits applied locally, operations queue up
   b. On reconnect: push queued ops, pull missed ops, auto-merge
   c. No conflict resolution UI needed — it just works

Server role (optional):
  - Relay: forwards operations between clients
  - Persistence: stores operation log / document snapshots
  - NOT an authority — any replica can independently merge
```

**CRDT trade-offs:**
- **Pro**: works offline indefinitely — merge is automatic on reconnect
- **Pro**: no central server required (peer-to-peer possible)
- **Pro**: no conflict resolution UI — every merge produces a valid result
- **Con**: metadata overhead — each character carries a unique ID + tombstone flag. A 10KB document might need 50-100KB of CRDT metadata
- **Con**: tombstone accumulation — deleted characters are never truly removed (needed for merge correctness). Requires periodic garbage collection ("compaction") when all replicas have seen the deletion
- **Con**: merge result may be semantically wrong even if structurally correct — e.g., User A writes "meet at 3pm", User B changes to "meet at 5pm", CRDT might merge to "meet at 35pm" (interleaved characters). Structurally valid, semantically nonsense. For prose this is rare; for structured data it matters more

**OT vs CRDT comparison:**

| | OT | CRDT |
|--|----|----|
| **Central server** | Required (coordinator) | Optional (relay only) |
| **Offline editing** | Limited — needs server to transform | Full — merge works without server |
| **Merge correctness** | Guaranteed by server ordering | Guaranteed by data structure design |
| **Metadata overhead** | Low (operations are small) | High (every element has unique ID) |
| **Complexity** | Transform functions (N×N pairs) | Data structure design + garbage collection |
| **Latency** | Depends on server round-trip | Instant local, eventual remote |
| **Battle-tested** | Google Docs (2010+) | Figma, Linear, Apple Notes (2020s+) |
| **Scaling** | Server bottleneck | Scales with peers |
| **Semantic correctness** | Server can validate | May produce valid-but-nonsensical merges |

**When to use which:**
- **OT**: real-time collaboration where server is always available and you need strong consistency (Google Docs, Notion)
- **CRDT**: offline-first apps where users may edit without connectivity and merge later (Figma, Linear, Apple Notes, local-first apps)
- **Neither** (field-level merge): simpler apps where concurrent same-field edits are rare (to-do apps, simple notes) — the engineering cost of OT/CRDT is not justified

**Libraries (if asked):**
- **Yjs**: popular CRDT library (JS/WASM, bindings for other languages). Powers many collaborative editors
- **Automerge**: Rust + WASM CRDT library, focus on local-first
- **Diamond Types**: high-performance CRDT in Rust
- **ShareDB**: OT library for Node.js (used with JSON documents)
- For Android/Kotlin: typically use Yjs or Automerge via a shared Rust/WASM core, or implement a simpler CRDT for structured data (not text)

> **Interview tip**: for a notes/to-do app interview, mention CRDTs exist and explain the concept, but recommend field-level merge for the scope of the problem. CRDTs are overkill unless the interviewer explicitly asks for real-time collaborative editing (Google Docs-style). Knowing *when not* to use a technology is a senior signal.

### Phase 4: Mobile Architecture

```
UI Layer
  ├─ NoteListScreen: LazyColumn + folder navigation
  │    └─ Observes Room Flow → auto-updates on sync or local edits
  ├─ NoteDetailScreen: TextField (auto-save on change)
  │    └─ Observes Room Flow → auto-updates on sync
  │    └─ Debounce saves (e.g., 1-2s after last keystroke)
ViewModel
  └─ NoteListViewModel — observes NoteFlowUseCase, triggers foreground sync
  └─ NoteDetailViewModel — uses NoteEditUseCase for debounced saves
Domain (UseCases)
  └─ NoteFlowUseCase — Flow from Room, UI observes this
  └─ NoteEditUseCase — debounce saves to Room, sets syncStatus = PENDING
  └─ DeleteUseCase — delete a note or folder (set syncStatus = DELETED)
Repository
  └─ NoteRepository — all reads from Room, writes to Room
  └─ SyncRepository — push (POST /sync/push) + pull (GET + POST /sync/pull)
Data Sources
  ├─ Remote: Retrofit (sync endpoints)
  └─ Local: Room (SOURCE OF TRUTH)
Background
  └─ WorkManager: periodic full sync (every few days)
  └─ WorkManager: push PENDING/DELETED items on network restored
```

**Data flow — Editing a note:**
```
1. User types in TextField
2. NoteEditUseCase debounces (1-2s after last keystroke) → save to Room
   Room write: update content + updatedAt + syncStatus = PENDING
3. UI updates immediately (NoteFlowUseCase observes Room Flow)
4. Fire a Worker / POST request to push to server
   - Online → server accepts → mark SYNCED, store remoteId if new
   - Offline → stays PENDING → Worker retries when network restored
```

**Data flow — Foreground sync:**
```
1. Screen starts (NoteList or NoteDetail)
2. NoteFlowUseCase delivers local cache from Room immediately → UI renders
3. ViewModel triggers foreground GET: /sync/item/{remoteId}
4. Response → INSERT OR REPLACE into Room
5. UI auto-updates via Room Flow
```

**Data flow — Background sync:**
```
WorkManager (periodic, every few days):
1. PUSH: SELECT * WHERE syncStatus IN ('PENDING', 'DELETED')
   → POST /sync/push → mark SYNCED
2. PULL: SELECT remoteId, updatedAt WHERE syncStatus = 'SYNCED'
   → POST /sync/pull → INSERT OR REPLACE + DELETE by deletedIds
```

**Data flow — Deleting a note:**
```
Note:
1. Set syncStatus = DELETED in Room
2. UI hides it (NoteFlowUseCase filters out DELETED)
3. POST to server → server deletes → hard-delete from Room

Folder (trickier — must delete all descendants):
1. BFS from the folder: read childrenIds → for each child folder, read its childrenIds → repeat
   Collects all direct + indirect children (notes and subfolders)
2. Set syncStatus = DELETED on the folder + ALL descendants in Room
3. UI hides them all
4. POST to server → server deletes → hard-delete from Room
```

**Key Trade-offs:**

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Source of truth | Server | Room (local) | **Room** — offline-first means local is truth, server is sync target |
| Sync protocol | Full fetch every time | Delta sync (updatedAfter) | Delta — users have many notes, full fetch is wasteful |
| Sync transport | REST polling | WebSocket push | REST for MVP — simpler; WebSocket if near-instant multi-device sync needed |
| Conflict resolution | Last-write-wins | Manual merge | Field-level merge + manual fallback — notes are user content, can't silently drop |
| Auto-save | On every keystroke | Debounced (1-2s) | Debounced — fewer Room writes, fewer sync triggers |
| Delete propagation | Hard delete | Soft delete | Soft delete — required for delta sync to discover deletions |
| Ordering (to-do items) | Integer position | Fractional indexing | Integer for simple lists; fractional if reordering is frequent and synced |
| Sync scope | All notes | On-demand (sync only when opened) | All notes if corpus is small (<1000); on-demand for large/media-heavy |
| Background sync | WorkManager periodic | None (only foreground) | WorkManager — ensures pending changes are pushed even if user forgets to open app |

**Gotchas:**
- **Clock skew**: `updatedAt` comparison relies on timestamps. Client clocks drift. Use **server-assigned timestamps** for sync comparison — client sends its changes, server stamps them with server time. Never trust client clocks for conflict detection
- **Sync loop prevention**: Device A syncs → server timestamp updates → Device B pulls → Device B now has "changed" items → pushes back to server → triggers Device A to pull... Use a `lastSyncedVersion` per item to distinguish "changed locally" from "received from sync"
- **Partial sync failure**: push succeeds for 8 of 10 items, then network drops. Must track sync status per item, not globally. Retry only the 2 failed items
- **Large initial sync**: fresh install with 5000 notes — paginate the pull, show progress, don't block UI. Let user start using the app while sync continues in background
- **Auto-save + conflict timing**: user is editing on Device A, auto-save triggers sync every 2s. Device B pulls mid-edit and sees partial content. Solution: don't sync notes that are actively being edited (debounce sync for the active note, or use a "dirty" flag that blocks sync while user is typing)
- **Process death during sync**: WorkManager handles this — work survives process death. But in-memory sync state is lost. Store sync progress in Room/DataStore so it can resume
- **Merge UI complexity**: showing two versions side-by-side is hard for long notes. Consider a diff view (highlight additions/deletions) rather than raw side-by-side
- **Offline duration**: if user is offline for weeks, delta sync may return thousands of changes. Server should detect this (e.g., >1000 changes since lastSync) and suggest a full sync instead
- **Attachment sync**: images/files are too large for the JSON sync payload. Sync metadata (URL, size) with notes, download actual files separately. Show placeholder until downloaded. Allow user to choose: sync attachments on Wi-Fi only

---

## 9. Notifications System
Two features: **Push notifications** (server-driven alerts) + **Notification center** (in-app history screen).

### Phase 1: Requirement Clarification

**Functional:**
- Push: types (social/transactional/system), deep link on tap, rich content, grouping/bundling
- Notification center: chronological list, unread/read states, badge count, mark-all-read
- Preferences: per-category toggle, per-item mute, quiet hours

**Non-functional:** reliability (must-deliver vs best-effort), offline handling, badge accuracy, dedup (FCM is at-least-once)

**Scope out:** email/SMS, scheduling, analytics

**Ambiguous traps:** badge sync across push + notification center, stale deep links (deleted content), foreground suppression (user already on screen), FCM ≠ storage (fire-and-forget)

### Phase 2: Data Model

```
// Core entity — used in both features:
//   Push: server sends this payload via FCM, client builds system notification from it
//   Notification center: stored in Room, displayed in a list
Notification {
  id: String
  type: NotificationType         // LIKE, COMMENT, FOLLOW, ORDER_UPDATE, SYSTEM, PROMO
  title: String                  // "John liked your post"
  body: String?                  // optional preview text
  imageUrl: String?              // avatar or content thumbnail
  deepLink: String               // "app://posts/123" — URI string, not typed route
  isRead: Boolean                // false on creation, true when user taps or marks as read
  createdAt: Instant
  groupKey: String?              // for OS-level bundling: "post_123_likes"
}

// Why deepLink is a URI string:
//   - Server can add new link types without client update
//   - Same scheme for push, in-app, and external deep links
//   - Client parses with a DeepLinkRouter — decoupled from nav graph

// Client state — Notification Center screen
NotificationCenterState {
  notifications: List<Notification>
  unreadCount: Int               // badge on tab
  nextCursor: String?
  isInitialLoading: Boolean
  isLoadingMore: Boolean
  error: String?
}

// Notification preferences — stored in DataStore + synced to server
NotificationPreferences {
  socialPush: Boolean            // likes, comments, follows
  messagesPush: Boolean
  ordersPush: Boolean
  promoPush: Boolean             // default OFF
  quietHoursEnabled: Boolean
  quietHoursStart: LocalTime
  quietHoursEnd: LocalTime
}

// Per-item mute — stored in Room
MutedItem {
  itemType: String               // "conversation", "post", "order"
  itemId: String
  mutedUntil: Instant?           // null = forever
}
```

### Phase 3: API Design

**Two groups of APIs for two features:**

```
// ─── NOTIFICATION CENTER (client fetches history) ───

// Fetch list — cursor-paginated, newest first
// WHY: FCM is fire-and-forget, doesn't store history.
//   Notification center needs persistent, paginated data from YOUR server.
GET /notifications?limit=20
GET /notifications?cursor={cursor}&limit=20

// Badge count — lightweight, called on app launch + foreground
GET /notifications/unread-count
→ { count: Int }

// Mark as read — user taps notification or "mark all"
PATCH /notifications/{id}/read
POST /notifications/mark-all-read

// ─── PUSH REGISTRATION (client registers device token) ───

// Register FCM token so server knows where to send pushes
POST /devices/register
Body: { token: String, platform: "ANDROID", deviceId: String }

// Unregister on logout
DELETE /devices/{deviceId}

// ─── PREFERENCES (client manages, server enforces) ───

GET /notifications/preferences
→ { preferences: NotificationPreferences }

PUT /notifications/preferences
Body: { preferences: NotificationPreferences }
```

**FCM payload — what the server sends (data-only message):**
```
{
  "to": "device_token_abc",
  "data": {                        // data-only — client controls display
    "notificationId": "n_789",
    "type": "LIKE",
    "title": "John liked your post",
    "body": "Your post about system design...",
    "imageUrl": "https://...",
    "deepLink": "app://posts/123",
    "groupKey": "post_123_likes"
  }
}
```

**Why `data` messages, not `notification` messages:**

| | `notification` message | `data` message |
|--|----------------------|----------------|
| **Foreground** | OS auto-displays (can't suppress) | `onMessageReceived()` — app decides |
| **Background** | OS auto-displays, tapping opens LAUNCHER | `onMessageReceived()` — app handles display + deep link |
| **Control** | Almost none | Full: channels, grouping, suppression, custom UI |

> Always use `data` messages. `notification` messages bypass your code when backgrounded.

### Phase 4: Mobile Architecture (Client Focus)

Three client responsibilities: **register for push**, **handle incoming notifications**, **manage preferences**.
Plus: **notification center screen** for in-app history.

---

#### 4A. FCM Registration — Token Lifecycle

```
What is the FCM token?
  - A unique string identifying THIS app installation on THIS device
  - Your server needs it to send pushes via FCM
  - NOT permanent — changes on reinstall, data clear, Google rotation

Registration flow:
  1. App launches → FirebaseMessaging.getInstance().token → get current token
  2. Send to server: POST /devices/register { token, platform, deviceId }
  3. Server stores (userId, token, deviceId) — uses token to push via FCM later

Two places you MUST handle token:

  // 1. Every app launch (Application.onCreate or after login)
  //    Why: token might have changed while app was killed
  FirebaseMessaging.getInstance().token.addOnSuccessListener { token ->
      repository.registerDevice(token)
  }

  // 2. FirebaseMessagingService.onNewToken()
  //    Why: called when token refreshes while app is alive
  class MyFirebaseService : FirebaseMessagingService() {
      override fun onNewToken(token: String) {
          // Not a coroutine scope — use CoroutineScope(IO) or WorkManager
          repository.registerDevice(token)
      }
  }

Logout cleanup:
  - DELETE /devices/{deviceId} → server stops sending to this device
  - FirebaseMessaging.getInstance().deleteToken() → invalidate locally
  - Without this: old user keeps getting pushes on this device
```

> **Gotcha**: `deviceId` (e.g., `Settings.Secure.ANDROID_ID`) prevents duplicate registrations. Without it, every app launch creates a new row. With it: `INSERT ... ON CONFLICT(deviceId) UPDATE token`.

> **Gotcha**: Multi-device — user on phone + tablet. Server stores ALL tokens, sends to ALL. Each device shows independently.

---

#### 4B. Handling Push Notifications

**Receiving and displaying:**
```kotlin
class NotificationService : FirebaseMessagingService() {

    override fun onMessageReceived(message: RemoteMessage) {
        // Called for BOTH foreground and background (data-only messages)
        // ~10 second budget before Android kills the service

        val data = message.data
        val type = data["type"]                    // "LIKE", "ORDER_UPDATE"
        val title = data["title"]                  // "John liked your post"
        val body = data["body"]
        val deepLink = data["deepLink"]            // "app://posts/123"
        val notificationId = data["notificationId"]
        val groupKey = data["groupKey"]

        // Check client-side preferences (defense in depth)
        if (!shouldShowNotification(type)) return

        // Check per-item mute
        val targetId = data["targetId"]
        if (targetId != null && mutedItemDao.isMuted(targetId)) return

        // Foreground suppression
        if (appIsInForeground()) {
            if (NavigationState.currentDeepLink == deepLink) return  // already viewing
            InAppBannerManager.show(title, body, deepLink)          // in-app banner
            return
        }

        // Background: build and show system notification
        showSystemNotification(notificationId, title, body, deepLink, groupKey, type)
    }

    private fun showSystemNotification(...) {
        val channelId = when (type) {
            "LIKE", "COMMENT", "FOLLOW" → "social"
            "MESSAGE"                   → "messages"
            "ORDER_UPDATE"              → "orders"
            "PROMO"                     → "promo"
            else                        → "general"
        }

        val intent = Intent(Intent.ACTION_VIEW, Uri.parse(deepLink)).apply {
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
        }
        val pendingIntent = PendingIntent.getActivity(
            this, notificationId.hashCode(), intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(this, channelId)
            .setSmallIcon(R.drawable.ic_notification)
            .setContentTitle(title)
            .setContentText(body)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true)
            .setGroup(groupKey)
            .build()

        // notificationId as tag prevents duplicates on FCM retry
        NotificationManagerCompat.from(this)
            .notify(notificationId, notificationId.hashCode(), notification)
    }
}
```

**Deep linking — user taps notification:**
```
Tap notification → PendingIntent fires → Activity receives intent with deepLink URI

// Main Activity:
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    intent.data?.let { DeepLinkRouter.navigate(navController, it) }
}

// Already running — new notification tap:
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    intent.data?.let { DeepLinkRouter.navigate(navController, it) }
}

object DeepLinkRouter {
    fun navigate(navController: NavController, uri: Uri) {
        when (uri.host) {
            "posts"   → navController.navigate(PostDetail(uri.lastPathSegment!!))
            "orders"  → navController.navigate(OrderDetail(uri.lastPathSegment!!))
            "chat"    → navController.navigate(ChatDetail(uri.lastPathSegment!!))
            "profile" → navController.navigate(Profile(uri.lastPathSegment!!))
            else      → navController.navigate(Home)
        }
    }
}
```

> **Gotcha — stale deep links**: content may have been deleted. Destination screen must handle "not found" gracefully.

**Notification channels (Android 8+):**
```kotlin
// Create once in Application.onCreate() — BEFORE showing any notification
fun createNotificationChannels(context: Context) {
    val manager = context.getSystemService(NotificationManager::class.java)
    listOf(
        NotificationChannel("messages", "Messages", IMPORTANCE_HIGH),      // sound + heads-up
        NotificationChannel("social", "Social", IMPORTANCE_DEFAULT),       // sound, no heads-up
        NotificationChannel("orders", "Order Updates", IMPORTANCE_DEFAULT),
        NotificationChannel("promo", "Promotions", IMPORTANCE_LOW),        // no sound
    ).forEach { manager.createNotificationChannel(it) }
}
```

> **Gotcha**: Channel importance is immutable after creation — only user can change it in system settings.

> **Gotcha (Android 13+)**: `POST_NOTIFICATIONS` runtime permission required before showing any notification.

---

#### 4C. Notification Preferences — Two-Level Control

```
Layer 1: Android system settings (OS-level — app CANNOT override)
  ├─ Per-app: user can disable ALL notifications
  ├─ Per-channel: user can mute "Promotions" in system settings
  └─ Read-only from app:
       NotificationManagerCompat.areNotificationsEnabled()
       manager.getNotificationChannel("social").importance

Layer 2: In-app preferences (app-level — stored in DataStore, synced to server)
  ├─ Per-category: "Receive push for likes?" toggle
  ├─ Per-item: "Mute this conversation" (Room table)
  └─ Quiet hours: "Don't disturb 10 PM – 8 AM"
```

**Why two layers?**
- OS-level: coarse, user controls outside your app, you can't override
- App-level: fine-grained, server enforces (doesn't send FCM → saves battery), client double-checks (defense in depth)
- Both work together: even if server sends, OS can block (channel muted). Even if OS allows, client can suppress (quiet hours, foreground)

**In-app preference sync:**
```kotlin
// DataStore for local reads + server sync
data class NotificationPreferences(
    val socialPush: Boolean = true,
    val messagesPush: Boolean = true,
    val ordersPush: Boolean = true,
    val promoPush: Boolean = false,       // default OFF for marketing
    val quietHoursEnabled: Boolean = false,
    val quietHoursStart: LocalTime = LocalTime.of(22, 0),
    val quietHoursEnd: LocalTime = LocalTime.of(8, 0),
)

// User toggles preference:
//   1. Save to DataStore (instant local effect)
//   2. PUT /notifications/preferences (server stops sending for disabled categories)
//
// On app launch:
//   GET /notifications/preferences → update DataStore
//   (handles cross-device sync: user changed preference on web)

// Client-side check in FirebaseMessagingService:
private fun shouldShowNotification(type: String): Boolean {
    val prefs = preferencesDataStore.get()
    return when (type) {
        "LIKE", "COMMENT", "FOLLOW" → prefs.socialPush
        "MESSAGE"                   → prefs.messagesPush
        "ORDER_UPDATE"              → prefs.ordersPush
        "PROMO"                     → prefs.promoPush
        else                        → true
    } && !isQuietHours(prefs)
}
```

**Per-item mute:**
```
// "Mute this conversation for 1 week"
// Local: Room table (MutedItem) for instant client-side filtering
// Remote: POST /mute → server stops sending pushes for this item

// In FirebaseMessagingService:
val targetId = data["targetId"]
if (targetId != null && mutedItemDao.isMuted(targetId)) return
```

**Guiding users to OS settings:**
```kotlin
// Problem: user muted at OS level, complains "I'm not getting notifications"
if (!NotificationManagerCompat.from(context).areNotificationsEnabled()) {
    // Show: "Notifications disabled. Tap to enable."
    startActivity(Intent(Settings.ACTION_APP_NOTIFICATION_SETTINGS).apply {
        putExtra(Settings.EXTRA_APP_PACKAGE, packageName)
    })
}

// Also detect per-channel:
val channel = manager.getNotificationChannel("messages")
if (channel.importance == NotificationManager.IMPORTANCE_NONE) {
    // "Message notifications are turned off."
}
```

**Full check chain — event to display:**
```
Server:
  1. Event happens (someone likes a post)
  2. Check app-level preferences → social push enabled? YES
  3. Check quiet hours → not quiet time? YES
  4. Check per-item mute → not muted? YES
  5. Send FCM data message to all user's device tokens

Client:
  6. FCM wakes FirebaseMessagingService.onMessageReceived()
  7. Check app-level preferences locally (defense in depth) → OK
  8. Check per-item mute locally → OK
  9. Check foreground → user not on this screen → OK
  10. Build notification on correct channel

OS:
  11. Check channel importance → not muted → SHOW
```

---

#### 4D. Notification Center Screen

```
Architecture:
  NotificationCenterScreen (LazyColumn)
    └─ observes Flow<List<Notification>> from Room
  NotificationViewModel
    ├─ notifications: StateFlow<NotificationCenterState>
    ├─ loadNotifications()     → GET /notifications → upsert into Room
    ├─ loadMore()              → GET /notifications?cursor={cursor} → upsert into Room
    ├─ markAsRead(id)          → update Room + PATCH /notifications/{id}/read
    ├─ markAllAsRead()         → update Room + POST /notifications/mark-all-read
    └─ unreadCount             → Room query: COUNT(*) WHERE isRead = false
  NotificationRepository
    ├─ Remote: Retrofit
    └─ Local: Room (source of truth for the list)
```

**Data flow — opening the screen:**
```
1. Screen opens → ViewModel starts collecting Flow<List<Notification>> from Room
2. Fetch latest: GET /notifications?limit=20 → upsert into Room
3. Room Flow emits → UI renders
4. Scroll to bottom → loadMore() → GET /notifications?cursor={cursor} → upsert into Room
```

**Data flow — push arrives while notification center is open:**
```
1. FCM delivers data message
2. FirebaseMessagingService inserts into Room
3. Room Flow emits updated list → UI adds new row at top automatically
4. No system notification shown (user is already in notification center)
```

**Data flow — user taps a notification:**
```
1. Mark as read: update Room (isRead = true) + PATCH /notifications/{id}/read
2. Navigate: DeepLinkRouter.navigate(notification.deepLink)
```

**Badge count sync:**
```
Local (fast): Room COUNT(*) WHERE isRead = false → updates badge immediately
Server (authoritative): GET /notifications/unread-count on app launch + foreground
  If server count ≠ local count → re-fetch notification list to reconcile
```

> **Why Room is source of truth**: FCM is fire-and-forget — no history. The list needs to survive process death, work offline, and paginate. Room provides all of this. Server is the authoritative source; Room is the local cache that powers the UI.

---

**Key Trade-offs:**

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| FCM payload | `notification` message | `data` message | Data — full control over display, suppression, grouping |
| Preference enforcement | Server-only | Server + client | Both — server saves battery, client is defense in depth |
| Preference storage | DataStore | Room | DataStore — simple KV, not relational data |
| Per-item mute storage | Server-only | Server + local Room | Both — local for instant filtering, server to stop sending |
| Token registration | Only `onNewToken()` | `onNewToken()` + every app launch | Both — token can change while app is killed |
| Deep link format | Typed routes | URI strings | URI strings — server-driven, decoupled from nav graph |
| Notification center data | In-memory only | Room | Room — survives process death, enables pagination + offline |
| Badge count | Local-only | Server-authoritative | Hybrid — local for speed, server sync for accuracy |

**Gotchas:**
- **FCM token refresh**: register in BOTH `onNewToken()` AND on every app launch
- **Duplicate pushes**: FCM is at-least-once. Use `notificationId` as notification tag
- **POST_NOTIFICATIONS (Android 13+)**: must request at runtime. If denied, notification center still works
- **Channel importance immutable**: only user can change after creation
- **10-second limit**: `onMessageReceived()` budget — don't do heavy work
- **Stale deep links**: handle "not found" gracefully
- **Logout cleanup**: `DELETE /devices/{deviceId}` + `deleteToken()` — or old user gets pushes

---

## 10. Auth / Onboarding
Login flows, OAuth, session management, multi-step signup

### Phase 1: Requirement Clarification
### Phase 2: Data Model
### Phase 3: API Design
### Phase 4: Mobile Architecture

---

## 11. Settings / Profile
Preferences, theme, account management

### Phase 1: Requirement Clarification
### Phase 2: Data Model
### Phase 3: API Design
### Phase 4: Mobile Architecture

---

## 12. File Download / Upload
Large file handling, resume, progress tracking

### Phase 1: Requirement Clarification

**Functional Requirements — Ask About:**
- **File types**: what can users upload/download? Images, videos, PDFs, docs, any arbitrary file? Max file size?
- **Upload flow**: single file or batch/multi-file? Pick from gallery, camera, file system? Drag-and-drop (tablet)?
- **Download flow**: tap to download? Auto-download? Download to where (app-internal, Downloads folder, share sheet)?
- **Progress**: show per-file progress bar? Overall batch progress? Speed/ETA display?
- **Resume**: can interrupted transfers resume from where they left off? Or restart from scratch?
- **Background**: do transfers continue when app is backgrounded? When device is locked?
- **Management UI**: list of active/completed/failed transfers? Cancel, pause, retry individual transfers?
- **Preview**: preview files before downloading (thumbnails, file metadata)?

**Non-Functional Requirements — Ask About:**
- **File size limits**: max upload size? (affects chunking strategy — files >100MB usually need chunking)
- **Concurrency**: how many simultaneous uploads/downloads? (battery vs speed trade-off)
- **Network**: Wi-Fi only option? Pause on metered network? Handle network transitions (Wi-Fi → cellular)?
- **Reliability**: must survive process death? App restart? Device reboot?
- **Storage**: check available disk space before download? Handle storage full mid-transfer?
- **Security**: files encrypted at rest? In transit (TLS)? Signed URLs with expiration?
- **Bandwidth**: compress before upload? Thumbnail generation client-side or server-side?
- **Analytics**: track upload/download success rates, speed, failure reasons?

**Scope Boundaries — Ask These:**
- Is the server storage layer (S3, GCS) in scope, or just the mobile client?
- Encryption at rest (client-side encryption before upload)?
- File versioning (upload a new version of an existing file)?
- Sharing / link generation?
- Cloud sync (auto-upload camera roll, like Google Photos)?
- Conflict handling (same file uploaded from two devices)?

**Things Interviewers Leave Ambiguous on Purpose:**
- Whether to use chunked upload or single PUT (depends on file size)
- How to handle interrupted uploads — resume from last chunk vs restart
- What happens when user kills the app mid-transfer
- Whether to use WorkManager vs foreground service for background transfers
- How to display progress for chunked uploads (per-chunk vs byte-level)

### Phase 2: Data Model

**Screens:** Transfer List (shows active/completed/failed) → File Browser (pick files to upload, view downloadable files)

```
// Core entity — represents a single file transfer (upload or download)
FileDownload {
  fileId: String                    // server generated file id
  fileName: String                  
  mimeType: String                  // image/jpeg, application/pdf, etc.
  status: TransferStatus
}

FileUpload {
  fileId: String                    // server generated file id
  fileName: String                  
  mimeType: String                  // image/jpeg, application/pdf, etc.
  status: TransferStatus
  Chunks: List<ChunkState>          // for mutipart upload 
}

enum TransferStatus {
  QUEUED,       // waiting to start (concurrency limit reached)
  IN_PROGRESS,  // actively transferring
  COMPLETED,    // successfully finished
  FAILED,       // error — retryable
  CANCELLED,    // user cancelled — not retryable (unless explicitly retried)
}

// For chunked uploads — tracks per-chunk progress
ChunkState {
  transferId: String
  chunkIndex: Int                   // 0-based
  offset: Long                      // byte offset in the file
  status: ChunkStatus               // PENDING, UPLOADED, FAILED
  etag: String?                     // server-returned ETag for this chunk (used for multipart complete)
}

enum ChunkStatus { PENDING, UPLOADED, FAILED }

// File metadata — what the server knows about an uploaded file
FileMetadata {
  id: String                        // server-assigned file ID
  name: String
  size: Long
  mimeType: String
  downloadUrl: String               // pre-signed download URL (may expire)
  thumbnailUrl: String?             // server-generated thumbnail for images/videos
}
```

**Chunked upload vs single PUT — when to use which:**
- **Single PUT**: files <10MB — simpler, one HTTP request, no chunk tracking
- **Chunked (multipart) upload**: files >10MB — resume support, parallel chunk uploads, survives interruptions
- Threshold is configurable — 5-10MB is typical. Some services (S3) require multipart for files >5GB

**Pre-signed URLs — why:**
- Client uploads directly to cloud storage (S3/GCS), not through your backend
- Backend generates pre-signed URL with embedded auth + expiration
- Saves backend bandwidth and CPU — it's just a signing operation, not a proxy
- Same pattern for downloads — backend returns a pre-signed download URL

### Phase 3: API Design

```
// ═══════════════════════════════════════════════════
// SMALL FILE UPLOAD (< 10MB) — single PUT
// ═══════════════════════════════════════════════════

// Step 1: Register file with backend (get fileId, NO pre-signed URL yet)
POST /files/register
Body: { fileName: "photo.jpg", fileSize: 2048000, mimeType: "image/jpeg" }
Response: {
  fileId: "f_123",                  // server-assigned, stored in Room immediately
}
// Why separate from URL generation: fileId is durable (never expires),
// pre-signed URL is ephemeral (expires in 15min-1hr).
// Register at enqueue time, get URL right before actual upload.

// Step 2: Get pre-signed URL (called by Worker right before upload starts)
GET /files/{fileId}/upload-url
Response: {
  uploadUrl: "https://s3.amazonaws.com/bucket/f_123?X-Amz-Signature=...",
  expiresAt: "2026-03-28T12:15:00Z"   // typically 15min-1hr
}
// Called INSIDE the Worker, not at enqueue time.
// If Worker is delayed (queued, no network, process killed), URL is fresh when it actually runs.
// On retry after failure, Worker calls this again → always gets a fresh URL.

// Step 3: Upload file directly to cloud storage
PUT {uploadUrl}
Headers: Content-Type: image/jpeg, Content-Length: 2048000
Body: <raw file bytes>
Response: 200 OK, ETag: "abc123"

// Step 4: Confirm upload with backend (so it records the file in DB)
POST /files/{fileId}/confirm
Body: { etag: "abc123" }
Response: { file: FileMetadata }

// ═══════════════════════════════════════════════════
// LARGE FILE UPLOAD (> 10MB) — chunked / multipart
// ═══════════════════════════════════════════════════

// Step 1: Initiate multipart upload (get fileId + uploadId, NO pre-signed URLs yet)
POST /files/multipart/init
Body: { fileName: "video.mp4", fileSize: 524288000, mimeType: "video/mp4" }
Response: {
  fileId: "f_456",
  uploadId: "mpu_789",               // S3 multipart upload ID
  chunkSize: 5242880,                 // 5MB per chunk
  totalChunks: 100,                   // client creates ChunkState entries in Room
}
// No pre-signed URLs returned here — they'd expire before all chunks upload.
// URLs are fetched per-chunk, right before each upload.

// Step 2: Create pre-signed URL for a specific chunk (called right before uploading that chunk)
POST /files/multipart/{fileId}/chunk-url
Body: { uploadId: "mpu_789", chunkIndex: 5 }
Response: {
  url: "https://s3.../f_456?partNumber=6&uploadId=mpu_789&X-Amz-Signature=...",
  expiresAt: "2026-03-28T12:15:00Z"
}
// Called by Worker right before each chunk upload → URL is always fresh.
// On resume after interruption, only fetches URLs for PENDING chunks.
// Can also batch: POST with chunkIndexes: [5, 6, 7] to get multiple URLs
// for parallel upload — but only for the next 2-3 chunks, not all at once.

// Step 3: Upload each chunk (can be parallel, 2-3 concurrent)
PUT {chunk.url}
Headers: Content-Length: 5242880
Body: <chunk bytes>
Response: 200 OK, ETag: "chunk_etag_0"
// Client stores the ETag per chunk in Room — needed for completion

// Step 4: Complete multipart upload
POST /files/multipart/complete
Body: {
  fileId: "f_456",
  uploadId: "mpu_789",
  parts: [
    { index: 0, etag: "chunk_etag_0" },
    { index: 1, etag: "chunk_etag_1" },
    // ...
  ]
}
Response: { file: FileMetadata }

// Resume after interruption:
// Worker restarts → query Room for PENDING chunks → fetch fresh URLs → upload
// No "refresh-urls" endpoint needed — same chunk-url endpoint works for initial + retry

// Abort (user cancelled or too many failures):
POST /files/multipart/abort
Body: { fileId: "f_456", uploadId: "mpu_789" }
// Server tells S3 to discard uploaded chunks — prevents orphaned storage costs

// ═══════════════════════════════════════════════════
// DOWNLOAD
// ═══════════════════════════════════════════════════

// Step 1: Get download URL
GET /files/{fileId}/download-url
Response: {
  url: "https://s3.amazonaws.com/bucket/f_123?X-Amz-Signature=...",
  expiresAt: "2026-03-28T12:00:00Z",
  fileSize: 524288000,
  fileName: "video.mp4",
  mimeType: "video/mp4",
}

// Step 2: Download with range support (for resume)
GET {url}
Headers: Range: bytes=10485760-    // resume from byte 10MB
Response: 206 Partial Content
  Content-Range: bytes 10485760-524287999/524288000
  Body: <remaining bytes>

// If server doesn't support Range → 200 with full file → restart from scratch

// ═══════════════════════════════════════════════════
// FILE LISTING
// ═══════════════════════════════════════════════════

GET /files?limit=20&cursor={cursor}
Response: {
  files: List<FileMetadata>,
  nextCursor: String?
}
```

**Pre-signed URL flow — why not proxy through backend:**
```
Option A: Proxy through backend
  Client → Backend → S3
  ❌ Backend becomes a bottleneck: receives full file, then re-uploads to S3
  ❌ Double bandwidth cost (client→backend + backend→S3)
  ❌ Backend must handle large request bodies (memory/disk pressure)
  ✅ Simpler client — just one endpoint

Option B: Pre-signed URLs (recommended)
  Client → Backend (get signed URL) → S3 (direct upload)
  ✅ Backend is lightweight — just signs URLs, no file bytes pass through
  ✅ Leverages S3/GCS edge infrastructure (CDN, multi-region)
  ✅ Client gets full upload/download speed directly to storage
  ❌ More complex client (two-step flow: get URL, then transfer)
  ❌ URL expiration — must handle refresh

→ Always use pre-signed URLs for production. Proxy is acceptable only for
  very small files or when you need server-side processing before storage.
```

**HTTP Range requests — how download resume works:**
```
First download attempt (interrupted at byte 10MB of 500MB file):
  GET https://s3.../file.mp4
  Response: 200 OK, Content-Length: 524288000
  Client writes bytes 0-10485759 to disk → network drops

Resume:
  GET https://s3.../file.mp4
  Range: bytes=10485760-
  Response: 206 Partial Content
  Content-Range: bytes 10485760-524287999/524288000
  Content-Length: 513802240
  Client APPENDS bytes to existing file starting at offset 10485760

How client knows where to resume:
  Check local file size → that's the offset
  Or track bytesTransferred in Room (survives process death)
```

**Resume behavior after pause/interruption:**
```
Single PUT (small file):
  ❌ No partial resume — S3/GCS PUT is atomic (all-or-nothing)
  → Restart upload from scratch on retry
  → Acceptable because file is small (<10MB)

Chunked multipart (large file):
  ✅ True resume — each chunk is an independent PUT
  → Room tracks per-chunk status (PENDING / UPLOADED)
  → On resume: query Room for PENDING chunks → upload only those
  → Already-UPLOADED chunks are skipped (ETags stored in Room)
  → Example: 500MB file, 100 chunks, paused at chunk 60
    Resume: skip chunks 0-59, upload chunks 60-99

This is the primary reason chunked upload exists — not just for large files,
but for RESUMABLE uploads. If resume is critical even for smaller files,
lower the chunking threshold (e.g., chunk anything >1MB).
```

**Idempotency for uploads:**
- Each chunk PUT to S3 is idempotent — re-uploading the same chunk with same content overwrites identically
- The `confirm` / `complete` step uses `uploadId` + `fileId` — server can deduplicate
- Client should store `fileId` in Room before starting upload — on retry, reuse the same `fileId`

**Why WorkManager (not coroutines or foreground service alone):**
```
Coroutine in ViewModel:
  ❌ Dies on process death
  ❌ Dies on config change (unless in viewModelScope, but still dies on process kill)
  ❌ No retry with backoff built in

Foreground Service alone:
  ✅ Keeps process alive while running
  ❌ Must manage lifecycle manually
  ❌ No built-in retry, constraints, or chaining
  ❌ Killed if user force-stops app

WorkManager:
  ✅ Survives process death — system reschedules the Worker
  ✅ Built-in retry with exponential backoff
  ✅ Constraints: requiresNetwork, requiresCharging, requiresStorageNotLow
  ✅ Can promote to foreground service (setForeground) for user-visible progress
  ✅ Chaining: upload chunks in sequence or parallel via WorkContinuation
  ✅ Unique work: enqueueUniqueWork prevents duplicate transfers

→ Use WorkManager as the orchestrator. If user needs real-time progress,
  Worker calls setForeground() to promote itself to a foreground service
  with a notification showing progress.
```

**Data flow — Upload (small file):**
```
1. User picks file
2. Read file metadata (name, size, mimeType) from ContentResolver
3. POST /files/register → get fileId (durable, never expires)
4. Create Transfer in Room
5. Enqueue UploadWorker with fileId as input
6. Worker runs:
   a. Update status → IN_PROGRESS in Room
   b. POST /files/{fileId}/upload-url → get FRESH pre-signed URL
      (generated right now, not at enqueue time — no expiration risk)
   c. Open InputStream from ContentResolver
   d. PUT to pre-signed URL with progress tracking:
      - Use OkHttp RequestBody that wraps InputStream
      - Override writeTo() to count bytes and update Room periodically
      - Call setForeground() with notification showing progress
   e. POST /files/{fileId}/confirm → server records the file
   f. Update status → COMPLETED in Room
```

**Data flow — Upload (large file, chunked):**
```
1. User picks file
2. File size > threshold → multipart path
3. Create Transfer in Room
4. Enqueue UploadWorker with fileId
5. Worker runs:
   a. Check Room: does this Transfer already have Chunks created
      - No (first run) → POST /files/multipart/init → get fileId, uploadId, chunkSize, totalChunks
        Create ChunkState entries in Room (one per chunk, all PENDING)
        Store uploadId + fileId in Transfer record
      - Yes (resume) → skip init, go straight to upload loop
   b. Upload loop:
      - Query Room for next PENDING chunk
      - POST /files/multipart/{fileId}/chunk-url → get FRESH pre-signed URL for this chunk
        (generated right now — no expiration risk regardless of how long the upload takes)
      - Read chunk bytes from file (seek to offset, read chunkSize bytes)
      - PUT to chunk's pre-signed URL
      - On success: mark chunk UPLOADED in Room, store ETag
      - Update Transfer.bytesTransferred in Room
      - Repeat until all chunks UPLOADED
   c. POST /files/multipart/complete with all ETags
   d. Update Transfer status → COMPLETED

   Parallel chunk upload (optional optimization):
   - Instead of sequential loop, launch 2-3 coroutines
   - Each coroutine picks next PENDING chunk from Room (use DB transaction to avoid double-picking)
   - Batch-fetch 2-3 URLs at once (POST with chunkIndexes: [5, 6, 7])
   - Speeds up upload on fast connections
   - Limit concurrency to avoid memory pressure (each chunk is 5MB in memory)
```

**Data flow — Download (with resume):**
```
1. User taps download
2. Create Transfer in Room (status = QUEUED, direction = DOWNLOAD, fileId stored)
   (No pre-signed URL yet — will be fetched by Worker)
3. Enqueue DownloadWorker with fileId
4. Worker runs:
   a. GET /files/{fileId}/download-url → get FRESH pre-signed URL
      (generated right now — no expiration risk from queue delay)
   b. Check if partial file exists locally:
      - Yes → resumeOffset = local file size
      - No → resumeOffset = 0
   c. GET pre-signed URL with Range: bytes={resumeOffset}-
   d. Response 206 → append to existing file
      Response 200 → server doesn't support range, restart from scratch
   e. Stream response body to file, tracking bytes:
      - Read from ResponseBody.source() in buffer chunks (8KB)
      - Write to FileOutputStream(append = resumeOffset > 0)
   f. Verify file integrity:
      - Check final file size matches expected
      - If server provides checksum (ETag, Content-MD5), verify hash
   g. Update status → COMPLETED in Room
   h. If download to public Downloads folder:
      - Use MediaStore API (Android 10+) or direct file write (older)
      - Notify MediaScanner so file appears in system file browser

   Resume after kill:
   - Worker restarts, reads Transfer from Room (fileId is there)
   - GET /files/{fileId}/download-url → fresh pre-signed URL
   - Checks local file size → that's the resume offset
   - GET with Range header → continues from where it left off
```

**File storage option**
```agsl
1. App-Specific Internal

  - Private to your app, no permission needed
  - Deleted when app uninstalled
  - Best for: app config, caches, sensitive data

  val file = File(context.filesDir, "data.json")
  // Write
  file.outputStream().use { /* write */ }
  // Read
  file.inputStream().use { /* read */ }

  2. App-Specific External
  
  - No permission needed (API 19+)
  - Deleted when app uninstalled
  - Best for: large files only your app needs

  val file = File(context.getExternalFilesDir(null), "video.mp4")
  // Write
  file.outputStream().use { /* write */ }
  // Read
  file.inputStream().use { /* read */ }

  3. Shared Storage (MediaStore)
  
   Survives app uninstall
  - Visible to other apps and the user
  - API 29+: no permission to write your own files; need READ_MEDIA_* to read others'
  - Best for: user-facing files (downloads, photos, videos)

  // Create
  val values = ContentValues().apply {
      put(MediaStore.Downloads.DISPLAY_NAME, "report.pdf")
  }
  val uri = contentResolver.insert(MediaStore.Downloads.EXTERNAL_CONTENT_URI, values)!!

  // Write
  contentResolver.openOutputStream(uri)?.use { /* write */ }
  // Read
  contentResolver.openInputStream(uri)?.use { /* read */ }

  4. User-Picked File (SAF — Storage Access Framework)

  // Launch picker
  val intent = Intent(Intent.ACTION_OPEN_DOCUMENT).apply {
      type = "*/*"
  }
  launcher.launch(intent)

  // In callback, receive uri
  contentResolver.openInputStream(uri)?.use { /* read */ }
```

**Key Trade-offs:**

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Upload strategy | Single PUT | Chunked multipart | Chunked for files >10MB — resume support, parallel chunks |
| Client→Storage path | Through backend (proxy) | Pre-signed URL (direct) | Pre-signed — saves backend bandwidth |
| Background execution | Coroutine | WorkManager | WorkManager — survives process death, built-in retry |
| Progress storage | Room | WorkManager setProgress | Room — durable, queryable, survives restart |
| Download resume | Restart from scratch | HTTP Range header | Range — saves bandwidth, better UX |
| Chunk concurrency | Sequential | 2-3 parallel | 2-3 parallel — faster on good connections, cap memory |
| Concurrency limit | Unlimited | Max 3 concurrent transfers | Max 3 — battery/memory trade-off |
| Network policy | Always transfer | Wi-Fi only default | Wi-Fi only for uploads >50MB, user-configurable |
| File integrity | Trust transfer | Checksum verify | Verify checksum — detect corrupted transfers |
| Partial file cleanup | Keep forever | Delete after 7 days if incomplete | Delete stale partials — prevent storage bloat |

**Gotchas:**
- **URI permission expiration**: `content://` URIs from the file picker may lose permission after activity/process restart. Persist URI permission with `takePersistableUriPermission()` or copy the file to app-internal storage before enqueuing the Worker
- **Pre-signed URL expiration**: URLs expire (typically 15min-1hr). The design avoids this by generating URLs right before each upload/download (inside the Worker, not at enqueue time). For chunked uploads, each chunk gets a fresh URL. If a single chunk takes longer than the expiry window (very large chunk on slow network), Worker detects 403 → re-fetches URL → retries that chunk
- **Memory pressure on large chunks**: reading a 5MB chunk into memory is fine, but don't load the entire file. Use streaming: `InputStream.read(buffer, 0, chunkSize)` with a fixed-size buffer, or use OkHttp's `RequestBody.create()` with a `Source` that reads from disk
- **Checksum mismatch**: if download completes but checksum doesn't match (corrupted transfer), delete the file and retry. Don't serve corrupted files to the user
- **Storage space**: check `StatFs.getAvailableBytes()` before starting a download. If insufficient, show an error before starting, not mid-download. Also handle the case where space runs out mid-transfer (other apps consumed space)
- **Foreground Worker notification**: WorkManager runs as a background task by default. To show progress and prevent the system from killing long transfers, call `setForeground(ForegroundInfo(...))` inside the Worker — this promotes it to a foreground service with an ongoing notification. On Android 12+, you must specify the foreground service type in `ForegroundInfo`. On Android 14+, you must also declare it in the manifest.
- **Battery optimization**: system may kill background Workers aggressively on some OEMs (Xiaomi, Samsung). Using `setForeground()` with an ongoing notification helps. Inform users if their device has aggressive battery settings
- **Duplicate uploads**: user might tap upload twice quickly. Use `enqueueUniqueWork(fileName + hash, KEEP)` to prevent duplicates. Or generate a content hash before upload and check server-side
- **Counting active transfers via tags**: tag Workers by type when enqueuing, then query by tag to get counts — useful for showing "3 uploads in progress" in the UI or enforcing concurrency limits
- **Large file thumbnail**: don't load a 500MB video into memory to generate a thumbnail. Use `MediaMetadataRetriever.getFrameAtTime()` for video, `BitmapFactory.Options.inSampleSize` for images, or let the server generate thumbnails after upload
- **Multipart abort cleanup**: if user cancels a multipart upload, call the abort endpoint. Otherwise, partially uploaded chunks remain on S3 indefinitely, accruing storage costs. Set up S3 lifecycle rules as a safety net to auto-delete incomplete multiparts after N days

---

## 13. AI Chat Interface
Streaming responses, conversation history, markdown rendering, token-based APIs

### Phase 1: Requirement Clarification

**Functional Requirements — Ask About:**
- Conversation management: create new conversation, view list of past conversations, delete conversation?
- Message types: text only? Image/file attachments in user messages? Or text-only for MVP?
- AI response behavior: streaming (token-by-token) or wait-for-complete? Stop generation mid-stream?
- Markdown rendering: code blocks with syntax highlighting? Tables? LaTeX math? Images in responses?
- Message actions: copy message, retry/regenerate response, edit user message and re-submit?
- Context window: does the user see token count? Warning when approaching limit?
- Model selection: single model or picker (GPT-4, Claude, etc.)?
- System prompt / persona: can the user configure it?

**Non-Functional Requirements — Ask About:**
- Latency: time-to-first-token target? (<500ms feels responsive)
- Streaming smoothness: token-by-token or chunked? (SSE vs WebSocket)
- Offline: can users view past conversations offline? Compose messages offline?
- Persistence: conversations stored server-side, client-side, or both?
- Scale: how many conversations per user? How many messages per conversation?
- Token usage: display token count? Rate limiting? Usage caps?
- Accessibility: screen reader for streaming text? Focus management?
- Analytics: track token usage, response latency, error rates?

**Scope Boundaries — Ask These:**
- Multi-modal input (image upload, voice) in scope?
- Conversation sharing / export?
- Collaborative conversations (multiple users)?
- Plugin / function calling display?
- Branching conversations (edit earlier message → fork)?

**Things Interviewers Leave Ambiguous on Purpose:**
- How to handle partial responses when network drops mid-stream
- Whether conversation history is sent with every request or managed server-side
- How to render markdown incrementally while streaming (partial code blocks)
- What happens when user sends a new message while AI is still responding

### Phase 2: Data Model

```
Conversation {
  id: String
  title: String                 // AI summaries
  model: AiModel { Sonnet 4.6, GPT 5.2 ... }                
  createdAt: Instant
  updatedAt: Instant            // last message time, used for sorting conversation list
}

Message {
  id: String                    
  conversationId: String
  role: MessageRole             // USER, ASSISTANT, SYSTEM
  content: String               // raw text (markdown for assistant messages)
  status: MessageStatus         // SENDING, STREAMING, COMPLETE, ERROR
  tokenCount: Int?              // token usage for this message (from API response)
  createdAt: Instant,
}

enum MessageRole { USER, ASSISTANT, SYSTEM }
enum MessageStatus { SENDING, STREAMING, COMPLETE, ERROR }
}

enum BlockType { THINKING, TEXT, TOOL_USE }
```

> **Note**: `streamingContent` is kept separate from `messages` list during streaming. Once streaming completes, it's merged into the final `Message` object with status=COMPLETE. This avoids triggering recomposition of the entire message list on every token.

> **Note**: Assistant message content is raw markdown stored as-is. Rendering to styled text happens at the UI layer. This keeps the data layer simple and allows different rendering strategies.

### Phase 3: API Design

```
// Conversation CRUD
// create new conversation
POST   /conversations/new
GET    /conversations?limit=20&cursor={cursor}

// Messages
GET    /conversations/{id}/messages?limit=50&cursor={cursor} 

SSE streaming {
  url: String,
  body: RequestBody, -> { message, model } 
  onEvent: (String) -> Unit,
  onComplete: () -> Unit,
  onError: (Throwable) -> Unit
}

// --- Thinking block (optional — AI's reasoning, shown as collapsible/dimmed) ---
data: {"type":"content_block_start","index":0,"content_block":{"type":"thinking","text":""}}
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","text":"Looking at the user's question..."}}
data: {"type":"content_block_delta","index":0,"delta":{"type":"thinking_delta","text":"I should use a Kotlin example..."}}
data: {"type":"content_block_stop","index":0}

// --- Text block (the actual response — raw markdown, client parses) ---
data: {"type":"content_block_start","index":1,"content_block":{"type":"text","text":""}}
data: {"type":"content_block_delta","index":1,"delta":{"type":"text_delta","text":"Monads are a design pattern. "}}
data: {"type":"content_block_delta","index":1,"delta":{"type":"text_delta","text":"```kotlin\nfun <T> flatMap("}}
data: {"type":"content_block_delta","index":1,"delta":{"type":"text_delta","text":"f: (T) -> Monad<T>)\n```"}}
data: {"type":"content_block_stop","index":1}

// --- Message complete (final signal with usage) ---
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},"usage":{"input_tokens":15,"output_tokens":230}}
data: [DONE]

// Key design points:
// - Content blocks are typed (thinking, text, tool_use) — client renders each differently
// - Within a text block, content is RAW MARKDOWN — server streams tokens as-is, client parses
// - index field tracks which block a delta belongs to (multiple blocks can interleave)
// - Tool use enables agentic behavior: AI can search, run code, call APIs mid-response

// Stop generation (client closes SSE connection — server detects and stops inference)
// No explicit API needed — closing the HTTP connection is the signal
```

**Protocol Choice — SSE vs WebSocket:**

**SSE (Server-Sent Events)** is a standard HTTP protocol where the server pushes data to the client over a long-lived HTTP connection:
1. Client makes a normal HTTP request
2. Server responds with `Content-Type: text/event-stream`
3. Server keeps the connection open and sends chunks of data as they become available
4. Each chunk is a `data:` line followed by the content

On Android, use OkHttp's `EventSource` to consume SSE — it gives callbacks as each `data:` line arrives, which you emit into a Kotlin `Flow`.
```agsl
.header("Accept", "text/event-stream")
```

| Aspect | SSE | WebSocket |
|--------|-----|-----------|
| Direction | Server → Client (one-way) | Bidirectional |
| Reconnection | Built-in auto-reconnect | Must implement manually |
| Protocol | Standard HTTP | Separate protocol (ws://) |
| Proxy/CDN support | Works through HTTP infra | Can be blocked by proxies |
| Best for | AI streaming (server pushes tokens) | Real-time chat (both sides send) |

> **Recommendation**: **SSE for AI chat**. The pattern is request-response with a streaming response body — fundamentally one-directional. User sends a POST request, server streams back tokens via SSE. No need for persistent bidirectional connection. This is what OpenAI, Anthropic, and most LLM APIs use.

> **Why not WebSocket?** WebSocket is better when *both* sides need to push data unpredictably (e.g., multiplayer game, collaborative editing). For AI chat, the client always initiates (sends message), and the server always responds (streams tokens). HTTP + SSE fits this naturally and works better with existing HTTP infrastructure (load balancers, CDNs, auth middleware).

**Context Management**: The server always manages the actual context window sent to the AI model (truncation, token counting, system prompts). The question is how the server gets the conversation history:
1. **Server stores history**: Client sends only the new message. Server looks up prior messages from its database and assembles the full context. Client payloads are small.
2. **Client sends history**: Client sends the full message array with each request. Server is stateless — no database needed. But payloads grow with conversation length.

> Most production apps (ChatGPT, Claude) use approach 1 — server stores history. For mobile this is clearly better: smaller payloads, no risk of client-side truncation bugs, and the server can apply token limits consistently.

### Phase 4: Mobile Architecture

```
UI Layer
  ├─ ConversationListScreen — LazyColumn of conversations, swipe-to-delete, FAB for new chat
  └─ ChatScreen
       ├─ LazyColumn (reversed) — messages rendered bottom-up, newest at bottom
       │    ├─ UserMessageBubble — plain text, right-aligned
       │    └─ AssistantMessageBubble — markdown rendered, left-aligned
       │         └─ MarkdownRenderer — code blocks, inline code, bold, lists, links
       ├─ StreamingIndicator — typing cursor / partial text during streaming
       └─ InputBar — TextField + send button (disabled while streaming) + stop button

ViewModel
  ├─ ConversationListViewModel
  │    └─ conversations: StateFlow<ConversationListState>
  └─ ChatViewModel
       └─ uiState: StateFlow<ChatScreenState>
            - sendMessage(text) → POST + consume SSE stream
            - stopGeneration() → cancel coroutine / close connection
            - retryLastMessage() → re-send last user message
            - loadMoreHistory() → paginate older messages

Repository
  └─ ChatRepository
       - createConversation() → POST /conversations
       - sendMessage(conversationId, content) → returns Flow<StreamEvent>
       - getConversations() → Flow from Room, synced with remote
       - getMessages(conversationId) → Flow from Room

Data Sources
  ├─ Remote: OkHttp SSE client (not Retrofit — Retrofit doesn't handle SSE well)
  │    └─ Uses OkHttp EventSource for streaming, Retrofit for REST endpoints
  └─ Local: Room (conversations + messages for offline viewing + fast cold start)
```

**Streaming Implementation Detail:**

```
// OkHttp SSE approach
fun streamResponse(conversationId: String, message: String): Flow<StreamEvent> = callbackFlow {
    val request = Request.Builder()
        .url("$BASE_URL/conversations/$conversationId/messages")
        .post(requestBody)
        .build()

    val eventSource = EventSources.createFactory(okHttpClient)
        .newEventSource(request, object : EventSourceListener() {
            override fun onEvent(eventSource: EventSource, id: String?, type: String?, data: String) {
                if (data == "[DONE]") { close(); return }
                val event = json.decodeFromString<StreamEvent>(data)
                trySend(event)
            }
            override fun onFailure(eventSource: EventSource, t: Throwable?, response: Response?) {
                close(t ?: IOException("SSE connection failed"))
            }
        })

    awaitClose { eventSource.cancel() }  // cancel SSE when Flow collector cancels (stop generation)
}
```

**ViewModel Streaming Consumption:**

```
fun sendMessage(text: String) {
    // 1. Add user message to Room + UI immediately
    // 2. Create placeholder assistant message with status=STREAMING
    // 3. Collect SSE flow, accumulate deltas into streamingContent
    // 4. On CONTENT_DONE: update message status=COMPLETE, save full content to Room
    // 5. On error: update message status=ERROR, save error message

    streamJob = viewModelScope.launch {
        _uiState.update { it.copy(streamingState = Streaming(content = "")) }
        val thinking = StringBuilder()
        val response = StringBuilder()

        repository.streamResponse(conversationId, text)
            .catch { e -> _uiState.update { it.copy(error = e.message, streamingState = Idle) } }
            .collect { event ->
                when (event) {
                    is BlockStart -> { /* track current block type if needed */ }
                    is BlockDelta -> {
                        // Append to the right buffer based on which block is active
                        // (use index to look up block type from the last BlockStart)
                        response.append(event.delta)
                        _uiState.update { it.copy(streamingState = Streaming(content = response.toString())) }
                    }
                    is BlockStop -> { /* block finished, could finalize thinking text */ }
                    is MessageDelta -> {
                        // Repository saves to Room internally when it emits this event
                        _uiState.update {
                            it.copy(
                                messages = it.messages + event.toMessage(),
                                streamingState = Idle,
                            )
                        }
                    }
                }
            }
    }
}

fun stopGeneration() {
    streamJob?.cancel()  // cancels Flow → awaitClose → eventSource.cancel()
    // Save whatever was accumulated so far as a COMPLETE message
}
```

#### Streaming Markdown Parsing Strategy — Re-parse on Every Token

The recommended approach is to **accumulate all tokens into a buffer and re-parse the entire buffer** on each new token. Unclosed blocks are treated as "still open" rather than broken.

**How it works token-by-token:**

Say the LLM streams: `"Here"` → `" is"` → `" code"` → `":\n\`\`"` → `"\`\nval"` → `" x ="` → `" 1\n\`"` → `"\`\`\n"`

| After Token | Buffer State | Rendered As |
|-------------|-------------|-------------|
| `"Here"` | `Here` | Paragraph: "Here" |
| `" is"` | `Here is` | Paragraph: "Here is" |
| `":\n\`\`"` | `Here is code:\n\`\`` | Paragraph + literal backticks |
| `"\`\nval"` | `Here is code:\n\`\`\`\nval` | Paragraph + **open code block** with "val" |
| `" 1\n\`"` | `...val x = 1\n\`` | Still open code block (only 1 backtick) |
| `"\`\`\n"` | `...val x = 1\n\`\`\`` | Paragraph + **closed code block** |

**Block boundary rules (CommonMark spec):**

The parser doesn't track state between re-parses. It walks line-by-line and the grammar determines block boundaries:

```
Fenced code block:
  OPEN:     line starts with ``` (or ~~~)
  CONTINUE: every subsequent line that is NOT a closing fence
  CLOSE:    line starts with ``` (matching or more backticks)

Paragraph:
  OPEN:     any non-blank line that doesn't match another block start
  CONTINUE: next line is also non-blank and not a block start
  CLOSE:    blank line, or a line that starts a new block

List item:
  OPEN:     line starts with - or 1.
  CONTINUE: subsequent lines indented to the item's content column
  CLOSE:    line at lower indentation or a blank line gap
```

**Unclosed block fallback rules:**
- Unclosed ` ``` ` → render as open code block (grows with each token)
- Unclosed `**` → render as open bold or treat as literal
- Unclosed `[link](` → render as literal text
- Ambiguous partial tokens (single `` ` ``) → render as-is, next re-parse self-corrects

**Why re-parsing the full buffer is cheap enough:**
- Typical full LLM response is ~2-4KB of markdown
- CommonMark parsing is O(n) in text length
- At ~50ms between tokens, plenty of time
- Compose skips recomposition if rendered tree hasn't structurally changed

**Compose integration:**
```kotlin
@Composable
fun StreamingMarkdown(flow: Flow<String>) {
    var buffer by remember { mutableStateOf("") }
    LaunchedEffect(flow) {
        flow.collect { token -> buffer += token }
    }
    MarkdownText(markdown = buffer) // re-parses on every buffer change
}
```

**Key Trade-offs:**

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Streaming protocol | SSE | WebSocket | SSE — simpler, matches LLM API pattern |
| SSE client | OkHttp EventSource | Retrofit | OkHttp — Retrofit has no SSE support |
| Markdown rendering | Compose-native | WebView | Compose-native — lighter, better UX |
| Streaming markdown | Plain text until done | Incremental parse | Incremental parse — users expect live-formatted code blocks |
| Message list layout | LazyColumn (reversed) | LazyColumn (normal + scroll to bottom) | Reversed — natural for chat, newest at bottom |
| Context management | Server-managed | Client sends full history | Server-managed — reduces payload, handles truncation |
| Conversation storage | Room | In-memory only | Room — offline viewing, fast cold start |
| Stop generation | Cancel HTTP connection | Explicit stop API | Cancel connection — SSE naturally supports this |

**Gotchas:**
- **Reversed LazyColumn**: Use `reverseLayout = true` so new messages appear at bottom. Item order in the list must also be reversed (newest first in the data list, or use `reverseLayout` which handles it)
- **Streaming recomposition**: Update `streamingContent` on each token. Use `derivedStateOf` or a separate composable for the streaming bubble to avoid recomposing the entire list
- **Network drop mid-stream**: Save accumulated content so far with status=ERROR. Offer retry button that re-sends the user message
- **Rapid token arrival**: SSE can fire hundreds of events per second. Batch updates with `conflate()` or update on a frame-rate cadence (every 16ms) to avoid excessive recomposition
- **Memory for long conversations**: Conversations with hundreds of messages need windowed loading. Load last N messages, paginate upward for history
- **Process death during streaming**: Streaming state is lost. On restore, check if last assistant message has status=STREAMING → mark as ERROR with "Response interrupted"
- **Copy message**: Copy raw markdown or rendered text? Offer both — long-press → "Copy text" (rendered) / "Copy markdown" (raw)
- **Code blocks**: Need horizontal scroll for wide code. Wrap in `horizontalScroll()` modifier. Add "Copy code" button on each code block
- **Auto-scroll**: While streaming, auto-scroll to bottom to follow new tokens. If user scrolls up (reading earlier messages), stop auto-scroll. Resume when user scrolls back to bottom
- **Input while streaming**: Disable send button during streaming. Show stop button instead. Some apps queue the next message — KISS, just disable

**Reversed LazyColumn Construction:**
```kotlin
val listState = rememberLazyListState()

LazyColumn(
    state = listState,
    reverseLayout = true  // anchors content to the bottom
) {
    items(messages.reversed()) { message ->  // reverse data so newest is at index 0
        MessageBubble(message)
    }
}
// With reverseLayout = true:
//   - index 0 = bottom of the screen (newest message)
//   - scrolling "up" reveals older messages
//   - new items naturally appear at the bottom without manual scroll
```

**Auto-Scroll During Streaming:**
```kotlin
// Track if user is "at the bottom"
val isAtBottom by remember {
    derivedStateOf { listState.firstVisibleItemIndex == 0 }  // reversed list: index 0 = bottom
}

// Auto-scroll to bottom when user is not already there
LaunchedEffect(isAtBottom) {
    if (!isAtBottom) {
        listState.animateScrollToItem(0)
    }
}
```

---

## Video Feed Sessions

### Design YouTube App — Andrey Tech (ex-Meta)
**Source:** [YouTube](https://www.youtube.com/watch?v=kiRSQAlUsn8) (~16 min)

#### Requirements
**Functional:**
- Video feed / home screen with video list
- Video playback (streaming)
- Background audio playback
- Picture-in-Picture (PiP) support

**Non-Functional:**
- Smooth scrolling in feed with video thumbnails
- Low-latency video start
- Adaptive bitrate for varying network conditions

#### API & Data Model

```agsl
// Data Model

VideoSummary {
    videoId: String,
    title: String,
    videoThumbnailUrl: [...],
    channelThumbnailUrl: String,
    viewCount: Int,
    postAt: Instant / Date,
}

VideoDetail {
    videoId: String,
    videoSummary: VideoSummary,
    description: String.
    commentCount: Int,
    channel: Channel,
}

Channel {
    channelId: String,
    channelName: String,
    channelThumbnailUrl: String,
    subscriberCount: Int,
}

LikeAction {
    isLiked: Boolean,
    videoId: String,
    userId: String
}

VideoPlayBack {
    videoId: String,
    menifestUrl: String,
}

// API
Get v1/videos?q=mostPolupar&region=us&cursor=abc
-> [ VideoSummary ] 

Get v1/video/detail/{videoId}
-> VideoDetail

POST v1/video/{videoId}/rate
request body: { LikeAction }

GET v1/video/{videoId}/rate
-> LikeAction

GET v1/video/{videoId}/playback:protocal=supportedProtocal
-> VideoPlayBack
```

#### Video Playback — HLS vs DASH

**AVOID download the whole video**

- download video in chunks (10 seconds)
- start playback after the first few chunk is downloaded
- download more chunks in background as user watches
- delete old chunk with not needed

| Protocol | Full Name | How It Works | Platform Support |
|----------|-----------|-------------|-----------------|
| **HLS** | HTTP Live Streaming | Video split into `.ts` segments listed in `.m3u8` playlist; client picks quality tier | Native on iOS/Android (`ExoPlayer` / `MediaPlayer`) |
| **DASH** | Dynamic Adaptive Streaming over HTTP | Similar segment approach with `.mpd` manifest; more flexible codec support | Android via `ExoPlayer`; iOS needs third-party |

**Key concepts:**
- Video is encoded at multiple bitrates/resolutions (e.g., 360p, 720p, 1080p)
- Manifest file lists available quality tiers and segment URLs
- Player monitors bandwidth and switches tiers mid-stream (**adaptive bitrate streaming**)
- Segments are small (2–10 sec) so quality switches are fast

#### High-Level Architecture (MVVM)
```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   UI Layer  │────▶│  ViewModel   │────▶│  Repository  │
│ (Compose /  │     │ (StateFlow)  │     │              │
│  Fragment)  │     └──────────────┘     └──────┬───────┘
└─────────────┘                                 │
                                    ┌───────────┴───────────┐
                                    │                       │
                              ┌─────▼─────┐          ┌─────▼─────┐
                              │  Remote   │          │   Local   │
                              │ (Retrofit)│          │  (Room /  │
                              └───────────┘          │   Cache)  │
                                                     └───────────┘
```

- **Feed screen**: ViewModel exposes `StateFlow<List<VideoItem>>`, UI renders in `LazyColumn`
- **Player screen**: Dedicated ViewModel manages playback state; uses `ExoPlayer` under the hood
- **Caching**: Thumbnail caching via Coil/Glide; video segment caching handled by ExoPlayer's built-in cache

#### Background Audio Playback

**Architecture**: ExoPlayer always lives inside a **foreground Service**, not the Activity. The Activity is just a UI shell that binds to the Service.

```
Activity (UI) ──binds──▶ MediaPlaybackService (foreground)
                              │
                         ExoPlayer instance
                              │
                         MediaSession (publishes state)
                              │
                         Notification (MediaStyle)
```

**Why the Service survives when the app is backgrounded:**

The Service is started with **both** `startForegroundService()` and `bindService()`:
- `startForegroundService()` gives the Service an **independent lifecycle** — it stays alive until `stopSelf()` is called, regardless of Activity binding
- `bindService()` gives the Activity a reference to communicate with the player
- When the user leaves the app → `unbindService()` is called, but the Service **stays alive** because it was started independently
- The persistent notification satisfies the foreground service requirement
- The Service only dies when it calls `stopSelf()` (e.g., user pauses and a timeout elapses, or the queue ends)

**Manifest declarations required:**
```xml
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<service
    android:name=".PlaybackService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="false" />
```

#### MediaSession

A **standard contract** between your player and everything external that wants to control or display playback — lock screen, notification, Bluetooth headphones, Google Assistant, Android Auto, etc.

**Two halves:**

| Component | Where | Role |
|-----------|-------|------|
| `MediaSession` | Service side | Source of truth — publishes state, receives commands |
| `MediaController` | Activity/UI side | Remote control — mirrors session state, sends commands |

**What MediaSession handles automatically (with Media3):**
- **Notification controls**: Auto-creates `MediaStyle` notification with play/pause/skip
- **Lock screen**: System reads session state, shows controls automatically
- **Headphone button**: System routes `ACTION_MEDIA_BUTTON` → session → player
- **Google Assistant**: "Hey Google, pause" → session callback → player
- **Audio focus**: Handled automatically when using `MediaSession`

**Media3 setup (minimal boilerplate):**
```kotlin
// Service side — ExoPlayer implements MediaSession.Player, so auto-syncs
val session = MediaSession.Builder(context, exoPlayer).build()

// Activity side — connect to session
val controllerFuture = MediaController.Builder(context, session.token).buildAsync()
controllerFuture.addListener({
    val controller = controllerFuture.get()
    controller.play()
    controller.seekTo(30_000)
}, MoreExecutors.directExecutor())
```

**Custom commands via callbacks:**
```kotlin
val session = MediaSession.Builder(context, player)
    .setCallback(object : MediaSession.Callback {
        override fun onConnect(session: MediaSession, controller: MediaSession.ControllerInfo)
            : MediaSession.ConnectionResult = AcceptedResultBuilder(session).build()

        override fun onCustomCommand(session: MediaSession, controller: MediaSession.ControllerInfo,
            customCommand: SessionCommand, args: Bundle): ListenableFuture<SessionResult> {
            if (customCommand.customAction == "FAVORITE") { /* handle */ }
            return Futures.immediateFuture(SessionResult(RESULT_SUCCESS))
        }
    }).build()
```

**MediaSessionCompat vs MediaSession (Media3):**
- `MediaSessionCompat` (`androidx.media`) — legacy, high boilerplate, manually sync player state
- `MediaSession` (`androidx.media3.session`) — **use this**, auto-syncs with ExoPlayer, wraps compat layer internally for backward compatibility

#### Picture-in-Picture
| Feature | Android Implementation |
|---------|----------------------|
| **Picture-in-Picture** | `Activity.enterPictureInPictureMode(PictureInPictureParams)` — declare `supportsPictureInPicture="true"` in manifest; handle `onPictureInPictureModeChanged` for UI adjustments |

**PiP gotchas on Android:**
- Must set `configChanges="screenSize|smallestScreenSize|screenLayout|orientation"` to avoid Activity recreation
- Use `RemoteAction` to add custom controls (play/pause, skip) in PiP window
- PiP is only available on API 26+

---

## 14. Mobile Metrics Collection (TikTok Shopping Foundation)

Collect metrics for user interactions

### Phase 1: Requirement Clarification

**Functional Requirements:**
- **Foundational SDK**: a shared API that any feature team can integrate — shopping, feed, live, etc.
- **Flexible metadata**: able to collect different event types with arbitrary key-value metadata (tap, scroll, impression, purchase, page view, error, etc.)
- **Manual instrumentation**: feature teams call `MetricsCollector.track(event)` for custom business events

**Non-Functional Requirements:**
- **Maximize collection**: collect as much as possible — dropping events is acceptable only under extreme pressure (battery/storage)
- **Minimal impact**: collection must not block UI thread or degrade scroll performance
- **Batching & compression**: events are batched and compressed before upload to reduce network usage
- **Offline resilience**: events survive app kill, process death, and no-network periods
- **Deduplication**: server-side dedup via event IDs — safe to retry uploads

**Things Interviewers Leave Ambiguous on Purpose:**
- How large can the local queue grow before dropping?
- Should events be sampled on high-traffic screens?
- Real-time vs near-real-time delivery expectations
- Privacy: which fields need scrubbing before upload?

### Phase 2: Data Model

```
MetricEvent {
  eventId: String               // UUID — used for dedup
  eventType: String             // "impression", "tap", "purchase", "page_view", "error"
  timestamp: Long               // epoch millis, captured at event time
  sessionId: String             // links events within a single user session
  userId: String?               // null if logged out
  metadata: Map<String, Any>    // arbitrary KV pairs — feature teams add what they need
                                // value type can be a Proto (protobuf) so the schema is shared
                                // across platforms (Android, iOS, web) and enforced at compile time
}

EventBatch {
  batchId: String
  events: List<MetricEvent>
  createdAt: Long
  retryCount: Int               // track how many times this batch was attempted
}

SessionInfo {
  sessionId: String
  startTime: Long
  deviceInfo: DeviceInfo        // OS version, device model, app version, locale
}

DeviceInfo {
  osVersion: String             // e.g. "Android 14"
  deviceModel: String           // e.g. "Pixel 8"
  appVersion: String            // e.g. "23.4.1"
  locale: String                // e.g. "en_US"
  networkType: String           // "wifi", "cellular", "none"
}
```

### Phase 3: API Design

```
// Single batch upload endpoint
POST /v1/metrics/batch
Headers: Content-Encoding: gzip, X-Idempotency-Key: {batchId}
Body: {
  batchId: String,
  sessionInfo: SessionInfo,
  events: [MetricEvent]
}
Response: { accepted: Int, duplicates: Int }
```

**Why POST not PUT?** Each batch is a new resource (append-only log). Idempotency handled via `batchId` header, so retries are safe.

### Phase 4: Mobile Architecture

**Presentation Layer — Auto-capture Setup**

```kotlin
// Feature teams integrate with one line:
MetricsCollector.track(
    eventType = "tap_add_to_cart",
    screenName = "ProductDetail",
    metadata = mapOf("productId" to "p_123", "price" to 29.99)
)

// Impression tracking via LazyColumn
@Composable
fun TrackImpression(eventType: String, metadata: Map<String, Any>, content: @Composable () -> Unit) {
    val visible = remember { mutableStateOf(false) }
    Box(modifier = Modifier.onGloballyPositioned {
        // check if >50% visible for >1 second
    }) {
        content()
    }
    LaunchedEffect(visible.value) {
        if (visible.value) {
            delay(1000) // 1s dwell threshold — only count impression if item was
                        // visible for ≥1s. Prevents fast-scroll fly-bys from inflating metrics.
                        // If user scrolls away before 1s, LaunchedEffect cancels (key changed),
                        // so track() never fires.
            MetricsCollector.track(eventType, metadata = metadata)
        }
    }
}
```

**Domain Layer — MetricsCollector (Singleton SDK)**

Two-tier storage: **in-memory Channel → Room (individual events) → batch from Room → WorkManager upload**

```kotlin
object MetricsCollector {

    private val eventChannel = Channel<MetricEvent>(capacity = Channel.BUFFERED)
    private val sessionId: String = UUID.randomUUID().toString()
    private const val BUFFER_FLUSH_SIZE = 20
    private const val BUFFER_TIMEOUT_MS = 5_000L  // flush partial buffer after 5s of no new events
    private const val FLUSH_INTERVAL_MS = 30_000L
    // Option A: ProcessLifecycleOwner.get().lifecycleScope
    //   + No setup — just works out of the box
    //   + Automatically tied to app process lifetime (ON_DESTROY never fires)
    //   - Couples SDK to Android lifecycle APIs — can't use in pure Kotlin/KMP modules
    //   - Hard to test — need Robolectric or instrumented tests to get a real LifecycleOwner
    //
    // Option B (recommended): Inject a singleton CoroutineScope
    //   + Testable — swap with TestScope in unit tests
    //   + Decoupled — no Android dependency, works in pure Kotlin / KMP modules
    //   + Caller controls dispatcher and SupervisorJob policy
    //   - Requires DI setup or manual init
    //   - Caller must ensure the scope lives long enough (app-level, not Activity-level)
    private lateinit var scope: CoroutineScope

    // Provide via Hilt:
    @Module @InstallIn(SingletonComponent::class)
    object CoroutineScopeModule {
         @Provides @Singleton
         fun provideAppScope(): CoroutineScope = CoroutineScope(SupervisorJob())
             // SupervisorJob: if one child coroutine fails (e.g. a single batch upload crashes),
             // it does NOT cancel sibling coroutines (e.g. the channel drain loop keeps running).
             // With a regular Job, one failure would cancel everything — the whole SDK stops collecting.
    }

    fun init(context: Context, scope: CoroutineScope) {
        this.scope = scope

        // Drain channel → batch-insert to Room using `select`
        // select waits for EITHER a new event OR a timeout — whichever fires first
        // No race condition on buffer: select runs EXACTLY ONE branch per iteration.
        // The other branch is skipped entirely — they never run concurrently.
        //   Iteration 1: item arrives   → onReceive runs  (onTimeout skipped)
        //   Iteration 2: item arrives   → onReceive runs  (onTimeout skipped)
        //   Iteration 3: 5s no item     → onTimeout runs  (onReceive skipped)
        //
        // Why not chunked()? chunked() only emits when buffer is full.
        // On a hot flow (Channel), partial buffers sit in memory forever if events
        // trickle in slowly. We need time-based flushing too.
        //
        // NOTE: no sampling here — sampling is the feature team's responsibility, not the SDK's.
        scope.launch(Dispatchers.IO) {
            val buffer = mutableListOf<MetricEventEntity>()

            while (true) {
                select {
                    eventChannel.onReceive { event ->
                        buffer.add(event.toEntity())
                        if (buffer.size >= BUFFER_FLUSH_SIZE) {
                            eventDao.insertAll(buffer.toList())
                            buffer.clear()
                        }
                    }
                    onTimeout(BUFFER_TIMEOUT_MS) {
                        if (buffer.isNotEmpty()) {
                            eventDao.insertAll(buffer.toList())
                            buffer.clear()
                        }
                    }
                }
            }
        }

        // WorkManager periodic task handles upload on its own schedule
        // No in-process flush loop needed — WorkManager survives process death
        MetricsUploadWorker.enqueuePeriodicFlush(context, FLUSH_INTERVAL_MS)
    }

    fun track(eventType: String, screenName: String = "", metadata: Map<String, Any> = emptyMap()) {
        val event = MetricEvent(
            eventId = UUID.randomUUID().toString(),
            eventType = eventType,
            timestamp = System.currentTimeMillis(),
            sessionId = sessionId,
            userId = UserManager.currentUserId,
            screenName = screenName,
            metadata = metadata
        )
        eventChannel.trySend(event) // non-blocking, never blocks UI thread
    }

    // Optional: trigger an immediate flush (e.g., on app background)
    // Runs a one-time WorkManager task outside the periodic schedule
    fun flushNow() {
        val request = OneTimeWorkRequestBuilder<MetricsUploadWorker>()
            .setConstraints(Constraints.Builder()
                .setRequiredNetworkType(NetworkType.CONNECTED)
                .build())
            .build()
        WorkManager.getInstance(context).enqueue(request)
    }
}
```

**Data Access Layer — Persistence & Upload**

```kotlin
// --- Individual event storage (tier 1: immediate persistence) ---

@Entity
data class MetricEventEntity(
    @PrimaryKey val eventId: String,
    val eventType: String,
    val timestamp: Long,
    val sessionId: String,
    val userId: String?,
    val screenName: String,
    val metadataJson: String      // serialized Map<String, Any>
)

@Dao
interface MetricEventDao {
    @Insert(onConflict = OnConflictStrategy.IGNORE)
    suspend fun insertAll(events: List<MetricEventEntity>)  // batch insert — single transaction

    @Query("SELECT * FROM MetricEventEntity ORDER BY timestamp ASC LIMIT :limit")
    suspend fun getOldest(limit: Int): List<MetricEventEntity>

    @Query("DELETE FROM MetricEventEntity WHERE eventId IN (:ids)")
    suspend fun deleteByIds(ids: List<String>)
}

// WorkManager worker — read from Room → upload → delete
// No batch table needed — just query events directly
// Mutex ensures only one upload runs at a time, even if periodic + flushNow() overlap
class MetricsUploadWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {

    companion object {
        // Shared Mutex across all worker instances — prevents concurrent getOldest/deleteByIds
        // from reading the same events when periodic and one-time workers overlap.
        //
        // Simpler alternative: remove flushNow() entirely and only use periodic workers.
        // Then no Mutex is needed since enqueueUniquePeriodicWork(KEEP) guarantees
        // only one periodic worker runs at a time. Prefer this simpler approach unless
        // immediate flush (e.g. on purchase) is a hard requirement.
        private val uploadMutex = Mutex()
    }

    override suspend fun doWork(): Result = uploadMutex.withLock {
        val events = eventDao.getOldest(limit = UPLOAD_BATCH_SIZE)
        if (events.isEmpty()) return@withLock Result.success()

        try {
            val payload = Json.encodeToString(events) // optional: gzip for 5-10x size reduction
            metricsApi.uploadBatch(UUID.randomUUID().toString(), payload)
            eventDao.deleteByIds(events.map { it.eventId })
            Result.success()
        } catch (e: Exception) {
            Result.retry() // WorkManager handles exponential backoff
        }
    }

    companion object {
        fun enqueuePeriodicFlush(context: Context, intervalMs: Long) {
            val request = PeriodicWorkRequestBuilder<MetricsUploadWorker>(
                    intervalMs, TimeUnit.MILLISECONDS
                )
                .setConstraints(Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build())
                .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.SECONDS)
                .build()
            WorkManager.getInstance(context)
                .enqueueUniquePeriodicWork("metrics_flush", ExistingPeriodicWorkPolicy.KEEP, request)
        }
    }
}
```

**Architecture Diagram:**

```
Feature Teams (Shopping, Feed, Live ...)
         │
         │  MetricsCollector.track(event)
         ▼
┌──────────────────────────────┐
│   MetricsCollector           │  ← Singleton, non-blocking Channel
│   (in-memory Channel)       │
│         │                    │
│         │ chunked(N)         │
│         ▼                    │
│   Room: MetricEventEntity    │  ← Batch-insert, survives process death
│                              │
│ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │  ← WorkManager periodic boundary
│                              │
│   WorkManager (periodic)     │  ← Runs every flushIntervalMs
│     1. read oldest N events  │
│     2. gzip + upload         │     POST /metrics/batch
│     3. delete uploaded by id │     on success only
│     (retry on failure)       │     exponential backoff
└──────────────────────────────┘
```

**Key Trade-offs:**

| Decision | Option A | Option B | Recommendation |
|----------|----------|----------|----------------|
| Buffering | In-memory only | Room DB | Room — survives process death, critical for "collect as much as possible". Single table, no separate batch table — just read N events, upload, delete by ID |
| Upload trigger | Time-based only (every 30s) | Time + size (30s or 50 events) | Both — prevents lag on busy screens, prevents stale events on idle screens |
| Serialization | JSON | Protobuf | JSON for simplicity in SDK; protobuf if bandwidth is a real constraint |
| Compression | None | gzip | gzip — typical 5-10x reduction on JSON event payloads |
| Sampling | SDK-level (hidden) | Feature-team-level (explicit) | Feature team — SDK should not silently drop events; hidden filtering causes confusing data loss for teams debugging their metrics |
| Queue overflow | Drop oldest | Drop newest | Drop oldest — newer events are more valuable for debugging |
| Threading | Coroutine Channel | HandlerThread | Channel — structured concurrency, backpressure built in |
| Worker | WorkManager | AlarmManager + Service | WorkManager — handles doze, battery optimization, guaranteed execution |

**Gotchas:**
- **UI thread safety**: `track()` must never block. Use `Channel.trySend()` (non-suspending) not `send()` (suspending)
- **Battery drain**: don't flush too frequently. 30s interval is a good default. Flush immediately on `onStop` lifecycle
- **Storage cap**: if Room grows beyond `maxQueueSize`, drop oldest batches. Prevent unbounded disk usage
- **Impression dedup**: same item scrolled in/out multiple times — dedup by `(eventType, itemId, sessionId)` within a time window
- **Privacy**: scrub PII before storing to Room. Don't log user input text as metadata
- **Config caching**: cache the metrics config locally with a TTL (e.g., 1 hour). Don't block event collection if config fetch fails — use defaults
- **Process death during flush**: batch is already in Room before WorkManager starts, so no data loss. WorkManager re-enqueues on restart



## Appendix: Common MIME Types Reference

**Images**
| MIME Type | Extension | Notes |
|-----------|-----------|-------|
| `image/jpeg` | .jpg, .jpeg | Photos |
| `image/png` | .png | Screenshots, graphics with transparency |
| `image/gif` | .gif | Animated images |
| `image/webp` | .webp | Modern, smaller than JPEG/PNG |
| `image/svg+xml` | .svg | Vector graphics |

**Video**
| MIME Type | Extension | Notes |
|-----------|-----------|-------|
| `video/mp4` | .mp4 | Most common video format |
| `video/webm` | .webm | Web-optimized |
| `video/quicktime` | .mov | Apple QuickTime |

**Audio**
| MIME Type | Extension | Notes |
|-----------|-----------|-------|
| `audio/mpeg` | .mp3 | Most common audio |
| `audio/wav` | .wav | Uncompressed |
| `audio/ogg` | .ogg | Open format |

**Documents**
| MIME Type | Extension | Notes |
|-----------|-----------|-------|
| `application/pdf` | .pdf | PDF |
| `application/msword` | .doc | Legacy Word |
| `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | .docx | Modern Word |
| `application/vnd.ms-excel` | .xls | Legacy Excel |
| `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | .xlsx | Modern Excel |

**Text**
| MIME Type | Extension | Notes |
|-----------|-----------|-------|
| `text/plain` | .txt | Plain text |
| `text/html` | .html | HTML |
| `text/csv` | .csv | CSV data |
| `text/markdown` | .md | Markdown |

**Archives**
| MIME Type | Extension | Notes |
|-----------|-----------|-------|
| `application/zip` | .zip | ZIP |
| `application/gzip` | .gz | GZIP |
| `application/x-tar` | .tar | TAR |

**Data / Other**
| MIME Type | Extension | Notes |
|-----------|-----------|-------|
| `application/json` | .json | JSON |
| `application/xml` | .xml | XML |
| `application/octet-stream` | (any) | Generic binary — fallback when type is unknown |

> **Format**: always `type/subtype`. The `type` is the broad category (image, video, audio, application, text), the `subtype` is the specific format.
>
> **On Android**: get a file's MIME type from `ContentResolver.getType(uri)`. For file extensions, use `MimeTypeMap.getSingleton().getMimeTypeFromExtension(ext)`.
