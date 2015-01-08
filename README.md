# Elastic-Harvest

Elastic-Harvest is a Nodejs implementation of the [JSON API Search Profile](https://github.com/agco-adm/json-api-search-profile).

This library ties together [Harvest](https://github.com/agco-adm/elastic-harvest) and Elasticsearch to offer the required [linked resource filtering and aggregation](https://github.com/agco-adm/json-api-search-profile/blob/master/public/profile.md) features.

Apart from that it also provides a number of helper functions to synchronize Harvest / Mongodb resources with an Elasticsearch backend.


## Features

- Aggregations : stats, extended_stats, top_hits, terms
- Primary and Linked resource filtering interop

## Roadmap

- Top_hits aggregation interop with JSON API features, inclusion and sparse fieldsets [#1](https://github.com/agco-adm/elastic-harvest/issues/1)
- More aggregations : min, max, sum, avg, percentiles, percentile_ranks, cardinality, geo_bounds, significant_terms, range, date_range, filter, filters, missing, histogram, date_histogram, geo_distance
- Reliable Harvest/Mongodb - Elasticsearch data synchronisation ( oplog based )
- Support adaptive queries, use the ES mapping file to figure out whether to use parent/child or nested queries / aggregations
- Use Harvest associations + ES mapping file to discover which Mongodb collections have to be synced rather than having to register them explicitly
- Bootstrap Elasticsearch with existing data from Harvest resources through REST endpoint
- Bootstrap Elasticsearch mapping file through REST endpoint

## Dependencies
 
TODO


## Usage

```js
//Hash of properties to related mongo collections
var collectionLookup = {
    "brand": "brand",
    "product_type": "product_type",
    "contract_type": "contract_type",
    "region": "region",
    "country": "country",
    "address_country": "country",
    "address_state_province": "state_province",
    "current_contracts": "contract",
    "phone_numbers": "phone_number",
    "business_hours": "business_hours",
    "current_offerings": "offering",
    "dealer_misc": "dealers_misc"
}
#####NB: collectionLookup deprecated in v0.0.10 - it's now automatically generated.
var Elastic_Search_URL = process.env.BONSAI_URL || "http://127.0.0.1:9200";
var Elastic_Search_Index = "dealer-api";
var type = "dealers";
```
#### Create elastic search endpoint (NB: api changed in v0.0.6)
```js
var dealerSearch = new ElasticFortune(fortune_app, Elastic_Search_URL,Elastic_Search_Index, type, collectionLookup);
fortune_app.router.get('/dealers/search', dealerSearch.route);
//Required to make the elastic search endpoint work properly
fortune_app.onRouteCreated('dealer').then(function(fortuneRoute){
    dealerSearch.setFortuneRoute(fortuneRoute);
});
```


#### Create an :after callback & sync elastic search after each item is posted to harvest
#####Note - only 1 "after" callback is allowed per endpoint, so if you enable autosync, you're giving it up to elastic-harvest.
```js
dealerSearch.enableAutoSync("dealer");
```


#### Alternative way to create an :after endpoint & sync elastic search. This approach gives you access to do more in the after callback.
```js
this.fortune_app.after("dealer", function (req, res, next) {
    if (req.method === 'POST' || (req.method === 'PUT' && this.id)) {
        return dealerSearch.expandAndSync(this);
    } else {
        return this;
    }
});
```    


#### Expand an object's links:
```js
dealerSearch.expandEntity(dealer);
```


#### Send an object to elastic search after expanding it's links:
```js
dealerSearch.expandAndSync(dealer);
```


#### Send an object to elastic search without expanding it's links:
```js
dealerSearch.sync(dealer);
```


#### Delete an object in elastic search: (added in 0.0.3)
```js
dealerSearch.delete(dealer.id);
```


#### Create an :after callback & keep your elastic search index up to date with PUTs and POSTs on linked documents. (added in 0.0.5)
#####Note - only 1 "after" callback is allowed per endpoint, so if you enable indexUpdateOnModelUpdate, you're giving it up to elastic-harvest.
```js
dealerSearch.enableAutoIndexUpdateOnModelUpdate("subdocumentsFortuneEndpoint","links.path.to.object.id");
e.g. dealerSearch.enableAutoIndexUpdateOnModelUpdate("brand","links.current_contracts.brand.id");
```


#### Update Elastic Search index when a related harvest model changes (added in 0.0.5)
```js
entity = this;
dealerSearch.updateIndexForLinkedDocument("links.path.to.object.id",entity);
```

#### Delete ES Index (added in 0.0.9)
```js
dealerSearch.deleteIndex().
```

#### Initialize ES Index (added in 0.0.9)
```js
dealerSearch.initializeIndex().
```

#### Initialize an elastic search mapping (added in 0.0.6, updated in 0.0.9)
```js
dealerSearch.initializeMapping(mappingObject).
```
v0.0.9 update provides automatic handling of missing-index errors.

The Mapping object can be loaded from a js file that looks like:
```js
module.exports= {
    "trackingPoints": {
        "properties": {
            "data": {
                "type": "nested"
            },
            "loc" : {
                "type" : "nested",
                "properties": {
                    "location" : {
                        "type" : "geo_point"
                    }
                }
            },
            "time" : {
                "type" : "date"
            },
            "links": {
                "type": "nested",
                "properties": {
                    "equipment": {
                        "type": "nested",
                        "properties": {
                            "model": {
                                "type": "nested",
                                "properties": {
                                    "brand":{
                                        "type": "nested",
                                        "properties": {
                                            "name":{
                                                "type": "string",
                                                "index": "not_analyzed"
                                            }
                                        }
                                    },
                                    "equipmentType":{
                                        "type": "nested",
                                        "properties": {
                                            "value":{
                                                "type": "string",
                                                "index": "not_analyzed"
                                            }
                                        }
                                    },
                                    "name":{
                                        "type": "string",
                                        "index": "not_analyzed"
                                    }
                                }
                            }
                        }
                    },
                    "duty": {
                        "type": "nested",
                        "properties": {
                            "status":{
                                "type": "string",
                                "index": "not_analyzed"
                            }
                        }
                    }
                }
            }
        }
    }
}
```
