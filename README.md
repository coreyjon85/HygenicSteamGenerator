# Steam Generator Project
DIY Low cost low pressure culinary steam generator for steaming rice.

# PLC control, safety monitoring, logging, and data display.
+ Using Beckoff and Ifm equipment, bring in hygienic sensor data over IO Link to EtherCAT to PLC interface. Run local control/safety loop. Transmit/Log/Display relevant information through MQTT/JSON to Home Assistant to log data in Influx DB for display in Grafana. 

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

# The Plan
+ Beckoff C6015: running Ubuntu Pro +PREEMPT_RT
+ IgH Ethercat Master: Controlling the fieldbus
+ Beckoff EtherCAT I/O (EL Terminals)
+ Structured IEC 61131-3 Control Logic
+ MQTT/ADS Bridge to publish process data
+ InfluxDB + Grafana: running on separate server to display dashboards
+ Home Assistant: Non-Critical monitoring & notifications
