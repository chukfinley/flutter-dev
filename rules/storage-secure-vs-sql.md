# Local storage: secure_storage for secrets, SQL for data

**Rule:** Always wire up [`flutter_secure_storage`](https://pub.dev/packages/flutter_secure_storage)
for **small secrets** (auth tokens, encryption keys, the current account, a
handful of settings). For anything that **grows** — chat history, AI
conversations, cached API data, lists of records — use a **local SQL database**
(`sqflite`, or `drift` for typed/reactive queries). Never dump bulk data into
secure storage.

## Problem (learned the hard way)

We once stored a large, ever-growing blob (basically the app's whole state) in
`flutter_secure_storage`. On **Android** it was fine. On **Linux desktop** the
app **froze**: secure_storage on desktop reads/writes the *entire* store on every
single `read`/`write`, and once the blob hit ~2 MB, every step re-serialized and
re-encrypted the whole 2 MB. Each tiny update took forever → UI freeze.

`flutter_secure_storage` is a **key–value secret vault**, not a database. It has
no querying, no partial reads, and its desktop/Linux backend is whole-file. It's
the wrong tool the moment data stops being tiny.

## Fix

Split by size and purpose:

| Data | Use |
|------|-----|
| Auth token, refresh token, encryption key (e.g. a 32-byte E2EE key) | `flutter_secure_storage` |
| The one logged-in account, a few flags | `flutter_secure_storage` |
| Chat / AI conversation history, messages, cached lists, anything paginated | **SQL DB** (`sqflite` / `drift`) |
| Larger structured state that changes often | **SQL DB** |

```yaml
dependencies:
  flutter_secure_storage: ^9.0.0
  sqflite: ^2.3.0            # raw SQLite
  # or, preferred for typed + reactive queries:
  drift: ^2.14.0
  sqlite3_flutter_libs: ^0.5.0
```

```dart
// secrets — tiny, fine
const secure = FlutterSecureStorage();
await secure.write(key: 'auth_token', value: token);

// bulk / growing — SQL, not secure_storage
final db = await openDatabase('app.db', version: 1, onCreate: (db, v) {
  return db.execute(
    'CREATE TABLE messages(id INTEGER PRIMARY KEY, chat_id TEXT, body TEXT, ts INTEGER)',
  );
});
await db.insert('messages', {'chat_id': c, 'body': text, 'ts': now});
```

If the bulk data is also **sensitive** (chats, AI history), encrypt the SQLite DB
(e.g. SQLCipher / a field-level AES key kept in `flutter_secure_storage`) instead
of trying to hold it all in the secret vault. That's the messenager approach:
32-byte key in secure_storage, everything else in encrypted SQLite.

## Notes / gotchas

- Rule of thumb: **if it can grow unbounded, it's SQL, not secure_storage.**
- `drift` gives you typed queries + reactive streams (pairs well with Riverpod);
  `sqflite` is fine when you want raw SQL and no code-gen.
- Don't reach for `shared_preferences` for secrets — it's plaintext.
- Desktop builds are where the secure_storage whole-file cost bites; test there,
  not only on Android.
