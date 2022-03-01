# Stream Data Generator

Built on top of the Data Generator, for scheduling continuous data streams from different schemas and sending it to different destinations such as message channels (Kafka, RabbitMQ), REST endpoints or JDBC databases. 

The stream-data-generator  is still WIP, though being used to generate the test data for all streaming-runtime sample use cases.

Here is sample configuration to configure two parallel treads to generate and send data to two Kafka topics: 

## Quick Start

```yaml title="user.yaml"
stream:
 data:
   generator:
     terminateAfter: 300s

     destinations:
       - name: kafka        
         bootstrap: 'localhost:9092'
         schemaRegistryServer: 'http://localhost:8081'

     streams:
       - streamName: kafka-stream-songs
         destination: kafka
         valueFormat: AVRO_SCHEMA_REGISTRY
         avroSchema: |-
           {
            "namespace": "com.tanzu.streaming.runtime.playsongs.avro",
            "type": "record",
            "name": "Song",
            "doc": "unique_on=song_id;to_share=song_id",
            "fields": [
                {"name": "song_id", "type": "long",   
                 "doc" : "#{number.number_between '1','1000'}"},
                {"name": "album",   "type": "string", 
                 "doc" : "#{ancient.hero} #{ancient.god}"},
                {"name": "artist",  "type": "string", 
                 "doc" : "#{artist.names}"},
                {"name": "name",    "type": "string", 
                 "doc" : "#{rock_band.name}"},
                {"name": "genre",   "type": "string", 
                 "doc" : "#{music.genres}"}
            ]
           }
         batch:
           size: 100
           initialDelay: 1ms
           messageDelay: 10ms


       - streamName: kafka-stream-playevents
         destination: kafka
         avroSchema: |-
           {
            "namespace": "com.tanzu.streaming.runtime.playsongs.avro",
            "type": "record",
            "name": "PlayEvent",
            "fields": [
              {"name": "song_id",  "type": "long", 
               "doc":"[[#shared.field('song.song_id')]]" },
              {"name": "duration", "type": "long", 
               "doc":"#{number.number_between '30000','1000000'}" }
            ]
           }
         valueFormat: JSON
         batch:
           size: 1
           initialDelay: 10ms
           delay: 100ms
           messageDelay: 100ms
```
