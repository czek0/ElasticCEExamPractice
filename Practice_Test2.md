# Practice Set 2
This set of practice questions uses data from the `kibana_sample_data_eccomerce` ddataset.

#### Question: 1
Create a dummy index named ecommerce_shard_test with a single shard and no replicas. This setup should cause the cluster health to turn yellow. After creating the index, verify the cluster health status. Then, fix the shard issue by increasing the number of replicas and verify that the cluster health returns to green.

Topics Covered: Diagnose shard issues and repair a cluster's health.

<details close>
    <summary><b>Solution</b></summary>

``` bash
PUT eccommerce_shard_test 
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  }
}

GET _cat/shards?v=true&h=index,shard,prirep,state,node,unassigned.reason&s=state
GET _cluster/allocation/explain
{
  "index": "eccommerce_shard_test", 
  "shard": 0, 
  "primary": true 
}

PUT eccommerce_shard_test/_settings
{
  "number_of_replicas": 1
}
```
</details>

#### Question: 2
Create an alias ecommerce_orders for the kibana_sample_data_ecommerce index called `ecommerce`. Write a query to search through this alias and retrieve all documents where the `customer_gender` is "MALE". Ensure that the alias is correctly used in the query.

Topics Covered: Define and use index aliases.

<details close>
    <summary><b>Solution</b></summary>

``` bash
POST _aliases
{
  "actions": [
    {
      "add": {
        "index": "kibana_sample_data_ecommerce",
        "alias": "ecommerce"
      }
    }
  ]
}

GET ecommerce/_search?size=1
GET ecommerce/_search
{
  "query": {
    "term": {
      "customer_gender": "MALE"
    }
  }
}

```
</details>

#### Question: 3
Define a data stream named `ecommerce_data_stream` using an index template called `ecommerce_template`. This template should apply to all indices matching the pattern `ecommerce_data_stream*` and include the following mappings:

* @timestamp: date type
* order_date: date type
* customer_id: keyword type

Once the data stream is created, add the following document to it:

```json
POST /ecommerce_data_stream/_doc/
{
  "@timestamp": "2024-08-05T09:28:48+00:00",
  "order_date": "2024-08-05T09:28:48+00:00",
  "customer_id": "38",
  "taxless_total_price": 36.98,
  "total_quantity": 2
}
```

Finally, perform a search query to verify that the data stream is working correctly by retrieving the document you just added.

Topics Covered: Define an index template that creates a new data stream, Verify data stream functionality.
<details close>
    <summary><b>Solution</b></summary>

``` Bash
PUT /_index_template/ecommerce_template
{
  "index_patterns": ["ecommerce_data_stream*"],
  "data_stream": { },
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "order_date": {
          "type": "date"
        },
        "customer_id": {
          "type": "keyword"
        }
      }
    }
  }
}

PUT /_data_stream/ecommerce_data_stream

POST /ecommerce_data_stream/_doc/
{
  "@timestamp": "2024-08-05T09:28:48+00:00",
  "order_date": "2024-08-05T09:28:48+00:00",
  "customer_id": "38",
  "taxless_total_price": 36.98,
  "total_quantity": 2
}

GET /ecommerce_data_stream/_search
{
  "query": {
    "match_all": {}
  }
}



```
</details>

#### Question: 4
Write and execute a search query on the `ecommerce` allias that retrieves all orders where the total quantity of products is greater than 5 or where the total price is greater than 100 EUR. Display only the `order_id`, `total_quantity`, and `taxful_total_price` fields in the results. Then, count how many orders match these criteria.

Topics Covered: Write and execute a search query that is a Boolean combination of multiple queries and filters.

<details close>
    <summary><b>Solution</b></summary>

``` bash
GET ecommerce/_search
{
  "_source": ["order_id","total_quantity","taxful_total_price"], 
  "query": {
    "bool": {
      "should": [
        {"range": {
          "total_quantity": {
            "gt": 5
          }
        }
        },
        {"range": {
          "taxful_total_price": {
            "gt": 100
          }
        }
        }
      ],
      "minimum_should_match": 1
    }
  },
  "aggs": {
    "count_matching criteria": {
      "cardinality": {
        "field": "order_id"
      }
    }
  }
}
```
</details>

#### Question: 5
Split the `kibana_sample_data_ecommerce` across two clusters.  `cluster1` has kibana_sample_data_ecommerce reindexed into `ecommerce_split1` that contains documents where the `taxful_total_price` is < 100. `cluster2` has `kibana_sample_data_ecommerce` reindexed into `ecommerce_split2` that contains documents where the `taxful_total_price` is >= 100. 
Implement a search query that retrieves orders from the kibana_sample_data_ecommerce index across two clusters. Filter for orders where the `manufacturer` is "Elitelligence" and calculate the average `taxful_total_price`. Show no results.

