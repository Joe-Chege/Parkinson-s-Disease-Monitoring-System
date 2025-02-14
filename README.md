# Parkinson-s-Disease-Monitoring-System



Let's create a script to set up the project for both the front end (Flutter) and back end (Flask) components. This script will include instructions to install dependencies, set up the environment, and run both the front end and back end.

### Project Structure
```
parkinsons_monitoring_system/
├── parkinsons_backend/
│   ├── app.py
│   ├── requirements.txt
│   └── venv/
├── parkinsons_frontend/
│   ├── android/
│   ├── ios/
│   ├── lib/
│   │   ├── main.dart
│   │   ├── pages/
│   │   │   ├── login_page.dart
│   │   │   ├── sensor_data_page.dart
│   │   ├── services/
│   │   │   └── socket_service.dart
│   ├── pubspec.yaml
│   └── ... (other Flutter project files)
└── setup.sh
```

### `setup.sh`
```sh name=setup.sh
#!/bin/bash

# Function to display a message
function print_message {
  echo "=================================================="
  echo "$1"
  echo "=================================================="
}

# Set up the backend
print_message "Setting up the backend..."

# Navigate to the backend directory
cd parkinsons_backend

# Create and activate a virtual environment
python3 -m venv venv
source venv/bin/activate

# Install backend dependencies
pip install -r requirements.txt

# Deactivate the virtual environment
deactivate

# Navigate back to the project root
cd ..

# Set up the frontend
print_message "Setting up the frontend..."

# Navigate to the frontend directory
cd parkinsons_frontend

# Get Flutter dependencies
flutter pub get

# Navigate back to the project root
cd ..

# Instructions to run the backend
print_message "To run the backend:"
echo "1. Navigate to the backend directory: cd parkinsons_backend"
echo "2. Activate the virtual environment: source venv/bin/activate"
echo "3. Run the Flask application: gunicorn -w 4 -b 0.0.0.0:8000 app:app"

# Instructions to run the frontend
print_message "To run the frontend:"
echo "1. Navigate to the frontend directory: cd parkinsons_frontend"
echo "2. Run the Flutter application: flutter run"

print_message "Setup complete!"
```

### Backend Files

#### `parkinsons_backend/app.py`
```python name=parkinsons_backend/app.py
from flask import Flask, request, jsonify
from flask_socketio import SocketIO, emit
from flask_cors import CORS
from pymongo import MongoClient
from dotenv import load_dotenv
import os
import tensorflow as tf

# Load environment variables
load_dotenv()

app = Flask(__name__)
app.config['SECRET_KEY'] = os.getenv('SECRET_KEY')
CORS(app)
socketio = SocketIO(app, cors_allowed_origins="*")

# MongoDB setup
mongo_client = MongoClient(os.getenv('MONGO_URI'))
db = mongo_client['parkinsons_db']

# TensorFlow model (placeholder)
model = tf.keras.Sequential([
    tf.keras.layers.Dense(64, activation='relu', input_shape=(10,)),
    tf.keras.layers.Dense(32, activation='relu'),
    tf.keras.layers.Dense(1, activation='sigmoid')
])
model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])

@app.route('/api/login', methods=['POST'])
def login():
    # Implement your login logic here
    # For simplicity, we'll just return a success message
    return jsonify({"message": "Login successful", "user_type": "patient"})

@app.route('/api/sensor-data', methods=['POST'])
def receive_sensor_data():
    data = request.json
    # Store sensor data in MongoDB
    db.sensor_data.insert_one(data)
    # Emit the data to connected clients
    socketio.emit('new_sensor_data', data)
    return jsonify({"message": "Sensor data received and stored"})

@socketio.on('connect')
def handle_connect():
    print('Client connected')

@socketio.on('disconnect')
def handle_disconnect():
    print('Client disconnected')

@socketio.on('sensor_data')
def handle_sensor_data(data):
    print(f"Received sensor data: {data}")
    # Process the data (e.g., run it through the AI model)
    processed_data = process_sensor_data(data)
    # Emit the processed data back to the client
    emit('processed_sensor_data', processed_data)

def process_sensor_data(data):
    # This is a placeholder for AI processing
    # In a real application, you'd use your TensorFlow model here
    # For now, we'll just return a dummy prediction
    dummy_input = tf.random.normal((1, 10))
    prediction = model.predict(dummy_input)
    return {"original_data": data, "ai_prediction": float(prediction[0][0])}

if __name__ == '__main__':
    socketio.run(app, host='0.0.0.0', port=8000, debug=False)
```

