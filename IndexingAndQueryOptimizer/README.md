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

-------------------------------------

## Basic Index Bounds

### Find One Document

#### define query and index

```javascript
db.c.find({x: 6}).limit(1);
db.c.ensureIndex({x: 1});
```

#### collection have only one

![Alt text](./8-1024.jpg)

-------------------------------------

![Alt text](./9-1024.jpg)

-------------------------------------

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

-------------------------------------

![Alt text](./13-1024.jpg)

-------------------------------------

#### Now we have duplicate x values

![Alt text](./14-1024.jpg)

-------------------------------------

![Alt text](./15-1024.jpg)

-------------------------------------

### Equality Match

#### define query and index

```javascript
db.c.find({x: 6});
db.c.ensureIndex({x: 1});
```

![Alt text](./17-1024.jpg)

-------------------------------------

![Alt text](./18-1024.jpg)

-------------------------------------

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

-------------------------------------

### Full Document Matcher

#### define query and index

```
db.c.find({x: 6, y: 1});
db.c.ensureIndex({x: 1});
```

![Alt text](./23-1024.jpg)

-------------------------------------

![Alt text](./24-1024.jpg)

-------------------------------------

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

-------------------------------------

### Range Match

#### define query and index

```javascript
db.c.find({x: {$gte: 4, $lte: 7}});
db.c.ensureIndex({x: 1});
```

![Alt text](./28-1024.jpg)

-------------------------------------

![Alt text](./29-1024.jpg)

-------------------------------------

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

-------------------------------------

### Exclusive Range Match

#### define query and index

```javascript
db.c.find(x: {$gt: 4, $lt: 7});
db.c.ensureIndex({x: 1});
```

![Alt text](./34-1024.jpg)

-------------------------------------

![Alt text](./35-1024.jpg)

-------------------------------------

Explain doesn't indicate that the range is exclusive

![Alt text](./36-1024.jpg)

-------------------------------------

But index keys matching the range bounds are not scanned because the bounds are exclusive.

![Alt text](./37-1024.jpg)

-------------------------------------

![Alt text](./38-1024.jpg)

-------------------------------------

### Multikeys

#### define query and index

```javascript
db.c.find({x: {$gt: 7}});
db.c.ensureIndex({x: 1});
```

![Alt text](./40-1024.jpg)

-------------------------------------

![Alt text](./41-1024.jpg)

-------------------------------------

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

-------------------------------------

![Alt text](./44-1024.jpg)

-------------------------------------

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

#### db.c.find({x: {'\$gt': 4, '\$lt': 7}})

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

-------------------------------------

![Alt text](./53-1024.jpg)

-------------------------------------

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

-------------------------------------

![Alt text](./56-1024.jpg)

-------------------------------------

### All Match

#### define query and index

```javascript
db.c.find({x: {$all: [3, 6]}});
db.c.ensureIndex({x: 1});
```

![Alt text](./58-1024.jpg)

-------------------------------------

![Alt text](./59-1024.jpg)

-------------------------------------

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

-------------------------------------

![Alt text](./62-1024.jpg)

-------------------------------------

### Limit

#### define query and index

```javascript
db.c.find({x: {$lt: 6}, y: 3}).limit(3);
db.c.ensureIndex({x: 1});
```

![Alt text](./64-1024.jpg)

-------------------------------------

![Alt text](./65-1024.jpg)

-------------------------------------

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

-------------------------------------

### Skip

```javascript
db.c.find({x: {$lt: 6}, y: 3}).skip(3);
db.c.ensureIndex({x: 1});
```

![Alt text](./69-1024.jpg)

-------------------------------------

![Alt text](./70-1024.jpg)

-------------------------------------

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

-------------------------------------

### Sort

```javascript
db.c.find({x: {$lt: 6}).sort({x: 1});
db.c.ensureIndex({x: 1});
```

![Alt text](./74-1024.jpg)

-------------------------------------

![Alt text](./75-1024.jpg)

-------------------------------------

`'cursor': 'BtreeCursor x_1'`

```javascript
db.c.find({x: {$lt: 6}).sort({y: 1});
db.c.ensureIndex({x: 1});
```

