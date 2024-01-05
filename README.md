# Twenty Arms, One Brain: Making a Swarm of Manipulators Work as a Single Unit

<div align="center">
<img src="images/so101-hero.png" alt="SO-101-LE ROBOT Industrial Swarm" width="520">
</div>

I plugged eleven USB devices into one host — seven manipulators and four cameras — and on the twelfth, everything collapsed. Not in software. Physically. The USB 2.0 controller ran out of bandwidth, the kernel spat `Not enough bandwidth for new device state`, and the twelfth manipulator simply didn't appear in the system. A camera sharing the same hub started dropping frames.

That was the moment I understood: "twenty arms" isn't marketing. It's an engineering problem. And it's far messier than it looks from the README.

---

The entire project fits in one paragraph. There are SO-101 manipulators — an open-source platform from The Robot Studio and Hugging Face, six degrees of freedom, STS3215 servos, SCS protocol on a 1 Mbaud bus. There's a software interface that accepts any number of these manipulators, assigns them roles (leader or follower), links them into pairs and groups, binds cameras, records operator demonstrations, and trains a neural policy on them. The trained policy runs in parallel on all followers. One hour of demonstration — twenty arms working simultaneously.

Sounds clean. Now try actually plugging it in.

## USB topology: why twenty arms don't scale linearly

Each SO-101 connects through a CH340 USB-Serial adapter. One adapter — one full-speed device (12 Mbit/s maximum, though actual traffic at 1 Mbaud on the SCS bus is well below that). Each camera — USB 2.0 with MJPEG stream: 320×240 at 30 fps is roughly 15–20 Mbit/s per camera, 640×480 — closer to 40.

A USB 2.0 hub shares 480 Mbit/s across all devices on one bus. That's the theoretical maximum; in practice, with protocol overhead, you get about 280–320 Mbit/s. Four cameras at 320×240 — already 60–80 Mbit/s. Seven CH340 adapters — another ~7 Mbit/s total (servos generate little traffic). Still manageable.

But here's the thing that isn't obvious. USB 2.0 devices plugged into a USB 3.0 hub still share only 480 Mbit/s between themselves. A 3.0 hub has a separate controller for 3.0 devices and a separate one for legacy 2.0. CH340 and most UVC cameras are 2.0 devices. Plug them into a 3.0 hub — and they land on the legacy controller with the same 480 Mbit/s limit. USB 3.0 adds nothing.

The solution I arrived at: a hub hierarchy distributed across root controllers.

A typical motherboard has 2–4 USB root controllers (Root Hubs), each with its own bandwidth. A PCIe USB card adds one or two more. The idea: spread devices across different root controllers so that each one carries no more than 4–5 cameras and 5–6 servo adapters. For twenty manipulators with two cameras each (40 cameras + 20 CH340), you need at minimum 5–6 independent root controllers. That's one PCIe USB 3.0 host controller with four ports plus the motherboard's built-in ports, plus possibly a second PCIe adapter.

`lsusb -tv` shows the tree. `lspci | grep USB` — how many physical controllers are available. That's the first thing you check before plugging in the twentieth hub.

I went through four topology iterations before finding a configuration that stably holds eleven cameras and fourteen servo adapters on one host without frame drops. For twenty arms with full camera coverage — you need either a second host or a PCIe expansion. That's reality, and I'd rather it surfaces here than after you've printed the twentieth set of parts.

<div align="center">
<img src="images/so101-pair.png" alt="SO-101 leader-follower pair" width="520">

*SO-101 leader (white) and follower (black). Same hardware, different roles.*
</div>

## Identification: why `/dev/ttyACM*` will destroy your system

Linux assigns serial device names in discovery order. Rebooted the machine — the order changed. Replugged one cable — `/dev/ttyACM3` became `/dev/ttyACM5`, and `/dev/ttyACM5` disappeared entirely. With three devices this is annoying. With twenty — the system is inoperable.

