---
name: mongoose-guidelines
description: Strict guidelines for integrating Mongoose with NestJS following Clean Architecture and DDD. Focus on performance, domain invariants, and protection against anemic models.
---

# Mongoose Guidelines (Clean Architecture & DDD)

## 1. What I do

- I establish **Schema Design** standards to prevent MongoDB from becoming a "JSON repository".

- I implement a clear separation between **Mongoose Documents** and **Domain Entities**.

- I ensure performance through the mandatory use of `.lean()` and atomic updates.

- I define the **Repositories** structure to isolate the infrastructure from the business logic.

## 2. When to use me

- When creating new NestJS modules that use MongoDB.

- During the refactoring of services that directly access the Mongoose `Model`.

- When defining indexing and data consistency strategies.

## 3. Rules and Standards (Mandatory)

### Right vs. Wrong

- **Right:** Use `Repository` to encapsulate the `Model`.

- **Wrong:** Inject `@InjectModel()` directly into Use Cases or Domain Services.

- **Right:** Use `.lean()` in all read operations that do not require the `.save()` method.

- **Wrong:** Return heavy instances of `HydratedDocument` to upper layers.

- **Right:** Perform atomic updates with `$set`, `$inc`, or `$push`.

- **Wrong:** Fetch the document, modify it in memory, and call `.save()` in highly concurrent flows.

### Architectural Constraints

- **Invariants:** The Schema must reflect the integrity constraints (Required, Enum, Min/Max, Unique).

- **Independence:** The domain layer should not know Mongoose types (e.g., `Types.ObjectId`). Use `string` and convert it in the Adapter.

## 4. Reference Examples

### Strict and Modular Schema
```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

@Schema({ timestamps: true, collection: 'users', versionKey: '__v' })
export class UserDocument extends Document {
  @Prop({ required: true, lowercase: true, trim: true, unique: true, index: true })
  email: string;

  @Prop({ required: true, enum: ['ACTIVE', 'BLOCKED'], default: 'ACTIVE' })
  status: string;
}

export const UserSchema = SchemaFactory.createForClass(UserDocument);
UserSchema.index({ status: 1, createdAt: -1 });
```

### Repository com Injeção por Token e Lean
```typescript
import { Injectable } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';

@Injectable()
export class MongooseUserRepository implements UserRepository {
  constructor(
    @InjectModel(UserDocument.name) private readonly userModel: Model<UserDocument>
  ) {}

  async findByEmail(email: string): Promise<User | null> {
    const doc = await this.userModel.findOne({ email }).lean().exec();
    if (!doc) return null;
    
    // Mapeia Plain Object para Entidade de Domínio (Rich Domain Model)
    return UserMapper.toDomain(doc);
  }

  async updateStatus(id: string, status: string): Promise<void> {
    await this.userModel.updateOne(
      { _id: id },
      { $set: { status } }
    ).exec();
  }
}
```
