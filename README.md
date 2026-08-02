# Steam Generator Project
DIY Low cost low pressure culinary steam generator for steaming rice.

# PLC control, safety monitoring, logging, and data display.
+ Using Beckoff and Ifm equipment, bring in hygienic sensor data over IO Link to EtherCAT to PLC interface. Run local control/safety loop. Transmit/Log/Display relevant information through MQTT/JSON to Home Assistant to log data in Influx DB for display in Grafana. 

# PLC Setup
+ Beckoff C6015-0010 Industrial PC.
  + Installed Ubuntu Server LTS (ensure the version supports realtime-kernel).
  + For us in 2026 - Ubuntu Pro needs 22.04 LTS  Jammy Jellyfish for real time kernel support.
  + For reference there is no clean downgrade path. If you are starting with an version after 22 verify real time kernel support has been added or reinstall the correct OS. 
  + Created Ubuntu Pro account
  + on Beckoff:
    + sudo pro attach [token]
    + sudo pro enable realtime-kernel
    + sudo reboot
