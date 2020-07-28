To fetch items from a resource, you need a GET request with the URL from the resource definition:

```
curl -X GET <resource URL returned by the OPTIONS request>
```

Have a look at [retrieving an OPTIONS request](index.md#retrieve-meta-data) for more details.

# Response structure

All GET requests will return a JSON object with the structure defined by [jsonapi.org](http://jsonapi.org/):

```javascript
{
    "meta": {
         "total": 100
    },
    "links": {
        "first": "http://localhost:8000/jsonapi/product?page[offset]=0",
        "prev": "http://localhost:8000/jsonapi/product?page[offset]=0",
        "next": "http://localhost:8000/jsonapi/product?page[offset]=25",
        "last": "http://localhost:8000/jsonapi/product?page[offset]=75",
        "self": "http://localhost:8000/jsonapi/product?page[offset]=0",
    },
    "data": [ {
        "type": "product",
        "id": "1",
        "attributes": {
             "product.id": "1",
             ...
             "product.status": "1"
        },
        "links": {
            "self": "http://localhost:8000/jsonapi/product/1"
        },
        "relationships": {
            "product/property": [ {
                "data": {
                    "id": "12",
                    "type": "product/property"
                }
            } ],
            "text": [  {
                "data": {
                    "id": "9",
                    "type": "text",
                    "attributes": {
                        "product.lists.id": "5",
                        ...
                        "product.lists.refid": "9",
                    }
                }
            } ]
        }
    },
    ...
    ],
    "included": [ {
        "id": "12",
        "type": "product/property",
        "attributes": {
            "product.property.id": "12",
            ...
            "product.property.value": "1234-ABCD-5678",
    }, {
        "type": "text",
        "id": "9",
        "attributes": {
           "text.id": "9",
            ...
            "text.content": "My product name"
        }
    },
    ...
    ]
}
```

The **"meta"** section will always contain a "total" property on success. It's value is the number of items found in total for the current search filter. In the **"links"** section, all necessary links for navigating through the result set are listed including the link for the current request.

**"data"** contains the product items and each item always contains the ID and the resource type. In the attributes section, you will find the same data as returned by the *toArray()* method for every item. An URL referencing the unique resource is included in "links". If related items are included, they will be referenced by their ID and type in the "relationships" section. For items that are associated via lists, they will also contain an "attributes" section with the data for the list table.

In the **"included"** section all related items are listed if you told the server to hand them over in the same request as well. It's structure is the same as for the "data" section. For more information, please have a look into the [related resources](#include-related-resources) section.

By default, the first 25 items available will be returned if no further GET parameters are sent along with the request. There are three parameters to modify that behavior:

* filter (conditions to retrieve specific items)
* sort (defines the order of the items in the result set)
* page (contains "start" and "limit" for getting slices of the result set)

# Filtering the result

Each resource offers custom search parameters to retrieve data for common use cases. Please have a look into the articles about products, basket, etc. for details about these parameters.

Additionally, you can use custom filters for each resource. The JSON API standard defines a parameter named "filter" that can contain arbitrary conditions. In the Aimeos API, you can use conditions for comparing values and combining them. To get all products items which are are selections:

=== "CURL"
    ```bash
    # filter[>][product.type]=select
    curl -X GET "http://localhost:8000/jsonapi/product?filter[%3E][product.type]=select"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[filter][%3E][product.type]=select"
    ```
=== "jQuery"
    ```javascript
    var params = {
        filter: {'>': {'product.type': 'select'}}
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

This is known as the "polish" notation because the operator comes first and than the two operands (key and value). There are several operators available:

* '=~' (%3D%7E) : Strings that starts with the given value
* '~=' (%7E%3D) : Strings that contains the given value
* '>' (%3E) : Greater than for date, date&time, integer and float values
* '>=' (%3E%3D) : Greater than and equal for date, date&time, integer and float values
* '<' (%3C) : Smaller than for date, date&time, integer and float values
* '<=' (%3C%3D) : Smaller than and equal for date, date&time, integer and float values
* '==' (%3D%3D) : Equal for boolean, date, date&time, integer, float and string values
* '!=' (%21%3D) : Not equal for boolean, date, date&time, integer, float and string values

!!! warning
    Don't forget to [nest the parameters](index.md#nested-parameters) if a prefix is sent in the meta data!

To combine several conditions into one request, you can combine two or more "compare" expressions by using a "combine" expression:

=== "CURL"
    ```bash
    # filter[&&][][>][product.type]=select
    # &filter[&&][][=~][product.label]=demo
    curl -X GET "http://localhost:8000/jsonapi/product?filter[%26%26][][%3E][product.type]=select&filter[%26%26][][%3D%7E][product.label]=demo"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[filter][%26%26][][%3E][product.type]=select&ai[filter][%26%26][][%3D%7E][product.label]=demo"
    ```
=== "jQuery"
    ```javascript
    var params = {
        filter: {'&&': [
            {'>': {'product.type': 'select'}},
            {'=~': {'product.label': 'demo'}}
        ]}
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

That would search for all product items which are selections AND whose labels starts with "demo". The available "combine" expressions are:

* '&&' (%26%26) : Combines expressions using an AND operator
* '||' (%7C%7C) : Combines expressions using an OR operator
* '!' (%21) : Negates an expression

The negation is a special case because it only accepts one "compare" condition while the others require more than one condition. Nevertheless, the value of the negation operator must be an array like for the other "combine" operators:

=== "CURL"
    ```bash
    # filter[!][][=~][product.label]=demo
    curl -X GET "http://localhost:8000/jsonapi/product?filter[%21][][%3D%7E][product.label]=demo"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[filter][%21][][%3D%7E][product.label]=demo"
    ```
=== "jQuery"
    ```javascript
    var params = {
        filter: {'!': [
            {'=~': {'product.label': 'demo'}}
        ]}
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

You can also create more complicated statements by nesting them like:

=== "CURL"
    ```bash
    # filter[&&][][!][][=~][product.label]=demo
    # &filter[&&][][||][][==][product.datestart]=
    # &filter[&&][][||][][>][product.datestart]=2000-01-01 00:00:00
    curl -X GET "http://localhost:8000/jsonapi/product?filter[%26%26][][%21][][%3D%7E][product.label]=demo&filter[%26%26][][%7C%7C][][%3D%3D][product.datestart]=&filter[%26%26][][%7C%7C][][%3E][product.datestart]=2000-01-01%2000:00:00"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[filter][%26%26][][%21][][%3D%7E][product.label]=demo&ai[filter][%26%26][][%7C%7C][][%3D%3D][product.datestart]=&ai[filter][%26%26][][%7C%7C][][%3E][product.datestart]=2000-01-01%2000:00:00"
    ```
=== "jQuery"
    ```javascript
    var params = {
        filter: {'&&': [
            {'!': [
                {'=~': {'product.label': 'demo'}}
            ]},
            {'||': [
                {'==': {'product.datestart': null}},
                {'>': {'product.datestart': '2000-01-01 00:00:00'}}
            }
        ]}
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

It combines the three expression by using the AND operator. In the second expression we tell the server that we want to get all items that doesn't start with "demo". The third expression is in this case an OR expression that specifies that "product.datestart" can be either be a null value or the start date must be after the beginning of the year 2000.

!!! warning
    Don't forget to [nest the parameters](index.md#nested-parameters) if a prefix is sent in the meta data!

# Sorting the result set

Some resources e.g. for products offer specific sorting like "price", "name" or "relevance". Others like for services use an implicit default sorting. Please have a look at the available sort criteria for the resources.

Additionally, you can use the sort parameter and the items keys for generic sorting. In the JSON API standard, the URL parameter "sort" is used to define the sorting of the result set:

=== "CURL"
    ```bash
    curl -X GET "http://localhost:8000/jsonapi/product?sort=product.label"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[sort]=product.label"
    ```
=== "jQuery"
    ```javascript
    var params = {
        sort: 'product.label'
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

This will return the results ordered by the product label. You can also tell the server to sort the result set in the reverse order by adding a minus symbol in front of the sort key:

=== "CURL"
    ```bash
    curl -X GET "http://localhost:8000/jsonapi/product?sort=-product.label"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[sort]=-product.label"
    ```
=== "jQuery"
    ```javascript
    var params = {
        sort: '-product.label'
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

Sorting by several keys is also possible if they are separated by a comma:

=== "CURL"
    ```bash
    curl -X GET "http://localhost:8000/jsonapi/product?sort=-product.status,product.id"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[sort]=-product.status,product.id"
    ```
=== "jQuery"
    ```javascript
    var params = {
        sort: '-product.status,product.id'
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

It would sort the result set by the product status (descending) and the product ID.

!!! warning
    Don't forget to [nest the parameters](index.md#nested-parameters) if a prefix is sent in the meta data!

# Retrieve slices of the result

By default, only the first 25 items are returned if nothing else is specified. To get more or less items and step through the result set, the JSON API standard uses the "page" parameter:

=== "CURL"
    ```bash
    curl -X GET "http://localhost:8000/jsonapi/product?page[offset]=0&page[limit]=100"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[page][offset]=0&ai[page][limit]=100"
    ```
=== "jQuery"
    ```javascript
    var params = {
        page: {
            offset: 0,
            limit: 100
        }
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

Thus, you will get the first 100 items instead. To get 100 items starting from the 300th one, simply increase the offset value:

=== "CURL"
    ```bash
    curl -X GET "http://localhost:8000/jsonapi/product?page[offset]=300&page[limit]=100"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[page][offset]=300&ai[page][limit]=100"
    ```
=== "jQuery"
    ```javascript
    var params = {
        page: {
            offset: 300,
            limit: 100
        }
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

!!! warning
    Don't forget to [nest the parameters](index.md#nested-parameters) if a prefix is sent in the meta data!

# Return specific fields only

If you only need the values of a few fields and want to reduce the amount of data transferred over the network, you can specify the fields that should be returned without the rest of the available ones. In the JSON API standard the parameter "fields" is used for this:

=== "CURL"
    ```bash
    curl -X GET "http://localhost:8000/jsonapi/product?fields[product]=product.id,product.label"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[fields][product]=product.id,product.label"
    ```
=== "jQuery"
    ```javascript
    var params = {
        fields: { product: "product.id,product.label" }
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

The generated request will then only return the ID and label of product items.

!!! warning
    Don't forget to [nest the parameters](index.md#nested-parameters) if a prefix is sent in the meta data!

# Include related resources

To minimize the number of requests, the Aimeos JSON API can add related resources to the response. For example, you can tell the server that it should not only return the list of products but also the texts and prices associated to these products. The JSON API uses the parameter "include" to specify the related resources:

=== "CURL"
    ```bash
    curl -X GET "http://localhost:8000/jsonapi/product?include=text,price"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[include]=text,price"
    ```
=== "jQuery"
    ```javascript
    var params = {
        include: 'text,price'
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

You can use the "include" parameter for all items that are associated via the lists to categories, products, services, attributes, media, prices and texts. This does also work for items from the same domain that have a parent/child relationship like product properties:

=== "CURL"
    ```bash
    curl -X GET "http://localhost:8000/jsonapi/product?include=text,product/property"
    curl -X GET "http://localhost:8000/jsonapi/product?ai[include]=text,product/property"
    ```
=== "jQuery"
    ```javascript
    var params = {
        include: 'text,product/property'
    };

	if(options.meta.prefix) { // returned from OPTIONS call
		params[options.meta.prefix] = params;
	}

    $.ajax({
        method: "GET",
        dataType: "json",
        url: options.meta.resources['product'], // returned from OPTIONS call
        data: params
    }).done( function( result ) {
        console.log( result.data );
    });
    ```

Then, the text items associated via the list table would be returned as well as the properties directly attached to the products.


!!! warning
    Don't forget to [nest the parameters](index.md#nested-parameters) if a prefix is sent in the meta data!