![Alt text](./78-1024.jpg)

-------------------------------------

![Alt text](./79-1024.jpg)

-------------------------------------

Results are sorted on the fly to match requested order.
The scanAndOrder field is only printed when its value is true.

![Alt text](./80-1024.jpg)

-------------------------------------

### Sort and scanAndOrder

- With "scanAndOrder" sort, all documents must be touched even if there is a limit spec.
- With scanAndOrder, sorting is performed in memory and the memory footprint is constrained by the limit spec if present.

### Count

```javascript
db.c.count({x: {$gte: 4, $lte: 7}});
db.c.ensureIndex({x: 1});
```

![Alt text](./83-1024.jpg)

-------------------------------------

We're just counting keys here, not loading the full documents.

![Alt text](./84-1024.jpg)

-------------------------------------

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

-------------------------------------

![Alt text](./87-1024.jpg)

-------------------------------------

![Alt text](./88-1024.jpg)

-------------------------------------

```javascript
{
    'isMultiKey': false,
    'indexOnly': true
}
```

![Alt text](./90-1024.jpg)

-------------------------------------

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

-------------------------------------

![Alt text](./95-1024.jpg)

-------------------------------------

![Alt text](./96-1024.jpg)

-------------------------------------

![Alt text](./97-1024.jpg)

-------------------------------------

- We track the set of documents that have been updated in the course of the current operation so they are only undated once.

## Compound Key Index Bounds

### Two Equality Bounds

```javascript
db.c.find({x: 5, y: 'c'});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./101-1024.jpg)

-------------------------------------

![Alt text](./102-1024.jpg)

-------------------------------------

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

-------------------------------------

### Equality and Set

```javascript
db.c.find({x: 5, y: {$in: ['c', 'f']}});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./107-1024.jpg)

-------------------------------------

![Alt text](./108-1024.jpg)

-------------------------------------

```javascript
{
    'indexBounds': {
        'x': [
            [5, 5]
        ],
        'y': [
            ['c', 'c'],
            ['f', 'f']
        ]
    },
    'nscanned': 3,
    'nscannedObjects': 2,
    'n': 2
}
```

![Alt text](./111-1024.jpg)

-------------------------------------

### Equality and Range

```javascript
db.c.find({x: 5, y: {$gte: 'd'}});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./113-1024.jpg)

-------------------------------------

![Alt text](./114-1024.jpg)

-------------------------------------

```javascript
{
    'indexBounds': {
        'x': [
            [5, 5]
        ],
        'y': [
            'd',
            {}
        ]
    },
    'nscanned': 2,
    'nscannedObjects': 2,
    'n': 2
}
```

![Alt text](./117-1024.jpg)

-------------------------------------

### Two Set Bounds

```javascript
db.c.find({x: {$in: [5, 9]}, y: {$in: ['c', 'f']}});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./119-1024.jpg)

-------------------------------------

![Alt text](./120-1024.jpg)

-------------------------------------

```javascript
{
    'indexBounds': {
        'x': [
            [5, 5],
            [9, 9]
        ],
        'y': [
            ['c', 'c'],
            ['f', 'f']
        ]
    },
    'nscanned': 5,
    'nscannedObjects': 3,
    'n': 3
}
```

![Alt text](./123-1024.jpg)

-------------------------------------

### Set and Range

```
db.c.find({x: {$in: [5, 9]}, y: {$lte: 'd'}});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./125-1024.jpg)

-------------------------------------

![Alt text](./126-1024.jpg)

-------------------------------------

```javascript
{
    'indexBounds': {
        'x': [
            [5, 5],
            [9, 9]
        ],
        'y': [
            ['', 'd']
        ]
    },
    'nscanned': 5,
    'nscannedObjects': 3,
    'n': 3
}
```

### Range and Equality

```
db.c.find({x: {$gte: 4}, y: 'c'});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./130-1024.jpg)

-------------------------------------

![Alt text](./131-1024.jpg)

-------------------------------------