Identification by the hardware serial number of the CH340 USB adapter. The number is burned into the chip at the factory. Doesn't depend on port, hub, boot order. The system maintains a profile database: serial number → manipulator name, role, calibration table, bound cameras. Plugged in a cable — serial number query — profile lookup — restored in milliseconds. Unplugged — loss registered — remaining pairs keep working.

Hot-plugging. Fifteen cycles in a row, seven devices, three hubs. Zero failures. That took me three evenings debugging udev rules and race conditions between device discovery and SCS bus initialization. Boring work. But without it, twenty arms is fiction.

## Pairs, trios, swarms: how orchestration works

The classic pair: one leader, one follower. Leader is physically free — register 40 of the SCS protocol (torque) set to zero, motors don't resist. The operator moves it by hand. Every 20 ms the system reads the current position from register 56 (Present Position, 12-bit, 0–4095) and writes it to register 42 (Goal Position) of each bound follower.

But "bound" doesn't have to mean one.

One leader — N followers — one operator drives a group. Recorded one demonstration — N arms reproduce in parallel. Each pair is an independent Python thread, 50 Hz. With twenty followers that's twenty threads, each with a 20 ms read-write cycle. Python's GIL? Not a problem: pyserial releases the GIL during system I/O calls, and actual CPU contention is minimal. I measured jitter on 14 parallel threads: standard deviation of the period — 0.8 ms. Acceptable.

Two leaders — one follower — experimental configuration. One operator sets XYZ position, the second controls gripper orientation. The system sums commands with weighted coefficients. Why? Imagine a task where one person holds a part steady (position) while another adjusts the tool approach angle (orientation). Two skills, two humans, one follower.

M leaders — N followers — full mesh. Two operators, each driving their group. Groups work in parallel on one node: one drills, the other fastens. Coordination through a shared timeline: both groups synchronized by timestamp to ±1 ms accuracy (local NTP sync, no internet required).

Reassignment — on the fly. Drag-and-drop in the web interface: drag a follower from group A to group B — without stopping the others.

## Cameras: not monitoring — the eyes of the neural network

Here's where many people stumble. Cameras in this system aren't for the operator to watch a video feed. Cameras are sensory input for training.

Each follower gets one or two cameras bound to it. A camera on the gripper (end-effector view) sees what the gripper sees. A top camera (workspace view) gives spatial context. Two angles — and the neural network gets rough scene geometry without a single lidar.

Tony Zhao et al. (Stanford, 2023) showed in the ACT paper that a model trained only on joint positions reproduces trajectories but doesn't generalize — move the object two centimeters and everything breaks. Add visual input — and the model starts reacting to object position. Servo positions are the robot's configuration space. Video is the task space. Two different things. You need both.

With twenty followers and two cameras each — forty video streams. Capture through ioctl (VIDIOC_DQBUF), bypassing OpenCV, which adds 3–5 ms latency and loads CPU on decompression. With forty streams, those 3–5 ms turn into a second of lost time per cycle.

Buffering — two frames per camera (V4L2 double buffering). Synchronization — by host timestamp: each frame is matched to the nearest servo position reading (±10 ms). For training at 50 Hz, that's sufficient.

## Replay and policy: two modes, same data

First mode — record and playback. Operator moves the leader. System writes: `[timestamp, servo_positions[6], camera_frames[]]`, 50 Hz. Hit play — followers replay recorded positions point by point. No neural networks. Works three minutes after connection. Used for trajectory debugging and data collection.

Second mode — trained policy. Recorded episodes (10–50 demonstrations) become training data. Format — LeRobotDataset v3.0 (Parquet + MP4, chunked episodes, streaming via Hugging Face Hub). The model takes current positions + camera frames as input, outputs target positions for the next step.

Architectures that work with this format right now:

