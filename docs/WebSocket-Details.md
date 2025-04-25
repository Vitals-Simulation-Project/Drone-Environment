---
title: WebSocket Details
layout: template
filename: WebSocket-Details.md
--- 

# **WebSocket Details**


# **Introduction**

This is a guide detailing how we are using a WebSocket server to communicate between the Unreal Engine project and the Python Client. 


# **Sockets and WebSockets Overview**

A socket is a connection between two services over a TCP/IP connection. It acts as a programmatic way to send and receive data across a network. Sockets are bidirectional (two way) and full duplex (can go both ways simultaneously).  A socket is composed of an IP address and a designated port number.

A WebSocket is a network protocol that builds on the idea of sockets. It is built for rapid real-time communication between services, and abstracts the complexity of socket management that you would find with regular sockets.


# **Why WebSockets?**

While developing the project, we learned that we needed a way to facilitate communication between both Unreal Engine and the Python Client. Although Colosseum provides an API that gets specific data FROM Unreal Engine TO Python, there is no way for the Unreal Engine UI team to get that information. This is especially problematic because when we provide waypoint overriding from the UI, we need to send data FROM Unreal Engine TO Python. 

Our solution is to use a WebSocket server that simply handles sending messages between Unreal Engine and the Python Client. That way, we can send and receive data both ways.

*(Side Note)* This was the approach that the previous team had done for their minimap. They had a Minimap Server Python script that sent messages using ROS to a Minimap built with Unity. There are two main differences about our approach:



* Our team ported the minimap from Unity to Unreal, eliminating the Unity dependency for the project.
* Our team is avoiding using ROS to handle data communication due to its setup complexity.


# **Use Cases**

As mentioned above, we are using WebSockets to communicate data between the UI and the Python Client. Below are all of the cases that require WebSocket communication in this project.

**Python Client→ UI**



* The Python Client can add a waypoint
* The Python Client can delete a waypoint
* The Python Client updates a drone’s target in the data table
* The Python Client updates a drone’s state in the data table

**UI → VLM**



* User can add a waypoint through the UI
* User can delete a waypoint through the UI


# **JSON Message Structure**

Every message sent will have the `MessageType` field. The `MessageType` field contains a string that acts as the identifier for what action to take.

Below is the JSON structure for each of the functions


## **Python Client → UI**

<span style="text-decoration:underline;">Add Waypoint:</span>


```
{
	"MessageType": "AddWaypoint",
	"X": float,
	"Y": float,
	"Z": float
}
```


<span style="text-decoration:underline;">Delete Waypoint:</span>


```
{
	"MessageType": "DeleteWaypoint",
	"ID": int
}
```


<span style="text-decoration:underline;">Update Drone Target:</span>


```
{
	"MessageType": "UpdateDroneTarget",
	"DroneID": int,
	"WaypointID": int
}
```


<span style="text-decoration:underline;">Update Drone State:</span>


```
{
	"MessageType": "UpdateDroneState",
	"DroneID": int,
	"State": string
}
```



## **UI → Python Client**

<span style="text-decoration:underline;">Start Simulation:</span>


```
{
	"MessageType": "StartSimulation",
}
```


<span style="text-decoration:underline;">Stop Simulation:</span>


```
{
	"MessageType": "StopSimulation"
}
```


<span style="text-decoration:underline;">Add Waypoint:</span>


```
{
	"MessageType": "AddWaypoint",
	"ID": int
	"X": float,
	"Y": float,
	"Z": float,
	"Priority": int
}
```


<span style="text-decoration:underline;">Delete Waypoint:</span>


```
{
	"MessageType": "DeleteWaypoint",
	"ID": int
}
```



# **WebSocket Server**

**websocket_server.py** is the Python script that acts as the “server”. It starts a WebSocket server on localhost:8765. The full URL to connect to this WebSocket server is **ws://localhost:8765**.

The WebSocket server is started as its own process inside the <span style="text-decoration:underline;">Mission Control</span> script using the **start_websocket_server()** function. This function is the entry point to setting up the WebSocket server. It creates a new asynchronous event loop that handles all async tasks within the WebSocket communication. This new event loop calls the **start()** function.

The **start()** function is what actually starts the WebSocket server. It sets the handler, host, and port values.

The **handler()** function is what a client will do when it connects to the server. Whenever a client sends a message to the server, the server will send the message asynchronously to every OTHER client in the **CLIENTS **set.


