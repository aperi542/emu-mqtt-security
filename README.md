# MQTT Authentication and Access Control for Simulated EMUs

## 1. Project Overview

The goal of this project is to learn how IoT devices communicate using MQTT and how authentication and access control can help secure that communication.

For this project, I used the Mosquitto MQTT broker on my MacBook to simulate two Electric Measurement Units (EMU_01 and EMU_02). I tested how these devices send measurements to different topics and how a monitor can receive those measurements.

I first tested the system without authentication to see how an unauthorized client could send messages. Then, I added usernames, passwords, and access control rules to restrict which topics each device could access.

## 2. MQTT Basics

MQTT is a messaging protocol that allows IoT devices to send and receive information. It uses four main parts:

- **Broker:** Receives messages and sends them to the correct subscribers.
- **Publisher:** A device or client that sends messages.
- **Subscriber:** A device or client that receives messages.
- **Topic:** A channel that organizes where messages are sent.

In this project, EMU_01 and EMU_02 act as publishers, Mosquitto is the broker, and the monitor acts as a subscriber.

The two EMUs use these topics:

- `emu/EMU_01/features`
- `emu/EMU_02/features`

The monitor subscribes to `emu/+/features`, which allows it to receive measurements from both EMUs.

## 3. Local Test Environment

I used my MacBook to set up a local MQTT test environment. I installed Homebrew and used it to install the Mosquitto MQTT broker and clients.

To install Mosquitto, I used:

```bash
brew install mosquitto
```

I created a project folder called `emu-mqtt-security` to store the configuration files and test results.

```bash
mkdir -p ~/emu-mqtt-security
cd ~/emu-mqtt-security
mkdir -p config results
```

I used three main MQTT commands:

- `mosquitto` to run the broker.
- `mosquitto_pub` to publish simulated measurements.
- `mosquitto_sub` to subscribe to topics and receive measurements.

I simulated two devices, EMU_01 and EMU_02, using Terminal commands instead of physical EMUs.

All tests were performed locally using `127.0.0.1` (localhost) and port `18883`. This kept the experiment on my own computer without connecting to real EMUs or production servers.


## 4. Insecure Baseline

Before adding security controls, I tested how MQTT communication works without authentication.

I created an insecure Mosquitto configuration file (`config/insecure.conf`) with the following settings:

```conf
listener 18883 127.0.0.1
allow_anonymous true
log_type all
```

The `allow_anonymous true` setting allowed clients to connect without a username or password. The broker was restricted to localhost so the test stayed on my own computer.

I started the broker using:

```bash
mosquitto -c config/insecure.conf -v
```

Next, I opened another Terminal window and subscribed to EMU_01's topic:

```bash
mosquitto_sub -h 127.0.0.1 -p 18883 -t "emu/EMU_01/features" -v
```

I then simulated EMU_01 sending a measurement:

```bash
mosquitto_pub -h 127.0.0.1 -p 18883 -t "emu/EMU_01/features" -m "voltage=120"
```

The subscriber received:

```text
emu/EMU_01/features voltage=120
```

This showed that a client could publish a measurement without providing any credentials.

The insecure configuration allowed unauthorized clients to send messages, which could be a security risk in a real EMU monitoring system.

After recording the result, I stopped the insecure broker before adding authentication and access control.


## 5. Authentication and Access Control

After testing the insecure broker, I added authentication and access control to restrict unauthorized clients.

### Password Authentication

I created separate test accounts for EMU_01, EMU_02, and the monitor using Mosquitto's password utility.

I used these commands to create the password file:

```bash
mosquitto_passwd -c config/passwords EMU_01
mosquitto_passwd config/passwords EMU_02
mosquitto_passwd config/passwords monitor
```

Each account has its own password. The password file is kept private and is not uploaded to GitHub.

I then updated the broker configuration (`config/secure.conf`) to require authentication:

```conf
listener 18883 127.0.0.1
allow_anonymous false
password_file config/passwords
acl_file config/acl
log_type all
```

The `allow_anonymous false` setting prevents clients from connecting without valid credentials.

### Access Control Lists (ACLs)

I also created an ACL file to control which MQTT topics each account can access.

The ACL rules in `config/acl` are:

```conf
user EMU_01
topic readwrite emu/EMU_01/features

user EMU_02
topic readwrite emu/EMU_02/features

user monitor
topic read emu/+/features
```

