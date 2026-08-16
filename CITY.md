# ASCII CITY

A walkable 3D city rendered entirely in ASCII characters, running in the browser.
Single file (`city.html`) — vanilla JS + CSS, zero dependencies, assets, or network requests.
Just open it.

## Controls

| Key | Action |
|---|---|
| WASD | Move |
| Mouse (click to capture) or ← → | Look around |
| Mouse Y or R / F | Look up / down (full ±90°) |
| SHIFT | Run |
| SPACE | Jump |

The day/night cycle runs continuously — a full day lasts ~2 minutes.

## Systems

### World (procedural)
A 96×96 grid generated at load with a seeded RNG: roads every 16 cells (4 wide, with
dashed centerlines), 12×12 city blocks, sidewalk rings, 1–4 buildings per block
(footprint + height + window seed + optional rooftop spire). Heights fall off from a
downtown core; one 34-story landmark; some blocks become parks; the center is a plaza
with two fountains. Each cell stores a ground type (road/sidewalk/apron/park/plaza)
and the building that owns it — that grid is the entire world.

### Renderer (software raycaster)
The scene is a 152×78 character grid in a `<pre>`; each frame rebuilds the text.
For every screen column a DDA march records the buildings its horizontal ray enters.
Each screen row is then its own 3D ray — a constant angular fan around the camera
pitch (clamped at ±90°), which is what makes full up/down look work. Per row the ray
resolves against that column's building list: facade (windows are a procedural grid,
lit at night), spire needle, flat ground, or sky. Everything is drawn as ASCII with
no color data: brightness ramps (` .:-=+*#`), face-orientation shading, distance fog,
and a procedural sky (sun/moon disc, stars, dusk haze). A CSS tint (pale day → amber
dusk → blue night) is the only color, applied to the whole buffer.

### Day/night
One scalar `timeT` (0..1 = 06:00..06:00) drives sun elevation, daylight, fog density,
window lighting (hash-bucketed so windows flicker on/off over time), the sky, and
the CSS tint.

### Player
Position, yaw, pitch, and jump height (simple gravity). Movement does circle-vs-box
collision against building cells with axis sliding, so you slide along walls. Head-bob
nudges the view while walking on the ground.

### Cars (16)
State: axis, direction, lane (right-hand traffic offset), position, speed. They drive
the road grid, randomly turn left/right/straight at intersections, brake for the car
ahead (car following), and U-turn at the world edge. They render as small ASCII
sprites (`[]`, `-[]-`, `oo` headlights at night) with proper perspective and
wall occlusion.

### NPCs (14)
State: position, heading, retarget timer. They wander to random adjacent walkable
cells (sidewalks/parks/plaza — never roads or buildings) and bob between `@` and `^`
while walking. Same projection/occlusion as cars.

### Engine loop
One `requestAnimationFrame` loop: fixed-clamped `dt`, then update player → cars →
NPCs → day/night → render → HUD (clock, day counter, compass, fps). Typical frame
cost is a few milliseconds, comfortably 60 fps.
