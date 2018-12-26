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
- The data is recorded with the following data:
-- The source IP of the event.
-- The date of the event.
-- The IP of the load balancer that received the event.
-- The destination `Host:` header of the request.
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
- The event is filtered for SSH auth failure (Initially only password, later on add pre-auth and other block types).
- The event is transformed to a versioned JSON type
- The JSON event is sent to a `sshd_json` NATS queue
Q: How do we handle different versions of SSH that change logs over time?


... TBC ...
Examples