Topics Covered: Write and execute a query that searches across multiple clusters, Write and execute metric and bucket aggregations. Update by Query API

<details close>
    <summary><b>Solution</b></summary>

``` bash
# Cluster 1
PUT ecommerce_split1

POST _reindex
{
  "source": {
    "index": "ecommerce"
  },
  "dest": {
    "index": "ecommerce_split1"
  }
}

# change ecommerce_split1
POST ecommerce_split1/_delete_by_query
{
  "query": {
    "range": {
      "taxful_total_price": {
        "gte": 100
      }
    }
  }
}

GET ecommerce_split1/_search
{
  "query": {
    "range": {
      "taxful_total_price": {
        "gte": 100
      }
    }
  }
}
# Cluster 2
PUT ecommerce_split2

POST _reindex
{
  "source": {
    "index": "kibana_sample_data_ecommerce"
  },
  "dest": {
    "index": "ecommerce_split2"
  }
}

# change ecommerce_split2
POST ecommerce_split2/_delete_by_query
{
  "query": {
    "range": {
      "taxful_total_price": {
        "lt": 100
      }
    }
  }
}

GET ecommerce_split2/_search
{
  "query": {
    "range": {
      "taxful_total_price": {
        "lt": 100
      }
    }
  }
}

# set up remote cluster (skip if you have done practice set 1)
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster2_remote": {
          "skip_unavailable": false,
          "mode": "sniff",
          "proxy_address": null,
          "proxy_socket_connections": null,
          "server_name": null,
          "seeds": [
            "node5:9304"
          ],
          "node_connections": 3
        }
      }
    }
  }
}

# check that kibana_sample_data_ecommerce.count = split1.count + split2.count
GET kibana_sample_data_ecommerce/_count
GET ecommerce_split1/_count
GET cluster2_remote:ecommerce_split2/_count
# 3725 + 950 = 4675 :) 


GET ecommerce/_mapping
GET ecommerce_split1,cluster2_remote:ecommerce_split2/_search?size=0
{
  "query": {
    "match": {
      "manufacturer": "Elitelligence"
    }
  },
  "aggs": {
    "Average_taxful_total_price": {
      "avg": {
        "field": "taxful_total_price"
      }
    }
  }
}


```
</details>

#### Question: 6
Write a search query on the `ecommerce` allias that uses a runtime field, `total_order_USD` to calculate the total value of orders in USD based on a conversion rate (e.g., 1 EUR = 1.2 USD). Use `taxless_total_price`. Filter for orders where the `total_order_USD` in USD is greater than 120. Ensure the runtime field is visible in the search results.

Topics Covered: Write and execute a search that utilizes a runtime field.

<details close>
    <summary><b>Solution</b></summary>

``` bash
GET kibana_sample_data_ecommerce/_search
{
  "_source": ["total_order_USD"], 
  "runtime_mappings": {
    "total_order_USD": {
      "type": "double",
      "script": {
        "source": "emit(doc['taxless_total_price'].value * 1.2);"
      }
    }
  },
  "query": {
    "range": {
      "total_order_USD": {
        "gt" : 120
      }
    }
  },
  "script_fields": {
    "total_order_USD": {
      "script": {
        "source": "doc['taxless_total_price'].value * 1.2"
      }
    }
  }
}

```
</details>

#### Question: 7
Write a query that searches for orders with the word "dress" in the `product_name` field and highlights the term in the search results. Ensure that the `product_name` field is displayed with the highlighted search term.

Topics Covered: Highlight the search terms in the response of a query.

<details close>
    <summary><b>Solution</b></summary>

``` bash
GET ecommerce/_search
{
  "query": {
    "match": { "products.product_name": "dress" }
  },
  "highlight": {
    "fields": {
      "products.product_name": {}
    }
  }
}
```
</details>

#### Question: 8
Define an ingest pipeline named `ecommerce_status_pipeline` that adds a field called `OrderStatus` based on the `taxful_total_price`. If the total price is greater than 100 EUR, set the status to "High Value"; otherwise, set it to "Regular". Reindex the `kibana_sample_data_ecommerce` index into a new index using this pipeline and verify the changes.

Topics Covered: Define and use an ingest pipeline that satisfies a given set of requirements, including the use of Painless to modify documents.


<details close>
    <summary><b>Solution</b></summary>

``` Bash
PUT _ingest/pipeline/ecommerce_status_pipeline
{
  "description": "adds a field called `OrderStatus` based on the `taxful_total_price`",
  "processors": [
    {
      "script": {
        "source": "ctx[\"OrderStatus\"] = \"\";\nif (ctx[\"taxful_total_price\"] > 100) {\n    ctx[\"OrderStatus\"] = \"High Value\";\n}\nelse {\n    ctx[\"OrderStatus\"] = \"Regular\";\n}\n\n"
      }
    }
  ]
}
# get an doc id to test the pipeline
GET ecommerce/_search?size=1


PUT ecommerce_with_orderstatus

POST _reindex
{
  "source": {
    "index": "kibana_sample_data_ecommerce"
  },
  "dest": {
    "index": "ecommerce_with_orderstatus",
    "pipeline": "ecommerce_status_pipeline"
  }
}
GET ecommerce_with_orderstatus/_mapping
```
</details>

