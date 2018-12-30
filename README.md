# AoG

AoG provides a real-time and historic dashboard of attacks of different sources.
Machine Learning is applied to the incoming events to analyze trends/groups.
This is an event based lambda architecture where raw events are stored permanently for self-healing purposes.

## Background
Events arrive raw, they are enriched with the source of the data, the time of receipt.
Storing the raw events allows us to make drastic changes to the architecture and heal the gaps in the data.
This means the microservices are state machines meant to have volatile storage and recalculate stats.

### Public layer
This is the input of data to our platform. An outter layer provides:
- Consistent trusted IPs
- Encrypted endpoints
- DNS endpoints with GeoLocation aware services such as Route53.
- Different DNS endpoints will be used to understand the data type, for example:
-- `http.input.domain.com`: A generic well known HTTP log receiver.
-- `ssh.input.domain.com`: A generic ssh input receiver.
-- `x-type.input.domain.com`: A custom event type.
-- input.domain.com: A general catch-all receiver.

#### Flow

- An event arrives as the body of an HTTP request.
- The event is recorded with the following data:

  - The source IP of the event.
  - The date of the event.
  - The IP of the load balancer that received the event.
  - The destination `Host:` header of the request.
- The API keys are validated.
- Throttling is enforced.
- The enriched event is sent to S3 or an object storage.
- The enriched event is also sent to a NATS queue depending on the destination `Host:` header of the request.

### Event categorizer

For the catch-all input ingress, the event is analyzed and matched against known event types (sshd, http).
When the event is recognized as a specific type it is sent to that queue's specific type.
Q: If the event data matches several possible protocol types, Should we match once or match several times?

### SSHD microservice 
Listens to the `sshd_raw` input topic.
- The event is filtered for SSH auth failure (Initially only password, later on add pre-auth and other attempt types).
- The event is transformed to a versioned JSON type
- The JSON event is sent to a `sshd_json` NATS queue
Q: How do we handle different versions of SSH that change logs over time?


Example:
Raw log from sshd:
`# systemctl status sshd
#[...]
Dec 29 17:21:44 archlinux sshd[29778]: Failed password for invalid user postgres from WW.XX.YY.ZZ port 33312 ssh2`

- The event is sent to the ssh endpoint. It is received as the body of the HTTP request:
`Dec 29 17:21:44 archlinux sshd[29778]: Failed password for invalid user postgres from WW.XX.YY.ZZ port 33312 ssh2`

- The ssh endpoint checks the data and sends it to the `sshd_raw` topic:
```json
{
  "receive_time": "<some time>",
  "source_ip": "<some ip>",
  "destination_host": "sshd.<some-domain>",
  "record_id": "XXX",
  "raw_line": "Dec 29 17:21:44 archlinux sshd[29778]: Failed password for invalid user postgres from WW.XX.YY.ZZ port 33312 ssh2`
",
  "event_type": "ssh_attack",
  "other_metadata":"...",
  "process_trail":[
     {
       "id": "sshd_raw_receiver-<kubectl replica id>",
       "processing_time": "<some time>",
       "version": "X.Y.Z"
     }
  ],
}
```

- The `sshd_raw` topic is read by the sshd microservice, it is enriched and sent to `sshd_json` topic as:
  Besides transforming to JSON, it also sends a request to the `geoip_lookup` microservice that returns lat/lon/country code/etc.
  Or should we put the GeoIP lookup after the `sshd_json` ?
```json
{
  "receive_time": "<some time>",
  "source_ip": "<some ip>",
  "destination_host": "sshd.<some-domain>",
  "record_id": "XXX",
  "raw_line": "Dec 29 17:21:44 archlinux sshd[29778]: Failed password for invalid user postgres from WW.XX.YY.ZZ port 33312 ssh2`
",
  "event_type": "ssh_attack",
  "other_metadata":"...",
  "event_data": {
    "sshd_auth_type": "password",
    "target_user": "postgres",
    "sshd_auth_failure_type": "password",
    "event_time": "2018-12-29 17:21:44",
    "source_ip": "WW.XX.YY.ZZ",
  },
  "is_valid": true,
  "process_trail":[
     {
       "id": "sshd_raw_receiver-<kubectl replica id>",
       "processing_time": "<some time>",
       "version": "X.Y.Z"
     },
     {
       "id": "sshd_raw_processor-<kubectl replica id>",
       "processing_time": "<some time>",
       "version": "X.Y.Z"
     }
  ],
}
```

- The `sshd_json` topic is read by various consumers:

  - The `s3_backup` consumer that sends the topic data to S3 (could be like fluentd sending every topic to S3), this helps for ML/Statistics.

    - The S3 partition uses /year=%Y/month=%m/day=%d/type=`event_type`/`more layers maybe IP grouping or geoIP grouping`/timestamp.json
    - The event records are stored after a while, gzipped.
  - The `ip_blacklister` microservice: 

    - keeps track of the severity/danger per IP and generate security group rules
    - keeps track of the last attack of an IP and drops the whitelist based on the severity
    - may be used to identify network ranges ?
    - when booting it needs a way to get a concise cache of attacks
    - groups events into specific security groups based on the `event_type` and XXX
```json
```
... TBC ...