ACT (Zhao et al., Stanford, 2023) — predicts a "chunk" of k future actions in one iteration. This fights compounding error: with step-by-step prediction, error accumulates, and after twenty steps the manipulator is already lost. Chunking is planning several steps ahead. Training — from two hours on one GPU.

Diffusion Policy (Chi et al., Columbia, 2023) — generates actions through reverse diffusion. Gaussian noise — dozens of denoising steps — target positions. Why this complexity? Because if a task can be solved two ways (go around the obstacle left or right), behavior cloning collapses to the mean and crashes into the wall. Diffusion Policy is a generative model — it picks one option instead of smearing.

SmolVLA (Shukor et al., Hugging Face, June 2025) — compact VLA, 64 visual tokens per frame, flow matching, fine-tuning in 8 hours on one A100. Trained on open data, supports SO-101 natively.

One recording — multiple architectures — no conversion.

## Now twenty inferences in parallel

Here's a question nobody asks until they try: how do you run twenty forward passes simultaneously?

ACT with a ResNet-18 encoder and transformer decoder — one forward pass on an RTX 3090 takes about 5–8 ms. Twenty sequential inferences — 100–160 ms. That's 6–10 Hz. For teleoperation at 50 Hz — doesn't work. For autonomous execution where the policy decides every 100–200 ms — tolerable, but tight.

Parallelization through batching: collect observations from all twenty followers into one batch (batch size = 20) and run one forward pass. On GPU, batching is practically free in time: 20 inferences batched — the same 8–12 ms as one. But there's a catch: you need to synchronize frame capture from all cameras before starting inference. With forty cameras, capture time spread reaches 15 ms. If you wait for the slowest camera — you add 15 ms to every cycle.

The alternative — distributed inference. Every 4–5 followers served by a separate Jetson Orin Nano (local inference, no network overhead). One host coordinates tasks, Jetsons run inference and control their groups. This eliminates the batching problem but adds clock synchronization between hosts. PTP (Precision Time Protocol) gives ±1 ms accuracy over Ethernet — sufficient for our cycle.

I've tested both approaches. Batching on one GPU works cleaner and simpler up to ~12 followers. Beyond that — the distributed architecture is more reliable.

## Task coordination: who drills while another holds

Twenty arms isn't twenty copies of one movement. It's an orchestra. One group holds the workpiece. Another drills. A third fastens. A fourth rotates the part for the next operation.

Coordination through a sequence of policies with shared state. Each follower group executes its own policy (trained on its own demonstrations). Between groups — a completion signal: group A finished its action — trigger for group B. Currently the trigger is manual (operator confirms through the interface) or timer-based. In development — a visual trigger: the workspace camera detects scene state (part in place / hole drilled / coating applied) and launches the next group automatically.

Swappable tooling amplifies this idea. Instead of the standard gripper — quick-release adapters. HVLP spray gun (sixth servo channel controls trigger through a linkage, no heat, predictable results). Screwdriver in a clamp with silicone damper.

Multiple groups, multiple tool types, visual synchronization between them. That's what "swarm" means. Not twenty copies of one gesture.

## Calibrating twenty units: the unsolved problem

STS3215 hobby servos — ±0.3° repeatability on good days. Industrial SCARA — ±0.02°, an order of magnitude better. One unit calibrates in a minute: zero offsets, nonlinearities — recorded, compensated. Twenty units, printed on different printers with different settings (extrusion temperature ±5°C, infill ±10%), with different servo batches — that's no longer calibration. It's a statistical problem.

Option one: automatic calibration through camera and reference points. The manipulator performs a series of calibration movements, the camera records actual positions, the system computes corrections. Requires ArUco markers or equivalent, fixed camera position, 2–3 minutes per unit.

Option two: domain randomization during training. Add noise to positions and kinematic parameters during training — and the model learns to compensate for physical scatter at the policy level. Tobin et al. (OpenAI, 2017) showed this works for sim-to-real transfer. Whether it works for arm-to-arm transfer across twenty handmade units — open question.

