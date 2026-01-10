## 1️⃣ Design schemas like relational tables (even in NoSQL)

MongoDB lets you be sloppy — Mongoose should stop you from being sloppy.

### Always define

- `required`
    
- `default`
    
- `enum`
    
- `min / max`
    
- `unique` (with index)
    

`const UserSchema = new Schema({   email: {     type: String,     required: true,     lowercase: true,     trim: true,     unique: true,     index: true,   },   status: {     type: String,     enum: ["ACTIVE", "BLOCKED", "DELETED"],     default: "ACTIVE",   }, }, { timestamps: true });`

If you don’t do this, MongoDB becomes a JSON dumpster.

---

## 2️⃣ Never expose raw Mongoose models to your business logic

The fastest way to destroy a backend is:

`UserModel.find(...) UserModel.update(...)`

spread everywhere.

Instead:

`Controller → Service → Repository → Mongoose`

Example:

`class UserRepository {   async findByEmail(email: string) {     return UserModel.findOne({ email }).lean();   } }`

Now:

- You can swap Mongo later
    
- You can test
    
- You can control queries
    
- You can add caching
    

---

## 3️⃣ Use `.lean()` by default

Mongoose documents are **slow**.  
They have getters, setters, virtuals, change tracking.

If you don’t need to call `.save()` on the result → use `lean()`.

`const user = await UserModel.findById(id).lean();`

This returns **plain JSON** → much faster.

Rule:

> Queries for reads = `lean()`

---

## 4️⃣ Never use `.save()` in high-traffic code

This is a silent performance killer:

`user.name = "John"; await user.save();`

This:

- Reads the full document
    
- Runs change tracking
    
- Rewrites entire doc
    

Use atomic updates:

`await UserModel.updateOne(   { _id: userId },   { $set: { name: "John" } } );`

This is:

- One round trip
    
- Atomic
    
- No race conditions
    

---

## 5️⃣ Always use indexes intentionally

Mongo without indexes = full collection scan = dead system.

In schema:

`UserSchema.index({ email: 1 }, { unique: true }); UserSchema.index({ status: 1, createdAt: -1 });`

Then run:

`mongoose.syncIndexes()`

in deploy.

Never rely on Mongo auto indexes.

---

## 6️⃣ Avoid populate for anything performance-critical

`populate()` looks nice. It’s dangerous.

`Order.find().populate("user")`

This runs:  
1 query for orders  
N queries for users

Instead:

- Either embed
    
- Or manually join with `$lookup`
    
- Or pre-denormalize key fields
    

Example:

`Order {   userId,   userEmail,   userName }`

Reads become O(1) instead of O(N).

---

## 7️⃣ Use transactions only when you truly need them

MongoDB transactions are:

- Slow
    
- Locking
    
- Fragile
    

Use them only for:

- Money
    
- Inventory
    
- Irreversible state
    

Wrap with:

`const session = await mongoose.startSession(); await session.withTransaction(async () => {   ... });`

But design first to avoid them.

---

## 8️⃣ Soft deletes instead of deletes

Never use `deleteOne()`.

Use:

`{ deletedAt: Date }`

And always query:

`{ deletedAt: null }`

This:

- Avoids data loss
    
- Allows recovery
    
- Keeps referential integrity
    

---

## 9️⃣ Version your documents

Add:

`version: { type: Number, default: 1 }`

When you change schema meaning, bump version and migrate lazily.

This prevents silent corruption over time.

---

## 10️⃣ Logging & Query visibility

Enable in non-prod:

`mongoose.set("debug", true);`

You should know:

- Which queries are slow
    
- Which are unindexed
    
- Which are exploding in count
    

---
