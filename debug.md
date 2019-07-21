# Debug Steps

## First, check shard, cluster and indices health

You can use `curl`, though remember you'll likely have to pass a username and password, and may want to pipe the results into `jq` or `fx` for readability, like so:

`curl -u elastic:elastic_password -X GET "localhost:9200/_cluster/health" | jq`

`curl -u elastic:elastic_password http://localhost:9200/_cat/shards`

`curl -u elastic:elastic_password -XPOST http://localhost:9200/INDEX_NAME/_settings { "index": { "number of replicas": 0} }`

etc.

(or from a remote machine by replacing `localhost` with your Elasticsearch box's IP, provided you've exposed it to the outside)

Or directly in Kibana's dev tools, where you won't need to pass along credentials:

- Indices health `GET _cat/indices?v`
- Cluster health `GET _cluster/health`
- Shard stores `GET _shard_stores?pretty`

## Check allocations by vluster

- `GET _cluster/allocation/explain?pretty`
- `GET _cluster/allocation/explain?include_disk_info=true`
- General: `GET _cat/allocation?v`

## Get unassigned shards

This seems to come up by default. More shards than nodes. The fix is to manually remove the unassigned shards, or to create index templates.

See which are unassigned:

`GET _cat/shards?h=index,shard,prirep,state,unassigned.reason | grep UNASSIGNED`

or

`GET _cat/shards | egrep (UNASSIGNED|INIT)`

or simply

`GET _cat/shards | grep unassigned`

You can also try forcing rerouting:

`POST _cluster/reroute?retry_failed`

and

`POST _cluster/reroute -d '{ "commands": [ { "allocate_empty_primary": { "index": "INDEX", "shard": 0 , "node": "NODE_ID", "accept_data_loss" : true } } ] }'`

...accept_data_loss...

## Getting settings

- General settings `GET _settings`
- Cluster settings `GET _cluster/settings`
- Specific index settings `GET INDEX_NAME/_settings`

### Change settings:

```
PUT INDEX_NAME/_settings
  {
    "index" : {
      "number_of_replicas" : "0"
    }
  }
```

Set number of replicas to 0 for a given index, to get rid of unassigned shards:

```
PUT INDEX_NAME/_settings
  {
    "index" : {
      "number_of_replicas" : "0"
    }
  }
```

## Index templates

Set some defaults for your indices. This is good for preventing unallocated shards, at the very least.

Making a default template for fluentd, APM and logstash logs:

```
PUT _template/default
{
  "order" : 0,
  "index_patterns" : [
    "fluentd-*",
    "apm-*",
    "logstash-*"
  ],
  "settings" : {
    "index" : {
      "number_of_shards" : "1",
      "number_of_replicas" : "0"
    }
  },
  "mappings" : { },
  "aliases" : { }
}
```

The template should also include a mapping (it's possible to simply match all), and you can set the number of replicas (0 fixes unallocated shards).


## Search

```
GET _search?pretty
{
  "query": {
    "term": {
      "your_term": {
        "value": "your_value"
      }
    }
  }
}
```