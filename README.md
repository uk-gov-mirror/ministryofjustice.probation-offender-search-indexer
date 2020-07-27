# probation-offender-search-indexer

The purpose of this service is two fold:
* Keep the Elastic Search  (ES) probation index up to date with changes from Delius
* Rebuild the index when required without an outage

## Architecture and design

### Offender updates

This service subscribes to the probation offender events, specifically the `OFFENDER_CHANGED` event

When this event is received the latest offender record is retrieved via the `community-api` and upserted into the offender index.

### Index rebuilds

This service maintains two indexes `probation-search-green` and `probation-search-blue` also know in the code as `BLUE` and `GREEN`.

In normal running one of these indexes will be "active" while the other is dormant and not in use. 

When we are ready to rebuild the index the "other" non active index is transitioned into a `BUILDING` state.

```
    PUT /probation-index/build-index
```
 
The entire Delius offender base is retrieved and over several hours the other index is fully populated. Once the operator realises the other index is fully rebuilt the active index is switched with 
```
    PUT /probation-index/mark-complete
```

#### Index switch

Given the state of the each index is itself held in ES under the `offender-index-status` index with a single "document" when the BLUE/GREEN indexes switch there are actually two changes:
* The document in `offender-index-status` to indicate which index is currently active
* The ES alias `offender` is switched to point at the the active index. This means external clients can safely use the `offender` index/alias without any knowledge of the the BLUE/GREEN indexes. 

### Running

`localstack` is used to emulate the AWS SQS and Elastic Search service. When running the integration test this will be started automatically. If you want the tests to use an already running version of `localstack` run the tests with the environment `AWS_PROVIDER=localstack`. This has the benefit of running the test quicker without the overhead of starting the `localstack` container.

Any commands in `localstack/setup-sns.sh` and `localstack/setup-es.sh` will be run when `localstack` starts, so this should contain commands to create the appropriate queues.

Running all services locally:
```bash
docker-compose up 
```
Queues and topics and an ES instance will automatically be created when the `localstack` container starts.

Running all services except this application (hence allowing you to run this in the IDE)

```bash
docker-compose up --scale probation-offender-search-indexer=0 
```

## Support

### Raw Elastic Search access

Access to the raw Elastic Search indexes is only possible from the Cloud Platform `probation-offender-search` family of namespaces. 

For instance 

```
curl http://aws-es-proxy-service:9200/_cat/indices
```
in any environment would return a list all indexes e.g.
```
green open probation-search-green l4Pf4mFcQ2eVhnqhNnCVGg 5 1 683351 366 311.1mb   155mb
green open probation-search-blue  hsuf4mFcQ2eVhnqhNnKjsw 5 1 683351 366 311.1mb   155mb
green open offender-index-status  yEhaJorgRemmWMONJjfHoQ 1 1      1   0  11.4kb   5.7kb
green open .kibana_1              jddPd_-XRBiqxOXGudoj7Q 1 1      0   0    566b    283b
```

### Rebuilding an index

To rebuild an index the credentials used must have the ROLE `PROBATION_INDEX` therefore it is recommend to use client credentials with the `ROLE_PROBATION_INDEX` added and pass in your username when getting a token.

The rebuilding of the index can be sped up by increasing the number of pods handling the reindex e.g.

```
kubectl -n probation-offender-search-dev scale --replicas=8 deployment/probation-offender-search-indexer
``` 
After obtaining a token for the environment invoke the reindex with a cUrl command or Postman e.g.

```
curl --location --request PUT 'https://probation-search-indexer-dev.hmpps.service.justice.gov.uk/probation-index/build-index' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <some token>>'
``` 

For production environments where access is blocked by inclusion lists this will need to be done from withing a Cloud Platform pod

Next monitor the progress of the rebuilding via the info endpoint e.g. https://probation-search-indexer-dev.hmpps.service.justice.gov.uk/info
This will return details like the following:

```
   "index-status": {
     "currentIndex": "NONE",
     "currentIndexStartBuildTime": null,
     "currentIndexEndBuildTime": null,
     "currentIndexState": "ABSENT",
     "otherIndexStartBuildTime": "2020-07-27T14:53:35",
     "otherIndexEndBuildTime": null,
     "otherIndexState": "BUILDING",
     "otherIndex": "GREEN"
   },
   "index-size": {
     "GREEN": 8,
     "BLUE": -1
   },
   "offender-alias": "",
   "index-queue-backlog": "260133"
```
 
 when `"index-queue-backlog": "0"` has reached zero then all indexing messages have been processed. Check that the dead letter queue is empty via the health check e.g https://probation-search-indexer-dev.hmpps.service.justice.gov.uk/health
 This should show the queues DLQ count at zero, e.g.
 ``` 
    "indexQueueHealth": {
      "status": "UP",
      "details": {
        "MessagesOnQueue": 0,
        "MessagesInFlight": 0,
        "dlqStatus": "UP",
        "MessagesOnDLQ": 0
      }
    },
 ```
  
 The indexing is ready to marked as complete using another call to the service e.g
 
 ``` 
curl --location --request PUT 'https://probation-search-indexer-dev.hmpps.service.justice.gov.uk/probation-index/mark-complete' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <some token>>'

 ```  

One last check of the info endpoint should confirm the new state, e.g.

```
   "index-status": {
     "currentIndex": "GREEN",
     "currentIndexStartBuildTime": "2020-07-27T14:53:35",
     "currentIndexEndBuildTime": "2020-07-27T16:53:35",
     "currentIndexState": "COMPLETE",
     "otherIndexStartBuildTime": null,
     "otherIndexEndBuildTime": null,
     "otherIndexState": "ABSENT",
     "otherIndex": "BLUE"
   },
   "index-size": {
     "GREEN": 2010111,
     "BLUE": -1
   },
   "offender-alias": "probation-search-green",
   "index-queue-backlog": "0"
```

Pay careful attention to `"offender-alias": "probation-search-green"` - this shows the actual index being used by clients.

### Deleting indexes in test

In rare circumstances where you need to test rebuilding indexes from scratch with an outage it is possible to delete the indexes as follows:

```
curl -X DELETE aws-es-proxy-service:9200/probation-search-green/_alias/offender?pretty
curl -X DELETE aws-es-proxy-service:9200/probation-search-green?pretty
curl -X DELETE aws-es-proxy-service:9200/probation-search-blue?pretty
curl -X DELETE aws-es-proxy-service:9200/offender-index-status?pretty
```


### Alias switch failure

It is possible for the green/blue index switching to become out of sync with the index the `offender` alias is pointing at. 
That could happen if the ES operation to update the `offender-index-status` succeeds but the ES operation to update the alias fails.

In that instance the alias must be switched manually. 

The current state of the alias can be seen from 
```
curl http://aws-es-proxy-service:9200/_cat/aliases
```
e.g.
```
.kibana  .kibana_1              - - - -
offender probation-search-green - - - -
```
shows that the current index is `probation-search-green`

To update the alias:
```
curl -X POST "aws-es-proxy-service:9200/_aliases?pretty" -H 'Content-Type: application/json' -d'
{
  "actions" : [
    { "add" : { "index" : "probation-search-blue", "alias" : "offender" } }
  ]
}
'
```
And then delete old alias
```
curl -X DELETE "aws-es-proxy-service:9200/probation-search-green/_alias/offender?pretty"

```
