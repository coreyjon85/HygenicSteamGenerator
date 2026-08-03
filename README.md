# Steam Generator Project
DIY Low cost low pressure culinary steam generator for steaming rice.

# PLC control, safety monitoring, logging, and data display.
+ Using Beckoff and Ifm equipment, bring in hygienic sensor data over IO Link to EtherCAT to PLC interface. Run local control/safety loop. Transmit/Log/Display relevant information through MQTT/JSON to Home Assistant to log data in Influx DB for display in Grafana. 

# The Plan
+ Beckoff C6015: running Ubuntu Pro +PREEMPT_RT
+ IgH Ethercat Master: Controlling the fieldbus
+ Beckoff EtherCAT I/O (EL Terminals)
+ Structured IEC 61131-3 Control Logic
+ MQTT/ADS Bridge to publish process data
+ InfluxDB + Grafana: running on separate server to display dashboards
+ Home Assistant: Non-Critical monitoring & notifications

# PLC Setup
+ Beckoff C6015-0010 Industrial PC.
  + Installed Ubuntu Server 22.04 (ensure the version supports realtime-kernel).
  + Ubuntu Pro needs 22.04 LTS  Jammy Jellyfish for real time kernel support.
  + Had to disable network cloud config, and manually create a correct netplan to get ethernet working
  + Sudo apt update && upgrade
   
  + Created Ubuntu Pro account
  + on Beckoff:
    + sudo pro attach [token]
    + sudo pro enable realtime-kernel
    + sudo reboot
    + sudo apt install rt-tests
    + sudo cyclictest
    + check avg and max latency - we had 75us for both, good enough.
    + Modify the netplan:
      + Port 2: enp1s0:
      + Port 1: enp2s0:
      + disable port 2 so that it can be set for the EtherCAT master port

+ Status Checkpoint
   + Ubuntu Server 22.04
   + Ubuntu Pro
   + PREEMPT_RT Kernel
   + Max Latency" 75us
   + Intel I210 NICS x2
   + Management NIC: enp2s0
   + EtherCAT NIC: enp1s0
   
+ IgH EtherCAT Master install
     sudo apt install -y \
    git \
    build-essential \
    autoconf \
    automake \
    libtool \
    pkg-config \
    flex \
    bison \
    libxml2-dev \
    libudev-dev \
    linux-headers-$(uname -r)
     
+ Verify the headers match RT kernerl
      + ls /usr/src/linux-headers-$(uname -r)


 + Clone IgH EtherCAT master
  + cd /usr/src
  + sudo git clone https://gitlab.com/etherlab.org/ethercat.git
  + cd ethercat 

  +  git describe --tags
  +  sudo ./bootstrap
  + ./configure \
    --enable-igb \
    --enable-generic \
    --disable-8139too \
    --disable-e100 \
    --disable-e1000 \
    --disable-r8169

  + make -j$(nproc)
  + make install
  + sudo ldconfig

 + Configure the EtherCAt mASTER
 + sudo nano /etc/ethercat.conf
      MASTER0_DEVICE="enp1s0"
      DEVICE_MODULES="generic"



+Acceptance test: PASS

Service:
  ethercat.service enabled
  ethercat.service active

Modules:
  ec_generic loaded
  ec_master loaded

Master:
  Phase: Idle
  Link: UP
  Slaves: 1
  Lost frames: 0
  Frame loss: 0.0%

Slave 0:
  EK1100 EtherCAT-Koppler
  State: PREOP 



+    Beckhoff C6015
+ │
+ └── Intel I210 (enp1s0)
+    │
+    ▼
+ ┌─────────────┐
+ │ EK1100      │
+ ├─────────────┤
+ │ EL1008      │
+ ├─────────────┤
+ │ EL2008      │
+ └─────────────┘

```
           C6015
                       │
          EtherCAT Master (IgH)
                       │
        ┌──────────────┴──────────────┐
        │                             │
     EK1100                      AL1333
        │                             │
        │                      IO-Link
        │                             │
   EL1008 EL2008              Sensors
                               │
                          Pressure
                          Temp
                          Level
                          Flow
```
We've demonstrated:

The PREEMPT_RT kernel is stable.
The Intel I210 is operating reliably with the IgH Generic driver.
The EtherCAT master starts automatically via systemd.
The bus topology is detected correctly after a reboot.
The E-bus communication through the EK1100 is functioning.
Additional terminals are automatically enumerated.
Communication is occurring without packet loss.

Stage 1 Acceptance Criterion: After a complete power cycle, the controller automatically starts the EtherCAT master, binds the Intel I210 interface, enumerates the EK1100, EL1008, and EL2008, reports zero lost frames, and requires no manual intervention.

+ Tomorrows plan:
           
Test Input/Output
  
