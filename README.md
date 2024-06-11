Creating a full system as you described involves various components, each with its specific setup and code. Below is a step-by-step guide and code snippets to build your system:

### 1. System Architecture Overview
- **Database**: MongoDB to store the status and results of Raspberry Pi.
- **Website**: Bootstrap for the frontend interface.
- **Backend**: A server running on a Debian VM to handle requests and compile code.
- **Raspberry Pi Cluster**: 10 Raspberry Pi 4 devices running Raspbian OS and K3S for container orchestration.
- **Router**: AX1800C for network connectivity.
- **Protocols**: MQTT for communication, HTTP for API requests.

### 2. Database Setup (MongoDB)
Install MongoDB on your VM or use a cloud-based MongoDB service.

```sh
sudo apt update
sudo apt install -y mongodb
sudo systemctl start mongodb
sudo systemctl enable mongodb
```

Create collections for storing Pi status and ping test results.

```js
use raspberry_cluster;
db.createCollection("pi_status");
db.createCollection("ping_results");
```

### 3. Backend Setup (Debian VM)
Install necessary packages and set up the backend server using Python Flask.

```sh
sudo apt update
sudo apt install -y python3 python3-pip build-essential mosquitto mosquitto-clients
pip3 install flask paho-mqtt pymongo
```

**Backend Server Code** (`server.py`):

```python
from flask import Flask, request, jsonify
import subprocess
import paho.mqtt.client as mqtt
import json
from pymongo import MongoClient
import os

app = Flask(__name__)
client = MongoClient('localhost', 27017)
db = client['raspberry_cluster']
pi_status_collection = db['pi_status']
ping_results_collection = db['ping_results']

MQTT_BROKER = 'your.mqtt.broker.address'
MQTT_PORT = 1883

def on_connect(client, userdata, flags, rc):
    print(f"Connected with result code {rc}")
    client.subscribe("pi/status")
    client.subscribe("pi/ping_result")

def on_message(client, userdata, msg):
    payload = json.loads(msg.payload)
    if msg.topic == "pi/status":
        pi_status_collection.update_one({"mac_address": payload['mac_address']},
                                        {"$set": payload}, upsert=True)
    elif msg.topic == "pi/ping_result":
        ping_results_collection.insert_one(payload)

mqtt_client = mqtt.Client()
mqtt_client.on_connect = on_connect
mqtt_client.on_message = on_message
mqtt_client.connect(MQTT_BROKER, MQTT_PORT, 60)
mqtt_client.loop_start()

@app.route('/ping', methods=['POST'])
def handle_ping_request():
    data = request.json
    c_code = f"""
    #include <stdio.h>
    #include <stdlib.h>
    int main() {{
        system("ping -c {data['num_packets']} {data['destination']}");
        return 0;
    }}
    """
    with open("ping_test.c", "w") as file:
        file.write(c_code)
    subprocess.run(["gcc", "ping_test.c", "-o", "ping_test"])
    with open("ping_test", "rb") as binary_file:
        binary_data = binary_file.read()
    mqtt_client.publish(f"pi/{data['raspberry_mac']}/ping_test", binary_data)
    return jsonify({"status": "Ping test sent"}), 200

@app.route('/pi_status', methods=['GET'])
def get_pi_status():
    status_list = list(pi_status_collection.find({}, {"_id": 0}))
    return jsonify(status_list)

@app.route('/ping_results', methods=['GET'])
def get_ping_results():
    results = list(ping_results_collection.find({}, {"_id": 0}))
    return jsonify(results)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### 4. Raspberry Pi Setup
Install necessary packages and set up MQTT client on each Raspberry Pi.

```sh
sudo apt update
sudo apt install -y mosquitto-clients gcc
pip3 install paho-mqtt
```

**MQTT Client on Raspberry Pi** (`pi_mqtt_client.py`):

```python
import paho.mqtt.client as mqtt
import os
import json
import time

MQTT_BROKER = 'your.mqtt.broker.address'
MQTT_PORT = 1883
MAC_ADDRESS = ':'.join(['{:02x}'.format((os.popen('cat /sys/class/net/wlan0/address').read().strip().split(":")[i])) for i in range(6)])

