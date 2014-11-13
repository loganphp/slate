## Using Views

Views are used to obtain data stored within a database.
Views are written using Javascript functions,
and work by letting you search for content that matches specific criteria.
The criteria are specified within the function,
or supplied as arguments to the function.

### Querying a view

-   **Method**: `GET /db/_design/<design-doc>/_view/<view-name>`
-   **Request**: None
-   **Response**: JSON of the documents returned by the view
-   **Roles permitted**: \_reader

Executes the specified `view-name` from the specified `design-doc` design document.

#### Query Arguments

> Example of retrieving a list of the first five documents from a database, applying the user-created `by_title` view:

```sourceCode
GET /<database>/_design/<design-doc>/_view/by_title?limit=5
Accept: application/json
Content-Type: application/json
```

> Example response:

```json
{
   "offset" : 0,
   "rows" : [
      {
         "id" : "3-tiersalmonspinachandavocadoterrine",
         "key" : "3-tier salmon, spinach and avocado terrine",
         "value" : [
            null,
            "3-tier salmon, spinach and avocado terrine"
         ]
      },
      {
         "id" : "Aberffrawcake",
         "key" : "Aberffraw cake",
         "value" : [
            null,
            "Aberffraw cake"
         ]
      },
      {
         "id" : "Adukiandorangecasserole-microwave",
         "key" : "Aduki and orange casserole - microwave",
         "value" : [
            null,
            "Aduki and orange casserole - microwave"
         ]
      },
      {
         "id" : "Aioli-garlicmayonnaise",
         "key" : "Aioli - garlic mayonnaise",
         "value" : [
            null,
            "Aioli - garlic mayonnaise"
         ]
      },
      {
         "id" : "Alabamapeanutchicken",
         "key" : "Alabama peanut chicken",
         "value" : [
            null,
            "Alabama peanut chicken"
         ]
      }
   ],
   "total_rows" : 2667
}
```

Argument | Description | Optional | Type | Default | Supported values
---------|-------------|----------|------|---------|-----------------
`descending` | Return the documents in 'descending by key' order. | yes | Boolean | false | 
`endkey` | Stop returning records when the specified key is reached. | yes | String or JSON array | | 
`endkey_docid` | Stop returning records when the specified document ID is reached. | yes | String | | 
`group` | Using the reduce function, group the results to a group or single row. | yes | Boolean | false | 
`group_level` | Only applicable if the view uses complex keys: keys that are JSON arrays. Groups reduce results for the specified number of array fields. | yes | Numeric | | 
`include_docs` | Include the full content of the documents in the response. | yes | Boolean | false | 
`inclusive_end` | Include rows with the specified endkey. | yes | Boolean | true | 
`key` | Return only documents that match the specified key. Note: Keys are JSON values, and must be URL encoded. | yes | String | | 
`limit` | Limit the number of returned documents to the specified count. | yes | Numeric | | 
`reduce` | Use the reduce function. | yes | Boolean | true | 
`skip` | Skip this number of rows from the start. | yes | Numeric | 0 | 
`stale` | Allow the results from a stale view to be used. This makes the request return immediately, even if the view has not been completely built yet. If this parameter is not given, a response is returned only after the view has been built. | yes | String | false | `ok`: Allow stale views.<br/>`update_after`: Allow stale views, but update them immediately after the request.
`startkey` | Return records starting with the specified key. | yes | String or JSON array | | 
`startkey_docid` | Return records starting with the specified document ID. | yes | String | | 

### Indexes

When a view is defined in a design document,
a corresponding index is also created,
based on the key information defined within the view.
The index helps improve performance when using the view to access documents,
for example when searching or selecting documents.
To save time,
the index is not populated with content until the view is used.

The index content is generated when the view is first used.
The index content is updated incrementally and automatically when the view is used again after any one of the following three situations has occurred:

-   A new document has been added to the database.
-   An existing document has been deleted from the database.
-   An existing document in the database has been updated or modified in some way.

When a view is being applied to the database,
a check is first made to see if any index updates should be performed.
If any updates are required,
they are completed before the view query is applied.

View indexes are rebuilt entirely when the view definition changes.
This ensures that changes to the view definitions are reflected in the view indexes.
To achieve this,
a 'fingerprint' of the view definition is created whenever the design document is updated.
If the fingerprint changes,
then the view indexes are completely rebuilt.

<aside class="notice">View index rebuilds occur whenever a change occurs to any one view from all the views defined in the design document.
For example,
if you have a design document with three views,
and you update the design document,
all three view indexes within the design document are rebuilt.</aside>

The results returned by a view are updated when the view is queried.
This means that there might be a delay in returning the results when the view is accessed.
A delay is affected by the number of documents in the database, and whether the view index does not exist or is not current because the database content has been modified.

It is not possible to eliminate these delays,
but you might reduce them by trying the following techniques:

-   Create the view definition in the design document in your database before inserting or updating documents. This causes incremental updates to the index when the view is accessed.
-   Manually force a view request from the database. Do this either before users are allowed to use the view, or by accessing the view manually after documents are added or updated.
-   Use `/db/_changes` to monitor for changes to the database. If a change occurs, access the view to force an update of the corresponding view index. See [Getting database changes](#get-changes) for more information.

<aside class="notice">None of these mitigations completely eliminate the need for the indexes to be rebuilt or updated when the view is accessed.</aside>

If speed of response is more important than having completely up-to-date data,
an alternative is to allow users to access an old version of the view index.
In effect,
the user has an immediate response from 'stale' index content,
rather than waiting for the index to be updated.
Depending on the document contents,
using a stale view might not return the latest information.
Nevertheless, a stale view returns the results of the view query quickly,
by using an existing version of the index.

For example, to access the existing stale view `by_recipe` in the `recipes` design document,
you would use a request similar to:
<code>/recipes/_design/recipes/_view/by_recipe?stale=ok</code>

### Accessing a stale view

Making use of a stale view has consequences.
In particular,
accessing a stale view:

1.	Does not trigger a rebuild of the view indexes, even if there have been changes since the last access.
2.	Returns the current (existing) version of the view index, if it exists.
3.	Returns an empty result set if the view index does exist.

A useful variation is to force an update to the index,
after a stale view has been returned.
Do this by supplying the `update_after` value to the `stale` parameter.
When `update_after` is supplied,
the update process is started when the view information has been returned to the client.

You can confirm that you are using stale view data by making use of the `update_seq` value returned in the view information.
The stale view data value can be compared to the current update sequence that is available in the database information ([Reading details about the database](#read)).
If the values are different,
you are using stale view data.

### Sorting Returned Rows

> Example of requesting the last five records by reversing the sort order:

```sourceCode
GET /<database>/_design/<design-doc>/_view/by_title?limit=5&descending=true
Accept: application/json
Content-Type: application/json
```

> Example response:

```json
{
   "offset" : 0,
   "rows" : [
      {
         "id" : "Zucchiniinagrodolcesweet-sourcourgettes",
         "key" : "Zucchini in agrodolce (sweet-sour courgettes)",
         "value" : [
            null,
            "Zucchini in agrodolce (sweet-sour courgettes)"
         ]
      },
      {
         "id" : "Zingylemontart",
         "key" : "Zingy lemon tart",
         "value" : [
            null,
            "Zingy lemon tart"
         ]
      },
      {
         "id" : "Zestyseafoodavocado",
         "key" : "Zesty seafood avocado",
         "value" : [
            null,
            "Zesty seafood avocado"
         ]
      },
      {
         "id" : "Zabaglione",
         "key" : "Zabaglione",
         "value" : [
            null,
            "Zabaglione"
         ]
      },
      {
         "id" : "Yogurtraita",
         "key" : "Yogurt raita",
         "value" : [
            null,
            "Yogurt raita"
         ]
      }
   ],
   "total_rows" : 2667
}
```

The data returned by a view query is in the form of an array.
Each element within the array is sorted using native UTF-8 sorting.
The sort is applied to the key defined in the view function.

The basic order of output is as follows:

Value | Order
------|------
`null` | First
`false` |
`true` |
Numbers |
Text (lowercase) |
Text (uppercase) |
Arrays (according to the values of each element, using the order given in this table) |
Objects (according to the values of keys, in key order using the order given in this table) | Last

You can reverse the order of the returned view information by setting the `descending` query value <code>true</code>.

### Specifying Start and End Values

> Example of filtering using `startkey` and `endkey` query arguments:

```sourceCode
GET /recipes/_design/recipes/_view/by_ingredient?startkey=%22carrots%22&endkey=%22egg%22
Accept: application/json
Content-Type: application/json
```

> Useful results would be returned, because "carrots" is alphabetically before "egg".

```sourceCode
```

> Example of <i>incorrect</i> filtering and reversing the order of output using the `descending` query argument:

```sourceCode
GET /recipes/_design/recipes/_view/by_ingredient?descending=true&startkey=%22carrots%22&endkey=%22egg%22
Accept: application/json
Content-Type: application/json
```

> The view request returns no entries, because "carrots" is alphabetically before "egg". The returned result is empty:

```json
{
   "total_rows" : 26453,
   "rows" : [],
   "offset" : 21882
}
```

> Example of <i>correct</i> filtering and reversing the order of output by using the `descending` query argument, and reversing the `startkey` and `endkey` query arguments:

```sourceCode
GET /recipes/_design/recipes/_view/by_ingredient?descending=true&startkey=%22egg%22&endkey=%22carrots%22
Accept: application/json
Content-Type: application/json
```

The `startkey` and `endkey` query arguments can be used to specify the range of values to be displayed when querying the view.

The sort direction is always applied first.
Next, filtering is applied using the `startkey` and `endkey` query arguments.
This means that it is possible to have empty view results because the sorting and filtering do not make sense in combination.

For example,
if you have a database that returns ten results when viewing with a `startkey` of "alpha" and an `endkey` of "beta",
you would get no results when reversing the order.
The reason is that the entries in the view are reversed before the key filter is applied.
Therefore the `endkey` of "beta" is seen before the `startkey` of "alpha", resulting in an empty list.

The solution is to reverse not just the sort order,
but also the `startkey` and `endkey` parameter values.

### Querying a view using a list of keys

> Example request to return all recipes, where the key for the view matches either "clear apple juice" or "lemonade":

```sourceCode
POST /recipes/_design/recipes/_view/by_ingredient
Content-Type: application/json

{
   "keys" : [
      "clear apple juice",
      "lemonade"
   ]
}
```

> Example returned view data, containing the standard view information, but only where the keys match:

```json
{
   "total_rows" : 26484,
   "rows" : [
      {
         "value" : [
            "Scotch collops"
         ],
         "id" : "Scotchcollops",
         "key" : "claret"
      },
      {
         "value" : [
            "Stand pie"
         ],
         "id" : "Standpie",
         "key" : "clear apple juice"
      }
   ],
   "offset" : 6324
}
```

-   **Method**: `POST /db/_design/design-doc/_view/view-name`
-   **Request**: List of keys to be returned from specified view
-   **Response**: JSON of the documents returned by the view
-   **Roles permitted**: \_reader

This method of requesting information from a database executes the specified `view-name` from the specified `design-doc` design document. However,
unlike using [`GET`](#querying-a-view) to access views,
the `POST` method allows you to specify the keys to use when retrieving the view results.
In all other aspects,
the `POST` method is identical to the [`GET`](#querying-a-view) API request.

### Multi-document Fetching

> Example request to obtain the full documents that match the listed keys:

```sourceCode
POST /recipes/_design/recipes/_view/by_ingredient?include_docs=true
Content-Type: application/json

{
   "keys" : [
      "claret",
      "clear apple juice"
   ]
}
```

> Example response, returning the full document for each recipe:

```json
{
   "offset" : 6324,
   "rows" : [
      {
         "doc" : {
            "_id" : "Scotchcollops",
            "_rev" : "1-bcbdf724f8544c89697a1cbc4b9f0178",
            "cooktime" : "8",
            "ingredients" : [
               {
                  "ingredient" : "onion",
                  "ingredtext" : "onion, peeled and chopped",
                  "meastext" : "1"
               },
            ...
            ],
            "keywords" : [
               "cook method.hob, oven, grill@hob",
               "diet@wheat-free",
               "diet@peanut-free",
               "special collections@classic recipe",
               "cuisine@british traditional",
               "diet@corn-free",
               "diet@citrus-free",
               "special collections@very easy",
               "diet@shellfish-free",
               "main ingredient@meat",
               "occasion@christmas",
               "meal type@main",
               "diet@egg-free",
               "diet@gluten-free"
            ],
            "preptime" : "10",
            "servings" : "4",
            "subtitle" : "This recipe comes from an old recipe book of 1683 called 'The Gentlewoman's Kitchen'. This is an excellent way of making a rich and full-flavoured meat dish in a very short time.",
            "title" : "Scotch collops",
            "totaltime" : "18"
         },
         "id" : "Scotchcollops",
         "key" : "claret",
         "value" : [
            "Scotch collops"
         ]
      },
      {
         "doc" : {
            "_id" : "Standpie",
            "_rev" : "1-bff6edf3ca2474a243023f2dad432a5a",
            "cooktime" : "92",
            "ingredients" : [
...            ],
            "keywords" : [
               "diet@dairy-free",
               "diet@peanut-free",
               "special collections@classic recipe",
               "cuisine@british traditional",
               "diet@corn-free",
               "diet@citrus-free",
               "occasion@buffet party",
               "diet@shellfish-free",
               "occasion@picnic",
               "special collections@lunchbox",
               "main ingredient@meat",
               "convenience@serve with salad for complete meal",
               "meal type@main",
               "cook method.hob, oven, grill@hob / oven",
               "diet@cow dairy-free"
            ],
            "preptime" : "30",
            "servings" : "6",
            "subtitle" : "Serve this pie with pickled vegetables and potato salad.",
            "title" : "Stand pie",
            "totaltime" : "437"
         },
         "id" : "Standpie",
         "key" : "clear apple juice",
         "value" : [
            "Stand pie"
         ]
      }
   ],
   "total_rows" : 26484
}
```

Combining a `POST` request to a given view, with the `include_docs=true` query argument, enables you to retrieve multiple documents from a database.
This technique is more efficient than using multiple [`GET`](#querying-a-view) API requests.

### Sending several queries to a view

-   **Method**: `POST /db/_design/design-doc/_view/view-name`
-   **Request**: A JSON document containing an array of query objects
-   **Response**: A JSON document containing an array of response object - one per query
-   **Roles permitted**: \_reader

This in an example of a request body:

The JSON object contains only the `queries` field, which holds an array of query objects. Each query object can have fields for the parameters of a query. The field names and their meaning are the same as the query parameters of a regular view request.

Here is an example of a response:

The JSON object contains only the `results` field, which holds an array of result objects - one for each query. Each result object contains the same fields as the response to a regular view request.