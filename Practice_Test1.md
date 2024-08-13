# Practice Set 1
This Practice Set uses the `kibana-sample-data-flights`index as part of Elastic's sample data.
#### Question: 1
 Define an index `flights_fixed` for the kibana_sample_data_flights dataset that includes a custom analyzer for the `Carrier` field called `case-analyzer`, allowing case-insensitive searches. It should use the `standard` tokenizer and make sure all characters are the same case. Provide the mapping and index creation request, check that the new mapping of `flights_fixed` show the custom analzyer for the `Carrier` field.

Topics Covered: Define an index that satisfies a given set of requirements, Define and use a custom analyzer.

<details close>
    <summary><b>Solution</b></summary>

```bash
PUT flights_fixed
{
  "settings": {
    "analysis": {
      "analyzer": {
        "case-analyzer": {
          "type": "custom", 
          "tokenizer": "standard",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "Carrier" : {
        "type": "text",
        "analyzer": "case-analyzer"
      }
    }
  }
}

POST flights_fixed/_analyze
{
  "analyzer": "case-analyzer",
  "text": "JetStar"
}

POST _reindex
{
  "source": {
    "index": "kibana_sample_data_flights"
  },
  "dest": {
    "index": "flights_fixed"
  }
}

GET flights_fixed/_mapping
```

</details>

#### Question: 2
 Delete `flights_fixed` and create a new dynamic template called `kibana_flights` for the kibana_sample_data_flights index that ensures all string fields are mapped as text and keyword, except for the Carrier field, which should be mapped as text with a lowercase analyzer. Reindex into `kibana_flights` and check that the mappings are correct

Topics Covered: Define and use a dynamic template that satisfies a given set of requirements.

<details close>
    <summary><b>Solution</b></summary>

``` bash

PUT kibana_flights/
{
  "settings": {
    "analysis": {
      "analyzer": {
        "case-analyzer": {
          "type": "custom", 
          "tokenizer": "standard",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_ip": {
          "match_mapping_type": "string",
          "match": "*",
          "unmatch" : "Carrier",
          "mapping" : {
            "type" : "keyword"
          }
        }
      }
    ],
    "properties": {
      "Carrier" : {
        "type": "text",
        "analyzer": "case-analyzer"
      }
    }
  }
}


POST _reindex
{
  "source": {
    "index": "kibana_sample_data_flights"
  },
  "dest": {
    "index": "kibana_flights"
  }
}

GET kibana_flights/_mapping

```

</details>



#### Question: 3
Define an Index Lifecycle Management (ILM) policy called `flights_ILM` for the kibana_sample_data_flights index with the following phases:

* Hot Phase: Roll over the index after 5 minutes.
* Warm Phase: Transition to the warm phase 3 minutes after the rollover, perform a force merge to 1 segment, and shrink the index to 1 shard.
* Cold Phase: Transition to the cold phase 6 minutes after the rollover, and set the number of replicas to 0.
* Delete Phase: Delete the index 8 minutes after the rollover.

Topics Covered: Define an Index Lifecycle Management policy for a time-series index.

<details close>
    <summary><b>Solution</b></summary>

```bash
PUT _ilm/policy/flights_ILM
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "5m"
          },
          "set_priority": {
            "priority": 100
          }
        },
        "min_age": "0ms"
      },
      "warm": {
        "min_age": "8m",
        "actions": {
          "set_priority": {
            "priority": 50
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
          }
        }
      },
      "cold": {
        "min_age": "11m",
        "actions": {
          "set_priority": {
            "priority": 0
          },
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "delete": {
        "min_age": "13m",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

</details>



#### Question: 4
Question: Write and execute a search query that retrieves flights that were either delayed by more than or equal to 60 minutes or canceled, where the destination was `US`. Configure the results to only display `DestCountry`, `Cancelled`, `FlighDelayMin`. Lastly, display a count of how many US flights had `Cancelled` set to True or False.

Topics Covered: Write and execute a search query that is a Boolean combination of multiple queries and filters.

<details close>
    <summary><b>Solution</b></summary>

```bash

GET kibana_flights/_search
GET kibana_flights/_search
{
  "_source": ["DestCountry", "Cancelled", "FlightDelayMin"], 
  "query": {
    "bool": {
      "should": [
        {"range": {
          "FlightDelayMin": {
            "gte": 60
          }
        }},
        {"term": {
          "Cancelled": true
        }}
      ],
      "minimum_should_match": 1,
      "filter": [
        {"term": {
          "DestCountry": "US"
        }}
      ]
    }
  },
  "sort": [
    {
      "FlightDelayMin": {
        "order": "asc"
      }
    }
  ],
  "aggs": {
    "sum_cancelled": {
      "terms": {
        "field": "Cancelled"
      }
      }
  }
}

```
</details>


#### Question: 5
Question: Implement a search query that retrieves flights from the kibana_sample_data_flights index across two clusters, filtering for flights originating from `"Tokyo Haneda International Airport"` and calculating the average ticket price.

Topics Covered: Write and execute a query that searches across multiple clusters, Write and execute metric and bucket aggregations.
<details close>
    <summary><b>Solution</b></summary>

```bash



####
POST kibana_sample_data_flights/_delete_by_query
{
  "query": {
    "match": {
      "DestCountry": "US"
    }
  }
}


GET kibana_sample_data_flights/_search
{
  "query": {
    "match": {
      "DestCountry": "US"
    }
  }
}



# In Cluster 2: change the index to only have flights to US
POST kibana_sample_data_flights/_delete_by_query
{
  "query": {
    "bool": {
      "must_not": [
        {"match": {
          "DestCountry": "US"
        }}
      ]
    }
  }
}




