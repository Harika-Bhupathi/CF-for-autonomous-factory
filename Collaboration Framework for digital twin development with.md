

In this  project, we will use MQTT for message queuing and brokering, facilitating communication between different components. Here’s the full implementation, including the use of PostgreSQL, Flask, Unity, WebGL, XAMPP, and MQTT.

### 1. Database Setup (PostgreSQL & XAMPP)
First, ensure you have PostgreSQL and XAMPP installed and running. Use XAMPP to manage the database server:

- **Install PostgreSQL**: [Download PostgreSQL](https://www.postgresql.org/download/)
- **Install XAMPP**: [Download XAMPP](https://www.apachefriends.org/index.html)

Once installed, create a database and the necessary tables:
#### SQL Script:
```sql
-- SQL script to create database and table
CREATE DATABASE industrial_automation;

\c industrial_automation

CREATE TABLE operations (
    id SERIAL PRIMARY KEY,
    description TEXT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2. Backend Code (Python with Flask and MQTT)

#### Required Libraries:
```bash
pip install psycopg2-binary flask flask-cors paho-mqtt
```

#### Python Script (`server.py`):

```python
import psycopg2
from flask import Flask, jsonify, request
from flask_cors import CORS
from threading import Thread
from paho.mqtt.client import Client
from time import sleep

app = Flask(__name__)
CORS(app)

MQTT_BROKER = 'broker.hivemq.com'  # Public MQTT Broker
MQTT_TOPIC = 'industrial/automation'

# Connect to PostgreSQL
def connect_to_db():
    return psycopg2.connect(
        dbname='industrial_automation', 
        user='admin', 
        host='localhost', 
        password='password'
    )

# Log operation to database
def log_operation(description):
    conn = connect_to_db()
    cursor = conn.cursor()
    query = "INSERT INTO operations (description) VALUES (%s)"
    cursor.execute(query, (description,))
    conn.commit()
    cursor.close()
    conn.close()

# MQTT Client Setup
mqtt_client = Client()

# MQTT Callback on Message
def on_message(client, userdata, message):
    payload = message.payload.decode()
    log_operation(payload)

mqtt_client.on_message = on_message
mqtt_client.connect(MQTT_BROKER)
mqtt_client.subscribe(MQTT_TOPIC)
mqtt_client.loop_start()

# Flask endpoint to get status
@app.route('/status', methods=['GET'])
def get_status():
    conn = connect_to_db()
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM operations ORDER BY timestamp DESC LIMIT 1")
    result = cursor.fetchone()
    cursor.close()
    conn.close()
    return jsonify({"last_operation": result})

# Flask endpoint to start process
@app.route('/start_process', methods=['POST'])
def start_process():
    process_thread = Thread(target=process_sequence)
    process_thread.start()
    return jsonify({"status": "Process started"})

# Simulating the manufacturing process
def process_sequence():
    simulate_amr_transport()
    amr_position_workpiece()
    start_3d_printing()
    move_to_cnc()
    start_cnc_machining()

# Simulation functions
def simulate_amr_transport():
    sleep(2)  # Simulate transport time
    log_operation("AMR transported to workstation")
    mqtt_client.publish(MQTT_TOPIC, "AMR transported to workstation")

def amr_position_workpiece():
    sleep(1)
    log_operation("Workpiece positioned in 3D printer")
    mqtt_client.publish(MQTT_TOPIC, "Workpiece positioned in 3D printer")

def start_3d_printing():
    sleep(5)  # Simulate printing time
    log_operation("3D printing completed")
    mqtt_client.publish(MQTT_TOPIC, "3D printing completed")

def move_to_cnc():
    sleep(2)
    log_operation("Workpiece moved to CNC machine")
    mqtt_client.publish(MQTT_TOPIC, "Workpiece moved to CNC machine")

def start_cnc_machining():
    sleep(4)  # Simulate machining time
    log_operation("CNC machining completed")
    mqtt_client.publish(MQTT_TOPIC, "CNC machining completed")

if __name__ == "__main__":
    app.run(port=5000)
```

In this script:

- The `Flask` application serves as an API endpoint for controlling and monitoring the process.
- Different endpoint `/start_process` kicks off the entire sequence in a separate thread.
### 3. Unity Integration

Unity will fetch the data from the backend Flask server and visualize it.

#### Unity Setup

- Install Unity Hub and create a new project.
- Configure the project to include WebGL.

#### Unity C# Script

Create a new C# script (e.g., `DataFetcher.cs`) and attach it to a game object.


```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.Networking;
using Newtonsoft.Json.Linq;
using uPLibrary.Networking.M2Mqtt;
using uPLibrary.Networking.M2Mqtt.Messages;
using System.Text;

public class DataFetcher : MonoBehaviour
{
    private string apiUrl = "http://localhost:5000/status";
    private MqttClient mqttClient;
    private string mqttBroker = "broker.hivemq.com";
    private string mqttTopic = "industrial/automation";

    void Start()
    {
        mqttClient = new MqttClient(mqttBroker);
        mqttClient.MqttMsgPublishReceived += OnMqttMessageReceived;
        mqttClient.Connect(System.Guid.NewGuid().ToString());
        mqttClient.Subscribe(new string[] { mqttTopic }, new byte[] { MqttMsgBase.QOS_LEVEL_AT_LEAST_ONCE });
        StartCoroutine(GetStatus());
    }

    private void OnMqttMessageReceived(object sender, MqttMsgPublishEventArgs e)
    {
        string message = Encoding.UTF8.GetString(e.Message);
        Debug.Log("MQTT Message Received: " + message);
        // Update the Unity scene based on the MQTT message
        UpdateStatus(message);
    }

    IEnumerator GetStatus()
    {
        while (true)
        {
            using (UnityWebRequest www = UnityWebRequest.Get(apiUrl))
            {
                yield return www.SendWebRequest();

                if (www.isNetworkError || www.isHttpError)
                {
                    Debug.Log(www.error);
                }
                else
                {
                    JObject json = JObject.Parse(www.downloadHandler.text);
                    string lastOperation = json["last_operation"].ToString();
                    Debug.Log("Last Operation: " + lastOperation);
                    
                    // Update the Unity scene based on the fetched data
                    UpdateStatus(lastOperation);
                }
            }
            yield return new WaitForSeconds(10);  // Fetch data every 10 seconds
        }
    }

    private void UpdateStatus(string status)
    {
        // Find a Text UI element in your scene (ensure there's a Text object with the "StatusText" tag)
        GameObject statusTextObject = GameObject.FindWithTag("StatusText");
        if (statusTextObject != null)
        {
            UnityEngine.UI.Text statusText = statusTextObject.GetComponent<UnityEngine.UI.Text>();
            statusText.text = $"Last Operation: {status}";
        }
    }
}
```

### Unity Scene Setup

1. **Create User Interface Elements:**
   - In the Unity Editor, add a `Canvas` to your scene.
   - Under the `Canvas`, add a `Text` element (use the UI > Text option).
   - Tag the `Text` element as "StatusText".

2. **Attach DataFetcher Script:**
   - Create a new empty GameObject in the scene.
   - Attach the `DataFetcher.cs` script to this GameObject.

### 4. Hosting Unity WebGL Build with XAMPP

1. **Build Settings in Unity:**
   - Go to `File` > `Build Settings`.
   - Select `WebGL` as the platform.
   - Click `Switch Platform`.
   - Click `Build` and choose a directory to save your WebGL build.

2. **Move the WebGL Build:**
   - Move the contents of the WebGL build folder to `C:\xampp\htdocs\`, and rename the folder to something appropriate (e.g., `digital_twin_simulation`).

3. **Start the Apache Server:**
   - Open the XAMPP control panel and start the Apache server.

4. **Access the Simulation:**
   - Open a web browser and navigate to `http://localhost/digital_twin_simulation` to view your Unity WebGL build.

### Final Workflow

- **Database and Server Management:** PostgreSQL serves as the database, while XAMPP manages the local server.
- **Backend and Control Logic:** The Python Flask server provides API endpoints and simulates the entire manufacturing process, logging each operation and publishing updates to the MQTT broker.
- **Real-Time Data Fetching and MQTT Messaging:** Unity retrieves data from the Flask server periodically and listens to MQTT messages to update the simulation in real-time.
- **WebGL Build:** The WebGL build hosted by XAMPP provides a web-accessible interface for monitoring the manufacturing process.

### Conclusion

This extensive setup integrates MQTT with PostgreSQL, Flask, Unity, WebGL, and XAMPP into a cohesive system. Following these comprehensive steps and using the provided code snippets, you can establish an autonomous industrial environment with real-time monitoring capabilities and bidirectional communication.  