```javascript
{
    'indexBounds': {
        'x': [
            [4, 1.7976931348623157e+30]
        ],
        'y': [
            ['c', 'c']
        ]
    },
    'nscanned': 7,
    'nscannedObjects': 2,
    'n': 2
}
```

High nscanned because every distinct value of x must be checked.

![Alt text](./133-1024.jpg)

-------------------------------------

![Alt text](./134-1024.jpg)

-------------------------------------

Every distinct value of x must be checked.

![Alt text](./135-1024.jpg)

-------------------------------------

### Range and Set

```
db.c.find({x: {$gte: 4}, y: {$in: ['c', 'a']}});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./137-1024.jpg)

-------------------------------------

![Alt text](./138-1024.jpg)

-------------------------------------

```javascript
{
    'indexBounds': {
        'x': [
            [4, 1.7976931348623157e+30]
        ],
        'y': [
            ['a', 'a'],
            ['c', 'c']
        ]
    },
    'nscanned': 7,
    'nscannedObjects': 3,
    'n': 3
}
```

![Alt text](./141-1024.jpg)

-------------------------------------

Every distinct value of x must be checked for y values 'a' and 'c'

![Alt text](./142-1024.jpg)

-------------------------------------

### Two Ranges (2D Box)

```javascript
db.c.find({x: {$gte: 3, $lte: 7}, y: {$gte: 'c', $lte: 'f'}});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./144-1024.jpg)

-------------------------------------

![Alt text](./145-1024.jpg)

-------------------------------------

![Alt text](./146-1024.jpg)

-------------------------------------

```javascript
{
    'indexBounds': {
        'x': [
            [3, 7]
        ],
        'y': [
            ['c', 'f']
        ]
    },
    'nscanned': 6,
    'nscannedObjects': 4,
    'n': 4
}
```

![Alt text](./149-1024.jpg)

-------------------------------------

![Alt text](./150-1024.jpg)

-------------------------------------

## $or

### Disjoint $or Criteria

```javascript
db.c.find({$or: [{x: 5}, {y: 'd'}]});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./153-1024.jpg)

-------------------------------------

```javascript
> db.c.find({$or: [{x: 5}, {y: 'd'}]})
{
    'clauses': [
        {
            'cursor': 'BtreeCursor x_1',
            'nscanned': 2,
            'nscannedObjects': 2,
            'n': 2,
            'millis': 0,
            'nYield': 0,
            'nChunkSkips': 0,
            'isMultiKey': false,
            'indexOnly': false,
            'indexBounds': {
                'x': [
                    [5, 5]
                ]
            }
        },
        {
            'cursor': 'BtreeCursor y_1',
            'nscanned': 2,
            'nscannedObjects': 2,
            'n': 1,
            'millis': 1,
            'nYield': 0,
            'nChunkSkips': 0,
            'isMultiKey': false,
            'indexOnly': false,
            'indexBounds': {
                'y': [
                    ['d', 'd']
                ]
            }
        }
    ],
    'nscanned': 4,
    'nscannedObjects': 4,
    'n': 3,
    'millis': 1
}
```

![Alt text](./156-1024.jpg)

-------------------------------------

```javascript
{
    'nscanned': 4,
    'nscannedObjects': 4,
    'n': 3,
    'millis': 1
}
```

![Alt text](./158-1024.jpg)

-------------------------------------

We have already scanned the x index for x:5.
So this document was returned already.
We don't return it again.

![Alt text](./159-1024.jpg)

-------------------------------------

### Unindexed $or Clause

```javascript
db.c.find({$or: [{x: 5}, {y: 'd'}]});
db.c.ensureIndex({x: 1}); // no index on y
```

Since y is not indexed, we must do a full collection scan to match y:'d'.
Since a full scan is required, we don't use the index on x to match x:5.

![Alt text](./161-1024.jpg)

-------------------------------------

### Eliminated $or Clause

```javascript
db.c.find({$or: [{x: {$gt: 2, $lt: 6}}, {x: 5}]});
db.c.ensureIndex({x: 1});
```

![Alt text](./163-1024.jpg)

-------------------------------------

The index range of the second clause is included in the index range of the first clause, so we use the first index range only.

![Alt text](./164-1024.jpg)

-------------------------------------

### Eliminated $or Clause with Differing Unindexed Criteria

```javascript
db.c.find({$or: [{x: {$gt: 2, $lt: 6}, y: 'c'}, {x: 5, y: 'd'}]});
db.c.ensureIndex({x: 1});
```

![Alt text](./166-1024.jpg)

-------------------------------------

![Alt text](./167-1024.jpg)

-------------------------------------

The index range for the first clause contains the index range for the second clause, so all matching is done using the index range for the first clause.

![Alt text](./168-1024.jpg)

-------------------------------------

### Overlapping $or Clauses

```javascript
db.c.find({$or: [{x: {$gt: 2, $lt: 6}}, {x: {$gt: 4, $lt: 7}}]});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./170-1024.jpg)

