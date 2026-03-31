# Cyclone DDS Network Configuration — FBOT Multi-Machine Setup

This document describes how to configure Cyclone DDS for reliable ROS 2 Humble communication between two machines connected via Ethernet cable.

## System Overview (Current Setup)

| Machine | Role | Interface | IP Address |
|---------|------|-----------|------------|
| Intel NUC | Camera publisher (Femtobolt) | `enp86s0` | `192.168.4.100` |
| Jetson Orin AGX | Perception / processing | `eno1` | `192.168.4.102` |

Both machines run Ubuntu 22.04 with ROS 2 Humble and `rmw_cyclonedds_cpp`.

## Problem

By default, Cyclone DDS may select the wrong network interface instead of the direct Ethernet link. This causes:

- `ddsi_udp_conn_write to udp/172.17.0.1:26660 failed with retcode -1` type of error
- Camera images and point clouds never arriving on the subscriber machine
- Excessive latency

## Step 1 — Install Cyclone DDS

ROS 2 Humble ships with eProsima Fast DDS as the default middleware. Cyclone DDS must be installed separately on **both machines**:

```bash
sudo apt install ros-humble-rmw-cyclonedds-cpp
```

To verify the installation:

```bash
dpkg -l | grep cyclone
```

You should see `ros-humble-rmw-cyclonedds-cpp` and `ros-humble-cyclonedds` in the output.

## Step 2 — Increase Linux Kernel Receive Buffer

This must be done **on both machines**. Without this, Cyclone DDS cannot allocate large enough socket buffers and will silently drop large messages.

### Temporary (until reboot)

```bash
sudo sysctl -w net.core.rmem_max=2147483647
sudo sysctl -w net.core.rmem_default=8388608
```

### Permanent

```bash
sudo tee /etc/sysctl.d/60-cyclonedds.conf << EOF
net.core.rmem_max=2147483647
net.core.rmem_default=8388608
EOF
sudo sysctl --system
```

## Step 3 — Cyclone DDS XML Configuration

Create a `cyclonedds_config.xml` file on each machine. The only difference between machines is the `NetworkInterface` name.
If the interface is different on your setup, change it accordingly. The IP addresses in the `Peers` section should match the actual IPs of both machines.

### NUC (`enp86s0`)

```xml
<CycloneDDS>
  <Domain>
    <General>
      <Interfaces>
        <NetworkInterface name="enp86s0"/>
      </Interfaces>
      <AllowMulticast>false</AllowMulticast>
    </General>
    <Discovery>
      <ParticipantIndex>auto</ParticipantIndex>
      <MaxAutoParticipantIndex>120</MaxAutoParticipantIndex>
      <Peers>
        <Peer Address="192.168.4.100"/>
        <Peer Address="192.168.4.102"/>
      </Peers>
    </Discovery>
    <Internal>
      <SocketReceiveBufferSize min="10MB"/>
      <Watermarks>
        <WhcHigh>500kB</WhcHigh>
      </Watermarks>
    </Internal>
  </Domain>
</CycloneDDS>
```

### Jetson Orin AGX (`eno1`)

```xml
<CycloneDDS>
  <Domain>
    <General>
      <Interfaces>
        <NetworkInterface name="eno1"/>
      </Interfaces>
      <AllowMulticast>false</AllowMulticast>
    </General>
    <Discovery>
      <ParticipantIndex>auto</ParticipantIndex>
      <MaxAutoParticipantIndex>120</MaxAutoParticipantIndex>
      <Peers>
        <Peer Address="192.168.4.100"/>
        <Peer Address="192.168.4.102"/>
      </Peers>
    </Discovery>
    <Internal>
      <SocketReceiveBufferSize min="10MB"/>
      <Watermarks>
        <WhcHigh>500kB</WhcHigh>
      </Watermarks>
    </Internal>
  </Domain>
</CycloneDDS>
```

### What each setting does

- **`NetworkInterface name="..."`** — Restricts DDS traffic to the specified Ethernet interface, preventing it from leaking to WiFi or Docker bridge networks.
- **`AllowMulticast=false`** — Disables multicast discovery. Without this, Cyclone may attempt to reach participants on other networks (e.g., `docker0`).
- **`Peers`** — Explicitly lists both machines' IPs (including itself) for unicast discovery. The machine's own IP is required so local nodes can discover each other when multicast is disabled.
- **`SocketReceiveBufferSize min="10MB"`** — Requests a 10 MB socket receive buffer from the OS. This is critical for receiving large messages (images, point clouds) without drops. Requires the `sysctl` change from Step 1.
- **`WhcHigh=500kB`** — Increases the writer history cache high-water mark, preventing the publisher from stalling when sending bursty large data.
- **`MaxAutoParticipantIndex=120`** — Allows up to 120 DDS participants on each machine, which is necessary when running many ROS 2 nodes.

## Step 4 — Environment Variables

Add these to `~/.bashrc` on both machines:

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
export CYCLONEDDS_URI=file:///path/to/cyclonedds_config.xml
```

Replace `/path/to/cyclonedds_config.xml` with the actual path to the XML file on each machine.

**Important:** After changing the XML or environment variables, always stop the ROS 2 daemon before relaunching nodes:

```bash
ros2 daemon stop
```

The daemon caches DDS configuration from its first startup and will ignore changes until restarted.

## Verifying the Configuration

### 1. Check the config trace

Add a `<Tracing>` block inside `<Domain>` temporarily:

```xml
<Tracing>
  <Verbosity>config</Verbosity>
  <OutputFile>cyclonedds_${CYCLONEDDS_PID}.log</OutputFile>
</Tracing>
```

Then launch a node and inspect the log. Confirm:

- `selected interfaces:` shows the correct interface
- `SocketReceiveBufferSize[@min]: 10 MiB` (not `default`)
- `socket receive buffer size set to 20971520 bytes` (not `failed to increase`)
- `add_peer_addresses:` lists only `192.168.4.100` and `192.168.4.102`

### 2. Test basic connectivity

```bash
# Verify Ethernet link on each machine
ethtool enp86s0   # NUC
ethtool eno1      # Jetson
# Look for "Speed: 1000Mb/s" and "Link detected: yes"

# Ping with large packets
ping -s 65000 -M do 192.168.4.102   # from NUC
ping -s 65000 -M do 192.168.4.100   # from Jetson
```

### 3. Test DDS without ROS

```bash
# On one machine
ddsperf sanity

# On the other
ddsperf sanity
# If "mean" latency > 100000 us, there are network-level issues
```

### 4. Test ROS 2 topic transfer

Launching a camera node is the easiest way to test this. On the NUC, run your camera publisher node (e.g., `ros2 launch realsense2_camera rs_camera.launch.py`). Then on the Jetson, check the topic frequency:

```bash
# On the Jetson (subscriber side)
ros2 topic hz /camera/color/image_raw
```