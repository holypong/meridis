# meridis - Background, Purpose & Glossary

English / [Japanese](README_concept.md)

This document explains the philosophy behind the project, its relationship with Physical AI, and the foundational terminology required for development.  
If you want to know "Why meridis?", "Why MuJoCo?", or "What possibilities does it open up?", this is the place to start.  
<BR>
This document also covers the relationship between this project and various technologies — Physical AI, Embodied AI, and more — along with the key terms you need to know for development.

## Table of Contents

- [Background](#background)
- [Purpose](#purpose)
  - [Research & Development](#research--development)
  - [Engineering](#engineering)
- [Key Features](#key-features)
- [Glossary](#glossary)
  - [Basic Terms](#basic-terms)
  - [Robot-Related Terms](#robot-related-terms)
  - [Technical Terms](#technical-terms)
  - [Physical AI Terms](#physical-ai-terms)
  - [Simulator-Related Terms](#simulator-related-terms)
  - [meridis-Specific Terms](#meridis-specific-terms)

---

## Background

- In recent years, the reach of AI has expanded beyond computers into the physical world.
- AI that has acquired a body and can physically act in the real world is called **Physical AI** or **Embodied AI**.
- The value of simulation environments — where you can repeat trials as many times as needed, safely validate behavior without damaging yourself or your surroundings — is growing rapidly as a prerequisite for deploying robots in the real world.
- Traditional approaches required implementing a different interface for each simulator, which increased development costs and reduced code reusability.
- There was also a need for a mechanism that allows multiple processes (simulation, real-robot control, AI control, monitoring tools) to access the same data simultaneously and exchange information with low latency.
- **meridis** solves these problems by using the high-speed in-memory database Redis as a common interface.

## Purpose

### Research & Development

- In Physical AI and Embodied AI research, it is important to efficiently validate algorithms under conditions close to reality.
- If algorithms validated in a simulation environment can be applied to real robots with minimal code changes, the research and development cycle can be dramatically accelerated.
- **meridis** combines a common data structure (Meridim90 format) with the standard technology Redis to achieve the following:
  - Apply control logic validated in simulation directly to real hardware
  - Reuse the same control code across multiple simulators (MuJoCo, Genesis, NVIDIA Isaac Sim, etc.)
  - Build highly reproducible experimental environments through data recording and playback
  - Realize digital twin (synchronized operation) between real hardware and simulation

### Engineering

- Robot development requires integrating multiple components: control systems, simulators, monitoring tools, and AI agents.
- Implementing proprietary protocols to exchange data between these components reduces maintainability and makes extension difficult.
- **meridis** uses Redis — an industry-standard in-memory database — as a common interface to achieve the following:
  - Execute commands from AI agents (via MCP server) using the same procedure for both simulation and real hardware
  - Develop and test each component (simulator, control system, monitoring tools) independently
  - Easy integration with existing systems (Redis client libraries are available in many languages)
  - Simple addition of real-time monitoring, logging, and debugging capabilities
  - Provide a foundation on which AI agents can automatically run the "think → try → evaluate" development loop

---

## Key Features

- **Bidirectional Integration of Simulation and Real Hardware**
  - **Sim2Real**: Send control commands from simulation to a real robot
  - **Real2Sim**: Reproduce real robot behavior in simulation
  - **Digital Twin**: Synchronize simulation and real hardware for simultaneous operation

- **UDP/Redis Bridge Manager (`meridis_manager.py`)**  
  Acts as a bridge connecting robot simulations, real robots, AI agents, and other external systems via a combination of **Redis** and UDP.

- **Rich Utility Tools**
  - Data transfer library (`redis_transfer.py`)
  - Data receive library (`redis_receiver.py`)
  - Real-time data visualization tool (`redis_plotter.py`)
  - PAD button-triggered data logger (`redis_logger.py`)

- **Common Data Structure (Meridim90)**  
  Provides a robot control data format defined as a numerical array of 90 elements.

- **Real-Time Data Exchange via Redis**  
  High-speed transmission and reception of control and status data between multiple systems via the in-memory database Redis.

- **Multi-Platform Support**  
  Confirmed to work on Windows / Linux / WSL / macOS.

---

## Glossary

### Basic Terms

**Meridian**<br>
An open-source system required when using meridis to control a real robot. It consists of a dedicated board and dedicated software. **If you are only using a simulator, Meridian is not required.**<br>
[What is the Meridian Project?](https://note.com/ninagawa123/n/ncfde7a6fc835)

**meridis**  
The name of this project. A portmanteau of "Meridim90 data format" and "Redis" — a collection of bridge tools for connecting robot simulations, real robots, and AI agents through a common interface. Proven with MuJoCo, Genesis AI, and Isaac Sim. Developed by holypong under the `#Meridian` project.

**Meridim90**  
The standard format for robot control data. Defined as a numerical array of 90 elements, it stores joint angles, sensor values, control flags, and more. Stored as fields `"0"` through `"89"` of a Redis hash and shared between processes.

**Redis**  
Short for Remote Dictionary Server. A high-speed in-memory database. Used to instantly exchange data between programs. In this project, it acts as the bridge between simulation and external systems. Supports a variety of data structures as a key-value store, including hashes, lists, and sets.

**MCP (Model Context Protocol)**  
A protocol for AI agents to interact with external tools and data sources in a standardized way. In this project, it is used (planned) to enable AI agents to control robots. An open standard proposed by Anthropic.

### Robot-Related Terms

**Sim2Real**  
Short for Simulation to Real. The process of applying behavior created and validated in a simulation environment to an actual (real) robot. Effective for balancing development efficiency and safety. In meridis, achieved using `mgr_sim2real.json`.

**Real2Sim**  
Short for Real to Simulation. The process of capturing the movements and sensor data of a real robot into a simulation environment. Used for real-hardware motion analysis and debugging. In meridis, achieved using `mgr_real2sim.json`.

**Digital Twin**  
A digital reproduction of a real-world object or system. In this project, it refers to the state in which a simulated robot and a real robot operate in synchronization. Achieved by enabling bidirectional data transfer (both `redis_to_udp` and `udp_to_redis`).

**Joint**  
Each movable part of a robot, analogous to a human's shoulder, elbow, or knee. Each joint has a defined angle and range of motion. In the Meridim90 array, target angles are stored at even indices and current angles at odd indices.

**IMU (Inertial Measurement Unit)**  
A sensor that measures a robot's orientation (tilt), angular velocity, and acceleration. In this project, data computed virtually within the simulation or retrieved from real hardware is stored in the Meridim90 array.

### Technical Terms

**In-Memory Database**  
A database that stores data in memory (RAM) rather than on disk. Reads and writes are extremely fast, but data is lost when power is cut (persistence options are available). Redis is the most prominent example of this type.

**Hash**  
One of Redis's data structures. Stores multiple field-name/value pairs. In this project, the 90 elements of Meridim90 are stored under field names `"0"` through `"89"`. Written with `HSET` and fully retrieved with `HGETALL`.

**Pub/Sub (Publish/Subscribe)**  
Redis's messaging feature. A Publisher sends a message to a channel, and Subscribers receive it. Used for real-time notifications and event-driven processing.

**UDP (User Datagram Protocol)**  
An internet protocol. Unlike TCP, it sends data without establishing a connection, making it fast but without delivery guarantees. UDP is often used in robot control where low latency is critical. In this project, it is used for communication with real robots.

**JSON (JavaScript Object Notation)**  
A lightweight text format for describing configuration and data. Used in this project for Redis connection settings (`mgr_sim2real.json`) and network settings (`network.json`). Easy for both humans to read and programs to process.

**Terminal / Command Line**  
A screen where you type commands to run programs. Examples include PowerShell on Windows and Terminal on macOS/Linux. In this document, used to run `python` and `redis-cli` commands.

### Physical AI Terms

**Physical AI**  
AI with a body that can act in the physical world. Refers to AI systems that operate in real space, such as robots and drones. Leans toward practical, real-world implementation. An important development field where simulation-to-real integration is critical.

**Embodied AI**  
AI that has a body (embodiment) and learns and acts by interacting with its environment. Often used interchangeably with Physical AI, though it leans more toward academic research. An AI research field that emphasizes intelligence acquired through embodiment.

**Reinforcement Learning**  
A technique in which AI learns optimal behavior through trial and error. Large numbers of trials are typically run in simulation and then applied to real hardware. Commonly used in combination with simulators such as MuJoCo, Genesis, and Isaac Sim.

### Simulator-Related Terms

**MuJoCo**  
Short for Multi-Joint dynamics with Contact. A physics engine for simulating the physical movement of robots and objects with high accuracy. Open-sourced by DeepMind and now freely available. Used as the physics engine in merimujoco.

**Why MuJoCo Was Chosen**  
While meridis can connect to multiple simulators, MuJoCo in particular is widely adopted in research and development for the following reasons:

- **Fast and accurate contact simulation**: Precisely and rapidly computes the contact dynamics of multi-joint robots, handling complex tasks such as locomotion and grasping.
- **Alignment with the research community**: Since DeepMind's open-sourcing, it has become the de facto standard simulator in reinforcement learning and motion control research papers.
- **Lightweight and highly extensible**: Its lightweight design runs on CPU alone, making it easy to run on a laptop. GPU parallelization technologies such as NVIDIA Warp can further accelerate it hundreds of times, supporting large-scale reinforcement learning and parallel simulation.
- **Integration with cutting-edge robotics AI**: Next-generation robotics platforms such as NVIDIA Isaac Sim/Lab and Genesis adopt MuJoCo as their core engine, enabling seamless integration with reinforcement learning libraries and immediate implementation of the latest algorithms from research papers.
- **Excellent built-in UI**: Comes with an interactive viewer featuring intuitive mouse-controlled camera operation, sliders for adjusting joint angles, and real-time display of physics parameters — greatly improving debugging efficiency and development speed.

**merimujoco**  
A robot simulator based on MuJoCo, designed from the ground up for integration with meridis. Sends and receives data via Redis, enabling seamless connection with real robots and AI agents. Developed by holypong.

**Genesis**  
A next-generation platform integrating physics simulation with AI robotics. Based on MuJoCo, it achieves ultra-high-speed simulation (up to 430,000 FPS) via GPU acceleration. Optimized for robot development in the generative AI era, with a design that completes physics simulation, rendering, and training within a single Python script.

**NVIDIA Isaac Sim / Isaac Lab**  
A robot development platform by NVIDIA. Isaac Sim is a simulation environment based on OpenUSD/Omniverse, which has traditionally used NVIDIA PhysX as its physics engine. Isaac Lab is a framework for training control policies through reinforcement learning and imitation learning on top of Isaac Sim. Recent versions also support next-generation GPU-accelerated physics simulation through integration with Newton.

**NVIDIA Newton**  
An open-source GPU-accelerated physics engine co-developed by NVIDIA, Google DeepMind, and Disney Research. Built on NVIDIA Warp and OpenUSD, it enables large-scale parallel simulation and differentiable physics. Supports Isaac Sim / Isaac Lab and MuJoCo Playground for humanoid robot motion control, reinforcement learning, and Sim2Real transfer.

### meridis-Specific Terms

**meridis_manager.py**  
A bridge program that manages bidirectional data transfer between Redis and UDP. The core tool for achieving real-time data exchange between simulation environments and real robots. The operating mode can be switched via a manager configuration file (JSON).

**redis_transfer.py**  
A library for writing data to the Redis server. Transfers Meridim90-format data to a Redis hash. Can be imported as the `RedisTransfer` class from other programs.

**redis_receiver.py**  
A library for reading data from the Redis server. Retrieves Meridim90 data stored in a Redis hash. Can be imported as the `RedisReceiver` class from other programs.

**redis_plotter.py**  
A visualization tool for displaying robot data stored in Redis as real-time graphs. Used to monitor and debug time-series changes in joint angles and foot positions. Displays animations using matplotlib.

**redis_logger.py**  
A logger tool that records Meridim90 data flowing through Redis to a CSV file, triggered by a PAD button value (Meridim90[15]). Accumulates data in a buffer only while the specified PAD button value matches, and saves to `log/logs-YYYYMMDDHHMM.csv` when the button value changes or the buffer limit (10,000 rows) is reached. Useful for collecting training data for machine learning.

**Redis Key**  
The name used to identify data within the Redis database.

**data_flow**  
A parameter in the manager configuration file. Controls the direction in which data flows.

---

## Further Reading

- **MuJoCo official documentation**: https://mujoco.readthedocs.io/
- **Redis official site**: https://redis.io/
- **merimujoco detailed manual**: https://github.com/holypong/merimujoco
- **Model Context Protocol**: https://modelcontextprotocol.io/
- **NVIDIA Isaac Sim**: https://developer.nvidia.com/isaac-sim
- **NVIDIA Isaac Lab**: https://isaac-sim.github.io/IsaacLab/
- **Genesis**: https://genesis-world.readthedocs.io/
- **Newton**: https://github.com/newton-physics/newton
