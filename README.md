# IMGD 4099-ST: Graphics, Simulation, and Aesthetics

This course explores digital simulations and strategies for interactive representation. Topics include:

- Parallel computing on graphics processing units (GPUs)  
- Realtime graphics techniques  
- The history of analog visuals (visual music, liquid projection, video synthesis etc.)  
- Interaction techniques for controlling digital simulations  

Students will examine and create digital simulations of real world phenonmenon, such as chemical diffusion models, models of artifical life, fluid simulations, and video feedback. The class is designed to encourage both technical and aesthetic exploration.  

All students will be expected to maintain a course website that contains links to all assignments. Videos should be posted to "permanent" locations (Vimeo, YouTube, bonus street cred for PeerTube) and linked in your course website. Details on this website can be found in the [onboarding assignment](./onboarding.md). 

Assignments and examples for this class will primarily use [WebGPU](https://cohost.org/mcc/post/1406157-i-want-to-talk-about-webgpu) / JavaScript, however, other environments and systems for GPU programming will be discussed and may be used for final projects; just talk to me about it first!

## Office hours
I will spend time on Discord answering questions asynchronously. Whenever possible, please post questions *publicly* in Discord so everyone can learn from the answers... and there's a good chance someone else in the class might be able to help with technical questions as well. My in-person office hours are from 3--5 on Tuesday, in Fuller Labs B20... please stop by! Some good reasons to stop by in-person office hours include:
- We can't debug your problems over Discord
- Brainstorming on final project / assignment ideas
- You want to talk about art / music / graphics / programming more generally
- You're interested in learning more about [my](http://charlie-roberts.com) [research](https://repl-wpi.github.io)
- You want to tell me about your research
- You'd like to sit and work a bit with me/other students nearby in case questions popup
- Anything else, really!

## Course Outline
A best guess is provided for how much time should be spent on each assignment. Some might take less time, some might take a bit more depending on how invested you become in the individual assignment. This schedule will be updated / changed on a weekly basis; consider it a rough outline of the course (especially at the start of the course!), not a contract.

### Week 1 8/30: Getting Started / GLSL Live Coding / Visual Music
- *Basic Intro to WGSL / Shader programmming* Assignment:  
    - Complete the [onboarding](./onboarding.md). Due 9/6 (2 hours).
- *Shader Live Coding and Visual Music*.  Assignments:  
    - Read / experiment with [The Book of Shaders](http://thebookofshaders.com) up through the lesson on [Noise](https://thebookofshaders.com/11/). Due 9/6 (4 hours).
    - Complete the Shader Live Coding Assignment, also due 9/6 (6 hours).  

### Week 2 9/6: Tech - Intro to WebGPU and Video synthesis history
Assignments:  
  - Read / experiment with [The Book of Shaders](http://thebookofshaders.com) up through the lesson on [Fractal Brownian Motion](https://thebookofshaders.com/13/). Due 9/13 (2 hours).
  - Complete the [WebGPU Intro / Video Processing Assignment](A2.video_processing.md). due on 9/13 (8 hours).  
  
### Week 3 9/13: Automata and Morphogenesis
- Cellular Automata  
- Complete the [Reaction Diffusion assignment](./A3.reaction_diffusion.md), due on 9/20.

### Week 4 9/20: Vertex Shaders and Particle Effects
- Go through the [Your first WebGPU app tutorial](https://codelabs.developers.google.com/your-first-webgpu-app#0), which will teach you to create a WebGPU application from the ground up, with no additional libraries. Give yourself about 4-5 hours for this, it's a long tutorial! Due 9/27
- Complete the [Particle effects assignment](./A4.particles.md)
   
### Week 5 9/27: Agent-based Simulations
- Langton's Ants   
- Physarum  
    - Complete the Physarum assignment, due on 10/4 (8 hours).
- Finalize final project idea (4 hours)

### Week 6 10/4: Interaction in the Digital Arts + Other ways to tame the GPU: CUDA / Game Engines
- OSC / MIDI
- Using mobile devices for remote control of simulations

### Week 7 10/11: Final Project Presentations &amp; Wrapup  

# Grades
Your course grade comes from three parts:

Assignments (55%)  
Final Project (35%)  
In-class assignments + attendance quizzes (10%)  

There will most likely be 4–5 assignments in the course in addition to the final project. I don't want anyone getting too far behind in the class as the technical material is sequential, but I'll do my best to help you figure out a reasonable schedule for completing your coursework if you fall behind. Get everything turned in!

# Attendance
Attendance is required... missing one class is equivalent to missing a 1/7th of the course. That said, life happens... please notify me in advance over Discord if you must absoluately must miss class so we can figure out how best to handle it.

# Academic Integrity
The goal of this class is to both create aesthetically interesting content and understand the code used to create it. In order to understand the code, you need to author it yourself. Copying and pasting code is not allowed in this class, unless explicitly stated by the instructor. If you have a question about this, please ask me.

Collaboration is encouraged in this class. There are many ways in which you can assist others without giving them code and answers. Providing low-level implemetation details (small code fragments that contribute to, but don't complete on their own, major portions of assignments) is great. For example, telling a classmate that they need to use the `smoothstep()` function in GLSL and showing them how to call it.
