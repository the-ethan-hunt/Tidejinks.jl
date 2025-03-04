# Tidejinks.jl

Hi-jinks for adding tides to ClimaOcean simulations.

`Tidejinks` provides 1) a way to download astronomical data about the position of the sun and moon at a particular time and
2) a function that will compute the tidal potential on an `Oceananigans` grid.

## Installation

Try

```julia
julia> using Pkg; Pkg.add("https://github.com/glwagner/Tidejinks.jl.git")
```

## Computing tidal potentials

Here's an example of computing the tidal potential due to the sun and moon's gravitational force
on a one-degree `LatitudeLongitudeGrid`:

```julia
using Oceananigans
using Tidejinks
using Dates
using SPICE
using GLMakie

grid = LatitudeLongitudeGrid(size=(360, 180, 1), longitude=(0, 360), latitude=(-90, 90), z=(0, 1))
Φ = Field{Center, Center, Nothing}(grid)
t = DateTime(1993, 1, 1, 1)

kernel_meta_file = "kernels.txt"
Tidejinks.wrangle_spice_kernels(kernel_meta_file)
SPICE.furnsh(kernel_meta_file)

Tidejinks.compute_tidal_potential!(Φ, t)

heatmap(Φ)
```

this produces

<img width="596" alt="image" src="https://github.com/user-attachments/assets/9b4b4233-c70f-4860-be18-fef8c2fffc08" />

Even more, this script:

```julia
using Oceananigans
using Tidejinks
using Dates
using GLMakie
using SPICE

kernel_meta_file = "kernels.txt"
Tidejinks.wrangle_spice_kernels(kernel_meta_file)
SPICE.furnsh(kernel_meta_file)

grid = LatitudeLongitudeGrid(size=(360, 180, 1), longitude=(0, 360), latitude=(-90, 90), z=(0, 1))
Φ = Field{Center, Center, Nothing}(grid)

t₀ = DateTime(1993, 1, 1, 1)
t₁ = t₀ + Day(60)
dt = Hour(3)
times = t₀:dt:t₁
Nt = length(times)

Φt = []
for t in times
    @info "Computing tidal potential at t = $t..."
    Tidejinks.compute_tidal_potential!(Φ, t)
    push!(Φt, deepcopy(Φ))
end

fig = Figure()
ax = Axis(fig[2, 1], xlabel="Longitude (deg)", ylabel="Latitude (deg)", aspect=2)
n = Observable(1)
Φn = @lift Φt[$n]
hm = heatmap!(ax, Φn)
Colorbar(fig[3, 1], hm, label="Tidal potential at Earth's surface (m² s⁻²)", vertical=false)
titlestr = @lift string("Time: ", times[$n])
Label(fig[1, 1], titlestr, tellwidth=false)

record(fig, "sixty_day_tidal_potential.mp4", 1:Nt, framerate=12) do nn
    @info "Drawing frame $nn of $Nt..."
    n[] = nn
end
```

produces

https://github.com/user-attachments/assets/25c989c3-98cc-4a28-a43c-5ec4c1f875cb

Note that the self-gravitational-attraction of the ocean changes the tidal potential significantly on Earth, but is not accounted for here.
Moreover, computing tidal amplitudes requires also carefully considering the drag between tidal velocities and the rough ocean bottom.

## Acknowledgements

`Tidejinks` uses data generated by NASA's
[Navigation and Ancillary Information Facility](https://naif.jpl.nasa.gov/naif/about.html)
via the [`SPICE.jl`](https://github.com/JuliaAstro/SPICE.jl) wrapper around NASA's
[Spice toolkit]([https://github.com/JuliaAstro/SPICE.jl](https://naif.jpl.nasa.gov/naif/index.html)).
An Nguyen and Joern Callies wrote precursor code to Tidejinks and helped with configuring the NAIF kernels.

