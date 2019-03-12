[![Build Status](https://travis-ci.org/Flowpack/Flowpack.ElasticSearch.ContentRepositoryAdaptor.svg)](https://travis-ci.org/Flowpack/Flowpack.ElasticSearch.ContentRepositoryAdaptor) [![Latest Stable Version](https://poser.pugx.org/flowpack/elasticsearch-contentrepositoryadaptor/v/stable)](https://packagist.org/packages/flowpack/elasticsearch-contentrepositoryadaptor) [![Total Downloads](https://poser.pugx.org/flowpack/elasticsearch-contentrepositoryadaptor/downloads)](https://packagist.org/packages/flowpack/elasticsearch-contentrepositoryadaptor)

# Neos Elasticsearch Adapter

This project connects the Neos Content Repository to Elasticsearch; enabling two
main functionalities:

* finding Nodes in Fusion / Eel by arbitrary queries
* Full-Text Indexing of Pages and other Documents (of course including the full content)

## Relevant Packages

* [Neos.ContentRepository.Search](https://www.neos.io/download-and-extend/packages/neos/neos-content-repository-search.html): provides common functionality for searching Neos Content Repository nodes. Does not contain a search backend.
* [Flowpack.ElasticSearch](https://www.neos.io/download-and-extend/packages/flowpack/flowpack-elasticsearch.html): provides common code for working with Elasticsearch
* [Flowpack.ElasticSearch.ContentRepositoryAdaptor](https://www.neos.io/download-and-extend/packages/flowpack/flowpack-elasticsearch-contentrepositoryadaptor.html): this package
* [Flowpack.SimpleSearch.ContentRepositoryAdaptor](https://www.neos.io/download-and-extend/packages/flowpack/flowpack-simplesearch-contentrepositoryadaptor.html): an alternative search backend (to be used instead of this package); storing the search index in SQLite
* [Flowpack.SearchPlugin](https://www.neos.io/download-and-extend/packages/flowpack/flowpack-searchplugin.html): search plugin for Neos

## Installation

```
composer require 'flowpack/elasticsearch-contentrepositoryadaptor'
// Not required, but can be used to learn how to integration the flowpack/elasticsearch-contentrepositoryadaptor in your project
composer require 'flowpack/searchplugin'
```
Ensure to update `<your-elasticsearch>/config/elasticsearch.yml` as explained below; then start Elasticsearch.

Finally, run `./flow nodeindex:build`, and add the search plugin to your page. It should "just work".

## Elastic version support

**HINT: the Master of this package only supports modern versions of ES. If you need ES 1.x or 2.x support, please [see the 4.x branch of this repository](https://github.com/Flowpack/Flowpack.ElasticSearch.ContentRepositoryAdaptor/tree/4.0#elastic-version-support).**

You can switch the Elastic driver by editing ```Settings.yaml```
(```Flowpack.ElasticSearch.ContentRepositoryAdaptor.driver.version```) with the following value:

* ```5.x``` to support Elastic 5.x

_Currently the Driver interfaces are not marked as API, and can be changed to adapt to future needs._

**Note:** When using Elasticsearch 5.x changes to the types may need to be done in your mapping.
More information on the [mapping in ElasticSearch 5.x](Documentation/ElasticMapping-5.x.md).

### Elasticsearch Configuration file elasticsearch.yml

There is a need, depending on your version of Elasticsearch, to add specific configuration to your
Elasticsearch Configuration File `<your-elasticsearch>/config/elasticsearch.yml`.

- [ElasticSearch 5.x](Documentation/ElasticConfiguration-5.x.md)

## Building up the Index

The node index is updated on the fly, but during development you need to update it frequently.

In case of a mapping update, you need to reindex all nodes. Don't worry to do that in production;
the system transparently creates a new index, fills it completely, and when everything worked,
changes the index alias.

```
./flow nodeindex:build

 # if during development, you only want to index a few nodes, you can use "limit"
./flow nodeindex:build --limit 20

 # in order to remove old, non-used indices, you should use this command from time to time:
./flow nodeindex:cleanup
```

### Advanced Configuration

By default the indexing processes all NodeTypes, but you can change this in your *Settings.yaml*:

```yaml
Neos:
  ContentRepository:
    Search:
      defaultConfigurationPerNodeType:
        '*':
          indexed: true
        'Neos.Neos:FallbackNode':
          indexed: false
        'Neos.Neos:Shortcut':
          indexed: false
        'Neos.Neos:ContentCollection':
          indexed: false
```

You need to explicitly configure the individual NodeTypes (this feature does not check the Super Type configuration).
But you  can use a special notation to configure a full namespace, `Acme.AcmeCom:*` will be applied for all node
types in the `Acme.AcmeCom` namespace. The most specific configuration is used in this order: 

- NodeType name (`Neos.Neos:Shortcut`)
- Full namespace notation (`Neos.Neos:*`)
- Catch all (`*`)

### Advanced Index Settings

If you need advanced settings you can define them in your *Settings.yaml*:

Example is from the Documentation of the used *Flowpack.ElasticSearch* Package

https://github.com/Flowpack/Flowpack.ElasticSearch/blob/master/Documentation/Indexer.rst

```yaml
Flowpack:
  ElasticSearch:
    indexes:
      default:
        twitter:
          analysis:
            filter:
              elision:
                type: elision
                articles: [ 'l', 'm', 't', 'qu', 'n', 's', 'j', 'd' ]
          analyzer:
            custom_french_analyzer:
              tokenizer: letter
              filter: [ 'asciifolding', 'lowercase', 'french_stem', 'elision', 'stop' ]
            tag_analyzer:
              tokenizer: keyword
              filter: [ 'asciifolding', 'lowercase' ]
```

If you use multiple client configurations, please change the *default* key just below the *indexes*.


## Doing Arbitrary Queries

We'll first show how to do arbitrary Elasticsearch Queries in Fusion. This is a more powerful
alternative to FlowQuery. In the long run, we might be able to integrate this API back into FlowQuery,
but for now it works well as-is.

Generally, Elasticsearch queries are done using the `Search` Eel helper. In case you want
to retrieve a *list of nodes*, you'll generally do:
```
nodes = ${Search.query(site)....execute()}
```

In case you just want to retrieve a *single node*, the form of a query is as follows:
```
nodes = ${q(Search.query(site)....execute()).get(0)}
```

To fetch the total number of hits a query returns, the form of a query is as follows:
```
nodes = ${Search.query(site)....count()}
```

All queries search underneath a certain subnode. In case you want to search "globally", you will
search underneath the current site node (like in the example above).

Furthermore, the following operators are supported:

As **value**, the following methods accept a simple type, a node object or a DateTime object.

* `nodeType('Your.Node:Type')`
* `exactMatch('propertyName', value)` -- supports simple types: `exactMatch('tag', 'foo')`, or node references: `exactMatch('author', authorNode)`
* `exclude('propertyName', value)` -- excludes results by property - the negation of exactMatch.
* `greaterThan('propertyName', value, [clauseType])` -- range filter with property values greater than the given value
* `greaterThanOrEqual('propertyName', value, [clauseType])` -- range filter with property values greater than or equal to the given value
* `lessThan('propertyName', value, [clauseType])` -- range filter with property values less than the given value
* `lessThanOrEqual('propertyName', value, [clauseType])` -- range filter with property values less than or equal to the given value
* `sortAsc('propertyName')` and `sortDesc('propertyName')` -- can also be used multiple times, e.g. `sortAsc('tag').sortDesc(`date')`
   will first sort by tag ascending, and then by date descending.
* `limit(5)` -- only return five results. If not specified, the default limit by Elasticsearch applies (which is at 10 by default)
* `from(5)` -- return the results starting from the 6th one
* `fulltext('searchWord', options)` -- do a query_string query on the Fulltext index using the searchword and additional [options](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-query-string-query.html) to the query_string

#### moreLikeThis(like, fields, options)

The More Like This Query (MLT Query) finds documents that are "like" a given text or a given set of documents.

* `like` Single value or an array of strings or nodes.
* `fields` An array of fields which are used to compare other docs with the given "like" definition.
* `options` Additional options for the `more_like_this` query. See the [elasticsearch documentation](https://www.elastic.co/guide/en/elasticsearch/reference/5.6/query-dsl-mlt-query.html) for what is possible.

Furthermore, there is a more low-level operator which can be used to add arbitrary Elasticsearch filters:

* `queryFilter("filterType", {option1: "value1"}, [clauseType])`

The optional argument `clauseType` defaults to "must" and can be used to specify the boolean operator of the [bool query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html). It has to be one of `must`, `should`, `must_not` or `filter`.	

At lowest level, there is the `request` operator which allows to modify the request in arbitrary manner. Note that the existing request is merged with the passed-in type in case it is an array:

* `request('query.filtered.query.bool.minimum_should_match', 1)`
* `request('query.filtered.query.bool', {"minimum_should_match": 1})`

In order to debug the query more easily, the following operation is helpful:

* `log()` log the full query on execution into the Elasticsearch log (i.e. in `Data/Logs/ElasticSearch.log`)

### Example Queries

#### Finding all pages which are tagged in a special way and rendering them in an overview

Use Case: On a "Tag Overview" page, you want to show all pages being tagged in a certain way

Setup: You have two node types in a blog called `Acme.Blog:Post` and `Acme.Blog:Tag`, both
inheriting from `Neos.Neos:Document`. The `Post` node type has a property `tags` which is
of type `references`, pointing to `Tag` documents.

Fusion setup:

```
 # for "Tag" documents, replace the main content area.
prototype(Neos.Neos:PrimaryContent).acmeBlogTag {
    condition = ${q(node).is('[instanceof Acme.Blog:Tag]')}
    type = 'Acme.Blog:TagPage'
}

 # The "TagPage"
prototype(Acme.Blog:TagPage) < prototype(Neos.Fusion:Collection) {
    collection = ${Search.query(site).nodeType('Acme.Blog:Post').exactMatch('tags', node).sortDesc('creationDate').execute()}
    itemName = 'node'
    itemRenderer = Acme.Blog:SingleTag
}
prototype(Acme.Blog:SingleTag) < prototype(Neos.Neos:Template) {
    ...
}
```

#### Making OR queries

There's no OR operator provided in this package, so you need to use a custom Elasticsearch query filter for that:

```
....queryFilter('bool', {should: [
    {term: {tags: tagNode.identifier}},
    {term: {places: tagNode.identifier}},
    {term: {projects: tagNode.identifier}}
]})
```

## Aggregations

Aggregation is an easy way to aggregate your node data in different ways. Elasticsearch provides a couple of different types of
aggregations. Check `https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html` for more
info about aggregations. You can use them to get some simple aggregations like min, max or average values for
your node data. Aggregations also allows you to build a complex filter for e.g. a product search or statistics.

**Aggregation methods**
Right now there are two methods implemented. One generic `aggregation` function that allows you to add any kind of
aggregation definition and a pre-configured `fieldBasedAggregation`. Both methods can be added to your TS search query.
You can nest aggregations by providing a parent name.

* `aggregation($name, array $aggregationDefinition, $parentPath = NULL)` -- generic method to add a $aggregationDefinition under a path $parentPath with the name $name.
* `fieldBasedAggregation($name, $field, $type = 'terms', $parentPath = '', $size = 10)` -- adds a simple filed based Aggregation of type $type with name $name under path $parentPath. Used for simple aggregations like sum, avg, min, max or terms. By default 10 buckets are returned.


### Examples

#### Add a average aggregation

To add an average aggregation you can use the fieldBasedAggregation. This snippet would add an average aggregation for
a property price:
```
nodes = ${Search.query(site)...fieldBasedAggregation("avgprice", "price", "avg").execute()}
```
Now you can access your aggregations inside your fluid template with
```
{nodes.aggregations}
```

#### Create a nested aggregation

In this scenario you could have a node that represents a product with the properties price and color. If you would like
to know the average price for all your colors you just nest an aggregation in your Fusion:
```
nodes = ${Search.query(site)...fieldBasedAggregation("colors", "color").fieldBasedAggregation("avgprice", "price", "avg", "colors").execute()}
```
The first `fieldBasedAggregation` will add a simple terms aggregation (https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-terms-aggregation.html)
with the name colors. So all different colors of your nodetype will be listed here.
The second `fieldBasedAggregation` will add another sub-aggregation named avgprice below your colors-aggregation.

You can nest even more aggregations like this:
```
fieldBasedAggregation("anotherAggregation", "field", "avg", "colors.avgprice")
```

#### Add a custom aggregation

To add a custom aggregation you can use the `aggregation()` method. All you have to do is to provide an array with your
aggregation definition. This example would do the same as the fieldBasedAggregation would do for you:
```
aggregationDefinition = Neos.Fusion:RawArray {
    terms = Neos.Fusion:RawArray {
        field = "color"
    }
}
nodes = ${Search.query(site)...aggregation("color", this.aggregationDefinition).execute()}
```

#### Product filter

This is a more complex scenario. With this snippet we will create a full product filter based on your selected Nodes. Imagine
an NodeTye ProductList with an property `products`. This property contains a comma separated list of sku's. This could also
be a reference on other products.

```
prototype(Vendor.Name:FilteredProductList) < prototype(Neos.Neos:Content)
prototype(Vendor.Name:FilteredProductList) {

    // Create SearchFilter for products
    searchFilter = Neos.Fusion:RawArray {
        sku = ${String.split(q(node).property("products"), ",")}
    }

    # Search for all products that matches your queryFilter and add aggregations
    filter = ${Search.query(site).nodeType("Vendor.Name:Product").queryFilterMultiple(this.searchFilter, "must").fieldBasedAggregation("color", "color").fieldBasedAggregation("size", "size").execute()}

    # Add more filter if get/post params are set
    searchFilter.color = ${request.arguments.color}
    searchFilter.color.@if.onlyRenderWhenFilterColorIsSet = ${request.arguments.color != ""}
    searchFilter.size = ${request.arguments.size}
    searchFilter.size.@if.onlyRenderWhenFilterSizeIsSet = ${request.arguments.size != ""}

    # filter your products
    products = ${Search.query(site).nodeType("Vendor.Name:Product").queryFilterMultiple(this.searchFilter, "must").execute()}

    # don't cache this element
    @cache {
        mode = 'uncached'
        context {
            1 = 'node'
            2 = 'site'
        }
    }
```

In the first lines we will add a new searchFilter variable and add your selected sku's as a filter. Based on this selection
we will add two aggregations of type terms. You can access the filter in your template with `{filter.aggregations}`. With
this information it is easy to create a form with some select fields with all available options. If you submit the form
just call the same page and add the get parameter color and/or size.
The next lines will parse those parameters and add them to the searchFilter. Based on your selection all products will
be fetched and passed to your template.


**Important notice**

If you do use the terms filter be aware of Elasticsearchs analyze functionality for strings. You might want to disable this
for all your filterable properties, or else filtering won't work on them properly:
```
'Vendor.Name:Product'
  properties:
    color:
      type: string
      defaultValue: ''
      search:
        elasticSearchMapping:
          type: "string"
          index: 'not_analyzed'
```

**Note:** When using Elasticsearch 5.x the mapping needs to be adjusted in a different way.
More information on the [mapping in ElasticSearch 5.x](Documentation/ElasticMapping-5.x.md).

## Sorting

This package adapts Elasticsearchs sorting capabilities. You can add multiple sort operations to your query.
Right now there are three methods you can use:

* `sortAsc('propertyName')`
* `sortDesc('propertyName')`
* `sort('configuration')`

Just append those method to your query like this:
```
# Sort ascending by property title

nodes = ${q(Search.query(site).....sortAsc("title").execute())}

# Sort for multiple properties

nodes = ${q(Search.query(site).....sortAsc("title").sortDesc("name").execute())}

# Custom sort operation

geoSorting = Neos.Fusion:RawArray {
    _geo_distance = Neos.Fusion:RawArray {
        latlng = Neos.Fusion:RawArray {
            lat = 51.512711
            lon = 7.453084
        }
        order = "plane"
        unit = "km"
        distance_type = "sloppy_arc"
    }
}
nodes = ${Search.query(site).....sort(this.geoSorting).execute()}

```
Check https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-sort.html for more configuration
options.

### Example with pagination and sort by distance

This is how a more complex example could look like. Imagine you a want to render a list of nodes and in addition to each
node you want to display the distance to a specific point.

First of all you have to define a property in your NodeTypes.yaml for your node where to store lat/lon information's:
```
'Vendor.Name:Retailer':
  properties:
    'latlng':
      type: string
      search:
        elasticSearchMapping:
          type: "geo_point"
```

Query your nodes in your Fusion:
```
geoSorting = Neos.Fusion:RawArray {
    _geo_distance = Neos.Fusion:RawArray {
        latlng = Neos.Fusion:RawArray {
            lat = 51.512711
            lon = 7.453084
        }
        order = "plane"
        unit = "km"
        distance_type = "sloppy_arc"
    }
}
nodes = ${Search.query(site).nodeType('Vendor.Name:Retailer').sort(this.geoSorting)}
```

Now you can paginate that nodes in your template. To get your actually distance for each node use
the `GetHitArrayForNodeViewHelper`:
```
{namespace cr=Neos\ContentRepository\Search\ViewHelpers}
{namespace es=Flowpack\ElasticSearch\ContentRepositoryAdaptor\ViewHelpers}

<cr:widget.paginate query="{nodes}" as="paginatedNodes">
    <f:for each="{paginatedNodes}" as="singleNode">
        {singleNode.name} - <es:getHitArrayForNode queryResultObject="{nodes}" node="{singleNode}" path="sort.0" />
    </f:for>
</cr:widget.paginate>

```

The ViewHelper will use \Neos\Utility\Arrays::getValueByPath() to return a specified path. So you can make use
of an array or a string. Check the documentation \Neos\Utility\Arrays::getValueByPath() for more information.

**Important notice**

The ViewHelper GetHitArrayForNode will return the raw hit result array. The path property allows you to access some
specific data like the the sort data. If there is only one value for your path the value will be returned.
If there is more data the full array will be returned by GetHitArrayForNode-VH. So you might have to use the
ForViewHelper to access your sort values.


## Fulltext Search / Indexing

When searching in a fulltext index, we want to show Pages, or, generally speaking, everything
which is a `Document` node. However, the main content of a certain `Document` is often not stored
in the node itself, but inside its (`Content`) child nodes.

This is why we need some special functionality for indexing, which *adds the content of the inner
nodes* to the `Document` nodes where they belong to, to a field called `__fulltext` and
`__fulltextParts`.

Furthermore, we want that a fulltext match e.g. inside a headline is seen as *more important* than
a match inside the normal body text. That's why the `Document` node not only contains one field with
all the texts, but multiple "buckets" where text is added to: One field which contains everything
deemed as "very important" (`__fulltext.h1`), one which is "less important" (`__fulltext.h2`),
and finally one for the plain text (`__fulltext.text`). All of these fields are configured with different `boost` values.

**For a search user interface, checkout the Flowpack.SearchPlugin package**

## Suggestions

Elasticsearch offers an easy way to get query suggestions based on your query. Check
`https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html` for more information about how
you can build and use suggestion in your search.

**Suggestion methods implemented**

There are two methods implemented. `suggestions` is a generic one that allows to build the suggestion query of your
dreams. The other method is `termSuggestions` and is meant for basic term suggestions. They can be added to your totaly
awesome TS search query.

* `suggestions($name, array $suggestionDefinition)` -- generic method to be filled with your own suggestionQuery
* `termSuggestions($term, $field = '_all', $name = 'suggestions'` -- simple term suggestion query on all fields

### Examples

#### Add a simple suggestion to a query

Simple suggestion that returns a suggestion based on the sent term

```
suggestions = ${Search.query(site)...termSuggestions('someTerm')}
```
You can access your suggestions inside your fluid template with
```
{nodes.suggestions}
```

### Add a custom suggestion

Phrase query that returns query suggestions

```
suggestionsQueryDefinition = Neos.Fusion:RawArray {
    text = 'some Text'
    simple_phrase = Neos.Fusion:RawArray {
        phrase = Neos.Fusion:RawArray {
            analyzer = 'body'
            field = 'bigram'
            size = 1
            real_world_error_likelihood = 0.95
            ...
        }
    }
}
suggestions = ${Search.query(site)...suggestions('my_suggestions', this.suggestionsQueryDefinition)}
```

## Calculate the maximum cache time

In order to set the maximum cache time of a fusion prototype that renders nodes fetched by `Search()`,
the nearest future value of the hiddenBeforeDateTime or hiddenAfterDateTime properties of all nodes in the result needs to be calculated.

	prototype(Acme.Blog:Listing) < prototype(Neos.Fusion:Collection) {
		@context.searchQuery = ${Search.query(site).nodeType('Acme.Blog:Post')}
	
	    collection = ${searchQuery.execute()}
	    itemName = 'node'
	    itemRenderer = Acme.Blog:Post
	    
	     @cache {
        	mode = 'cached'
			maximumLifetime = ${searchQuery.cacheLifetime()}
			
        	entryTags {
          	map = ${'NodeType_Acme.Blog:Post'}
        	}
    	}
	}


## Advanced: Configuration of Indexing

**The default configuration supports most usecases and often may not need to be touched, as this package comes
with sane defaults for all Neos data types.**

**Note:** When using Elasticsearch 5.x changes to the your mapping may be needed. More information
on the [mapping in ElasticSearch 5.x](Documentation/ElasticMapping-5.x.md).

Indexing of properties is configured at two places. The defaults per-data-type are configured
inside `Neos.ContentRepository.Search.defaultConfigurationPerType` of `Settings.yaml`.
Furthermore, this can be overridden using the `properties.[....].search` path inside
`NodeTypes.yaml`.

This configuration contains two parts:

* Underneath `elasticSearchMapping`, the Elasticsearch property mapping can be defined.
* Underneath `indexing`, an Eel expression which processes the value before indexing has to be
  specified. It has access to the current `value` and the current `node`.

Example (from the default configuration):
```
 # Settings.yaml
Neos:
  ContentRepository:
    Search:
      defaultConfigurationPerType:

        # strings should just be indexed with their simple value.
        string:
          elasticSearchMapping:
            type: string
          indexing: '${value}'
```

```
 # NodeTypes.yaml
'Neos.Neos:Timable':
  properties:
    '_hiddenBeforeDateTime':
      search:

        # a date should be mapped differently, and in this case we want to use a date format which
        # Elasticsearch understands
        elasticSearchMapping:
          type: DateTime
          format: 'date_time_no_millis'
        indexing: '${(node.hiddenBeforeDateTime ? Date.format(node.hiddenBeforeDateTime, "Y-m-d\TH:i:sP") : null)}'
```

If your nodetypes schema defines custom properties of type DateTime, you have got to provide similar configuration for
them as well in your `NodeTypes.yaml`, or else they will not be indexed correctly.

There are a few indexing helpers inside the `Indexing` namespace which are usable inside the
`indexing` expression. In most cases, you don't need to touch this, but they were needed to build up
the standard indexing configuration:

* `Indexing.buildAllPathPrefixes`: for a path such as `foo/bar/baz`, builds up a list of path
  prefixes, e.g. `['foo', 'foo/bar', 'foo/bar/baz']`.
* `Indexing.extractNodeTypeNamesAndSupertypes(NodeType)`: extracts a list of node type names for
  the passed node type and all of its supertypes
* `Indexing.convertArrayOfNodesToArrayOfNodeIdentifiers(array $nodes)`: convert the given nodes to
  their node identifiers.



## Advanced: Fulltext Indexing

In order to enable fulltext indexing, every `Document` node must be configured as *fulltext root*. Thus,
the following is configured in the default configuration:

```
'Neos.Neos:Document':
  search:
    fulltext:
      isRoot: true
```

A *fulltext root* contains all the *content* of its non-document children, such that when one searches
inside these texts, the document itself is returned as result.

In order to specify how the fulltext of a property in a node should be extracted, this is configured
in `NodeTypes.yaml` at `properties.[propertyName].search.fulltextExtractor`.

An example:

```
'Neos.Neos.NodeTypes:Text':
  properties:
    'text':
      search:
        fulltextExtractor: '${Indexing.extractHtmlTags(value)}'

'My.Blog:Post':
  properties:
    title:
      search:
        fulltextExtractor: '${Indexing.extractInto("h1", value)}'
```


## Fulltext Searching / Search Plugin

**For a search user interface, checkout the Flowpack.SearchPlugin package**


## Working with Dates

As a default, Elasticsearch indexes dates in the UTC Timezone. In order to have it index using the timezone
currently configured in PHP, the configuration for any property in a node which represents a date should look like this:

```
'My.Blog:Post':
  properties:
    date:
      search:
        elasticSearchMapping:
          type: 'date'
          format: 'date_time_no_millis'
        indexing: '${(value ? Date.format(value, "Y-m-d\TH:i:sP") : null)}'
```

This is important so that Date- and Time-based searches work as expected, both when using formatted DateTime strings and
when using relative DateTime calculations (eg.: `now`, `now+1d`).

If you want to filter items by date, e.g. to show items with date later than today, you can create a query like this:

```
${...greaterThan('date', Date.format(Date.Now(), "Y-m-d\TH:i:sP"))...}
```

For more information on Elasticsearch's Date Formats,
[click here](http://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html).


## Working with Assets / Attachments

If you want to index attachments, you need to install the [Elasticsearch Attachment Plugin](https://github.com/elastic/elasticsearch-mapper-attachments).
Then, you can add the following to your `Settings.yaml`:

```
Neos:
  ContentRepository:
    Search:
      defaultConfigurationPerType:
        'Neos\Media\Domain\Model\Asset':
          elasticSearchMapping:
            type: attachment
          indexing: ${Indexing.indexAsset(value)}

        'array<Neos\Media\Domain\Model\Asset>':
          elasticSearchMapping:
            type: attachment
          indexing: ${Indexing.indexAsset(value)}
```

## Configurable Elasticsearch Mapping

(included in version >= 2.1)

If you want to fine-tune the indexing and mapping on a more detailed level, you can do so in the following way.

First, configure the index settings as you need them, e.g. configuring analyzers:

```
Flowpack:
  ElasticSearch:
    indexes:
      default:
        'neoscontentrepository': # This index name must be the same as in the Neos.ContentRepository.Search.elasticSearch.indexName setting
          analysis:
            filter:
              elision:
                type: 'elision'
                articles: [ 'l', 'm', 't', 'qu', 'n', 's', 'j', 'd' ]
            analyzer:
              custom_french_analyzer:
                tokenizer: 'letter'
                filter: [ 'asciifolding', 'lowercase', 'french_stem', 'elision', 'stop' ]
              tag_analyzer:
                tokenizer: 'keyword'
                filter: [ 'asciifolding', 'lowercase' ]
```

Then, you can change the analyzers on a per-field level; or e.g. reconfigure the _all field with the following snippet
in the NodeTypes.yaml. Generally this works by defining the global mapping at `[nodeType].search.elasticSearchMapping`:

```
'Neos.Neos:Node':
  search:
    elasticSearchMapping:
      _all:
        analyzer: custom_french_analyzer
```

Hint: If this leads to error message like:

    mapper [_all] has different [analyzer], mapper [_all] is used by multiple types

you have different (node) types that do not have the same analyzer. Internally Elasticsearch uses the same
configuration for all fields of the same name, even if they are in different types. Use the `nodeindex:showmapping`
command to check for any node type that does not have `\_all` configured as expected and adjust it as well.

## Change the default Elastic index name

If you need to run serveral (different) neos instances on the same elasticsearch server you will need to change the Configuration/Settings.yaml indexName for each of your project.

So `./flow nodeindex:build` or `./flow nodeindex:cleanup` won't overwrite your other sites index.

```
Neos:
  ContentRepository:
    Search:
      elasticSearch:
        indexName: useMoreSpecificIndexName
```

## Debugging

In order to understand what's going on, the following commands are helpful:

* use `./flow nodeindex:showMapping` to show the currently defined Elasticsearch Mapping
* use the `.log()` statement inside queries to dump them to the Elasticsearch Log
* the logfile `Data/Logs/ElasticSearch.log` contains loads of helpful information.

**Settings.yaml**

1. Change the base namespace for configuration from `Flowpack.ElasticSearch.ContentRepositoryAdaptor`
   to `Neos.ContentRepository.Search`. All further adjustments are made underneath this namespace:
2. (If it exists in your configuration:) Move `indexName` to `elasticSearch.indexName`
3. (If it exists in your configuration:) Move `log` to `elasticSearch.log`
4. search for `mapping` (inside `defaultConfigurationPerType.<typeName>`) and replace it by
   `elasticSearchMapping`.
5. Inside the `indexing` expressions (at `defaultConfigurationPerType.<typeName>`), replace
   `ElasticSearch.` by `Indexing.`.

**NodeTypes.yaml**

1. Replace `elasticSearch` by `search`. This replaces both `<YourNodeType>.elasticSearch`
   and `<YourNodeType>.properties.<propertyName>.elasticSearch`.
2. search for `mapping` (inside `<YourNodeType>.properties.<propertyName>.search`) and replace it by
   `elasticSearchMapping`.
3. Replace `ElasticSeach.fulltext` by `Indexing`
4. Search for `ElasticSearch.` (inside the `indexing` expressions) and replace them by `Indexing.`

Created by Sebastian Kurfürst; [contributions by Karsten Dambekalns, Robert Lemke and others](https://github.com/Flowpack/Flowpack.ElasticSearch.ContentRepositoryAdaptor/graphs/contributors).
