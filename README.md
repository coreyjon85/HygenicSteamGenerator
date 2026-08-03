# Steam Generator Project
DIY Low cost low pressure culinary steam generator for steaming rice.

# Disclaimer: 
None of this is necessary. Some contactors, a few switches, some old school mechanical back-ups (over pressure/vacuum relief), maybe a few other odds and ends - but that would probably get you close. This is a dive down the rabbit hole to explore safety, control, Data in the brewery. This is excessive and no one should follow this. Also, the end goal is to make steam, and even low pressure steam can do some serious bodily harm. Be safe, be smart. 

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
SITREP:

+ The PREEMPT_RT kernel is stable.
+ The Intel I210 is operating reliably with the IgH Generic driver.
+ The EtherCAT master starts automatically via systemd.
+ The bus topology is detected correctly after a reboot.
+ The E-bus communication through the EK1100 is functioning.
+ Additional terminals are automatically enumerated.
+ Communication is occurring without packet loss.

Stage 1 Acceptance Criterion: After a complete power cycle, the controller automatically starts the EtherCAT master, binds the Intel I210 interface, enumerates the EK1100, EL1008, and EL2008, reports zero lost frames, and requires no manual intervention.

+ Tomorrows plan:
           
Test Input/Output

Some useful ethernet commands:
systemctl is-enabled ethercat.service
systemctl is-active ethercat.service
lsmod | grep '^ec_'
sudo /usr/local/bin/ethercat master
sudo /usr/local/bin/ethercat slaves

got a permissions issue with my logged in user and the ethercat commands
lets check permissions:
ls -l /dev/EtherCAT0
crw------- 1 root root 235, 0 Aug  3 15:09 /dev/EtherCAT0

gonna create a ethercat group and udev rule
sudo groupadd -f ethercat
sudo usermod -aG ethercat "$USER"
sudo nano /etc/udev/rules.d/99-ethercat.rules
KERNEL=="EtherCAT[0-9]*", GROUP="ethercat", MODE="0660"
sudo udevadm control --reload-rules
sudo systemctl restart ethercat.service

ls -l /dev/EtherCAT0
crw-rw---- 1 root ethercat 235, 0 Aug  3 16:01 /dev/EtherCAT0

reboot
check
groups
ethercat should now show up

now back to checking the EtherCAT system with: EtherCAT bus inspection commands
ethercat slaves -v
ethercat pdos
ethercat cstruct
  
```
C6015 / IgH Master
  └─ Slave 0: EK1100
      └─ Slave 1: EL1008, 8 digital inputs
          └─ Slave 2: EL2008, 8 digital outputs

EL1008: 8 input bits  = 1 byte
EL2008: 8 output bits = 1 byte

EL1008
Channel 1 → 0x6000:01
Channel 2 → 0x6010:01
...
Channel 8 → 0x6070:01

EL2008
Channel 1 → 0x7000:01
Channel 2 → 0x7010:01
...
Channel 8 → 0x7070:01
```
