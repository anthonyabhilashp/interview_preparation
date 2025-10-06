Add your notes below:


Stateless 
“Each webhook service instance is stateless; scaling is managed by HPA in Kubernetes using metrics like request count or CPU utilization. Kafka offsets guarantee at-least-once delivery, so new instances can safely pick up from the queue.”


We use separate Kafka topics (or clusters) for ingestion and delivery to isolate concerns: ingestion topics are high-throughput, append-only, while delivery topics include retries and may have different retention policies.



Each Postgres instance has streaming replication for HA, and we can shard by tenant or endpoint ID to scale writes when we hit millions of records.

Verification (HMAC)

The sender (producer) computes HMAC(secret, body + timestamp).

Your system recomputes that using the stored secret and compares the signatures.

Include a timestamp tolerance (e.g., ±5 min) to prevent replay.