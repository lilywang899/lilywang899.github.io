---
title: Projects
permalink: /projects/
---

## Robot driver station + agent

A teleoperation system for a robot arm, split into two programs. The **driver station** runs on the operator's machine: a joystick feeds a driver event handler, which updates the GUI's control and status panels and hands commands to a robot protocol handler that sends them over UDP. The **robot agent** runs on the robot side: it receives those packets through a socket, decodes them in a message handler, and dispatches them to a service handler that talks to the robot over MQTT. Robot state flows back along the same path to the driver station's robot info panel.

[Read the project summary (PDF)]({{ '/assets/projects/arm-project-summary.pdf' | relative_url }})
