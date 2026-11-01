# MQTT Security Test Results

All five tests were performed locally using the Mosquitto MQTT broker and simulated EMU measurements.

| Test | Expected Behavior | Observed Behavior | Result |
|---|---|---|---|
| T1 | EMU_01 can publish to its own topic | The monitor received `voltage=120` from EMU_01. | Passed |
| T2 | Client without credentials is blocked | The broker returned `Connection Refused: not authorised`. The unauthorized message was not received. | Passed |
| T3 | EMU_01 cannot publish to EMU_02's topic | The broker logged `Denied PUBLISH`. The monitor did not receive the unauthorized message. | Passed |
| T4 | EMU_01 cannot read EMU_02's topic | EMU_01 received no data, while the authorized monitor received `voltage=230`. | Passed |
| T5 | Monitor can read both topics but cannot publish | The monitor received measurements from both EMUs. Its attempt to publish was denied. | Passed |

## Supporting Evidence

- T1: `T1-authorized-publish.png`
- T2: `T2-unauthorized-publish.png`
- T3: `T3-cross-topic-blocked.png`
- T4: `T4-unauthorized-subscribe.png`
- T5: `T5-monitor-read-write-test.png`

## Overall Result

All five tests produced the expected results. Authentication prevented clients without valid credentials from connecting, and ACL rules restricted which topics each client could publish or subscribe to.
