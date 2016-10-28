# Stream position checkpoint

The library provides a mechanism to restart the stream from a recent *checkpoint*, to be resilient
to restarts and crashes.

*Checkpoints* are saved automatically, with a configured frequency, on a storage provided.

For instance, the stream position can be saved every 15 seconds in Azure blobs, or 
(soon) a custom backend.

To store checkpoints in Azure blobs the configuration looks like this:

```
iothub-checkpointing {
  enabled = true
  frequency = 15s
  countThreshold = 1000
  timeThreshold = 30s
  storage {
    rwTimeout = 5s
    backendType = "AzureBlob"
    namespace = "iothub-react-checkpoints"
    azureblob {
      lease = 15s
      useEmulator = false
      protocol = "https"
      account = "..."
      key = "..."
    }
  }
}
```

Soon it will be possible to plug in custom storage backends implementing a simple 
[interface](src/main/scala/com/microsoft/azure/iot/iothubreact/checkpointing/Backends/CheckpointBackend.scala)
to read and write the stream position.

There is also one API parameter to enabled/disable the mechanism, for example:

```scala
val start = java.time.Instant.now()
val withCheckpoints = false

IoTHub.source(start, withCheckpoints)
    .map(m => jsonParser.readValue(m.contentAsString, classOf[Temperature]))
    .filter(_.value > 100)
    .to(console)
    .run()
```

# Checkpointing behavior

### Configuration

The following table describes the impact of the settings within the `iothub-checkpointing` 
configuration block. For further information, you can also check the 
[reference.conf](src/main/resources/reference.conf) file.

| Setting | Type | Example | Description |
|---------|------|---------|-------------|
| **enabled**             | bool                 | true        | Global switch to enable/disable the checkpointing feature. This value overrides the API parameter "withCheckpoints".            |
| **frequency**           | duration             | 15s         | How often to check if the offset in memory should be saved to storage. The check is scheduled for each partition individually.  |
| **countThreshold**      | int                  | 1000        | How many messages to stream before saving the position. The setting is applied to each partition individually. The value should be big enough to take into account buffering and batching. |
| **timeThreshold**       | duration             | 60s         | In case of low traffic (i.e. when not reaching countThreshold), save the stream position that is older than this value.|
| storage.**rwTimeout**   | duration             | 5000ms      | How long to wait, when writing to the storage, before triggering a storage timeout exception.                                         |
| storage.**backendType** | string or class name | "AzureBlob" | Currently only "AzureBlob" is supported. The name of the backend, or the class FQDN, to use to write to the storage. This provides a way to inject custom storage logic. |
| storage.**namespace**   | string               | "mycptable" | The table/container which will contain the checkpoints data. When streaming data from multiple IoT hubs, you can use this setting, to store each stream position to a common storage, but in separate tables/containers. | 

### Runtime

The following table describes the system behavior, based on **API parameters** and stored **state**.

| Checkpointing | Start point | Saved position | Behavior |
|:---:|:---:|:-------:|---|
| No  | No  | No      | The stream starts from the beginning
| No  | No  | **Yes** | The stream starts from the beginning (**the saved position is ignored**)
| No  | Yes | No      | The stream starts from the 'start point' provided
| No  | Yes | **Yes** | The stream starts from the 'start point' provided (**the saved position is ignored**)
| Yes | No  | No      | The stream starts from the beginning
| Yes | No  | **Yes** | **The stream starts from the saved position**
| Yes | Yes | No      | The stream starts from the 'start point' provided
| Yes | Yes | **Yes** | **The stream starts from the saved position**

Legend:
* **Checkpointing**: whether checkpointing (saving the stream position) is enabled or disabled
* **Start point**: whether the client provides a starting position (date or offset) or ask for all 
the events from the beginning
* **Saved position**: whether there is a position saved in the storage 