-------------------------------------

```javascript
> db.c.find({$or: [{x: {$gt: 2, $lt: 6}}, {x: {$gt: 4, $lt: 7}}]});
{
    'clauses': [
        {
            'cursor': 'BtreeCursor x_1',
            'nscanned': 3,
            'nscannedObjects': 3,
            'n': 3,
            'millis': 0,
            'nYield': 0,
            'nChunkSkips': 0,
            'isMultiKey': false,
            'indexOnly': false,
            'indexBounds': {
                'x': [
                    [2, 6]
                ]
            }
        },
        {
            'cursor': 'BtreeCursor x_1',
            'nscanned': 1,
            'nscannedObjects': 1,
            'n': 1,
            'millis': 1,
            'nYield': 0,
            'nChunkSkips': 0,
            'isMultiKey': false,
            'indexOnly': false,
            'indexBounds': {
                'x': [
                    [6, 7]
                ]
            }
        }
    ],
    'nscanned': 4,
    'nscannedObjects': 4,
    'n': 4,
    'millis': 1
}
```

The index range scanned for the previous clause is removed.

![Alt text](./173-1024.jpg)

-------------------------------------

![Alt text](./174-1024.jpg)

-------------------------------------

### 2D Overlapping $or Clauses

```javascript
db.c.find({
    $or: [
        {x: {$gt: 2, $lt: 6}, y: {$gt: 'b', $lt: 'f'}},
        {x: {$gt: 4, $lt: 7}, y: {$gt: 'b', $lt: 'e'}}
    ]
});
db.c.ensureIndex({x: 1, y: 1});
```

![Alt text](./176-1024.jpg)

-------------------------------------

```javascript
> db.c.find({
    $or: [
        {x: {$gt: 2, $lt: 6}, y: {$gt: 'b', $lt: 'f'}},
        {x: {$gt: 4, $lt: 7}, y: {$gt: 'b', $lt: 'e'}}
    ]
});

{
    'clauses': [
        {
            'cursor': 'BtreeCursor x_1_y_1',
            'nscanned': 4,
            'nscannedObjects': 3,
            'n': 3,
            'millis': 1,
            'nYield': 0,
            'nChunkSkips': 0,
            'isMultiKey': false,
            'indexOnly': false,
            'indexBounds': {
                'x': [
                    [2, 6]
                ],
                'y': [
                    ['b', 'f']
                ]
            }
        },
        {
            'cursor': 'BtreeCursor x_1_y_1',
            'nscanned': 0,
            'nscannedObjects': 0,
            'n': 0,
            'millis': 1,
            'nYield': 0,
            'nChunkSkips': 0,
            'isMultiKey': false,
            'indexOnly': false,
            'indexBounds': {
                'x': [
                    [6, 7]
                ],
                'y': [
                    ['b', 'e']
                ]
            }
        }
    ],
    'nscanned': 4,
    'nscannedObjects': 3,
    'n': 3
}
```

The index range scanned for the previous clause is removed.

![Alt text](./179-1024.jpg)

-------------------------------------

We only have to scan the remainder here.

![Alt text](./180-1024.jpg)

-------------------------------------

### Overlapping $or Clauses

- Rule of thumb for n dimensions: We subtract earlier clause boxes from current box when the result is a/some box(es).

![Alt text](./181-1024.jpg)

