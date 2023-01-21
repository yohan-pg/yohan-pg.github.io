# 21-01-2023

## Baseline config

Here is a complete description of the config we used yesterday. The main hyperparameters are:

$$
(\alpha_{lr}, \gamma_{decay}, \lambda_{time}, \lambda_{deform}, \#_{particles}, \#_{grid\_cells}, \#_{frames}) =
(1e^{-4}, 0.1, 0.0, 0.0, 10^5, 42, 100)
$$

Where $\alpha_{lr}$ is the learning rate, $\gamma_{decay}$ is the learning rate decay (shared between the voxels and the particles), $\lambda_{time}$ is the temporal regularizer weight, $\lambda_{deform}$ is the deformation regularizer weight, $\#_{particles}$ is the total number of particles if the polytope is disabled, $\#_{grid\_cells}$ is the number of grid cells in the particle acceleration structure (from which the particle radius is computed), and  $\#_{frames}$ is the number of frames.

- Training goes from $32^3$ to $300^3$ voxels over 100k iterations, upsampling progressively during the first 50k; all scenes train on 100 frames (the videos are 4 seconds at 25fps but the capture was 2 seconds at 50fps).
- The TensoRF settings are all at low values (# of voxels is half, # of features is divided by 3, MLP width is halved, # of feature tensor components is halved, image resolution is halved). This speeds up training ~2x compared to the final version. Other params like the voxel and MLP learing rates are left to default values.
- The particle radius is adjusted automatically based on the grid cell count (so a high grid cell count implies smaller particles). 
- The grid cell number and the number of particles are given for the whole scene: they are rescaled as a function of the polytope size (such that the polytope "crops out" particles but does not affect how many are placed on the object).
- The polytopes are axis-aligned cubes placed to cover the entire motion of object (how far they go outside of the objects depends on the alignment).

- The learning rate is divided by the polytope size (this compensates for the fact that our particles are stored local coordinates normalized to [-1, 1])
  
- The time regularizer is averaged over the # of frames, and multiplied by the polytope size (again, to compensate for the fact that particles are in local coordinates). At each step, it is evaluated for all particles over all frames, but an Adam step is taken for all particles across all frames. Averaging over the # of frames may be incorrect.
  
- The deformation regularizer is averaged over the # of particles. At each step, it is evaluated on the current frame only, but an Adam step is taken for all particles across all frames. Averaging over the # of particles may be incorrect, and we may need to adjust based on the frame count.

- By default TensorRF performs empty space skipping using both an occupancy grid and by dropping sampled where their integration weight is less than 1e-4. Both of these are now fully disabled within the polytope, which slows down training considerably (maybe ~2x but I'm not sure exactly), but fixed training on higher frame counts.

- The view direction component is set to 0 within the polytope, but kept outside of it (otherwise it uses it to cheat for the moving objects, at least for the sphere).

- The ray bending now skips points outside of the polytope. This speeds up training ~2x, depending on the polytope size, and gave identical results on the synthetic scene. 

- When sampling points exit the polytope after being deformed by the ray bending, their alpha is set to 0. This avoids the object duplication and is also done during training.

- In the grid, there is a hard limit of 8 particles per cell. During rendering, we increase it to 12, but this cannot be done during training (when we go over the limit, there is flickering).


## Baseline results

Here are the results for the baseline, alongside the polytopes used.


### Plant A
$\mathrm{polytope} = (0.33, (0.08,-0.05,-0.06))$

<video autoplay loop controls><source src="./baseline/plantA.mp4"></video>


### Plant B
$\mathrm{polytope} = (0.15, (-0.1012,0.1,0.0333))$

<video autoplay loop controls><source src="./baseline/plantB.mp4"></video>

### Pendulum
$\mathrm{polytope} = (0.15, (-0.11,0.04,0.04))$

<video autoplay loop controls><source src="./baseline/pendulum.mp4"></video>

### Synthetic
$\mathrm{polytope} = (0.175, (0,0,0.18))$

<video autoplay loop controls><source src="./baseline/synthetic.mp4"></video>

Since plant B has a problem, it was omitted in the followup experiments.

## Experiments

The baseline hyperparameters $(\alpha_{lr}, \gamma_{decay}, \lambda_{time}, \lambda_{deform}, \#_{particles}, \#_{grid\_cells}, \#_{frames})$ used were:

$(1e^{-4}, 0.1, 0.0, 0.0, 100^4, 40, 100)$

While for the sphere, the best hyperparameters were:

$(1e^{-4}, 0.1, 1.0, 1.0, 50^4, 28, 100)$

The goal of these experiments is to 1) solve the flickering/jiggling 2) unify both configs 3) make sure it works with regularizers, because they are needed for the modal analysis. Finally, we will want to 4) make sure that everything works with a higher # of frames.  

