
# POSIX Software-In-The-Loop (SIL) Automotive Simulator

> **Disclaimer**: This project was developed based on the architectural specifications provided by the RaceUP Formula Student team for their Software-In-the-Loop environment. The core ECU safety logic (smu.cpp, ebs.cpp) is based on their domain model, while the POSIX SIL infrastructure, multithreading synchronization, CI/CD pipelines, and GTest suites were entirely architected and implemented by me to showcase hard real-time C++ practices.

## Overview
This repository contains a host-based **Software In the Loop (SIL)** environment for two vehicle Electronic Control Units (ECUs):
- **Sensor Management Unit (SMU):** Reads pedal sensors and publishes normalized values/faults.
- **Emergency Brake System (EBS):** Evaluates critical vehicle state (pressures, speeds, mission flags) to safely control the autonomous braking valves.

The goal of this project was to implement a **thread-safe, POSIX-based simulation** around the fixed ECU interfaces, ensuring real-time constraints, race-free state management, and reliable black-box testing without requiring actual microcontrollers or SocketCAN hardware.

## Key Technical Implementations
As the developer of the SIL infrastructure, my main contributions include:

1. **Thread-Safe Simulator (`src/sil.cpp`):** Implemented a Producer-Consumer architecture using POSIX threads, `std::mutex`, and `std::condition_variable` with bounded queues and a "drop-oldest" policy to prevent priority inversion and simulate real-time CAN bus behavior.
2. **Lifecycle Management:** Engineered clean thread initialization, synchronization, and safe shutdown/join mechanisms without relying on busy-waiting (100% CPU usage) or infinite waits.
3. **Comprehensive Testing:** Developed a robust suite of GTest black-box tests asserting both nominal behavior and edge-case safety states (e.g., sensor track shorts, 250ms message staleness, watchdog faults).
4. **CI/CD & DevOps:** Built an automated GitHub Actions workflow to enforce C++/Python linting and execute incremental builds. 
5. **Containerization & Sanitization:** Added Podman/Docker support for reproducible builds and integrated **ThreadSanitizer (TSAN)** to mathematically prove the absence of data races and deadlocks.

---

## Build and Test Instructions

**Requirements:**
- CMake 3.25 or newer
- A C++20 compiler
- Ninja

Run the complete local workflow with:
```bash
cmake --workflow --preset sil
```
### Running via Docker/Podman
To ensure a reproducible build environment, the entire build process and test suite can be run inside a Docker or Podman container. 

Run the following command from the root of the repository:

Using Podman:
```bash
podman run --rm -v "$PWD":/workspace -w /workspace ubuntu:24.04 bash -c "apt-get update && apt-get install -y cmake ninja-build g++ && cmake --workflow --preset sil"
```

>[!NOTE]
>If you are using a RHEL-based distribution (Fedora, CentOS) and have SELinux enabled, you may need to [adjust the volume mount options](https://docs.docker.com/engine/storage/bind-mounts/#configure-the-selinux-label) to avoid permission issues when running the container, by adding `:z` or `:Z` to the volume mount, like so:
>```bash
>podman run --rm -v "$PWD":/workspace:z -w /workspace ubuntu:24.04 bash -c "apt-get update && apt-get install -y cmake ninja-build g++ && cmake --workflow --preset sil"
>```
>Alternatively, you can run the container with elevated privileges using `--privileged` or temporarily disable SELinux using `sudo setenforce 0`, but this is generally discouraged for security reasons.

Using Docker:
```bash
docker run --rm -v "$PWD":/workspace -w /workspace ubuntu:24.04 bash -c "apt-get update && apt-get install -y cmake ninja-build g++ && cmake --workflow --preset sil"
```


### Running with ThreadSanitizer (TSAN)
To run the test suite with ThreadSanitizer enabled inside Podman/Docker to detect race conditions:
```bash
podman run --rm -v "$PWD":/workspace:z -w /workspace ubuntu:24.04 bash -c "apt-get update && apt-get install -y cmake ninja-build g++ && CXXFLAGS='-fsanitize=thread -g -fno-omit-frame-pointer' LDFLAGS='-fsanitize=thread' TSAN_OPTIONS='second_deadlock_stack=1' cmake --workflow --preset sil"
```

---

## ECU Domain Logic (Application Under Test)

### SMU (Sensor Management Unit) Behavior
The SMU reads inverted raw 12-bit ADC values and normalizes them. It must:
- Detect an APP (Accelerator Pedal Position) track short (< 50 counts difference).
- Detect an APP plausibility fault (> 10% normalized difference).
- Set throttle to zero whenever an APP fault is detected.
- Publish a `SMU_SENSE` frame every cycle (Throttle %, Brake %, Fault Code).

### EBS (Emergency Brake System) Behavior
The EBS runs every 10 ms and receives critical input frames (`VEHICLE_STATUS`, `BRAKE_PRESSURE`, `EBS_COMMAND`) which **become stale after 250 ms**. 

The EBS can close the ASNode (Safety Relay) only when all strict conditions are met, including:
- All critical input frames are fresh.
- ASMS is active, valves powered, watchdog healthy, ready-to-drive active.
- Pneumatic pressures between 7-10 bar; Hydraulic between 15-99 bar.
- Vehicle speed $\le 5$ dm/s.

When safe, the EBS opens the ASNode and engages the emergency valves. When nominal, it closes the ASNode and releases the valves.
