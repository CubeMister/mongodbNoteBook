# MongoDB Indexing and Query Optimizer Details

@(Mongodb)[Indexing|Optimizer]

`链接地址`：[http://www.slideshare.net/mongodb/mongodb-indexing-the-details](http://www.slideshare.net/mongodb/mongodb-indexing-the-details)

[TOC]

## What will we cover?

- Many details of how indexing and the query optimizer work
- A full understanding of these details is not required to use mongo, but this knowledge can be helpful when making optimizations.
- We'll discuss functionality of Mongo 1.8 (for our purposes pretty similar to 1.6 and almost identical to 1.7 edge).
- Much of the material will be presented through examples.
- Diagrams are to aid understanding - some details will be left out.
- Basic index bounds
- Compound key index bounds
- Or queries
- Automatic index selection
- We're going to try and cover this material interactively - please volunteer your thoughts on what mongo should do in given scenarios when I ask.
- Pertinent questions are welcome, but please keep off topic or specialized questions until the end so we don't lose momentum.

## Btree（just a conceptual diagram）

![Alt text](./mongodb-indexing-the-details-5-1024.jpg)

## Basic Index Bounds

### Find One Document

#### define query and index

```javascript
db.c.find({x: 6}).limit(1);
db.c.ensureIndex({x: 1});
```

#### collection have only one

![Alt text](./8-1024.jpg)

![Alt text](./9-1024.jpg)

```javascript
{
    'indexBounds': {
        'x': [
            [6, 6]
        ]
    },
    'nscanned': 1, // 扫描的行数
    'nscannedObjects': 1, // 扫描的对象
    'n': 1 // 返回的行数
}
```

![Alt text](./12-1024.jpg)

![Alt text](./13-1024.jpg)

#### Now we have duplicate x values

![Alt text](./14-1024.jpg)

![Alt text](./15-1024.jpg)

### Equality Match

#### define query and index

```javascript
db.c.find({x: 6});
db.c.ensureIndex({x: 1});
```

![Alt text](./17-1024.jpg)

![Alt text](./18-1024.jpg)

```javascript
{
    'indexBounds': {
        'x': [
            [6, 6]
        ]
    },
    'nscanned': 3,
    'nscannedObjects': 3,
    'n': 3
}
```

![Alt text](./21-1024.jpg)

### Full Document Matcher

#### define query and index

```
db.c.find({x: 6, y: 1});
db.c.ensureIndex({x: 1});
```

![Alt text](./23-1024.jpg)

![Alt text](./24-1024.jpg)

```javascript
{
    'indexBounds': {
        'x': [
            [6, 6]
        ]
    },
    'nscanned': 3,
    'nscannedObjects': 3,
    'n': 1
}
```

Documents for all matching keys scanned, but only one document matched on non index keys.

![Alt text](./26-1024.jpg)

### Range Match

#### define query and index

```javascript
db.c.find({x: {$gte: 4, $lte: 7}});
db.c.ensureIndex({x: 1});
```

![Alt text](./28-1024.jpg)

![Alt text](./29-1024.jpg)

```javascript
{
    'indexBounds': {
        'x': [
            [4, 7]
        ]
    },
    'nscanned': 4,
    'nscannedObjects': 4,
    'n': 4
}
```

![Alt text](./32-1024.jpg)

### Exclusive Range Match

#### define query and index

```javascript
db.c.find(x: {$gt: 4, $lt: 7});
db.c.ensureIndex({x: 1});
```

![Alt text](./34-1024.jpg)

![Alt text](./35-1024.jpg)

Explain doesn't indicate that the range is exclusive

![Alt text](./36-1024.jpg)

But index keys matching the range bounds are not scanned because the bounds are exclusive.

![Alt text](./37-1024.jpg)

![Alt text](./38-1024.jpg)

### Multikeys

#### define query and index

```javascript
db.c.find({x: {$gt: 7}});
db.c.ensureIndex({x: 1});
```

![Alt text](./40-1024.jpg)

![Alt text](./41-1024.jpg)

```javascript
{
    'indexBounds': {
        'x': [
            [7, 1.7976931348623157e+308]
        ]
    },
    'nscanned': 2,
    'nscannedObjects': 2,
    'n': 1
}
```

All keys in valid range are scanned, but the matcher rejects duplicate documents making n == 1.

![Alt text](./43-1024.jpg)

![Alt text](./44-1024.jpg)

### Range Types

#### define query and index

- Explicit inequality

#### define query

```javascript
db.c.find({x: {$gt: 4, $lt: 7}});
db.c.find({x: {$gt: 4}});
db.c.find({x: {$ne: 4}});
```

- Regular expression prefix

```javascript
db.c.find({x: /^a/});
```

- Data type

```javascript
db.c.find({x: /a/})
```

#### db.c.find({x: {\$gt: 4, \$lt: 7}})

```javascript
{
    'indexBounds': {
        'x': [
            [4, 7]
        ]
    }
}
```

#### db.c.find({x: {\$gt: 4}})

```javascript
{
    'indexBounds': {
        'x': [
            [4, 1.7976931348623157e+308]
        ]
    }
}
```

#### db.c.find({x: {\$ne: 4}})

```javascript
{
    'indexBounds': {
        'x': [
            [{
                '$minElement': 1
            }, 4],
            [4, {
                '$maxElement': 1
            }]
        ]
    }
}
```

#### db.c.find({x: /^a/})

```javascript
{
    'indexBounds': {
        'x': [
            ['a', 'b'],
            [/^a/, /^a/]
        ]
    }
}
```

#### db.c.find({x: /a/})

```javascript
{
    'indexBounds': {
        'x': [
            ['', {}],
            [/a/, /a/]
        ]
    }
}
```

### Set Match

#### define query and index

```javascript
db.c.find({x: {$in: [3, 6]}});
db.c.ensureIndex({x: 1});
```

![Alt text](./52-1024.jpg)

![Alt text](./53-1024.jpg)

```javascript
{
    'indexBounds': {
        'x': [
            [3, 3],
            [6, 6]
        ]
    },
    'nscanned': 3,
    'nscannedObjects': 2,
    'n': 2
}
```

Why is nscanned 3?
This is an algorithmic detail we'll discuss more later, but when there are disjoint ranges for a key nscanned may be higher than the number of matching keys.

![Alt text](./55-1024.jpg)

![Alt text](./56-1024.jpg)

### All Match

#### define query and index

```javascript
db.c.find({x: {$all: [3, 6]}});
db.c.ensureIndex({x: 1});
```

![Alt text](./58-1024.jpg)

![Alt text](./59-1024.jpg)

The first entry in the \$all match array is always used for index bounds. Note this may not be the least numerous indexed value in the \$all array.

```javascript
{
    'indexBounds': {
        'x': [
            [3, 3]
        ]
    },
    'nscanned': 1,
    'nscannedObjects': 1,
    'n': 1
}
```

![Alt text](./60-1024.jpg)

![Alt text](./62-1024.jpg)

### Limit

#### define query and index

```javascript
db.c.find({x: {$lt: 6}, y: 3}).limit(3);
db.c.ensureIndex({x: 1});
```

![Alt text](./64-1024.jpg)

![Alt text](./65-1024.jpg)

```javascript
{
    'indexBounds': {
        'x': [
            [-1.7976931348623157e+308, 6]
        ]
    }
}
```

Scan until three matches are found, then stop;

![Alt text](./67-1024.jpg)

### Skip

```javascript
db.c.find({x: {$lt: 6}, y: 3}).skip(3);
db.c.ensureIndex({x: 1});
```

![Alt text](./69-1024.jpg)

![Alt text](./70-1024.jpg)

```javascript
{
    'indexBounds': {
        'x': [
            [-1.7976931348623157e+308, 6]
        ]
    }
}
```

All skipped documents are scanned.

![Alt text](./72-1024.jpg)

### Sort

```javascript
db.c.find({x: {$lt: 6}).sort({x: 1});
db.c.ensureIndex({x: 1});
```

![Alt text](./74-1024.jpg)

![Alt text](./75-1024.jpg)

`'cursor': 'BtreeCursor x_1'`

```javascript
db.c.find({x: {$lt: 6}).sort({y: 1});
db.c.ensureIndex({x: 1});
```

![Alt text](./78-1024.jpg)

![Alt text](./79-1024.jpg)

Results are sorted on the fly to match requested order.
The scanAndOrder field is only printed when its value is true.

![Alt text](./80-1024.jpg)

### Sort and scanAndOrder

- With "scanAndOrder" sort, all documents must be touched even if there is a limit spec.
- With scanAndOrder, sorting is performed in memory and the memory footprint is constrained by the limit spec if present.

### Count

```javascript
db.c.count({x: {$gte: 4, $lte: 7}});
db.c.ensureIndex({x: 1});
```

![Alt text](./83-1024.jpg)

We're just counting keys here, not loading the full documents.

![Alt text](./84-1024.jpg)

- With some operators the full document must be checked. Some of these cases:
    - $all
    - $size
    - array match
    - Negation - $ne, $nin, $not, etc.
        - With current semantics, all multikey elements must match negation constraints
- Multikey de duplication works without loading full document

### Covered Indexes

Id would be returned by default, but isn't in the index so we need to exclude to return only indexed fields.

![Alt text](./86-1024.jpg)

![Alt text](./87-1024.jpg)

![Alt text](./88-1024.jpg)

```javascript
{
    'isMultiKey': false,
    'indexOnly': true
}
```

![Alt text](./90-1024.jpg)

```javascript
> db.c.find({x: 6}, {x: 1, _id: 0}).explain();

{
    'cursor': 'BtreeCursor x_1',
    'nscanned': 1,
    'nscannedObjects': 1,
    'n': 1,
    'millis': 0,
    'nYields': 0,
    'nChunkSkips': 0,
    'isMultiKey': true,
    'indexOnly': false,
    'indexBounds': {
        'x': [
            [6, 6]
        ]
    }
}
```

Currently we set isMultiKey to true the first time we save a doc where the field is a multikey array.
But when all multikey docs are removed we don't reset isMultiKey.
This can be improved.

```javascript
{
    'isMultiKey': true,
    'indexOnly': false
}
```

### Update

```javascript
db.c.find({x: {$gte: 4, $lte: 7}}, {$set: {x: 2}});
db.c.ensureIndex({x: 1});
```

![Alt text](./94-1024.jpg)

![Alt text](./95-1024.jpg)

![Alt text](./96-1024.jpg)

![Alt text](./97-1024.jpg)

- We track the set of documents that have been updated in the course of the current operation so they are only undated once.

## Compound Key Index Bounds

### Two Equality Bounds

```javascript
db.c.find({x: 5, y: 'c'});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./101-1024.jpg)

![Alt text](./102-1024.jpg)

```javascript
{
    'indexBounds': {
        'x': [
            [5, 5]
        ],
        'y': [
            ['c', 'c']
        ]
    },
    'nscanned': 1,
    'nscannedObjects': 1,
    'n': 1
}
```

![Alt text](./105-1024.jpg)

### Equality and Set

```javascript
db.c.find({x: 5, y: {$in: ['c', 'f']}});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./107-1024.jpg)

![Alt text](./108-1024.jpg)


