---
layout: project-detail
name: "Playing tic-tac-toe with Roboarm"
image: "/assets/img/roboarm.png" 
description: "Creating a program for roboarm to be capable of playing tic-tac-toe with physical tiles"
tags:
    - "Computer Vision"
    - "Roboarm"
    - "Finished"
    - "Software"
    - "Robotics"
---
## Project description 
Created a program for **Kinova Gen3lite** roboarm that can play tic-tac-toe autonomously. The system tracks the board state, calculates optimal moves, and physically places pieces using computer vision and robotic control.

### Key technologies 
* Python 
* OpenCV 
* gRPC
* AprilTag

### Technical challenges
The main challenge was achieving accurate positioning despite robot, camera and real-world
limitations. Solved this using AprilTag markers for coordinate reference and
implementing error correction for reliable tiles placement.

### Results
The program successfully beats its creator in fair tic-tac-toe matches,
handles real-world coordinate transformations and maintains game state.

<a href="https://github.com/robocy-lab/tic-tac-robo">Project repository</a>