### 1. Stronger learning rate recay:
$(1e^{-4}, 0.01, 0.0, 0.0, 10^5, 42, 100)$

<video autoplay loop controls><source src="./1_slower/plant_A.mp4"></video>

<video autoplay loop controls><source src="./1_slower/pendulum.mp4"></video>

<video autoplay loop controls><source src="./1_slower/synthetic.mp4"></video>

### 2. Smaller learning rate:
$(5e^{-5}, 0.1, 0.0, 0.0, 10^5, 42, 100)$

**All following experiments will use this learning rate.**

<video autoplay loop controls><source src="./2_decay/plant_A.mp4"></video>

<video autoplay loop controls><source src="./2_decay/pendulum.mp4"></video>

<video autoplay loop controls><source src="./2_decay/synthetic.mp4"></video>

### 3. With both regularizers at 1.0:
$(5e^{-5}, 0.1, 1.0, 1.0, 10^5, 42, 100)$

<video autoplay loop controls><source src="./3_both/plant_A.mp4"></video>

<video autoplay loop controls><source src="./3_both/pendulum.mp4"></video>

<video autoplay loop controls><source src="./3_both/synthetic.mp4"></video>

### 4. With both regularizers at 0.1:
$(5e^{-5}, 0.1, 0.1, 0.1, 10^5, 42, 100)$

<video autoplay loop controls><source src="./4_less/plant_A.mp4"></video>

<video autoplay loop controls><source src="./4_less/pendulum.mp4"></video>

<video autoplay loop controls><source src="./4_less/synthetic.mp4"></video>

### 5. With the time regularizer only:
$(5e^{-5}, 0.1, 1.0, 0.0, 10^5, 42, 100)$

<video autoplay loop controls><source src="./5_time/plant_A.mp4"></video>

<video autoplay loop controls><source src="./5_time/pendulum.mp4"></video>

<video autoplay loop controls><source src="./5_time/synthetic.mp4"></video>

### 6. With the deformation regularizer only:
$(5e^{-5}, 0.1, 0.0, 1.0, 10^5, 42, 100)$

<video autoplay loop controls><source src="./6_deform/plant_A.mp4"></video>

<video autoplay loop controls><source src="./6_deform/pendulum.mp4"></video>

<video autoplay loop controls><source src="./6_deform/synthetic.mp4"></video>

### 7. 250 frames, unregularized:
$(5e^{-5}, 0.0, 0.0, 0.0, 10^5, 42, 250)$


<video autoplay loop controls><source src="./7_250frames/pendulum.mp4"></video>

<video autoplay loop controls><source src="./7_250frames/synthetic.mp4"></video>

### 8. 250 frames, regularized:
$(5e^{-5}, 0.0, 1.0, 1.0, 10^5, 42, 250)$


<video autoplay loop controls><source src="./7_250frames_reg/pendulum.mp4"></video>

<video autoplay loop controls><source src="./7_250frames_reg/synthetic.mp4"></video>


## TODO

Here are the things left to do for the trainings:

**HIGH RISK**
- Fix plant B; it looks like the polytope is wrong, but we need to make sure.
- Some results are failing for the sphere. I need to verify if this is because the # of particles and the # of grid cells changed or because I misconfigured something else. Yesterday increasing the # of particles seemed to work, but I had increased the # of grid cells way more. 
- Adjust the losses and the training schedule correctly as a function of the # of frames. We might want to pick a fixed duration for all animations to make this easier (but lets see how it goes). 
- Make sure we can train with an increased particle count to see how it impacts the analysis.
- Fix the flickering completely; this should not be too hard. On the sphere, it was fixed by using 50000 particles and a grid size of 28, by the deformation regularizer, and by increasing the cell limit during rendering. I'm pretty sure this can be fixed with minor tweaks; the randomization during training was implemented but never tested yet. 
  
**LOW RISK**
- Overly large polytope sizes slow down the training considerably, especially since the empty space skipping is disabled in the polytope. Possibilities include rectangular polytopes, or fixing the empty space skipping. For the latter I think the problem is that once space is marked empty, is is empty forever. I think we could keep the occupancy grid disabled, but use a probabilistic check when discarding points based on their sampling weight (because it should take very few steps to go back above the clipping threshold of 1e-4).
- Try a few more learning rate schedules since it makes such a huge difference.
- Try the final high quality config, to make sure everything is OK (it was fine for the synthetic scene).
- Make sure the training works on more scenes.