-------------------------------------

![Alt text](./182-1024.jpg)

-------------------------------------

### $or TODO

- Use indexes on $or fields to satisfy a sort specification SERVER-1205
- Use full query optimizer to select $or clause indexes in getMore SERVER-1215
- Improve index range elimination (handling some cases where remainder is not a box)

## Automatic Index Selection (Query Optimizer)

### Optimal Index

- find({x: 5})
    - index {x: 1}
    - index: {x: 1, y: 1}
- find({x: 5}).sort({y: 1})
    - index {x: 1, y: 1}
- find({}).sort({x: 1})
    - index {x: 1}
- find({x: {$gt: 1, $lt: 7}}).sort({x: 1})
    - index {x: 1}

- Rule of Thumb
    - No scanAndOrder
    - All fields with index useful constraints are indexed
    - If there is a range or sort it is the last field of the index used to resolve the query
- If multiple optimal indexes exist, one chosen arbitrarily.
- These same criteria are useful when you are designing your indexes.

### Multiple Candidate Indexes

- find({x: 4, y: 'a'})
    - index {x: 1} or {y: 1} ?
- find({x: 4}).sort({y: 1})
    - index {x: 1} or {y: 1}?
    - Note: {x: 1, y: 1} is optimal
- find({x: {\$gt: 2, \$lt: 7}, y: {\$gt: 'a', \$lt: 'f'}})
    - index {x: 1, y: 1} or {y: 1, x: 1}?
- The only index selection criterion is nscanned
- find({x: 4, y: 'a'})
    - Index {x: 1} or {y: 1} ?
    - If fewer documents match {y: 'a'} than {x: 4} then nscanned for {y: 1} will be less so we pick {y: 1}
- find({x: {\$gt: 2, \$lt: 7}, y: {\$gt: 'b', \$lt: 'f'}})
    - Index {x: 1, y: 1} or {y: 1, x: 1}?
    - If fewer distinct values of 2 < x < 7 than distinct values of 'b' < y < 'f' then {x: 1, y: 1} chosen (rule of thumb)
- The only index selection criterion is nscanned
- Pretty good, but doesn't cover every case, eg
    - Cost of scanAndOrdervs ordered index
    - Cost of loading full document vs just index key
    - Cost of scanning adjacent btree  keys vs non adjacent keys/documents

### Competing Indexes

- At most one query plan per index
- Run in interleaved fashion
- Plans kept in a priority queue ordered by nscanned. We always continue progress on plan with lowest nscanned.
- Run until one plan returns all results or enough results to satisfy the initial query request (based on soft limit spec / data size requirement for initial query).
- We only allow plans to compete in initial query. In getMore, we continue reading from the index cursor established by the initial query.

### "Learning" a Query Plan

- When an index is chosen for a query the query's "pattern" and nscanned are recorded
    - find({x: 3, y: 'c'})
        - {Pattern: {x: 'equality', y: 'equality'}, Index: {x: 1}, nscanned: 50}
    - find({x: {\$gt: 5}, y: {\$lt: 'z'}})
        - {Pattern: {x: 'gt bound', y: 'lt bound'}, Index: {y: 1}, nscanned: 500}
- When a new query matches the same pattern, the same query plan is used
    - find({x: 5, y: 'z'})
        - Use index {x: 1}
    - find({x: {\$gt: 20}, y: {\$lt: 'b'}})
        - Use index {y: 1}

### "Un-Learning" a Query Plan

- 100 writes to the collection
- Indexes added / removed

### Bad Plan Insurance

- If nscanned for a new query using a recorded plan is much worse than the recorded nscanned for an earlier query with the same pattern, we start interleaving other plans with the current plan.
- Currently "much worse" means 10x

### Query Planner

- Ad hoc heuristics in some cases
- Seem to work decently in practice

### Feedback

- Large and small scale optimizer features are generaly prioritized based on user input.
- Please use jira to request new features and vote on existing feature requests.

### Thanks!

Feature Requests
jira.mongodb.org

Support
groups.google.com/group/mongodb-user

Next up:
Sharding Details with Eliot