# **Python Client**

The <span style="text-decoration:underline;">Parent Controller</span> script handles WebSocket communication on the Python Client side. Inside the **perform_startup_sequence()** function, a background thread is created to run **start_websocket_client_thread()**.

The **start_websocket_client_thread()** function, like the WebSocket server, creates a new asynchronous event loop that handles all async tasks within the WebSocket communication. This new event loop calls the **connect_to_websocket_server()** function.

The **connect_to_websocket_server()** is an asynchronous function to connect the client to the server. It then performs an infinite loop of receiving messages from the server and putting the message in the **RECEIVED_UI_DATA_QUEUE** to be processed later.

When the Parent Controller enters its **perform_startup_sequence()** function, it will wait for a <span style="text-decoration:underline;">StartSimulation</span> message from the UI. This is sent when the user selects a difficulty.

After the user selects a difficulty, the Parent Controller enters its **receive_initial_waypoints()** function, where it will process all of the incoming AddWaypoint messages that come from the UI.

When the Parent Controller enters its continuous **loop()** function, each iteration starts with fetching any WebSocket messages in the **RECEIVED_UI_DATA_QUEUE** and processing it inside **process_websocket_message()**.

**The process_websocket_message()** function handles the messages sent by the UI client. There are three potential messages to process from the UI:



1. <span style="text-decoration:underline;">AddWaypoint</span>: Creates a new waypoint and pushes it onto the Waypoint Priority Queue
2. <span style="text-decoration:underline;">DeleteWaypoint</span>: Deletes an existing waypoint from the Waypoint Priority Queue. If a drone’s target is deleted, this also sets its current target to None.
3. <span style="text-decoration:underline;">StopSimulation</span>: Fires the SHUTDOWN_EVENT that is shared across all processes 


# **UI Client**

The UI client starts with **BP_VitalsSimulationGameInstance**. This is the GameInstance defined for the project under Project Settings > Maps and Modes. A GameInstance is a high-level manager for running the game. It contains overridable **Init()** and **Shutdown()** methods that are called once during the lifecycle of the game.

**BP_VitalsSimulationGameInstance** is derived from the **VitalsSimulationGameInstance **C++ file. This blueprint only defines three event handlers:



1. An event to get the reference to **BP_WaypointManager**
2. An event to get the reference to **WBP_DroneTable**
3. An event to get the reference to **WBP_HUD**

These events are called from WebSocketManager when needed.

**VitalsSimulationGameInstance.cpp** is the core GameInstance file. It contains the following:



* **Init():** GameInstance override, creates and initializes the **WebSocketManager **object
* **Shutdown():** GameInstance override, cleans up the WebSocketManager

It also contains wrapper methods for sending messages. This is so that other blueprints can call these methods once they get a reference to the GameInstance class.

**WebSocketManager.cpp** is what handles all WebSocket communication. It is initialized within **VitalsSimulationGameInstance.cpp**. Below is a list of the public methods that are called by the wrapper methods of the GameInstance:



* **Initialize():** Continuously probe for connection to the server. Once connected, it sets up WebSocket event handlers.
* **Close():** Closes the client connection
* **SendAddWaypointMessage():** Sends an AddWaypoint message to the server
* **SendDeleteWaypointMessage():** Sends a DeleteWaypoint message to the server
* **SendStartSimulationMessage():** Sends a StartSimulation message to the server
* **SendStopSimulationMessage():** Sends a StopSimulation message to the server

When a message is received from the WebSocket server, the message is parsed as a JSON and is sent to a handler method based on the <span style="text-decoration:underline;">MessageType</span>** **property.

All message handlers in WebSocketManager are set up so that they call some other event in a specific Blueprint. In other words, they are calling a Blueprint-defined event from a C++ file. In general, they perform the following operations:



1. Get the associated data from the JSON message
2. Get a function pointer to the event from the Blueprint you want to call
3. Create a struct containing the parameters for the event
4. Call ProcessEvent with the function pointer and the parameters

For example, this is the general outline for when the UI receives an AddWaypoint message from the WebSocket server:



1. Get the X, Y, Z coordinates from the message
2. Get a function pointer to the **AddWaypointEvent** found in **BP_WaypointManager**
3. Create a struct that matches the **AddWaypointEvent** parameters
4. Call ProcessEvent with the function pointer and the parameters.

This will fire **BP_WaypointManager**’s **AddWaypoint Event** directly from C++.
