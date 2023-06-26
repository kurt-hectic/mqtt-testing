# WIS2 MQTT QoS testing
When subscribing to a topic, a client may have the requirement to not loose messages which were published during a possible downtime of the client, such as reboot, crash or software update.
MQTT QoS flags are intended to make sure notifications are delivered according to intended quality requirements.
How do they work in practice?

# summary
For a lossless subscription the following parameters are required (mosquitto and mosquitto_sub/pub parameters in brackets)

 * subscriber to supply client_id (--id myclientid)
 * subscriber to disable clean session (--disable-clean-session)
 * subscriber to enable QoS >= 1 (--qos 1)
 * publisher to publish with QoS >=1 (--qos 1)
 * broker to enable QoS (max_qos 1)
 * broker to queue sufficient messages (max_queued_messages 10000, max_queued_bytes 0)

# setup 
testing mosquitto broker and testing container with mosquitto_sub/pub started using ```docker-compose up```.
Two bash sessions opened in testing container ```docker exec -it CONTAINERID bash ```

For each test: 10 messages published in first session ```for x in {1..10}; do sleep 1; mosquitto_pub -h mosquitto --qos 1 -t test -m "message ${x}" ; done```

To test subscription, two mosquitto_sub invocations are used, separated by a pause. The parameter -C 3 is used to terminate the first invocation after 3 received messages. Since the pause between the invocations is 10 seconds, the publisher has stopped publishing by the time the second mosquitto_sub is invoked.

## test 1
Client does not supply (--disable-clean-session)
```bash
root@ff5c637814b4:/# mosquitto_sub -h mosquitto -t test --qos 1  -C 3 --id testclient  && echo "disconnected and sleeping"  ; sleep 10 ; echo "reconnecting" && mosquitto_sub -h mosquitto -t test --qos 1 --id tes
tclient
message 1
message 2
message 3
disconnected and sleeping
reconnecting
```
The second subscription does not receive messages published during the subscription pause.

## test 2
Client supplies (--disable-clean-session)
```bash
root@ff5c637814b4:/# mosquitto_sub -h mosquitto -t test --qos 1  --id testclient --disable-clean-session   -C 3 && echo
"disconnected and sleeping"  ; sleep 10 ; echo "reconnecting" && mosquitto_sub -h mosquitto -t test --qos 1 --id testcli
ent --disable-clean-session
message 1
message 2
message 3
disconnected and sleeping
reconnecting
message 4
message 5
message 6
message 7
message 8
message 9
message 10
```

The second subscription receives messages published during the subscription pause.

## test 3
As in test 2, but publisher uses (--qos 0).
```bash
root@ff5c637814b4:/# mosquitto_sub -h mosquitto -t test --qos 1  --id testclient --disable-clean-session   -C 3 && echo "disconnected and sleeping"  ; sleep 10 ; echo "reconnecting" && mosquitto_sub -h mosquitto -t test --qos 1 --id testclient --disable-clean-session
message 1
message 2
message 3
disconnected and sleeping
reconnecting
```

The second subscription does not receive messages published during the subscription pause.

## test 4
As in test 2, but mosquitto broker uses config options (max_queued_messages 3)
```bash
root@ff5c637814b4:/# mosquitto_sub -h mosquitto -t test --qos 1  --id testclient --disable-clean-session   -C 3 && echo "disconnected and sleeping"  ; sleep 10 ; echo "reconnecting" && mosquitto_sub -h mosquitto -t test --qos 1 --id testclient --disable-clean-session
message 1
message 2
message 3
disconnected and sleeping
reconnecting
message 4
message 5
message 6
```

The second subscription receives only 3 messages published during the subscription pause.

# discussion
Given that Global Brokers publish messages with at least (--qos 1), a client intending to recover messages published during its downtime, can use (--qos 1 --disable-clean-session --id myclientid) to tell the Global Broker to queue messages. The length of time a subscriber can not be connected is determined by the number and size of messages queued by the broker for each session (max_queued_messages and max_queued_bytes in mosquitto).

A large number of messages will require enough memory resources from the broker. A broker may grant a large queue to privileged subscribers, such as Global Caches, provided it can be configured to only provide this to select subscribers, or otherwise expose itself of a queue exhaustion attack or DoS, whereby numerous clients (everyone/everyone) connect and disconnect with (--disable-clean-session).