Option three: online adaptation. The policy adjusts to a specific unit in the first few seconds of operation. Elegant on paper. In practice — requires meta-learning (MAML or analogs), and I haven't seen this in the context of cheap manipulators.

I don't know which of the three will work. Most likely — a combination of the first and second. But this is one of the reasons the project exists. The SO-101 isn't a finished product. It's a prototyping platform for problems that in five years will be solved on industrial hardware. The control interface, data format, training pipeline — that part scales. Swap the driver, recalibrate the kinematics, fine-tune the model — and under the hood sits entirely different hardware. The architecture stays the same.

## Stack

Python 3.12: FastAPI, pyserial, scservo_sdk. React 18 + Vite 6 on the frontend. V4L2 through ioctl. LeRobot — the entire ML pipeline. LeRobotDataset v3.0. Docker, docker-compose, works offline. GPU: CUDA for training, Jetson Orin Nano for distributed inference.

## What works, what doesn't

Works: auto-discovery, hot-plugging, arbitrary pairs, calibration, profiles, camera binding, record and replay on 14 followers simultaneously.

In development: batched inference on 20 followers, distributed architecture on Jetson, visual triggers between groups.

Not tested: online adaptation for scatter compensation.

Staring at that stack list — pyserial to FastAPI to LeRobot to CUDA — I realized at some point that the entire path from raw register reads to autonomous transformer inference lives in one project, and that's basically a robotics course by accident. And all of it runs on one device you can print at home and assemble in an afternoon. That's not a normal situation. Normally you'd need three different labs and five different courses to touch all of these topics. Here it's one arm and a USB cable.

So I started putting together a course. Free, no paywall — I just want more people building on this stuff. It's aimed at high school students, university undergrads, hobbyists, career switchers, whoever. I'm filming a video course and talking to a couple of universities about integrating it into their curriculum. None of that has launched yet. It's in progress.

The mechatronics part is where I'm spending the most time right now. Printing the arm, wiring the SCS bus, sending raw read/write commands to register 56 and watching the encoder value change as you physically rotate a joint. Then writing register 42 and watching the motor move to the target. Then setting register 40 to zero and feeling the arm go limp in your hand. There's something about that sequence — reading a register, writing a register, feeling the physical result — that makes embedded systems click in a way that no simulator reproduces. And then you wire the PID wrong and the joint oscillates at 10 Hz and nearly shakes itself off the table, and suddenly gain tuning isn't a textbook problem anymore.

The ML module is the other one I'm building with real depth. Recording demonstrations, formatting them into LeRobotDataset v3.0, training an ACT policy, deploying it back onto the arm, watching it fail on an object that's 3 cm to the left of where it was during training — and then understanding *why* it failed, what the attention maps are looking at, why action chunking helps with compounding error. That loop from "I moved the arm by hand" to "the arm moves by itself and picks up the right object" — that's a hard thing to create in a classroom. The SO-101 makes it cheap enough to try.

Forward and inverse kinematics, DH parameters, workspace singularities — those are in there too, but I haven't decided on the right sequencing yet. Part of me wants to start with forward kinematics as theory, part of me thinks it's better to let people break things first and derive the math from the wreckage. Computer vision covers Zhang calibration, YOLOv8 fine-tuning on custom datasets, ByteTrack. Systems engineering — USB topology, V4L2 capture, Docker, FastAPI. Those sections are more modular, people can jump in where they need to.

Anyway. I'm filming. The course will be structured as a series of project-based modules, each one ending with something that moves. You'll need one SO-101 kit, a USB camera, and a Linux machine with a GPU (or a Jetson if you want to go the edge route). I'll post updates as modules get finished.

---

*The SO-101 is an open-source arm designed by The Robot Studio in collaboration with Hugging Face, WowRobo, Seeed Studio, and PartaBot.*
