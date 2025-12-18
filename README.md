# BERK Showcase

**Real-World Embedded Systems Projects Implemented in BERK**

This repository contains production-grade reference implementations demonstrating BERK's capabilities in safety-critical embedded systems, real-time control, and industrial applications.

---

## Repository Purpose

BERK Showcase serves as a collection of complete, deployable projects that illustrate:

- RTOS-semantic programming patterns at compile-time
- Hardware Abstraction Layer (HAL) integration across multiple platforms
- Sensor fusion algorithms suitable for aerospace and robotics applications
- Protocol implementations for telemetry and industrial communication
- Task-based concurrent architectures with deterministic timing guarantees

Each project is self-contained with full documentation, source code, hardware specifications, and test suites.

---

## Projects

### iAts - Intelligent Antenna Tracking System

A complete redesign of an Arduino-based antenna tracker for FPV/UAV ground stations, demonstrating migration from traditional embedded C to BERK's RTOS-semantic paradigm.

**Technical Specifications:**

| Parameter | Value |
|-----------|-------|
| Target Platform | ESP32-WROOM-32 |
| Sensor Suite | MPU6050 (IMU), HMC5883L (Magnetometer) |
| Actuators | Dual MG995 Servos (Pan/Tilt) |
| Communication | Bluetooth SPP, UART (9600-115200 baud) |
| Protocols | NMEA 0183, MAVLink v1/v2, LTM |
| Update Rate | 50 Hz control loop |
| Pointing Accuracy | < 2 degrees RMS |

**Key Implementations:**

- Extended Kalman Filter for 9-DOF sensor fusion
- PID controllers with anti-windup and derivative filtering
- Haversine-based navigation calculations
- Lock-free SPSC ring buffers for inter-task communication
- Compile-time task scheduling validation

**Documentation:**

- [Project Overview](iAts/README.md)
- [Hardware Schematic](iAts/docs/hardware_schematic.md)
- [PCB Layout Guide](iAts/docs/pcb_layout.md)
- [Performance Analysis](iAts/docs/performance_analysis.md)

---

## Architecture Principles

All projects in this repository adhere to the following design principles:

1. **Determinism**: All timing constraints are expressed and verified at compile-time
2. **Safety**: Memory safety through ownership semantics; no dynamic allocation in hot paths
3. **Portability**: HAL-based hardware abstraction enabling cross-platform deployment
4. **Testability**: Comprehensive unit and integration test coverage
5. **Documentation**: Complete technical documentation suitable for certification processes

---

## Requirements

To build and run these projects:

- BERK Compiler v0.9.4 or later
- Rust toolchain (stable)
- Platform-specific toolchain (ESP-IDF for ESP32 projects)
- Hardware as specified per project

---

## Related Resources

| Resource | Description |
|----------|-------------|
| [BERK Language](https://github.com/ArslantasM/berk) | Core compiler and language implementation |
| [BERK Test Suite](https://github.com/ArslantasM/berk-test) | Language test cases and CI infrastructure |
| [Language Documentation](https://arslantasm.github.io/berk/) | Complete language reference |
| [Standard Library API](https://arslantasm.github.io/berk-stdlib-docs/) | Stdlib function reference |

---

## Contributing

Contributions of additional showcase projects are welcome. Projects should:

- Demonstrate non-trivial embedded systems applications
- Include complete hardware and software documentation
- Provide reproducible build and test procedures
- Follow BERK coding conventions and safety guidelines

---

## License

This repository is licensed under the GNU General Public License v3.0. See [LICENSE](LICENSE) for details.

---

## Contact

For technical inquiries regarding these implementations, please open an issue in this repository or contact the maintainers through the main BERK project.
