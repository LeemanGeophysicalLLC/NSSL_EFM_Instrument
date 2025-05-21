# NSSL EFM Instrument

This repository serves as the top-level hub for the National Severe Storms Laboratory (NSSL)  
balloon-borne electric field vector measurement instrument. Documentation for the project  
is hosted here via GitHub Pages on the `gh-pages` branch. In addition, this repository  
includes links to several related repositories that contain the hardware designs, firmware,  
and mechanical components associated with the instrument.

## Motor Control
One of the paddles on the instrument houses a motor driver, which reads motor speed  
from an encoder and controls power to the motor to maintain a constant rotation rate  
of the instrument. We also monitor the current supplied to the motor and the  
temperature of the control board, logging all of this data to a rolling log file.  
This log is used for debugging and diagnosing any flight-related failures.

* [PCB Respository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Motor_Control_PCB)
* [Firmware Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Motor_Control_Firmware) 

## Paddle Orientation Measurement
In the paddle opposite the motor driver, an orientation sensor based on a simple IMU  
provides rough data on the instrument’s orientation over time. This allows analysis  
of pendulum motion and other movements of the full instrument package. The system is  
designed for easy sensor replacement, and all orientation data are logged to a  
microSD card.

* [PCB Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Orientation_PCB)
* [Firmware Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Orientation_Firmware)

## Rotating Electronics
The rotating electronics collect the majority of the scientific data during flight.  
This includes the relative orientation of the sensor spheres, electrical charge flow  
between the spheres, GPS position, temperature, pressure, and humidity. All data are  
logged to an onboard microSD card for post-flight analysis. These measurements are  
essential for understanding the electric field structure and environmental conditions  
surrounding the instrument throughout its ascent and descent.

* [PCB Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Rotating_PCB)
* [Firmware Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Rotating_Firmware)

## Analog Charge Amplifier
The analog charge amplifier performs the actual coulomb counting for the electric field  
measurement and filters the signal before passing it to the rotating electronics for  
digitization, synchronization, and logging. This board closely follows the design of  
the original instrument and includes a small microcontroller that allows temperature-  
independent control of the filter's cutoff frequency.

* [PCB Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Analog_PCB)
* [Firmware Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Analog_Firmware)

## Tracker
The tracker provides GPS location data to instrument operators via a satellite  
connection. It is designed to enable easy location of the instrument package during  
and after flight, supporting both flight termination decisions and efficient recovery  
of the full payload. The tracker uses a progressive rate strategy to optimize battery  
life, providing high-rate position updates during the most dynamic and critical phases  
of the flight.

* [PCB Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Tracker_PCB)
* [Firmware Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Tracker_Firmware)

## Mechanical
The EFM also includes a range of mechanical components that support and protect the  
instrument during flight. These include machined parts, 3D-printed housings, laser-cut  
mounts, and structural elements made from foam and composite materials. The design  
prioritizes strength, low weight, and ease of assembly in the field. Detailed models,  
drawings, and documentation are maintained to ensure consistent fabrication and  
support future revisions.

[Mechanical Repository](https://github.com/LeemanGeophysicalLLC/NSSL_EFM_Mechanical)
