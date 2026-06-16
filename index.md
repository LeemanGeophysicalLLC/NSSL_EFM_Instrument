# NSSL EFM Instrument

The Electric Field Mapper (EFM) is a balloon-borne instrument developed by the  
National Severe Storms Laboratory (NSSL) to measure the vector electric field in the  
atmosphere during flight. It is designed to operate in severe weather conditions and  
collect high-resolution data on electric field structure, environmental parameters, and  
instrument dynamics.

The EFM consists of multiple integrated subsystems, including rotating sensor spheres,  
precision charge amplifiers, GPS tracking, orientation sensing, and onboard data  
logging. Rotating electronics handle data collection from key sensors, including charge  
transfer, GPS, temperature, pressure, and humidity. Motor control electronics regulate  
rotation rate and monitor system health, while an IMU-based orientation paddle tracks  
the instrument's motion throughout flight.

A satellite-based tracker provides real-time GPS updates for monitoring and recovery.  
Mechanical components—ranging from 3D-printed mounts to machined parts and  
lightweight foams—support and protect the system under demanding conditions.

This documentation site includes user guides, design files, firmware references, and  
mechanical documentation to support continued development, deployment, and analysis  
of the EFM system.

## Parts List
* [Parts list for subcomponents and major elements](parts_listings.md)

## Subsystem Documentation
* [Motor Control](motor_control/motor_control.md)

## Repositories

### Motor Control
* [Firmware](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Motor_Control_Firmware)
* [PCB](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Motor_Control_PCB)

### Rotating Electronics
* [Firmware](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Rotating_Firmware)
* [PCB](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Rotating_PCB)

### Orientation Electronics
* [Firmware](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Orientation_Firmware)
* [PCB](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Orientation_PCB)

### Tracker
* [Firmware](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Tracker_Firmware)
* [PCB](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Tracker_PCB)

### Analog (charge amplifier)
* [Firmware](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Analog_Firmware)
* [PCB](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Analog_PCB)

### Other
* [Documentation (this repo)](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Instrument)
* [Mechanical](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Mechanical)
