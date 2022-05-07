---
layout: post
title:  "Lumenarium"
date:   2022-05-07 03:41:30 -0700
categories: [project]
---
## Reasoning
- wanted something faster than the tools we'd been using
- high degree of customizability
- 

## Architecture
This wasn't arrived at ahead of time, but through a process of discovery.

Platform Layer
- exposes a common interface to the application. To port to a new platform, just need to create a os.h and os.c file for the platform.
  - graphics are a bit of a complication. 
  - WebAssembly - can't rely on multithreading, so some of the platform details leak into the application

Engine
- Sculpture
  - map leds to 3d coordinates and map leds to output buffers
- Output
  - take leds and send them over various kinds of connections to actual led strips
  - Protocols Supported
    - Networked SACN
    - USB UART
    - Theoretically, SACN could be sent over USB and UART could be sent over the network since the output system just treats them like blobs of data with a destination.
- Patterns
  - basic animation engine - can blend between multiple patterns

Editor
- Notable feature - you can run the executable without the editor and the memory footprint decreases by almost 80%
- Only used for previewing patterns, debugging the sculpture, etc.

User Space
- a set of function prototypes that a user application can tie into to customize the application

## Usage
- Blumen Lumen, Silver Spring MD
- Incenter, Burning Man '22