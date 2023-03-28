---
output: github_document
---

<!-- README.md is generated from README.Rmd. Please edit that file -->



# ideal-astronomies-celestial-vanilla

<!-- badges: start -->
<!-- badges: end -->

The base code of this system is almost identical to [Ideal Astronomies](https://github.com/paezha/ideal-astronomies) but instead of black-and-white I sample colors from an astronomy photo, which are then randomized. This is the image that I use for sampling colors:
<img src="figure/fig01-1.png" alt="plot of chunk fig01" width="300px" />

The base code drew heavily from Tyler Morgan-Wall's tutorial to [rayrender Saturn](https://www.tylermw.com/tutorial-visualizing-saturns-appearance-from-earth-in-r/). The system uses [{rayimage}](https://www.rayimage.dev/) and [{rayrender}](https://www.rayrender.net/index.html) packages. Packages {here} and {glue} are used for file management:

```r
library(glue)
library(here)
library(rayimage)
library(rayrender)
```

## Generate a random seed


```r
seed <- sample.int(100000000, 1)
```

## Simulate rings

To simulate the rings I use as a template Tyler's code for processing the [texture]() of the rings of Saturn. A four-dimensional array simulates a slice of the rings, 125 pixels in width and 2048 pixels in depth (this would be the ring in the direction away from the planet). 

Saturn's rings have several subdivisions with different widths and thicknesses. The simulated rings here have three sections: first, second, and third rings (`fr`, `sr`, and `tr`, respectively), and each will have a different parameter for the transparency of the ring (coded in `alpha`). The padding and other parameters are copied from Tyler's code, where they are accurate representations of the dimensions of the rings of Saturn.

```r
set.seed(seed)

full_ring_slice <- array(1, c(125, 2048, 4))

fr <- 250 + sample.int(250, 1)
sr <- 800 + sample.int(500, 1)
tr <- 2048 - fr - sr

alpha <- c(runif(fr, 0.25, 0.45), runif(sr, 0.45, 0.95), runif(tr, 0.25, 0.35))

for(i in 1:2048){
  full_ring_slice[, i, 4] <- runif(125, 0, alpha[i])
}

half_ring_slice = render_resized(full_ring_slice, dims = c(125, 3926/2))

inc = (139826 - 66900)/(3926/2)
padding = 66900/inc
full_width = ncol(half_ring_slice)
```

This function (copied from Tyler's code) reads the slice of the ring and returns the values for color and transparency:

```r
return_texture = function(i, j, k) {
  distanceval = (sqrt((i - full_width-1)^2 + (j - full_width - 1)^2) + 1 ) * (padding + full_width)/full_width
  frac = distanceval - floor(distanceval)
  if(distanceval <= padding + full_width - 1 && distanceval > padding + 1) {
    half_ring_slice[64, distanceval - padding, k] * (1 - frac) + 
      half_ring_slice[64, distanceval + 1 - padding, k] * frac
  } else {
    0
  }
}
```

A texture matrix is initialized and the transparency is read from the slice of ring with the function `return_texture`:

```r
texture_mat <- array(1,
                     c(2 * (full_width),
                       2 * (full_width),
                       4))

for(i in 1:nrow(texture_mat)) {
  for(j in 1:ncol(texture_mat)) {
    texture_mat[i,j,4] = return_texture(i, j, 4)
  }
}

texture_mat_small = render_resized(texture_mat,
                                   mag = 0.2)
```

List of sampled colors:

```r
set.seed(seed)

color <- sample(c("#a8a18f",
                  "#d5c9a8",
                  "#e6d5a9",
                  "#f6e2b1",
                  "#eeddaf"),
                1)
```

## Render scene

The rings are the most tricky part of the code. The rest simply involves rayrendering the celestial bodies (here a "planet" and a "satellite"): 

```r
set.seed(seed)

# Choose if the rings will be white or black matter
texture_mat_small[, , 1:3] <- sample(c(0, 1), 1)

# Create the celestial model by generating a disk that will use the texture of the simulated rings
celestial_model <- disk(radius = runif(1, 2, 2.5), 
                     inner_radius = runif(1, 1.2, 1.5),
     material=diffuse(color = color, 
                      sigma = 90,
                      image_texture = texture_mat_small)) |>
  # Add the main body, that is the "planet" in this system 
  add_object(ellipsoid(a = 1,
                       c = 1,
                       b = 1,
                       material = diffuse(color = color))) |>
  # Add the second body, that is the "satellite" in this system 
  add_object(ellipsoid(a = 0.05, 
                       c = 0.05,
                       b = 0.05,
                       x = runif(1, 2, 3.5), 
                       y = runif(1, 2, 3.5), 
                       z = runif(1, 0.2, 0.5),
                       material = diffuse(color = color))) |>
  # Group the objects and rotate them at random; this is analog to changing the perspective from which the system is observed
  group_objects(angle=c(runif(1, -45, 45),
                        runif(1, 0, 90), 
                        runif(1, 0, 90)), 
                order_rotation = c(1, 2, 3))

intensity <- 50 + sample.int(50, 1)

# Take the celestial model and add a source of light
celestial_model |>
  add_object(sphere(x = sample(c(-1, 1), 1) * runif(1, 5, 15), 
                    y = sample(c(-1, 1), 1) * runif(1, 5, 15), 
                    z = sample(c(-1, 1), 1) * runif(1, 5, 15),
                    material = light(intensity = intensity))) |>
  # Render the scene and save to file
  render_scene(filename = glue("outputs/ideal-astronomy-cv-{intensity}-{color}-{seed}.png"),
               fov = 18,
               samples = 300,
               # The point of observation is chosen at random; here what changes is how close to the planet this point is
               lookfrom = c(runif(1, 15, 20), 0, 0),
               lookat = c(0, mean(celestial_model$y), mean(celestial_model$z)),
               width = 2000,
               height = 1600,
               clamp_value = 1.1,
               sample_method = "sobol")
#> --------------------------Interactive Mode Controls---------------------------
#> W/A/S/D: Horizontal Movement: | Q/Z: Vertical Movement | Up/Down: Adjust FOV | ESC: Close
#> Left/Right: Adjust Aperture  | 1/2: Adjust Focal Distance | 3/4: Rotate Environment Light 
#> P: Print Camera Info | R: Reset Camera |  TAB: Toggle Orbit Mode |  E/C: Adjust Step Size
#> K: Save Keyframe | L: Reset Camera to Last Keyframe (if set) | F: Toggle Fast Travel Mode
#> Left Mouse Click: Change Look At (new focal distance) | Right Mouse Click: Change Look At
```

<img src="outputs/ideal-astronomy-cv-62-#d5c9a8-30269429.png" alt="plot of chunk unnamed-chunk-9" width="800px" />
