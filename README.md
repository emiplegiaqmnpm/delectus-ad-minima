[![CircleCI](https://circleci.com/gh/tandg-digital/@emiplegiaqmnpm/delectus-ad-minima/tree/master.svg?style=svg)](https://circleci.com/gh/tandg-digital/@emiplegiaqmnpm/delectus-ad-minima/tree/master) [![Coverage Status](https://coveralls.io/repos/github/tandg-digital/@emiplegiaqmnpm/delectus-ad-minima/badge.svg?branch=master)](https://coveralls.io/github/tandg-digital/@emiplegiaqmnpm/delectus-ad-minima?branch=master)

# What is @emiplegiaqmnpm/delectus-ad-minima?
@emiplegiaqmnpm/delectus-ad-minima is a plugin for the [objection.js](https://github.com/Vincit/objection.js) ORM. It's designed to allow powerful filters and aggregations on your API.

Some examples of what you can do include:

#### 1. Filtering on nested relations
For example, if you have the models _Customer_ belongsTo _City_ belongsTo _Country_, we can query all _Customers_ where the _Country_ starts with `A`.

#### 2. Eagerly loading data
Eagerly load a bunch of related data in a single query. This is useful for getting a list models e.g. _Customers_ then including all their _Orders_ in the same query.

#### 3. Aggregation and reporting
Creating quick counts and sums on a model can speed up development significantly. An example could be the _numberOfOrders_ for a _Customer_ model.

# Shortcuts

* [Changelog](doc/CHANGELOG.md)
* [Recipes](doc/RECIPES.md)
* [Aggregation](doc/AGGREGATIONS.md)

# Installation

`npm i @emiplegiaqmnpm/delectus-ad-minima --save`

> @emiplegiaqmnpm/delectus-ad-minima >= 1.0.0 is fully backwards compatible with older queries, but now supports nested [and/or filtering](#logical-expressions) as well as the new objection.js object notation. The 1.0.0 denotation was used due to these changes and the range of query combinations possible. In later major versions of @emiplegiaqmnpm/delectus-ad-minima, the top level "where" and "require" filters will be deprecated.

# Usage

The filtering library can be applied onto every _findAll_ REST endpoint e.g. `GET /api/{Model}?filter={"limit": 1}`

A typical express route handler with a filter applied:
```js
const { buildFilter } = require('@emiplegiaqmnpm/delectus-ad-minima');
const { Customer } = require('./models');

app.get('/Customers', function(req, res, next) {
  buildFilter(Customer)
    .build(JSON.parse(req.query.filter))
    .then(customers => res.send(customers))
    .catch(next);
});
```

Available filter properties include:
```js
// GET /api/Customers
{
  // Filtering and eager loading
  "eager": {
    // Top level $where filters on the root model
    "$where": {
      "firstName": "John"
      "profile.isActivated": true,
      "city.country": { "$like": "A" }
    },
    // Nested $where filters on each related model
    "orders": {
      "$where": {
        "state.isComplete": true
      },
      "products": {
        "$where": {
          "category.name": { "$like": "A" }
        }
      }
    }
  },
  // An objection.js order by expression
  "order": "firstName desc",
  "limit": 10,
  "offset": 10,
  // An array of dot notation fields to select on the root model and eagerly loaded models
  "fields": ["firstName", "lastName", "orders.code", "products.name"]
}
```

> The `where` operator from < v1.0.0 is still available and can be combined with the `eager` string type notation. The same is applicable to the `require` operator. For filtering going forward, it's recommended to use the objection object-notation for eager loading along with `$where` definitions at each level.

# Filter Operators

There are a number of built-in operations that can be applied to columns (custom ones can also be created). These include:

1. **$like** - The SQL _LIKE_ operator, can be used with expressions such as _ab%_ to search for strings that start with _ab_
2. **$gt/$lt/$gte/$lte** - Greater than and Less than operators for numerical fields
3. **=/$equals** - Explicitly specify equality
4. **$in** - Whether the target value is in an array of values
5. **$exists** - Whether a property is not null
6. **$or** - A top level _OR_ conditional operator

For any operators not available (eg _ILIKE_, refer to the custom operators section below).

#### Example

An example of operator usage
```json
{
  "eager": {
    "$where": {
      "property0": "Exactly Equals",
      "property1": {
        "$equals": 5
      },
      "property2": {
        "$gt": 5
      },
      "property3": {
        "$lt": 10,
        "$gt": 5
      },
      "property4": {
        "$in": [ 1, 2, 3 ]
      },
      "property5": {
        "$exists": false
      },
      "property6": {
        "$or": [
          { "$in": [ 1, 2, 3 ] },
          { "$equals": 100 }
        ]
      }
    }
  }
}
```

#### Custom Operators

If the built in filter operators aren't quite enough, custom operators can be added. A common use case for this may be to add a `lower case LIKE` operator, which may vary in implementation depending on the SQL dialect.

Example:

```js
const options = {
  operators: {
    $ilike: (property, operand, builder) =>
      builder.whereRaw('?? ILIKE ?', [property, operand])
  }
};

buildFilter(Person, null, options)
  .build({
    eager: {
      $where: {
        firstName: { $ilike: 'John' }
      }
    }
  })
```

The `$ilike` operator can now be used as a new operator and will use the custom operator callback specified.

# Logical Expressions
Logical expressions can be applied to both the `eager` and `require` helpers. The `where` top level operator will eventually be deprecated and replaced by the new `eager` [object notation](https://vincit.github.io/objection.js/#relationexpression-object-notation) in objection.js.

#### Examples using `$where`
The `$where` expression is used to "filter models". Given this, related fields between models can be mixed anywhere in the logical expression.

```json
{
  "eager": {
    "$where": {
      "$or": [
        { "city.country.name": "Australia" },
        { "city.code": "09" }
      ]
    }
  }
}
```

Logical expressions can also be nested
```json
{
  "eager": {
    "$where": {
      "$and": {
        "name": "John",
        "$or": [
          { "city.country.name": "Australia" },
          { "city.code": { "$like": "01" }}
        ]
      }
    }
  }
}
```

Note that in these examples, all logical expressions come _before_ the property name. However, logical expressions can also come _after_ the property name.

```json
{
  "eager": {
    "$where": {
      "$or": [
        { "city.country.name": "Australia" },
        {
          "city.code": {
            "$or": [
              { "$equals": "12" },
              { "$like": "13" }
            ]
          }
        }
      ]
    }
  }
}
```

The `$where` will apply to the relation that immediately precedes it in the tree, in the above case "city". The `$where` will apply to relations of the eager model using dot notation. For example, you can query `Customers`, eager load their `orders` and filter those orders by the `product.name`. Note that `product.name` is a related field of the order model, not the customers model.

# Aggregations

[Aggregations](doc/AGGREGATIONS.md) such as _count, sum, min, max, avg_ can be applied to the queried model.

Additionally for any aggregations, you can use them in other expressions above including:

* Filtering using `$where`
* Ordering using `order`

For more detailed descriptions of each feature, refer to the [aggregations section](doc/AGGREGATIONS.md).

Transform a basic aggregation like this on a `GET /Customers` endpoint:

```js
{
  "eager": {
    "$aggregations": [
        {
          "type": "count",
          "alias": "numberOfOrders",
          "relation": "orders"
        }
    ]
  }
}
```

...into a result set like this:

```json
[
  {
    "firstName": "John",
    "lastName": "Smith",
    "numberOfOrders": 10
  },{
    "firstName": "Jane",
    "lastName": "Bright",
    "numberOfOrders": 5
  },{
    "firstName": "Greg",
    "lastName": "Parker",
    "numberOfOrders": 7
  }
]
```