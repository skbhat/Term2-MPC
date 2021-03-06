# CarND-Controls-MPC

<p align="center">
  <img src="./images/rep_im.jpg" alt="car_img">
</p>

MPC differs from PID in two ways:

1. In PID project we did not have the future trajectory of the path
2. In PID project we did not use model for vehicle dynamics

With the above two additions we can compute the control parameters by predicting the vehicle position in future.  The overall framework is as folows:

1. Fit a 3rd degree polynomial to the provided points in the trajectory ahead.
2. Use the polynomial to compute cross track error and orientation error of the present state
3. Starting from the present state of the vehicle, predict the furture error as a function of cotrol parameters
4. Compute the actual values for control parameters by minimizing the error.

Following set of equations are used for predicting vehicle state:
<p align="center">
  <img src="./images/eqn1.jpg" alt="car_img">
</p>

In step(3), we have to take care of the latency.  We have to start our computation from the state in which vehicle will be after latency.  According to my understanding the future position and cross track error must be compute as follows:

double lt_x = v*latency*cos(A);

double lt_y = v*latency*sin(A);

double lt_cte = cte + v*sin(A)*latency;

where A is the difference between current orientation of the vehicle and the orientation of the trajectory. But in practice it did not work as well as the equations used in [this](https://github.com/awbrown90/CarND-MPC-Project).  Hence I used those equations (while being unclear about why it should work better).

## Final values of parameters and weight of cost functions

Final values are as follows:

```c++
size_t N = 10;
double dt = 0.1;
double ref_v = 60;

// Minimize cross track error and try to maintain a reference speed.
for (unsigned int t = 0; t < N; t++) {
      fg[0] += 5000*CppAD::pow(vars[cte_start + t], 2);
      fg[0] += 5000*CppAD::pow(vars[epsi_start + t], 2);
      fg[0] += CppAD::pow(vars[v_start + t] - ref_v, 2);
}

// Minimize the use of actuators.
for (unsigned int t = 0; t < N - 1; t++) {
	fg[0] += 5*CppAD::pow(vars[delta_start + t], 2);
	fg[0] += 5*CppAD::pow(vars[a_start + t], 2);
}

// Minimize the value gap between sequential actuations.
for (unsigned int t = 0; t < N - 2; t++) {
	fg[0] += 50*CppAD::pow(vars[delta_start + t + 1] - vars[delta_start + t], 2);
	fg[0] += 50*CppAD::pow(vars[a_start + t + 1] - vars[a_start + t], 2);
}

```
## How I arrived at these values
I try to obtain a sufficently long predicted path (green curve shown in the simulator) which aligns well with the actual path.  A 'misfit' occurs when the predicted path deviates from the actual path.  Ideally, it is better to have dt as small as possible.  But when dt becomes smaller the time horizon (N*dt) becomes smaller and the foresight we have reduces.  In order to increase the time horizon, if we increase N, then it adds computational cost and may not execute the controls within the given time.  Moreover, since there is a latency of 0.1s, any dt<0.1 will lead to predictions which do not take into account the effect of previous actuation.  Hence I experimented with dt values in range 0.1 to 0.2.  For a fixed value of dt, I iterated through the following steps.

1. If there is any misfit at the beginning of the trajectory then increase the weight of Cross Track Error component.
2. If there is any misfit at the sharp corners, then decrease dt
3. If the length of the predicted trajectory is less then increase N (vice-versa)
4. If the vehicle is moving comfortably, then increase reference velocity ref_v

The vehicle passes the test even with ref_v = 100.  But I kept it at 60 because in the past it failed the test in the reviewer's simulator.

---

# Original Readme
## Dependencies

* cmake >= 3.5
 * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets 
    cd uWebSockets
    git checkout e94b6e1
    ```
    Some function signatures have changed in v0.14.x. See [this PR](https://github.com/udacity/CarND-MPC-Project/pull/3) for more details.
* Fortran Compiler
  * Mac: `brew install gcc` (might not be required)
  * Linux: `sudo apt-get install gfortran`. Additionall you have also have to install gcc and g++, `sudo apt-get install gcc g++`. Look in [this Dockerfile](https://github.com/udacity/CarND-MPC-Quizzes/blob/master/Dockerfile) for more info.
* [Ipopt](https://projects.coin-or.org/Ipopt)
  * Mac: `brew install ipopt`
       +  Some Mac users have experienced the following error:
       ```
       Listening to port 4567
       Connected!!!
       mpc(4561,0x7ffff1eed3c0) malloc: *** error for object 0x7f911e007600: incorrect checksum for freed object
       - object was probably modified after being freed.
       *** set a breakpoint in malloc_error_break to debug
       ```
       This error has been resolved by updrading ipopt with
       ```brew upgrade ipopt --with-openblas```
       per this [forum post](https://discussions.udacity.com/t/incorrect-checksum-for-freed-object/313433/19).
  * Linux
    * You will need a version of Ipopt 3.12.1 or higher. The version available through `apt-get` is 3.11.x. If you can get that version to work great but if not there's a script `install_ipopt.sh` that will install Ipopt. You just need to download the source from the Ipopt [releases page](https://www.coin-or.org/download/source/Ipopt/) or the [Github releases](https://github.com/coin-or/Ipopt/releases) page.
    * Then call `install_ipopt.sh` with the source directory as the first argument, ex: `sudo bash install_ipopt.sh Ipopt-3.12.1`. 
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [CppAD](https://www.coin-or.org/CppAD/)
  * Mac: `brew install cppad`
  * Linux `sudo apt-get install cppad` or equivalent.
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/self-driving-car-sim/releases).
* Not a dependency but read the [DATA.md](./DATA.md) for a description of the data sent back from the simulator.


## Basic Build Instructions


1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.

## Tips

1. It's recommended to test the MPC on basic examples to see if your implementation behaves as desired. One possible example
is the vehicle starting offset of a straight line (reference). If the MPC implementation is correct, after some number of timesteps
(not too many) it should find and track the reference line.
2. The `lake_track_waypoints.csv` file has the waypoints of the lake track. You could use this to fit polynomials and points and see of how well your model tracks curve. NOTE: This file might be not completely in sync with the simulator so your solution should NOT depend on it.
3. For visualization this C++ [matplotlib wrapper](https://github.com/lava/matplotlib-cpp) could be helpful.

## Editor Settings

We've purposefully kept editor configuration files out of this repo in order to
keep it as simple and environment agnostic as possible. However, we recommend
using the following settings:

* indent using spaces
* set tab width to 2 spaces (keeps the matrices in source code aligned)

## Code Style

Please (do your best to) stick to [Google's C++ style guide](https://google.github.io/styleguide/cppguide.html).

## Project Instructions and Rubric

Note: regardless of the changes you make, your project must be buildable using
cmake and make!

More information is only accessible by people who are already enrolled in Term 2
of CarND. If you are enrolled, see [the project page](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/f1820894-8322-4bb3-81aa-b26b3c6dcbaf/lessons/b1ff3be0-c904-438e-aad3-2b5379f0e0c3/concepts/1a2255a0-e23c-44cf-8d41-39b8a3c8264a)
for instructions and the project rubric.

## Hints!

* You don't have to follow this directory structure, but if you do, your work
  will span all of the .cpp files here. Keep an eye out for TODOs.

## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to we ensure
that students don't feel pressured to use one IDE or another.

However! I'd love to help people get up and running with their IDEs of choice.
If you've created a profile for an IDE that you think other students would
appreciate, we'd love to have you add the requisite profile files and
instructions to ide_profiles/. For example if you wanted to add a VS Code
profile, you'd add:

* /ide_profiles/vscode/.vscode
* /ide_profiles/vscode/README.md

The README should explain what the profile does, how to take advantage of it,
and how to install it.

Frankly, I've never been involved in a project with multiple IDE profiles
before. I believe the best way to handle this would be to keep them out of the
repo root to avoid clutter. My expectation is that most profiles will include
instructions to copy files to a new location to get picked up by the IDE, but
that's just a guess.

One last note here: regardless of the IDE used, every submitted project must
still be compilable with cmake and make./
