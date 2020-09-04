# How we manage iterations to our Elasticsearch Mapping

## The Architecture

We have a KCL (Kinesis Consumer Library) python application that consumes data from CDC (Change Data Capture) streams (thanks, [Maxwell](https://github.com/zendesk/maxwell)).
We also have a small Sinatra application that acts as the search proxy.  It negotiates a secure connection with Elasticsearch, and converts our UI-generated query DSL 
into an Elasticsearch compatible query.

## How we reindex

Everything is going well and fine, until we decide we need to change the mapping, uh oh!  

Our KCL supports two indices: Current and Next. 
When our KCL application boots up, it creates/updates both Current and Next indices.  If the Next index is new, it will fire off a Reindex task that is performed asynchronously.
Additionally, the KCL application is capable of updating and removing records from both indices, simultaneously. 


The asyncronous reindex task can be monitored:

```
>> GET /_tasks
<<
{
                ...
                "5clUSFrfRKGbMEN8-5Octg:3166453": {
                    "node": "5clUSFrfRKGbMEN8-5Octg",
                    "id": 3166453,
                    "type": "transport",
                    "action": "indices:data/write/reindex",
                    "start_time_in_millis": 1599257753595,
                    "running_time_in_nanos": 544111332490,
                    "cancellable": true,
                    "headers": {}
                },
                ...
}
```

Monitor that lovely task for completion.  When it's done, we can head on over to our search application and point it at the new version!  

Here's a recap:
1.  In the KCL application github repository, create a new index (fancy-index-v2, since we are currently on fancy-index-v1)
2.  Update the KCL's "next index" variable from `fancy-index-v1` to `fancy-index-v2`.  Since they were previously the same, we weren't updating twice, but skipping the second update/delete
3.  Deploy!  The deployment will create a new index, start updating both indices, and kick off a reindex task.
4.  Monitor your elasticsearch instance for the completion of the reindex task
5.  Finish that cup of coffee, then you are free to deploy a new version of your search application to point at `fancy-index-v2`
6.  You can now rollout the UI of your new search functionality
7.  You can now update the KCL aplpication github repo, setting current and next index equal to `fancy-index-v2`
8.  You can now safely delete `fancy-index-v1`, assuming now other applications are consuming data from it.
