---
layout: post
title:  "The Explorer Lab"
date:   2022-05-07 03:41:30 -0700
categories: [project]
---
![Image Alt Text](/assets/images/explorerlab_hero.jpeg)
We were approached by Verizon Innovative Learning to design and build an experience that would get low-income middle school students excited about careers in STEM. A team proposed the idea of a "real life magic school bus", "a field trip that comes to your school". I joined when it came time to design and build the game and bus.

## Design
![Image Alt Text](/assets/images/explorerlab_00.jpg)
The game consisted of 2 systems, and several phases. The bus has 16 TV screens instead of windows - these are controlled by a single PC located at the back of the bus. We used these to provide the illusion of flying throughout the solar system, finally arriving at Mars. During the flight we introduce the idea of "the search for life" and why its important. On landing, NASA calls the bus saying they've lost the Curiosity rover and need the kids to go find it. At this point each student gets a tablet to design their own rovers. Once designed, the rover designs are sent over a local network to the PC at the back of the bus. This allows the kids rovers to drive off their tablet screens and out the windows into the Martian terrain around them.

The meat of the experience is the actual search for Curiosity. Students repeatedly encounter challenges, they formulate solutions, modify their rovers to meet the new specifications (this is their super power in the experience) and then test their new rovers against the harsh terrain of Mars. Along the way, they have opportunities to learn about the planet, the research efforts we've undertaken, and how some things we've learned on Mars have taught us about our home planet, Earth. (These were largely informed by an absolutely inspiring collaboration with Tanya Harrison, who came out and worked with us for a few weeks to flesh out the science content in the game)

Various elements of the game required building real time networked systems (as students collect evidence, it shows up on the windows), and heavily relied on performant graphical systems (rendering 16 4k videos at 30fps for the introductory flight to Mars, not to mention the martian landscape was a real time simulation). Building and maintaining these systems was one of my primary responsibilities, along with implementing much of the tablet rover engineering game.

## Technical Setup

## Ms. Frizzle's Mechanic
![Image Alt Text](/assets/images/explorerlab_01.jpg)
We worked with West Coast Customs (of Pimp My Ride fame) to build the bus and install the game into it. Because of my familiarity with the game's system requirements, I was in the shop with them almost every day for 6 months, and ended up specifying, sourcing, and installing much of the buses electrical systems (pictured above).
In the end the bus consisted of:
- 16 4k television screens for windows
- 4k individually addressable leds for ambience, controlled by the game
- A Windows PC to run the Mars simulation
- 16 Android tablets, networked to the PC to run the rover engineering game on
- 5 Web Cameras, mounted outside the bus so that the TV screens could operate like actual windows at the start of the experience
- 7.1 Channel Surround Sound with subwoofers under the students seats for maximum "liftoff" rumble

## Long Term Support