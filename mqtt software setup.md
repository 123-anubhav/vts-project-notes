Ah! I see the issue. Your Docker Compose is trying to **mount a Mosquitto config file** from:

```
./mosquitto/config/mosquitto.conf
```

But this file **doesn’t exist on your host**, which is why Mosquitto fails to start properly. Also, the MQTT Web client (`mqttx-web`) won’t connect if the broker isn’t running.

Here’s a full working setup step by step:

---

### 1️⃣ Create the directory structure

From your project folder (where `docker-compose.yml` is):

```bash
mkdir -p mosquitto/config
```

---

### 2️⃣ Create a working `mosquitto.conf`

Create the file `mosquitto/config/mosquitto.conf`:

```bash
nano mosquitto/config/mosquitto.conf
```

Add the following content (simple working config with MQTT + WebSockets):

```conf
# Default listener for MQTT
listener 1883
protocol mqtt

# WebSocket listener
listener 9001
protocol websockets

# Allow anonymous connections (for testing only)
allow_anonymous true

# Persistence (optional)
persistence true
persistence_location /mosquitto/data/
log_dest stdout
```

---

### 3️⃣ Update Docker Compose

Your `docker-compose.yml` is mostly fine. Just **add volumes for data/logs** (optional, but recommended):

```yaml
version: '3'
services:
  mosquitto:
    image: eclipse-mosquitto:latest
    container_name: mqtt-broker
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config/mosquitto.conf:/mosquitto/config/mosquitto.conf
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log

  mqttx-web:
    image: emqx/mqttx-web:latest
    container_name: mqttx-web-client
    ports:
      - "8081:80"
    depends_on:
      - mosquitto
```

---

### 4️⃣ Start the stack

From the same folder as `docker-compose.yml`:

```bash
docker-compose up -d
```

Check logs:

```bash
docker logs -f mqtt-broker
```

You should see:

```
mosquitto version x.x starting
Opening ipv4 listen socket on port 1883.
Opening websockets listen socket on port 9001.
```

---

### 5️⃣ Test MQTT connection

* MQTT CLI / MQTT Explorer can connect to:

```
Host: localhost
Port: 1883
WebSocket: ws://localhost:9001
```

* MQTTX Web: Open `http://localhost:8081` and connect to your broker using:

```
Host: mqtt-broker (or localhost)
Port: 1883
```

---


---
when publish something then topic will display with message in mqtt explorer
in ubuntu cmd test
# Publish
mosquitto_pub -h localhost -t "mytopic" -m "test message"

# Subscribe
mosquitto_sub -h localhost -t "#"
---

Do you want me to do that?
