I tried to follow a "recipe" for an infinite grid shader, but after hitting a bug and not being able to fix it because I didn't understand the code, I decided to write it from scratch myself so I could understand it properly.

In the process I found better implementations than the original one I was following, and better than my own.

In the end I came up with a different, simpler approach, although more limited in some aspects, that ticks all my boxes at the moment.

You can see the code [here](https://gitlab.com/ludusestars/Talos3D/-/commit/f15d291f764021ec4759a17dd601ad4448430ab1).
Here's a detailed step-by-step explanation of how it works.

> ⚠️ DISCLAMER 1:
> The goal of this article is just to document and organise my thought process while I was trying to understand the code behind another implementation.
> If you want something robust / production-worthy, you should read [this](https://bgolus.medium.com/the-best-darn-grid-shader-yet-727f9278b9d8) instead.
![](this_one_is_mine.png)

> ⚠️ DISCLAIMER 2:
> I'm using MSL (Metal Shading Language) because I like it and because my toy renderer uses Metal as the graphics API anyway, but it should be pretty straightforward to translate it to GLSL or HLSL.

> ⚠️ DISCLAIMER 3:
> This article will only focus on the shader part, but the setup in the host (CPU) side should be very simple regardless of API:
> - Set everything up to render a quad (2 triangles)
> - Enable depth testing
> - Disable face culling (we want to be able to see it from bellow too)
> - Because it's transparent it should be rendered _after_ the opaque objects
 ---
## 1. Finite version
### 1.1 Vertex Shader
```C++
static constant auto grid_size = 1.0f;
static constant float4 positions[4]{
    { -0.5, 0.0,  0.5, 1.0 },
    {  0.5, 0.0,  0.5, 1.0 },
    { -0.5, 0.0, -0.5, 1.0 },
    {  0.5, 0.0, -0.5, 1.0 } };

struct VertexOut
{
    float4 position [[ position ]] [[ invariant ]];
    float2 coords;
};

vertex
VertexOut main(
	uint id [[vertex_id]],
	float4x4 view_proj [[buffer(N)]])
{
	auto world_pos = positions[id];
	world_pos.xyz *= grid_size;

	return{
		.position = view_proj * world_pos,
		.coords = world_pos.xz };
}
```
This will create a quad of size 1x1 centred at the origin.

If we output the normalised coordinates in the fragment shader...
```C++
using FragmentIn = VertexOut;

fragment
float4 main(FragmentIn frag [[stage_in]])
{
    auto c = float4(frag.coords/grid_size + 0.5, 0.0, 1.0);
    return c;
}
```

...you'll get something like this:
![](quad.png)
### 1.2 Fragment Shader
#### 1.2.0 Modulo
> ⚠️ IMPORTANT in case you're using MSL or HLSL

[GLSL](https://registry.khronos.org/OpenGL-Refpages/gl4/html/mod.xhtml) defines its modulo function as `x - y * floor(x/y)`
which produces this type of repeating pattern:
![](glsl_mod.png)
However, both [MSL](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf) and [HLSL](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-fmod) define it as `x - y * trunc(x/y)`,
which produces this:
![](fmod.png)

Since I'm using MSL, I'll add a template function that mimics GLSL's `mod`:
```C++
template<typename T, typename U>
constexpr T mod(T x, U y) { return x - y * floor( x / y ); }
```
For HLSL older than 2021 you'll have to rely on macros:
```C
#define mod(x,y) ((x) - (y) * floor((x)/(y)))
```
#### 1.2.1 Draw the cells
We'll subdivide the grid into 2 types of cells:
- Big 1x1m cells
- Small 0.1x0.1 (sub)cells
To make it clearer to see, I'm increasing the plane size to 2x2m
```C++
static constant auto grid_size = 2.0f;

static constant auto cell_size = 1.0f;
static constant auto half_cell_size = cell_size * 0.5f;

static constant auto subcell_size = 0.1f;
static constant auto half_subcell_size = subcell_size * 0.5f;
```

We start by calculating the coordinates inside the cells and subcells.

1. First, displace the plane coordinates so the world's origin is in a corner rather than in the middle of the (sub)cell.
2. Then get the coordinates inside the (sub)cell
```C++
auto cell_coords    = mod( frag.coords + half_cell_size,    cell_size    );
auto subcell_coords = mod( frag.coords + half_subcell_size, subcell_size );
```

If we normalise and output this as a colour, we'll get something like this:

| Cell UVs                  | Subcell UVs          |
| ------------------------- | -------------------- |
| ![cell_uvs](cell_uvs.png) | ![](subcell_uvs.png) |

Next we calculate the distances in U and V to the (sub)cell's edge.
The coordinates we calculated before are in the [0, (sub)cell_size] range so:
1. First, we transform them into the [-(sub)cell_size/2, (sub)cell_size/2] range so the center of the (sub)cell has the (0,0) coordinates.
2. Then, the distance to the edge is simply the absolute of these new coordinates

```C++
auto distance_to_cell    = abs( cell_coords    - half_cell_size    );
auto distance_to_subcell = abs( subcell_coords - half_subcell_size );
```

Now it's time to draw the lines.
Comparing that distance to the (sub)cell line thickness, we determine if we should draw the line or not:
```C++
static constant auto cell_line_thickness    = 0.01f;
static constant auto subcell_line_thickness = 0.001f;

static constant auto cell_colour    = float4( 0.75, 0.75, 0.75, 0.5 );
static constant auto subcell_colour = float4(  0.5,  0.5,  0.5, 0.5 );

...

// In the fragment shader
auto color = float4(0);
if ( any( distance_to_subcell < subcell_line_thickness * 0.5 ) ) color = subcell_color;
if ( any( distance_to_cell    < cell_line_thickness    * 0.5 ) ) color = cell_color;

return color;
```

The line thicknesses are halved because only half the line is within a given (sub)cell, as the other half is in the neighbouring (sub)cell

However, this has obvious issues (I made the plane opaque and the lines yellow to make it clearer):
![](first_grid.png)
#### 1.2.2 Get uniform lines

So the problem here is that, due to perspective, the coordinates can change drastically from one fragment to another, meaning that the exact coordinates that fall within the line might be skipped from one fragment to the next.

An easy solution is to increase the line width by how much the coordinates change across fragments.
Thankfully, we get a very handy tool to get that change: partial derivatives.
If you don't know what they are or how they work, I recommend reading [this article](https://www.aclockworkberry.com/shader-derivative-functions/).

```C++
auto d = fwidth(frag.coords);
auto adjusted_cell_line_thickness    = 0.5 * ( cell_line_thickness    + d );
auto adjusted_subcell_line_thickness = 0.5 * ( subcell_line_thickness + d );

auto color = float4(0);
if ( any( distance_to_subcell < adjusted_subcell_line_thickness ) ) color = subcell_color;
if ( any( distance_to_cell    < adjusted_cell_line_thickness ) )    color = cell_color;

return color;
```

![](corrected_grid.png)

And this is how it looks with dimensions 100x100m and the actual colour:
![](corrected_grid_2.png)

### 1.3 Fade out
This already looks pretty good, but you can see issues in the distance: mainly aliasing and [Moiré patterns](https://en.wikipedia.org/wiki/Moir%C3%A9_pattern).
There's a lot of literature about how to fix these kind of issues, but I decided to take a different approach...

...just fade out the grid around the camera so you can't see them!

![](lazy_filtering.png)

First, we need the camera position.
Let's make the necessary changes in the vertex shader:
```C++
struct VertexIn
{
	float4x4 view_proj;
	float4 camera_pos;
};

struct VertexOut
{
    float4 position [[ position ]] [[ invariant ]];
    float3 camera_pos [[ flat ]];
    float2 coords;
};

vertex
VertexOut main(
	uint id [[vertex_id]],
	constant VertexIn& vert_in [[buffer(N)]])
{
	auto world_pos = positions[id];
	world_pos.xyz *= grid_size;

	return {
		.position   = vert_in.view_proj * world_pos,
		.camera_pos = vert_in.camera_pos,
		.coords     = world_pos.xz };
}
```

Note the `[[ flat ]]`: this is an attribute that disables interpolation, so we have the same value for all fragments.

After that we calculate the distance to the camera, and use that to interpolate:
```C++
static const auto max_fade_distance = 25.0f;

...

// In the fragment shader
float opacity_falloff;
{
	auto distance_to_camera = length(frag.coords - frag.camera_pos.xz);
	opacity_falloff = smoothstep(1.0, 0.0, distance_to_camera / max_fade_distance);
}

return color * opacity_falloff;
```

I've found that a 25m fade radius works pretty well at a camera height of 1m, but it's too small when the camera is very high, and too big if it's very low.
So the fade radius will now be adjusted by height, keeping that 1:25 ratio.

```C++
static constant auto height_to_fade_distance_ratio = 25.0f;

...

// In the fragment shader
auto fade_distance = abs(frag.camera_pos.y) * height_to_fade_distance_ratio;
opacity_falloff = smoothstep(1.0, 0.0, distance_to_camera / fade_distance);
```

However, if the camera gets very close to the plane, that radius will approach zero, so I added a minimum radius.
And, if the camera gets very high, you can see the shape of the quad, so I added a maximum.

```C++
static constant auto min_fade_distance = grid_size * 0.05f;
static constant auto max_fade_distance = grid_size * 0.5f;

...

// In the fragment shader
auto fade_distance = abs(frag.camera_pos.y) * height_to_fade_distance_ratio;
{
	fade_distance = max(fade_distance, min_fade_distance);
	fade_distance = min(fade_distance, max_fade_distance);
}
opacity_falloff = smoothstep(1.0, 0.0, distance_to_camera / fade_distance);
```

Et voilà!
![](final_grid.png)

---
## 2. Make it "infinite"
We're almost done, but at the moment you can (eventually) move outside of the plane.
Thankfully, it has a very easy solution, just move the plane *and the UVs* with the camera. Think of it as a moving treadmill.
![](treadmill_cat.gif)

In the vertex shader:
```C++
auto world_pos = positions[id];
world_pos.xyz *= grid_size;
world_pos.xz  += vert_in.camera_pos.xz;
}
```

An important detail is to ignore the Y component of the camera position. We only want the grid to move *horizontally*, it shouldn't change when you move vertically.

Of course, in the host you'll need to add the camera position to the buffer too.

---
## 4. Conclusions and future work
Even though there're objectively better approaches, this solution has scratched my brain itch (for now) and provides a good enough gizmo for my purposes.

The process of reinventing the wheel was a bit frustrating at times, but I'm glad I did. I wouldn't be happy having a copy-pasted piece of code that I don't understand and thus can't debug.
I learn best by doing so it was a great exercise.

Also, the process of writing this article forced me to dissect the code to be able to provide step by step examples. That uncovered a series of bugs and misunderstandings in my renderer and the shader itself. As a result, the version explained here is significantly different from the first one I wrote.

In terms of future work, there's still a bunch of stuff to be done:

- Fix the fade to black in the distance
- Actually filter the grid instead of using that hacky fade
- Colour the X/Z axis
- Add a Y axis at 0,0?

However, at the time of writing this, neither sound very fun, and I have a loong list of other features that do.
At the end of the day, this is a toy project and its sole purpose is to have fun while I experiment and learn.
![](does_not_spark_joy.png)
So I consider it done (for now).

---
## 📚 References
- [The Best Darn Grid Shader (Yet)](https://bgolus.medium.com/the-best-darn-grid-shader-yet-727f9278b9d8)
- [An introduction to shader derivative functions](https://www.aclockworkberry.com/shader-derivative-functions/)
- [GLSL's mod](https://registry.khronos.org/OpenGL-Refpages/gl4/html/mod.xhtml)
- [HLSL's fmod](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-fmod)
- [MSL spec](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf)
