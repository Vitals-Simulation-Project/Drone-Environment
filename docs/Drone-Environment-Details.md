---
title: Drone Environment Details
layout: template
filename: Drone-Environment-Details.md
--- 

# Drone Environment Details


# **Introduction**

This document includes notes about everything to do with the Drone Environment Unreal Project.

Currently, the Drone Environment repository is hosted on [Azure DevOps](https://dev.azure.com/Zach-Sally/VITALS-Drone-Environment/_git/Drone-Environment) due to Git LFS storage concerns with GitHub. Once a billing plan is set with our budget provided by Lockheed Martin, we will likely switch back to hosting the repository on our [GitHub](https://github.com/Vitals-Simulation-Project).


# **Unreal Engine Programming Workflow**

Below is just a quick guide to programming with Unreal Engine if you have never worked with it before.


## Terminology



* **Actor:** An object that can be placed in the level (Equivalent to GameObject in Unity)
* **Actor Component:** Component that adds additional functionality to Actor
* **Blueprints:** Visual Scripting language
* **GameInstance:** A high-level manager class for an instance of a running game
* **Widgets:** The term for UI components


## C++ Compilation

You can use Live Coding (bottom right button in editor) for small tweaks (function logic). Live Coding should be on by default, but can be toggled by going to Edit > Editor Preferences > Live Coding. Live Coding lets you compile C++ code without needing to close the Unreal Editor. 

Build the Solution in VS for large structural changes (new classes, inheritance, etc). A guideline to whether you should do Live Coding or Building the Solution is whether or not a header file is changed. If a header file is changed, then you should Build the Solution, otherwise Live Coding will suffice. You will still need to build the solution every so often even when using Live Coding, as Live Coding may introduce bugs. Also, if you have Live Coding enabled, you will have to close the Unreal Editor every time you want to Build Solution.

**Note:** I personally disabled Live Coding, opting to just stick with Building the Solution in VS instead. Disabling Live Coding lets you build through VS without needing to close the Unreal Editor.


## C++/Blueprint Workflow

In general, you will create a C++ class with exposed functions and properties. This is where you define them. The C++ files are broken down into two parts:



* MyClass.h - This is the header file that acts as an interface for other C++ classes to use. It declares the properties and functions and exposes them based on the assigned macro (BlueprintCallable, Blueprintable, etc)
* MyClass.cpp - This is where you will implement the functionality of the functions.

Then, you will create a Blueprint that derives from the C++ class. The Blueprint uses the properties and functions defined in the C++ class and extends upon them. Blueprints typically handle the more visual/auditory aspects (Animations, Sound, Environmental Effects)


# **File Structure**

Below is an outline of the files in the Unreal Project



* <span style="text-decoration:underline;">Config </span>- Where project settings are stored
* <span style="text-decoration:underline;">Content </span>- Where your assets are stored
    * <span style="text-decoration:underline;">Brian</span> - Contains assets related to Brian (Animation, Skeleton, Blueprints, etc)
    * <span style="text-decoration:underline;">Environment</span> - Everything related to the Landscape (Textures, Animals, etc)
    * <span style="text-decoration:underline;">Maps </span>- Contains the Landscape Map (Level) + its build data
    * <span style="text-decoration:underline;">POI</span> - Contains the Actor Component for Points of Interest that should show on the map, as well as the Blueprint for a waypoint actor
    * <span style="text-decoration:underline;">UI</span> - Contains everything related to the UI
        * <span style="text-decoration:underline;">Fonts</span> - Contains fonts used for text widgets
        * <span style="text-decoration:underline;">Icons</span> - Icon textures used for points of interest on maps
        * <span style="text-decoration:underline;">MainMenu</span> - Contains Main Menu Widget
        * <span style="text-decoration:underline;">MapUI</span> - Contains assets related to Minimap and LargeMap
        * <span style="text-decoration:underline;">Waypoints</span> - Contains assets related to the Drone and Waypoint data tables
* <span style="text-decoration:underline;">Source </span>- Where your C++ Code is stored
    * <span style="text-decoration:underline;">DroneEnvironment.Build.cs</span> - Defines dependency modules
    * <span style="text-decoration:underline;">DroneEnvironment.cpp</span> - Core game module
    * <span style="text-decoration:underline;">VitalsSimulationGameInstance.cpp</span> - Manages the instance of the project
    * <span style="text-decoration:underline;">WebSocketManager.cpp</span> - Handles WebSocket communication between UI and the WebSocket server.
* <span style="text-decoration:underline;">Plugins </span>- Where Plugins are stored
    * AirSim is **NOT **tracked by git. You must copy your own Plugins folder that was generated when you built Colosseum.


# **Brian**

Brian is the guy the drones are searching for. The current project has 3 difficulty modes: Easy, Medium, and Hard, which the user can select on start.

There are 5 “Easy” Brians. Each one is located near a more likely point of interest (man-made building at low altitude). 

There are 4 “Medium” Brians. Each one is located near a less likely point of interest (man-made building at higher altitudes and more remote positions).

There are 4 “Hard” Brians. Each one is located in a place not near any points of interest, and are typically higher in altitude.

When a user starts the Drone Environment, a menu will automatically pop up. On pressing Start, the user will be prompted to select a difficulty. On selecting a difficulty, the general process is as follows:



1. TargetSpawner binds event SpawnDifficultyTarget to Select Difficult Event delegate on BeginPlay
2. When event is fired (a difficulty button is selected), event passes in a string (Easy, Medium, Hard) that is switched
3. TargetSpawner performs the following sequence:
    1. Get All Actors of Class (RescueTarget&lt;DIFFICULTY>), and spawn a random one out of the list, destroying the other unselected actors.
    2. Destroy all actors of the other difficulty classes


# **Map**

The map is located in *Content/Maps/LandscapeMap*. 


## Dimensions and Coordinate Systems

Dimensions in Unreal Engine are in centimeters.

The coordinate system in Unreal Engine is different based on whether you are in Editor or Play Mode. In Editor Mode, there is an absolute world origin point at (0, 0, 0). In Play Mode, the world origin point is where the **PlayerStart** actor is placed.

Coordinate values with respect to Editor Mode:



* **Bottom Left**: (-32,000, 0)
* **Upper Right**: (195,000, -195,000)

Dimensions with respect to Play Mode:



* **PlayerStart**: (15,408, -103,271)
* **Bottom Left:** (-47,000, 102,000)
* **Upper Right:** (180,000, -92,000)

Regardless, the map is **227,000 cm x 195,000 cm**. This ratio is used to convert world coordinates to the respective minimap and large map coordinates.


## Minimap

Everything involving the minimap is located in *Content/UI/MapUI/Minimap*.


## Large Map

Everything involving the large map is located in *Content/UI/MapUI/LargeMap*.


## Points of Interest

Everything involving the icons/waypoints on the maps is located in *Content/POI*.
