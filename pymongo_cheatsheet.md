# PyMongo Cheatsheet

A quick reference guide for MongoDB operations using PyMongo (Python MongoDB driver).

## Table of Contents

- [Installation](#installation)
- [Connection](#connection)
  - [Basic Connection](#basic-connection)
  - [Connection with Authentication](#connection-with-authentication)
  - [MongoDB Atlas Connection](#mongodb-atlas-connection)
  - [Connection with Options](#connection-with-options)
  - [Close Connection](#close-connection)
- [Create Operations (Insert)](#create-operations-insert)
  - [Insert One Document](#insert-one-document)
  - [Insert Many Documents](#insert-many-documents)
  - [Insert with Custom _id](#insert-with-custom-_id)
- [Read Operations (Query)](#read-operations-query)
  - [Find One Document](#find-one-document)
  - [Find All Documents](#find-all-documents)
  - [Query Operators](#query-operators)
  - [Projection (Select Fields)](#projection-select-fields)
  - [Sorting](#sorting)
  - [Limit and Skip](#limit-and-skip)
  - [Count Documents](#count-documents)
  - [Distinct Values](#distinct-values)
- [Update Operations](#update-operations)
  - [Update One Document](#update-one-document)
  - [Update Many Documents](#update-many-documents)
  - [Update Operators](#update-operators)
  - [Array Update Operators](#array-update-operators)
  - [Replace One Document](#replace-one-document)
  - [Upsert (Update or Insert)](#upsert-update-or-insert)
  - [Find and Modify](#find-and-modify)
- [Delete Operations](#delete-operations)
  - [Delete One Document](#delete-one-document)
  - [Delete Many Documents](#delete-many-documents)
- [Aggregation](#aggregation)
  - [Basic Aggregation Pipeline](#basic-aggregation-pipeline)
  - [Common Aggregation Operators](#common-aggregation-operators)
- [Indexes](#indexes)
  - [Create Index](#create-index)
  - [List Indexes](#list-indexes)
  - [Drop Index](#drop-index)
- [Bulk Operations](#bulk-operations)
- [Transactions (MongoDB 4.0+)](#transactions-mongodb-40)
- [Common Patterns](#common-patterns)
  - [Pagination](#pagination)
  - [Check if Document Exists](#check-if-document-exists)
  - [Get or Create](#get-or-create)
  - [Safe Delete with Return](#safe-delete-with-return)
- [Error Handling](#error-handling)
- [Tips](#tips)

## Installation

```bash
pip install pymongo
```

## Connection

### Basic Connection

```python
from pymongo import MongoClient

# Local connection
client = MongoClient('localhost', 27017)
# Or using URI
client = MongoClient('mongodb://localhost:27017/')

# Access database
db = client['database_name']
# Or
db = client.database_name

# Access collection
collection = db['collection_name']
# Or
collection = db.collection_name
```

### Connection with Authentication

```python
# URI with credentials
client = MongoClient('mongodb://username:password@localhost:27017/')

# Or specify auth separately
client = MongoClient(
    host='localhost',
    port=27017,
    username='user',
    password='pass',
    authSource='admin'
)
```

### MongoDB Atlas Connection

```python
# Atlas connection string
client = MongoClient('mongodb+srv://username:password@cluster.mongodb.net/')
```

### Connection with Options

```python
client = MongoClient(
    'mongodb://localhost:27017/',
    maxPoolSize=50,
    serverSelectionTimeoutMS=5000,
    connectTimeoutMS=10000
)
```

### Close Connection

```python
client.close()
```

## Create Operations (Insert)

### Insert One Document

```python
# Insert single document
document = {
    "name": "John Doe",
    "age": 30,
    "email": "john@example.com"
}

result = collection.insert_one(document)
print(f"Inserted ID: {result.inserted_id}")
```

### Insert Many Documents

```python
# Insert multiple documents
documents = [
    {"name": "Alice", "age": 25, "city": "NYC"},
    {"name": "Bob", "age": 35, "city": "LA"},
    {"name": "Carol", "age": 28, "city": "Chicago"}
]

result = collection.insert_many(documents)
print(f"Inserted IDs: {result.inserted_ids}")
```

### Insert with Custom _id

```python
document = {
    "_id": "custom_id_123",
    "name": "Jane",
    "status": "active"
}

collection.insert_one(document)
```

## Read Operations (Query)

### Find One Document

```python
# Find first matching document
document = collection.find_one({"name": "John Doe"})
print(document)

# Find by _id
from bson.objectid import ObjectId
document = collection.find_one({"_id": ObjectId("507f1f77bcf86cd799439011")})
```

### Find All Documents

```python
# Find all documents
cursor = collection.find()
for doc in cursor:
    print(doc)

# Find with filter
cursor = collection.find({"age": {"$gte": 30}})
for doc in cursor:
    print(doc)
```

### Query Operators

```python
# Greater than / Less than
collection.find({"age": {"$gt": 25}})  # greater than
collection.find({"age": {"$gte": 25}}) # greater than or equal
collection.find({"age": {"$lt": 30}})  # less than
collection.find({"age": {"$lte": 30}}) # less than or equal

# Not equal
collection.find({"status": {"$ne": "inactive"}})

# In / Not in
collection.find({"city": {"$in": ["NYC", "LA", "Chicago"]}})
collection.find({"city": {"$nin": ["Boston", "Miami"]}})

# AND condition (implicit)
collection.find({"age": {"$gte": 25}, "city": "NYC"})

# OR condition
collection.find({"$or": [{"age": {"$lt": 25}}, {"city": "LA"}]})

# EXISTS
collection.find({"email": {"$exists": True}})

# Regex
collection.find({"name": {"$regex": "^John"}})
```

### Projection (Select Fields)

```python
# Include specific fields (exclude _id explicitly if needed)
collection.find({}, {"name": 1, "age": 1, "_id": 0})

# Exclude specific fields
collection.find({}, {"password": 0, "internal_id": 0})
```

### Sorting

```python
# Sort ascending
collection.find().sort("age", 1)  # or pymongo.ASCENDING

# Sort descending
collection.find().sort("age", -1)  # or pymongo.DESCENDING

# Multiple sort fields
collection.find().sort([("age", -1), ("name", 1)])
```

### Limit and Skip

```python
# Limit results
collection.find().limit(10)

# Skip results (pagination)
collection.find().skip(20).limit(10)

# Combine sort, skip, limit
collection.find({"status": "active"}).sort("created_at", -1).skip(0).limit(20)
```

### Count Documents

```python
# Count all documents
count = collection.count_documents({})

# Count with filter
count = collection.count_documents({"age": {"$gte": 30}})

# Estimated count (faster but less accurate)
count = collection.estimated_document_count()
```

### Distinct Values

```python
# Get distinct values for a field
cities = collection.distinct("city")
print(cities)  # ['NYC', 'LA', 'Chicago']
```

## Update Operations

### Update One Document

```python
# Update first matching document
result = collection.update_one(
    {"name": "John Doe"},  # filter
    {"$set": {"age": 31, "status": "updated"}}  # update
)

print(f"Matched: {result.matched_count}, Modified: {result.modified_count}")
```

### Update Many Documents

```python
# Update all matching documents
result = collection.update_many(
    {"status": "pending"},  # filter
    {"$set": {"status": "active"}}  # update
)

print(f"Matched: {result.matched_count}, Modified: {result.modified_count}")
```

### Update Operators

```python
# $set - Set field value
collection.update_one({"_id": doc_id}, {"$set": {"age": 30}})

# $unset - Remove field
collection.update_one({"_id": doc_id}, {"$unset": {"temp_field": ""}})

# $inc - Increment numeric value
collection.update_one({"_id": doc_id}, {"$inc": {"views": 1}})

# $mul - Multiply numeric value
collection.update_one({"_id": doc_id}, {"$mul": {"price": 1.1}})

# $rename - Rename field
collection.update_one({"_id": doc_id}, {"$rename": {"old_name": "new_name"}})

# $min - Update if new value is less than current
collection.update_one({"_id": doc_id}, {"$min": {"lowest_score": 50}})

# $max - Update if new value is greater than current
collection.update_one({"_id": doc_id}, {"$max": {"highest_score": 100}})

# $currentDate - Set to current date
collection.update_one({"_id": doc_id}, {"$currentDate": {"last_modified": True}})
```

### Array Update Operators

```python
# $push - Add element to array
collection.update_one(
    {"_id": doc_id},
    {"$push": {"tags": "python"}}
)

# $push with $each - Add multiple elements
collection.update_one(
    {"_id": doc_id},
    {"$push": {"tags": {"$each": ["mongodb", "database"]}}}
)

# $addToSet - Add if not exists
collection.update_one(
    {"_id": doc_id},
    {"$addToSet": {"tags": "unique_tag"}}
)

# $pull - Remove matching elements
collection.update_one(
    {"_id": doc_id},
    {"$pull": {"tags": "old_tag"}}
)

# $pop - Remove first (-1) or last (1) element
collection.update_one(
    {"_id": doc_id},
    {"$pop": {"tags": 1}}
)
```

### Replace One Document

```python
# Replace entire document (keeps _id)
new_document = {
    "name": "John Smith",
    "age": 32,
    "email": "john.smith@example.com"
}

result = collection.replace_one(
    {"name": "John Doe"},
    new_document
)
```

### Upsert (Update or Insert)

```python
# Insert if not exists, update if exists
result = collection.update_one(
    {"email": "new@example.com"},
    {"$set": {"name": "New User", "status": "active"}},
    upsert=True
)

if result.upserted_id:
    print(f"Inserted new document with ID: {result.upserted_id}")
```

### Find and Modify

```python
# Find one and update (returns original or modified document)
doc = collection.find_one_and_update(
    {"name": "John"},
    {"$inc": {"counter": 1}},
    return_document=True  # ReturnDocument.AFTER for new, .BEFORE for old
)

# Find one and replace
doc = collection.find_one_and_replace(
    {"name": "John"},
    {"name": "John", "age": 30, "updated": True}
)

# Find one and delete
doc = collection.find_one_and_delete({"name": "John"})
```

## Delete Operations

### Delete One Document

```python
# Delete first matching document
result = collection.delete_one({"name": "John Doe"})
print(f"Deleted count: {result.deleted_count}")
```

### Delete Many Documents

```python
# Delete all matching documents
result = collection.delete_many({"status": "inactive"})
print(f"Deleted count: {result.deleted_count}")

# Delete all documents in collection
result = collection.delete_many({})
```

## Aggregation

### Basic Aggregation Pipeline

```python
# Aggregation example
pipeline = [
    {"$match": {"status": "active"}},  # Filter
    {"$group": {                        # Group
        "_id": "$city",
        "count": {"$sum": 1},
        "avg_age": {"$avg": "$age"}
    }},
    {"$sort": {"count": -1}},          # Sort
    {"$limit": 5}                       # Limit
]

results = collection.aggregate(pipeline)
for result in results:
    print(result)
```

### Common Aggregation Operators

```python
# $match - Filter documents
{"$match": {"age": {"$gte": 25}}}

# $project - Select/transform fields
{"$project": {"name": 1, "age": 1, "adult": {"$gte": ["$age", 18]}}}

# $group - Group and aggregate
{"$group": {
    "_id": "$category",
    "total": {"$sum": "$amount"},
    "average": {"$avg": "$score"},
    "max": {"$max": "$value"},
    "min": {"$min": "$value"}
}}

# $sort - Sort results
{"$sort": {"field": 1}}  # 1 for ascending, -1 for descending

# $limit - Limit results
{"$limit": 10}

# $skip - Skip results
{"$skip": 20}

# $unwind - Deconstruct array field
{"$unwind": "$tags"}

# $lookup - Join with another collection
{"$lookup": {
    "from": "other_collection",
    "localField": "field_id",
    "foreignField": "_id",
    "as": "joined_data"
}}
```

## Indexes

### Create Index

```python
# Single field index
collection.create_index("email")

# Compound index
collection.create_index([("name", 1), ("age", -1)])

# Unique index
collection.create_index("email", unique=True)

# Text index
collection.create_index([("description", "text")])
```

### List Indexes

```python
indexes = collection.list_indexes()
for index in indexes:
    print(index)
```

### Drop Index

```python
collection.drop_index("email_1")
```

## Bulk Operations

```python
from pymongo import InsertOne, DeleteOne, ReplaceOne, UpdateOne

# Bulk write operations
requests = [
    InsertOne({"name": "Alice", "age": 25}),
    UpdateOne({"name": "Bob"}, {"$set": {"age": 36}}),
    DeleteOne({"name": "Charlie"}),
    ReplaceOne({"name": "David"}, {"name": "David", "age": 40})
]

result = collection.bulk_write(requests)
print(f"Inserted: {result.inserted_count}, Modified: {result.modified_count}")
```

## Transactions (MongoDB 4.0+)

```python
# Start a session
with client.start_session() as session:
    with session.start_transaction():
        collection.insert_one({"name": "Alice"}, session=session)
        collection.update_one(
            {"name": "Bob"},
            {"$inc": {"balance": -100}},
            session=session
        )
        # Automatically commits or aborts on exception
```

## Common Patterns

### Pagination

```python
def get_page(page_num, page_size=20):
    skip = (page_num - 1) * page_size
    return collection.find().skip(skip).limit(page_size)
```

### Check if Document Exists

```python
exists = collection.count_documents({"email": "test@example.com"}, limit=1) > 0
```

### Get or Create

```python
doc = collection.find_one({"email": email})
if not doc:
    doc = {"email": email, "name": name}
    collection.insert_one(doc)
```

### Safe Delete with Return

```python
deleted_doc = collection.find_one_and_delete({"_id": doc_id})
if deleted_doc:
    print(f"Deleted: {deleted_doc}")
```

## Error Handling

```python
from pymongo.errors import DuplicateKeyError, ConnectionFailure

try:
    collection.insert_one({"_id": 1, "name": "Test"})
except DuplicateKeyError:
    print("Document with this _id already exists")
except ConnectionFailure:
    print("Failed to connect to MongoDB")
```

## Tips

- Always use connection pooling (default in PyMongo)
- Create indexes for frequently queried fields
- Use projection to limit returned fields
- Use `explain()` to analyze query performance: `collection.find().explain()`
- Close connections when done or use context managers
- Use bulk operations for multiple writes
- Consider using aggregation for complex queries instead of loading all data into Python