#### Question: 9
Create a snapshot repository named ecommerce_snapshots and define a snapshot lifecycle policy called `daily_snapshot_policy` to automatically take snapshots of the `kibana_sample_data_ecommerce` index every day at 4:30 PM. Manually create a snapshot of the kibana_sample_data_ecommerce index using this snapshot repository, then delete the `kibana_sample_data_ecommerce` index. Mount the snapshot as a searchable snapshot named `searchable_snap_ecommerce`, and ensure that the restored index is searchable by retrieving all documents where the `total_quantity` is greater than 1. Afterward, clean up by deleting the cache and the mounted searchable snapshot. Finally, restore the snapshot into a new index named `kibana_sample_data_ecommerce`.

Topics Covered: Backup and restore a cluster and/or specific indices, Configure a snapshot to be searchable.

<details close>
    <summary><b>Solution</b></summary>

``` bash
# Create the snapshot
PUT _snapshot/ecommerce_snapshots
{
  "type": "fs",
  "settings": {
    "location": "/mnt/elastic_backups"
  }
}
# Define the lifecycle policy
PUT /_slm/policy/daily_snapshot_policy
{
  "schedule": "0 30 16 * * ?",  # 4:30 PM daily
  "name": "<daily-snapshot-{now/d}>",
  "repository": "ecommerce_snapshots",
  "config": {
    "indices": ["kibana_sample_data_ecommerce"],
    "ignore_unavailable": true,
    "include_global_state": false
  },
  "retention": {
    "expire_after": "30d",  # Keep snapshots for 30 days
    "min_count": 1,
    "max_count": 30
  }
}
# Manually Create the Snapshot
PUT _snapshot/ecommerce_snapshots/snapshot1
{
  "indices": "kibana_sample_data_ecommerce"
}


# Delete the old index
DELETE kibana_sample_data_ecommerce

# Mount the snapshot for searching
POST /_snapshot/ecommerce_snapshots/snapshot1/_mount?wait_for_completion=true
{
  "index": "kibana_sample_data_ecommerce", 
  "renamed_index": "searchable_snap_ecommerce", 
  "index_settings": { 
    "index.number_of_replicas": 0
  },
  "ignore_index_settings": [ "index.refresh_interval" ] 
}

# check the index is in the searchable snapshots
GET /searchable_snap_ecommerce/stats

GET searchable_snap_ecommerce/_search
{
  "query": {
    "range": {
      "total_quantity": {
        "gt": 1
      }
    }
  }
}


# cleanup and delete
POST /_searchable_snapshots/cache/clear

DELETE searchable_snap_ecommerce


# Restore the snapshot
POST _snapshot/ecommerce_snapshots/snapshot1/_restore
{
  "indices": "kibana_sample_data_ecommerce"
}

# Verify
GET kibana_sample_data_ecommerce/_count








```
</details>

#### Question: 10
Write and execute two queries on the `kibana_sample_data_ecommerce` index that demonstrates the difference between sibling and nested aggregations. 
For each query. calculate the average `taxful_total_price` for each product category.
In query 1, display the product that the maxium average price. Your results for query 1 should display each category and the maxium average price.
In query 2, sort the average price for each category by `desc` and return the product with the maxium average price. Your results for query 2 should only display the product with the maxium average price.

Show no hits. Have each query in sepeate `GET` calls. 

Compare your results, are they the same product and price?

Topics Covered: Write and execute aggregations that contain sub-aggregations, Differentiate between sibling and nested aggregations.

<details close>
    <summary><b>Solution</b></summary>

``` Bash
# Query 1
GET /kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "product_categories": {
      "terms": {
        "field": "products.category.keyword"
      },
      "aggs": {
        "avg_taxful_price": {
          "avg": {
            "field": "taxful_total_price"
          }
        }
      }
    },
    "maxium_avg_price": {
      "max_bucket": {
        "buckets_path": "product_categories>avg_taxful_price" 
      }
    }
  }
}

# Query 2
GET /kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "product_categories": {
      "terms": {
        "field": "products.category.keyword"
      },
      "aggs": {
        "avg_taxful_price": {
          "avg": {
            "field": "taxful_total_price"
          }
        },
        "avg_bucket_sort": {
            "bucket_sort": {
              "sort": [
              { "avg_taxful_price": { "order": "desc" } } 
            ],
              "size": 1                                
            }
          }
      }
    }
  }
}

```
</details>
