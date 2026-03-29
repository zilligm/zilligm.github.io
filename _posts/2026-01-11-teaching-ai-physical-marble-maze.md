---
layout: posts
title: "Teaching an AI to Solve a Physical Marble Maze - From 3D Printing to Reinforcement Learning"
date: 2026-01-11
published: true
description: "A real-world RL project combining 3D printing, embedded firmware, computer vision, and continuous control."
---

Originally posted on LinkedIn: [Can reinforcement learning solve a physical marble maze?](https://www.linkedin.com/pulse/can-reinforcement-learning-solve-physical-marble-maze-guilherme-zilli-a6ure/?trackingId=eOztbQfERGyqg07T7AzuuA%3D%3D)

![The marble maze setup in action](/assets/images/rl-maze/maze-gif.gif)

A couple of years ago, I set myself a simple goal: Train an agent to control a physical marble maze and guide the marble from start to finish.

At the time, I had just finished a Reinforcement Learning specialization from the University of Alberta on Coursera. I had some understanding of the theory, had played with a few simulated environments, but I wanted something more concrete, something that challenged me to deal with all the messy parts that don’t show up in textbooks.

Around the same time, I had bought a 3D printer and was browsing Printables when I came across a marble maze design. It originally used knobs that you could turn by hand. I modified it to attach two small stepper motors, wrote some Arduino code, and used a joystick to move it around. Once that worked, the obvious next step was: what if a learning algorithm could do this instead of me?

That was the beginning of a project that would end up taking me three winters.

**From a toy to a real system**

What started as “a fun RL experiment” quickly turned into a full hardware-software system. The maze is entirely 3D printed. Two small stepper motors tilt it along two axes. An Arduino controls the motors and communicates with my computer over USB. The computer runs the reinforcement learning algorithm and sends back target positions for the motors.

Above the maze, I mounted a camera. I placed four colored markers in the corners of the maze and trained a small YOLO model to detect them. Using OpenCV’s projective transform, I “flatten” the camera image so it looks like a top-down square view of the maze, even if the camera isn’t perfectly aligned.

That corrected image then goes into a second YOLO model, trained to detect the marble. From that, I compute the marble’s position. The learning agent never sees the image itself, only numbers: marble position and maze angles.

All of this runs in a loop, continuously:

- Read camera
- Estimate marble position
- Decide how to tilt the maze
- Move the motors
- Repeat

**Why not just simulate it?**

I could have done this in a physics simulator. It would have been much easier. But I didn’t want something that only worked in an idealized world. I wanted to see what happens when:

- motors are slow and imprecise
- 3D printed parts are slightly loose
- vision algorithms sometimes miss the marble
- everything is delayed by a few hundred milliseconds

I wanted to feel how reinforcement learning behaves when it has to deal with the real world. That turned out to be far more challenging and far more interesting than I expected.

**Three winters of progress**

I didn’t build this in one long sprint. I worked on it mostly during the winters, which naturally split the project into three phases.

**Year 1** was all about making the system exist at all: getting the maze built, motors working, Arduino talking to the computer, and the camera seeing something usable.

**Year 2** was when things got serious. I started collecting data and training models. That’s when I realized that even “small” datasets become huge when you’re collecting state transitions from a real system. Pickle files quickly became unmanageable, so I moved everything into a MariaDB database running in Docker. That decision ended up being crucial, because it let me collect and store a lot more data from many experiments.

**Year 3** was when things finally started to come together. By then, the codebase had grown large enough that just understanding it again each winter was a challenge. Using GitHub Copilot to refactor and document things gave me back a lot of mental bandwidth. I could focus on experiments instead of fighting my own code.

**When it almost didn’t work**

There were many times when the agent looked like it was learning, only to completely fall apart later. I would see behavior improving, then diverging. I kept wondering: Is the model too small? Is the reward wrong? Is the data too noisy? Is there a bug somewhere I can’t find?

I also knew that DDPG (the algorithm I was using) has a reputation for being unstable, especially in real-world systems. More than once, I asked myself whether I was just being naïve, trying to make something work that simply wouldn’t. The hardest part wasn’t writing code, it was not knowing why something wasn’t working.

**What “working” looks like**

Last week, I finally got it to work. The agent can get the marble from the start of the maze to the goal in a small number of steps (in the best cases, as few as eight). More importantly, it can recover when it makes a bad move, adjusting the maze to bring the marble back on track. It’s not perfect. The motion isn’t always smooth, and there’s still a lot of room for improvement. But it’s real, and it’s learned.

I’m already planning the next iteration:

- faster controllers
- better motors
- smoother control
- and trying other learning models

But even in its current form, this project taught me something that simulations never did: Getting reinforcement learning to work in the real world is not about clever algorithms — it’s about engineering, data, and patience.


I’m planning to write a couple of other posts to go deeper into how the learning system itself works: how I designed the hardware, how I set up the environment and the agents, how I trained the models, and so on. I also plan to upload the code to my GitHub once I get time to polish a bit more.

If you are interested in learning more about any aspect for the system, please leave me a comment or a DM and I will do my best to cover it in the upcoming posts. Also, if you tried to implement some RL models in a real-system, I’d be glad to hear more about your experience, your biggest challenges, and how you overcame them. I would appreciate any feedback or advices to further improve my setup or try out other models.

**Project resources:**

- [Marble Maze RL repository](https://github.com/zilligm/marble-maze-rl)
- [Original printable maze model](https://www.printables.com/model/741739-snap-together-marble-maze-with-step-motor)