#### `parkinsons_backend/requirements.txt`
```plaintext name=parkinsons_backend/requirements.txt
Flask==2.0.2
flask-socketio==5.1.1
flask-cors==3.0.10
pymongo==3.12.1
gunicorn==20.1.0
tensorflow==2.8.0
python-dotenv==0.19.2
```

### Frontend Files

#### `parkinsons_frontend/lib/main.dart`
```dart name=parkinsons_frontend/lib/main.dart
import 'package:flutter/material.dart';
import 'package:parkinsons_monitoring_system/services/socket_service.dart';
import 'package:parkinsons_monitoring_system/pages/login_page.dart';
import 'package:parkinsons_monitoring_system/pages/sensor_data_page.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatefulWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  _MyAppState createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  final SocketService _socketService = SocketService();

  @override
  void initState() {
    super.initState();
    _socketService.connect();
  }

  @override
  void dispose() {
    _socketService.disconnect();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Parkinson\'s Monitoring System',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        visualDensity: VisualDensity.adaptivePlatformDensity,
      ),
      home: const LoginPage(),
      routes: {
        '/sensor-data': (context) => const SensorDataPage(),
      },
    );
  }
}
```

#### `parkinsons_frontend/lib/pages/sensor_data_page.dart`
```dart name=parkinsons_frontend/lib/pages/sensor_data_page.dart
import 'package:flutter/material.dart';
import 'package:parkinsons_monitoring_system/services/socket_service.dart';

class SensorDataPage extends StatefulWidget {
  const SensorDataPage({Key? key}) : super(key: key);

  @override
  _SensorDataPageState createState() => _SensorDataPageState();
}

class _SensorDataPageState extends State<SensorDataPage> {
  final SocketService _socketService = SocketService();

  void _sendSensorData() {
    Map<String, dynamic> sensorData = {
      'patientId': 'patient123',
      'sensorType': 'tremor',
      'value': 0.75,
      'timestamp': DateTime.now().toIso8601String(),
    };
    _socketService.sendSensorData(sensorData);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Sensor Data')),
      body: Center(
        child: ElevatedButton(
          onPressed: _sendSensorData,
          child: const Text('Send Sensor Data'),
        ),
      ),
    );
  }
}
```

#### `parkinsons_frontend/lib/services/socket_service.dart`
```dart name=parkinsons_frontend/lib/services/socket_service.dart
import 'package:socket_io_client/socket_io_client.dart' as IO;

class SocketService {
  late IO.Socket socket;
  static const String _serverUrl = 'http://10.0.2.2:5000';  // Use this for Android emulator
  // static const String _serverUrl = 'http://localhost:5000';  // Use this for web or iOS simulator

  void connect() {
    socket = IO.io(_serverUrl, <String, dynamic>{
      'transports': ['websocket'],
      'autoConnect': false,
    });

    socket.connect();

    socket.on('connect', (_) {
      print('Connected to server');
    });

    socket.on('disconnect', (_) {
      print('Disconnected from server');
    });

    socket.on('new_sensor_data', (data) {
      print('Received new sensor data: $data');
      // Handle the new sensor data (e.g., update UI)
    });

    socket.on('processed_sensor_data', (data) {
      print('Received processed sensor data: $data');
      // Handle the processed sensor data (e.g., update UI with AI predictions)
    });
  }

  void sendSensorData(Map<String, dynamic> data) {
    socket.emit('sensor_data', data);
  }

  void disconnect() {
    socket.disconnect();
  }
}
```

#### `parkinsons_frontend/pubspec.yaml`
```yaml name=parkinsons_frontend/pubspec.yaml
name: parkinsons_monitoring_system
description: A new Flutter project.

publish_to: 'none' # Remove this line if you want to publish to pub.dev

version: 1.0.0+1

environment:
  sdk: ">=2.12.0 <3.0.0"

dependencies:
  flutter:
    sdk: flutter
  socket_io_client: ^2.0.2
  firebase_core: ^2.15.1
  firebase_auth: ^4.9.0
  http: ^1.1.0
  provider: ^6.0.5
  fl_chart: ^0.63.0
  shared_preferences: ^2.2.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^2.0.0

flutter:
  uses-material-design: true
```

### Running the Script

To set up the project, run the `setup.sh` script:
```sh
chmod +x setup.sh
./setup.sh
```

This script will set up both the front end and back end of your project, install necessary dependencies, and provide instructions for running the applications.