These rules allow:

- EMU_01 to publish and subscribe to its own topic.
- EMU_02 to publish and subscribe to its own topic.
- The monitor to read measurements from both EMUs without publishing messages.

This prevents one EMU from accessing another EMU's measurements and restricts the monitor to read-only access.

### Starting the Secure Broker

After creating the password file and ACL rules, I started the secure broker using:

```bash
mosquitto -c config/secure.conf -v
```

I then repeated the MQTT tests to check whether authorized clients could communicate and unauthorized clients were blocked.


## 6. Security Tests and Results

After setting up authentication and access control, I ran five tests to check whether the MQTT broker allowed authorized clients and blocked unauthorized activity.

All tests were performed locally using simulated measurements.

### Test Results

| Test | Description | Expected | Observed |
|---|---|---|---|
| T1 | EMU_01 publishes to its own topic with valid credentials | Allowed | The monitor received `voltage=120`. |
| T2 | Client without credentials attempts to publish | Blocked | The broker refused the connection with a `not authorised` error. |
| T3 | EMU_01 attempts to publish to EMU_02's topic | Blocked | The broker logged `Denied PUBLISH`, and the monitor did not receive the unauthorized message. |
| T4 | EMU_01 attempts to subscribe to EMU_02's topic | Blocked / no data | EMU_01 did not receive EMU_02's measurement, while the authorized monitor received `voltage=230`. |
| T5 | Authorized monitor subscribes to both EMUs | Allowed (read-only) | The monitor received measurements from both EMUs. Its attempt to publish was denied. |

### Test Commands

The following commands were used to reproduce the five tests. Each command assumes that the secure broker is running on localhost, port `18883`.

Before running the tests, the appropriate test account passwords must be entered into the Terminal session:

```bash
read -s "EMU01_PASS?Enter EMU_01 password: "
read -s "EMU02_PASS?Enter EMU_02 password: "
read -s "MONITOR_PASS?Enter monitor password: "
```

Start an authorized monitor subscriber in a separate Terminal window:

```bash
mosquitto_sub -h 127.0.0.1 -p 18883 -u monitor -P "$MONITOR_PASS" -t "emu/+/features" -v
```

**T1 — Authorized publishing**

```bash
mosquitto_pub -h 127.0.0.1 -p 18883 -u EMU_01 -P "$EMU01_PASS" -t "emu/EMU_01/features" -m "voltage=120" -q 1 -d
```

**T2 — Publishing without credentials**

```bash
mosquitto_pub -h 127.0.0.1 -p 18883 -t "emu/EMU_01/features" -m "voltage=999" -q 1 -d
```

**T3 — Publishing to another EMU's topic**

```bash
mosquitto_pub -h 127.0.0.1 -p 18883 -u EMU_01 -P "$EMU01_PASS" -t "emu/EMU_02/features" -m "voltage=999" -q 1 -d
```

**T4 — Subscribing to another EMU's topic**

Run this command in a separate Terminal window:

```bash
mosquitto_sub -h 127.0.0.1 -p 18883 -u EMU_01 -P "$EMU01_PASS" -t "emu/EMU_02/#" -v -d -W 120
```

While that subscriber is running, publish a measurement using EMU_02:

```bash
mosquitto_pub -h 127.0.0.1 -p 18883 -u EMU_02 -P "$EMU02_PASS" -t "emu/EMU_02/features" -m "voltage=230" -q 1 -d
```

The authorized monitor should receive the message, but EMU_01 should not.

**T5 — Monitor read-only access**

Use the monitor subscriber shown above to receive messages from both EMUs. Publish an authorized message from each device using its own credentials.

Then attempt to publish using the monitor account:

```bash
mosquitto_pub -h 127.0.0.1 -p 18883 -u monitor -P "$MONITOR_PASS" -t "emu/EMU_01/features" -m "voltage=999" -q 1 -d
```

The broker should deny this publication because the monitor has read-only permission.

### Evidence

Screenshots of the Terminal output and broker logs are saved in the `results` folder.

- `T1-authorized-publish.png`
- `T2-unauthorized-publish.png`
- `T3-cross-topic-blocked.png`
- `T4-unauthorized-subscribe.png`
- `T5-monitor-read-write-test.png`

All five tests produced the expected results. Authentication prevented clients without valid credentials from connecting, and the ACL rules restricted access to the appropriate topics.