GET kibana_sample_data_flights/_search
{
  "query": {
    "bool": {
      "must_not": [
        {"match": {
          "DestCountry": "US"
        }}
      ]
    }
  }
}


# In Cluster 1:
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

GET cluster2_remote:kibana_sample_data_flights/_search

GET kibana_sample_data_flights,cluster2_remote:kibana_sample_data_flights/_search
{
  "query": {
  "term": {
    "Origin": "Tokyo Haneda International Airport"
  }
  },
  "aggs": {
    "avg_ticket_price": {
      "avg": {
        "field": "AvgTicketPrice"
      }
    }
  }
}

```
</details>

#### Question: 6
Question: Write a search query on `kibana_sample_data_flights` that uses a runtime field to calculate the total flight time in hours and filters for flights longer than 10 hours.  Create a scripted field to display the total flight time in hours.

Topics Covered: Write and execute a search that utilizes a runtime field.
<details close>
    <summary><b>Solution</b></summary>

```bash

GET kibana_sample_data_flights/_search?
{
  "_source": ["FlightNum", "total_flight_time"], 
  "runtime_mappings": {
    "total_flight_time": {
      "type": "double",
      "script": {
        "source": "emit(doc['FlightTimeMin'].value / 60.0 )"
      }
    }
  },
  "query": {
    "range" : {
      "total_flight_time" : {
        "gte": 10.0
      }
    }
  },
  "script_fields": {
    "total_flight_time": {
      "script": {
        "source": "doc['FlightTimeMin'].value / 60.0"
      }
    }
  }
}

GET kiba/_search


GET kibana_sample_data_flights
```
</details>



#### Question: 7
Question: Write a query that searches for flights with the word "Weather Delay" in the FlightDelayType field and highlights the term in the search results.

Topics Covered: Highlight the search terms in the response of a query.
<details close>
    <summary><b>Solution</b></summary>

```bash

GET kibana_sample_data_flights/_search
{
  "query": {
    "term": {
      "FlightDelayType": {
        "value": "Weather Delay"
      }
    }
  },
  "highlight": {
    "fields": {"FlightDelayType": {}}
  }
}

```
</details>

#### Question: 8
Define an ingest pipeline that adds a field called FlightStatus based on the FlightDelay and Cancelled fields. If a flight is delayed, the status should be "Delayed"; if canceled, "Canceled"; otherwise, "On Time". Use Painless scripting. Then, remove the `FlightDelay` and `Cancelled` fields. Reindex `Kibana_sample_data_flights` into a new index using this pipeline.

Topics Covered: Define and use an ingest pipeline that satisfies a given set of requirements, including the use of Painless to modify documents.
<details close>
    <summary><b>Solution</b></summary>

```bash

PUT _ingest/pipeline/flightstatus_pipeline
{
  "processors": [
  {
    "script": {
      "source": "String response = \"\";\nif (ctx['FlightDelay'] == true) {\n    response = \"Delayed\"\n} else if (ctx['Cancelled'] == true) {\n    response = \"Cancelled\"\n} else {\n    response = \"On time\"\n}\n\nctx['Flight Status'] = response;"
    }
  },
  {
    "remove": {
      "field": [
        "FlightDelay",
        "Cancelled"
      ]
    }
  }
  ]
}

PUT flights_with_status
POST _reindex 
{
  "source" :
  {
    "index" : "kibana_sample_data_flights"
  },
  "dest" : 
  {
    "index": "flights_with_status",
    "pipeline": "flightstatus_pipeline"
  }
}

GET flights_with_status/_search?size=1
```
</details>

#### Question: 9
Delete `kibana_sample_data_flights` and set up cross-cluster replication for the kibana_sample_data_flights index from your previous cluster2_remote and ensure that the data is backed up using a snapshot repository. Name your new index `flights_replicated`.
<details close>
    <summary><b>Solution</b></summary>

```bash

# Register a new snaprepo in StackManagement>snaprepositories

GET _snapshot/flights_repo

PUT _snapshot/flights_repo/my-snap
{
  "indices": "kibana_sample_data_flights"
}


DELETE kibana_sample_data_flights


PUT /flights_replicated/_ccr/follow
{
  "remote_cluster": "cluster2_remote",
  "leader_index": "kibana_sample_data_flights",
  "max_read_request_operation_count": 5120,
  "max_outstanding_read_requests": 12,
  "max_read_request_size": "32mb",
  "max_write_request_operation_count": 5120,
  "max_write_request_size": "9223372036854775807b",
  "max_outstanding_write_requests": 9,
  "max_write_buffer_count": 2147483647,
  "max_write_buffer_size": "512mb",
  "max_retry_delay": "500ms",
  "read_poll_timeout": "1m"
}

GET flights_replicated/_search
```
</details>

#### Question: 10
Create a search template for querying flights with pagination. The query should return results for flights on a specific day of the week and allow for paginated results based on the user's input. Test it with the `query_number` set to 2 and get the results on the third page with 10 documents per page

Topics Covered: Implement pagination of the results of a search query, Define and use a search template.
<details close>
    <summary><b>Solution</b></summary>

```bash
PUT _scripts/day_of_week_search
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "match": {
          "dayOfWeek": "{{query_number}}"
        }
      },
      "from": "{{from}}",
      "size": "{{size}}"
    }
  }
}

GET flights_replicated/_search/template
{
  "id": "day_of_week_search",
  "params": {
    "query_number": 2,
    "from": 20,
    "size": 10
  }
}
```
</details>
