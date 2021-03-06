# Software implementation

CHON is coded completely in C++. Its foundation is built on the Allolib multimedia library, developed by the Allosphere Research Group at UCSB and runs on Linux, MacOS, and Windows. CHON can simulate the physics of networks of up to 100 coupled harmonic oscillators moving in up to 3 dimensions. CHON can sonify the oscillator network with its internal audio engine, but it is designed to be part of a modular workflow, providing control data to external applications.

## Libraries

Much of CHON's low-level functionality is provided by the Allolib library[@allosphere_research_group_allolib_2021], which in turn is powered by various other C++ libraries. These libraries wrap many functions that provide access to the system CHON is running on. These libraries are cross-platform, which allows CHON to be built on Linux, MacOS, or Windows, without needing to account for idiosyncrasies in those operating systems in the code of CHON itself.

Allolib is a C++ library developed by the Allosphere Research Group at UCSB. It provides a framework for cross-platform development of interactive multimedia applications and tools. The features of Allolib itself that CHON primarily makes use of is its structure for initializing an application and running audio and video threads (allowing for communication between threads), its management of parameters, and its convenient wrapping of low-level functions and libraries.

The audio input and output streams are handled by the RtAudio library[@scavone_rtaudio_nodate]. RtAudio provides an API for cross-platform, real-time audio input and output. Window creation and graphics API are provided by OpenGL and GLFW[@noauthor_glfw_nodate]. Interaction with the physical simulation (i.e. the ability to drag the particles on screen with the mouse) was provided by Tim Woods' "pickable" classes in Allolib. The GUI is provided by the ImGui (Dear ImGui) library by Omar Cornut[@cornut_imgui_2021]. Some synthesis classes such as basic sine oscillator and reverb were borrowed from the Gamma library.

## Physics computation

The heart of CHON is a real-time and interactive simulation of a mass-spring system. There are many solutions to the mass-spring system suitable for different purposes, but for CHON to be both real-time and interactive, I needed to use a numerical method for solving the differential equation of motion for each particle at each call of a synchronous thread. 

The physics simulation of CHON employs a forward Euler method. This method was chosen for its simplicity and relatively low computational cost. The forward Euler method solves numerical integration using an explicit method. This means that the state of the system at every time step of the simulation is calculated from the state at the previous time step. Essentially, the forward Euler method is an efficient and simple way to quantize the physics involved in CHON so that it can be calculated at every step in the simulation.

The equation of motion for each particle is:

$$ v_{now} = \frac {v_{prev} + \frac {\vec F_{net}} {m} - bv_{prev} } {h} $$ 

Where $v_{now}$ is the velocity or amount to move the particle this frame, $v_{prev}$ is the velocity of the particle in the previous frame, $\vec F_{net}$ is the net force on the particle from neighboring particles, $m$ is the mass of the particle, $b$ is the damping constant, and $h$ is the frames per second of the simulation.

The net force on a particle $\vec F_{net}$ is calculated every frame based on the difference in the displacement between the particle and each of its neighbors. This difference in displacement is equivalent to the amount the spring is stretched or compressed relative to its equilibrium state. The equation for $\vec F_{net}$ on a particle looks like:

$$ \vec F_{net[x]} = k(x_2 - x_1) $$
$$ \vec F_{net[y]} = k(y_2 - y_1) $$
$$ \vec F_{net[z]} = k(z_2 - z_1) $$

For efficiency, the opposite of this force is immediately added to the neighboring particle. This effectively halves the number of times this $\vec F_{net}$ value needs to be calculated because each particle only needs to calculate the force from the next particle, not the previous one.

I made a *Particle* class to keep track of parameters relating to each individual particle such as 3D arrays for velocity, acceleration, equilibrium position, and current displacement from equilibrium. When a particle is created, the equilibrium position is set according to how many particles there are in the CHON system. All other physical parameters are initialized to zero.