def on_connect(client, userdata, flags, rc):
    print(f"Connected with result code {rc}")
    client.subscribe(f"pi/{MAC_ADDRESS}/ping_test")

def on_message(client, userdata, msg):
    with open("ping_test", "wb") as binary_file:
        binary_file.write(msg.payload)
    os.system("chmod +x ping_test")
    result = os.popen("./ping_test").read()
    result_json = json.dumps({"mac_address": MAC_ADDRESS, "result": result})
    client.publish("pi/ping_result", result_json)

def send_status():
    status = {"mac_address": MAC_ADDRESS, "status": "online", "timestamp": int(time.time())}
    client.publish("pi/status", json.dumps(status))

client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message
client.connect(MQTT_BROKER, MQTT_PORT, 60)
client.loop_start()

while True:
    send_status()
    time.sleep(60)  # Send status every minute
```

### 5. Frontend Setup
Use Bootstrap for the web interface.

**Index Page (`index.html`)**:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Raspberry Pi Cluster Dashboard</title>
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.3.1/css/bootstrap.min.css">
</head>
<body>
    <div class="container">
        <h1 class="mt-5">Raspberry Pi Cluster Dashboard</h1>
        <div id="pi-status" class="mt-4"></div>
        <h2 class="mt-5">Ping Test</h2>
        <form id="ping-form" class="mt-3">
            <div class="form-group">
                <label for="raspberry_mac">Raspberry Pi MAC Address</label>
                <input type="text" class="form-control" id="raspberry_mac" required>
            </div>
            <div class="form-group">
                <label for="num_packets">Number of Packets</label>
                <input type="number" class="form-control" id="num_packets" required>
            </div>
            <div class="form-group">
                <label for="destination">Destination Domain</label>
                <input type="text" class="form-control" id="destination" required>
            </div>
            <button type="submit" class="btn btn-primary">Ping</button>
        </form>
        <div id="ping-results" class="mt-4"></div>
    </div>

    <script src="https://code.jquery.com/jquery-3.3.1.min.js"></script>
    <script>
        function updatePiStatus() {
            $.get('/pi_status', function(data) {
                let html = '<table class="table"><thead><tr><th>MAC Address</th><th>Status</th><th>Timestamp</th></tr></thead><tbody>';
                data.forEach(function(pi) {
                    html += `<tr><td>${pi.mac_address}</td><td>${pi.status}</td><td>${new Date(pi.timestamp * 1000).toLocaleString()}</td></tr>`;
                });
                html += '</tbody></table>';
                $('#pi-status').html(html);
            });
        }

        function updatePingResults() {
            $.get('/ping_results', function(data) {
                let html = '<table class="table"><thead><tr><th>MAC Address</th><th>Result</th></tr></thead><tbody>';
                data.forEach(function(result) {
                    html += `<tr><td>${result.mac_address}</td><td><pre>${result.result}</pre></td></tr>`;
                });
                html += '</tbody></table>';
                $('#ping-results').html(html);
            });
        }

        $('#ping-form').submit(function(e) {
            e.preventDefault();
            const data = {
                raspberry_mac: $('#raspberry_mac').val(),
                num_packets: $('#num_packets').val(),
                destination: $('#destination').val()
            };
            $.post('/ping', JSON.stringify(data), function(response) {
                alert(response.status);
            }, 'json');
        });

        setInterval(updatePiStatus, 5000);  // Update Pi status every 5 seconds
        setInterval(updatePingResults, 5000);  // Update ping results every 5 seconds
        updatePiStatus();
        updatePingResults();
    </script>
</body>
</html>
```

### 6. MQTT Broker Setup
Ensure the MQTT broker is running and accessible by both the backend server and Raspberry Pis.

```sh
sudo apt update
sudo apt install -y mosquitto
sudo systemctl start mosquitto
sudo systemctl enable mosquitto
```

### Final Steps
1. **Run the backend server**: `python3 server.py`
2

. **Deploy the frontend**: Serve `index.html` using a web server like Nginx or Apache.
3. **Deploy the MQTT client on each Raspberry Pi**: `python3 pi_mqtt_client.py`

This setup provides a basic yet comprehensive implementation of the system you described. Each component (web, backend, database, Raspberry Pi) is interconnected using MQTT and HTTP protocols, ensuring smooth communication and data handling.