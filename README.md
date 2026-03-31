# fbot_config_files

Configuration files for BORIS, including hardware device rules and DDS network setup for multi-machine ROS 2 communication.

## Contents

### `udev_rules/`

Udev rules that create stable symlinks for the robot's USB devices:

- `90-face-neck.rules` — Face and neck Arduinos (`/dev/ttyFACE`, `/dev/ttyNECK`)
- `90-shark-mb.rules` — Shark mobile base
- `99-interbotix-udev.rules` — Interbotix manipulator
- `99-sensors.rules` — Hokuyo LIDAR sensors
- `99-webcam.rules` — USB webcam

Install rules by copying them to `/etc/udev/rules.d/` and reloading:

```bash
sudo cp udev_rules/*.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### `dds_config/`

Cyclone DDS XML configurations for reliable ROS 2 Humble communication between the Intel NUC and Jetson Orin AGX over a direct Ethernet link. See [`dds_config/README.md`](dds_config/README.md) for setup instructions.

### `run.sh`

Helper script used by udev rules to invoke ROS 2 commands.