The physics simulation of CHON takes place in the visual thread. Since the visual thread runs at 60 frames per second, this means that the step size of the simulation is 1/60th of a second (16.7ms). Moving the simulation to a higher frequency thread, such as the audio thread, would increase the accuracy of the simulation by reducing the step size, but the computation cost would increase proportionally. This would limit the feasible complexity of the CHON system so that only a few particles could be simulated consistently in real-time. Conversely, increasing the step size would allow for greater complexity in the CHON system (more particles), but reduce the simulation accuracy. It is absolutely essential that CHON be real-time, so the balancing factors are the number of particles CHON can handle vs. the accuracy of the simulation. The 16.7ms update rate of the visual thread was deemed an appropriate compromise, allowing for smoothly simulating up to 100 particles on my laptop.

The fact that the audio thread, which updates at a much higher rate, uses this data, caused some audible quantization noise in early experiments. For this reason, I implemented a basic smoothing class called *SmoothValue*, which acts like a low-pass filter. This allows the audio thread to smoothly change the synthesis control data from the visual thread between visual thread calls.

## Sound engine

CHON's internal sound engine serves to quickly and easily sonify the physical simulation. Though more powerful and flexible sound design is possible by pairing CHON with an external audio application, the internal sound engine in CHON does create some fairly compelling sonic results. There are two main paradigms under which the sound engine operates: *Continuous* and *Trigger*.

The continuous paradigm is represented by the "additive synth" in the sound engine. When the particle count is 1, this synth generates a sine tone at the user selected fundamental frequency. The synth generates an  additional sine tone at integer multiples of that fundamental for each particle beyond 1, so each particle has a sine tone associated with it. By default, this synth simply plays a harmonic spectrum. If the user couples the FM or AM engine of the additive synth to one of the 3 axes (x, y, or z), then the sound becomes more interesting. The AM coupling ties the amplitude of the sine tone of each particle to the displacement of that particle (in whatever axis is chosen). The FM coupling ties the width of the FM effect to the displacement of the particle.

Both of these couplings work the same in the way they couple to the particles movement. Each particle's displacement is stored as a public variable in the Particle class. This is updated on each visual frame, and is read by the audio thread if the AM or FM coupling is active. If the particle moves in an axis that is not coupled, it will not affect the sound. For example, if AM is coupled to the y axis, but the particle is moving in the x axis, then there will be no audible effect. On the other hand, if AM is coupled to the x axis, and the particle moves in the x axis, then the amplitude will gradually increase and decrease with the particle's displacement. This is the reason for calling it the "continuous" paradigm.

Each particle has a sine wave for their own FM synthesis, the modulator wave. The value of this wave is added to the particle's main oscillator's frequency (the carrier wave) at every sample. The modulator wave's frequency is equal to the particle's carrier frequency multiplied by a user-defined ratio. The amplitude of the modulator wave is determined by a user-defined value, and is additionally modulated by the movement of the particle. This way, the user controls the intensity of the effect of the particle's movement on the sound. If the user-defined FM width is set to zero, then the frequency of the particle will rise and fall with the particle's displacement, rather than at the FM frequency.

The AM synth is much simpler. The amplitude of the particle's main oscillator is multiplied by the displacement of the particle. If the particle is at rest, then the amplitude is zero and there is no sound. When the particle is moved from equilibrium, the amplitude increases. In this way, the activity of the synth is tied to the activity of the physical simulation.

The trigger paradigm is represented by the bell synth. The bell synth is a sine tone tuned according to user-defined settings, with an amplitude envelope governed by a 1-second ADSR. The ADSR envelope is triggered whenever the particle crosses its equilibrium point in the selected axis. This allows cascading effects where each particle triggers its bell sound and pushes the next particle to trigger its bell sound and so on.

## OSC

OSC (Open Sound Control) can be used to send control data between audio applications[@wright_matt_opensoundcontrol_nodate]. In CHON, OSC allows for the displacement of each particle to be sent out to control parameters on another audio application. In actuality, the data can be used for any purpose.

The OSC output of CHON can be turned on or off by the user. OSC messages are broadcast in the format: 
\begin{center}

/dispX/NUM x

\end{center}

Where NUM is the number of the particle and x is the displacement in the x axis. The displacement in all 3 axes can be sent out simultaneously. For the displacement in the y and z axes, the format is "/dispY/NUM y" and "/dispZ/NUM z", respectively.

The address and port over which CHON broadcasts the OSC data can be changed by the user at will. When these settings are changed, the OSC server is restarted.

Some examples of using CHON to control external audio applications are discussed in Chapter 6.
