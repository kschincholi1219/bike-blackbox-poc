# bike-blackbox-poc
A comprehensive Proof of Concept for a dual-camera bike dashcam system with local SD card storage, GPS tracking, and graceful power-loss recovery. Includes system architecture, storage optimization for 1-hour rolling video retention, modular software design, implementation roadmap, and validation test plans.

## 📐 Architecture Diagrams

Professional PlantUML UML diagrams are available in **[docs/ARCHITECTURE-DIAGRAMS.md](docs/ARCHITECTURE-DIAGRAMS.md)**, covering:

- **System Architecture** – Hardware/software component model (Raspberry Pi 4B, cameras, GPS, SD card)
- **Software Services** – Class/component diagram for all 5 core Python services
- **Data Flow Pipelines** – Video capture, GPS ingest, and storage management flows
- **Boot Recovery Sequence** – Startup validation and graceful shutdown lifecycle
- **Circular Buffer State Machine** – File lifecycle states and atomic deletion
- **GPS Time Synchronisation** – Boot sync, drift monitoring, and scheduled resync

Individual `.puml` files are in [`docs/diagrams/`](docs/diagrams/).

## 📚 Documentation

Full documentation index: **[docs/README.md](docs/README.md)**
