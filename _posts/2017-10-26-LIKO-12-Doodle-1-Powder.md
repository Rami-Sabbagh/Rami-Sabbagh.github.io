---
title: "LIKO-12 Doodle #1 - PowderV1.1"
categories:
  - LIKO-12
  - Doodles
  - Lua
  - LÖVE

tags:
  - LIKO-12
  - Doodle
  - Powder
  - Sandbox
  - Lua
  - LÖVE
---

(The Powder Doodle GIF)

_A powder sandbox in LIKO-12_

Creating a powder sandbox in LIKO-12, is not hard at all, it only took me 2 hours to write the whole doodle.

First, I had to find a way to store the powder canvas, and for this I used an _imagedata_, but you may be asking, what an imagedata is ?

---

The LIKO-12 GPU offers 2 amazing features called _images_ and _imagedatas_:

- **Images:** They are like sprites (Sprites are actually images internaly), that you can directly draw to the screen, but without being able to edit them.

- **ImageDatas:** They are the _data_ of an image that you can read pixels from, set pixels to, encode, export to a png, etc.., But without the ability to draw them.

---

So here I want to create an image with size of the screen, but leaving some space at the buttom for a tool bar, I can simply achive this by a single call !

```lua
local sw, sh = screenSize() --Returns the size of the screen, so you won't ever have to remember the resolution of LIKO-12 screen :P
local cw, ch = sw, sh-8 --The powder canvas size, with 8 free pixels from the bottom for the toolbar.

local cimg = imagedata(cw,ch) --The imagedata of the powder canvas, we can easily create one by a single call.
```

And creating a particle is easy then

```lua
---local sw, sh = screenSize()
---local cw, ch = sw, sh-8

---local cimg = imagedata(cw,ch)

local parts = {} --A list of particle to update/move.

--Particle X, Particle Y, Particle Color
local function createParticle(x,y,c)
	if x < 0 or y < 0 or x > cw-1 or y > ch-1 then return end --Out of bounds.
	cimg:setPixel(x,y,c) --Set the pixel in the powder canvas.
	parts[#parts+1] = {x,y} --This way is faster than table.insert()
end
```

---

Next I have to hook the createParticle with the mouse, but wait, what about the mobile devices with multitouch ??
I can simply handle this by storing each touch position in a table, and simulating the mouse as a touch on desktops.

```lua
Controls("touch") --I have to set the controls type to touch for it to work ! (It defaults to the controllers).
cursor("point") --Set the mouse cursor, 'point' is one of the default DiskOS cursors, note that the cursor is not visible on mobile.

---local sw, sh = screenSize()
---local cw, ch = sw, sh-8

---local cimg = imagedata(cw,ch)

---local parts = {}
local touch = {} --The touches table.

---local function createParticle(x,y,c)
---  if x < 0 or y < 0 or x > cw-1 or y > ch-1 then return end
---	cimg:setPixel(x,y,c)
---	parts[#parts+1] = {x,y}
---end

--Touch Events--
--They should be made globals for LIKO-12 to call them.

--Called when the screen is touched.
function _touchpressed(id,x,y)
	touch[id] = {x,y}
end

--Called when a touch moves.
function _touchmoved(id,x,y)
	touch[id][1] = x
	touch[id][2] = y
end

--Called when a touch is released.
function _touchreleased(id,x,y)
	touch[id] = nil
end

--Mouse Events--
--Note: LIKO-12 simulates touch as a mouse by default.

--Called when a mouse button is pressed
function _mousepressed(x,y,button,istouch)
	if istouch then return end --We already handle touch events.
	if button ~= 1 then return end --I only want to handle the left mouse button.
	_touchpressed(0,x,y)
end

--Called when the mouse moves.
function _mousemoved(x,y,dx,dy,istouch)
	if istouch then return end --We already handle touch events.
	if not touch[0] then return end --Because the mouse button is not held down.
	_touchmoved(0,x,y)
end

--Called when a mouse button is released
function _mousereleased(x,y,button,istouch)
	if istouch then return end --We already handle touch events.
	if button ~= 1 then return end --I only want to handle the left mouse button.
	_touchreleased(0,x,y)
end
```

Now I will create a ticks system and call createParticle in it.

```lua
---Controls("touch")
---cursor("point")

---local sw, sh = screenSize()
---local cw, ch = sw, sh-8

---local cimg = imagedata(cw,ch)

---local parts = {}
---local touch = {}

---local function createParticle(x,y,c)
---  if x < 0 or y < 0 or x > cw-1 or y > ch-1 then return end
---	cimg:setPixel(x,y,c)
---	parts[#parts+1] = {x,y}
---end

local function updateTouch()
	for touchid, pos in pairs(touch) do
		createParticle(pos[1],pos[2],7) --Create white particles
	end
end

local function tick()
	updateTouch()
end

--Hook the tick function with LIKO-12 update
function _update(dt) --The delta time between the update calls.
	tick()
end

--Touch Events--
--[[
...
]]

--Mouse Events--
--[[
...
]]
```

Let's draw the powder canvas now

```lua
--[[
---Controls("touch")
---cursor("point")

...

---local function tick()
---	updateTouch()
---end
]]

--Draws the powder canvas.
local function drawSandbox()
	cimg:image():draw(0,0) --Convert it to an image and draw it, it works superfast, even on mobile.
end

---function _update(dt)
---	tick()
---end

function _draw()
	clear() --Clear the pervious screen.
	drawSandbox()
end

--Touch Events--
--[[
...
]]

--Mouse Events--
--[[
...
]]
```

---

It's time for a test run !

(GIF 1)

---

After that I had to write the particles movement function, and call it in tick:

```lua
--[[
Controls("touch")

...

local function updateTouch() ... end
]]

local function updateParticle(x,y)
	local pcol = cimg:getPixel(x,y) --The particle color
	if cimg:getPixel(x,y+1) == 0 then --If the pixel under the particle is already filled
```
(IMAGE 1)
```lua
		--Move the particle
		cimg:setPixel(x,y+1,pcol):setPixel(x,y,0)
			
		--Return the new position
		return x,y+1
		
	else --Otherwise check the left and right pixels under the particle
	
		if cimg:getPixel(x-1,y+1) == 0 then --Check if the particle can move to the left pixel under it.
```
(IMAGE 2)
```lua
			--Move the particle
			cimg:setPixel(x-1,y+1,pcol):setPixel(x,y,0)
			
			--Return the new position
			return x-1,y+1
			
		elseif cimg:getPixel(x+1,y+1) == 0 then --If not then heck if the particle can move to the right pixel under it.
```
(IMAGE 3)
```lua
			--Move the particle
			cimg:setPixel(x+1,y+1,pcol):setPixel(x,y,0)
			
			--Return the new position
			return x+1,y+1
			
		end
	end
end

local function updateSandbox()
	local nparts, tid = {}, 1 --The particles that are still moving.
	for i=1,#parts do
		local nx, ny = updateParticle(unpack(parts[i]))
		if nx then
			nparts[tid] = {nx,ny}
			tid = tid + 1
		end
		--If a particle didn't move then it's in a static state -> there's no longer a need to update/move it !
	end
	parts = nparts
end

local function tick()
	updateTouch()
	updateSandbox()
end

--[[
function _update() ... end

....

End-Of-File
]]
```