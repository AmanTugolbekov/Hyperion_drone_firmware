# Hyperion — firmware

Flight controller and gripper firmware for the [Hyperion](https://github.com/AmanTugolbekov/Hyperion)
smart drone.

> **Status: not started.** This repository currently holds the design the
> firmware has to satisfy, not an implementation. It is published early because
> the interface it must honour is already fixed and tested — see below.

---

## What this has to do

The aircraft side of a system whose other half already exists and flies. The
[ground station](https://github.com/AmanTugolbekov/Hyperion_drone_controller_app)
is written, verified against real hardware, and will not need changing: if this
firmware honours the agreed protocol, the existing app flies it unchanged.

Four jobs:

1. **Hold the link.** Accept control frames over Wi-Fi at 25 Hz and cut the
   motors if they stop arriving.
2. **Fly.** Turn stick values into motor commands, holding altitude so the
   pilot's throttle means climb and descend rather than raw power.
3. **Grip.** Drive the gripper servo on command — open, close, and report back
   whether something is actually held.
4. **Report.** Send battery, altitude and link quality to the pilot.

## The interface

Specified in [docs/PROTOCOL.md](https://github.com/AmanTugolbekov/Hyperion/blob/main/docs/PROTOCOL.md)
in the umbrella repository. The short version:

| | |
|---|---|
| Transport | UDP, port 7099, aircraft at `192.168.1.1` |
| Control frame | 9 bytes, `03 66 roll pitch throttle yaw flags CS 99` |
| Rate | every 40 ms, continuously |
| Neutral | 128 on every axis, throttle included |
| Checksum | XOR of roll, pitch, throttle, yaw and flags |
| Camera | RTSP MJPEG on TCP 7070, path `/webcam` |

Three requirements that are easy to miss and break everything:

- **The control stream is a dead-man switch.** Stop hearing frames, stop the
  motors. This is a safety property, not a convenience.
- **Commands arrive repeated.** A flag such as take-off rides on ten
  consecutive frames because single packets are lost routinely. Act once.
- **Emergency stop must bypass every queue.** It cuts the motors immediately,
  which means the aircraft falls — correct behaviour, and the reason it is the
  only red control in the app.

## Planned layout

```
src/
├── link/        Wi-Fi socket, frame parsing, checksum, failsafe timer
├── flight/      attitude loop, altitude hold, motor mixing
├── gripper/     servo control, grip detection
├── telemetry/   battery, altitude, link quality
└── main.c
docs/
```

## Hardware target

Not yet chosen. The decision is coupled to the
[airframe](https://github.com/AmanTugolbekov/Hyperion_drone_hardware) and to
whether an existing flight stack is adapted or the control loop is written from
scratch.

Candidates under consideration: an ESP32 for its integrated Wi-Fi, or an STM32
paired with a separate Wi-Fi module for more headroom in the control loop.

## Contributing

The protocol document is the thing to read first. It records not only what the
wire format is, but how it was established and which two platform behaviours
make a correct implementation look broken.
