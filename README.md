# pico8-helpers

Helper functions and snippets for Pico 8 development

## Constants

Pick and choose

```lua
INT_MAX = 32767
PI = 3.1415926
HALF_PI = 1.5707963
TAU = 6.2831853
DELTA_60 = 0.0166667
DELTA_30 = 0.0333333
```

## Math Helpers

```lua
-- Trig tangent
function tan(a)
	return sin(a) / cos(a)
end

-- Sigmoid function; [0,1] -> [0,1]. Useful for animations
-- WARNING this is performance heavy
function sigmoid(t)
	return 1.01357 * (1 / (1 + 2.71828 ^ (-10 * t + 5)) - 0.006693)
end
```

## Pico 8 Helpers
```lua
-- Initialize many variables with fewer tokens
-- ex: var1, var2, var3 = us"red,green,blue"
function us(s, d)
	return unpack(split(s, d or ","))
end

-- dget with default value
function dgetd(i, d)
	local v = dget(i)
	return v != 0 and v or d
end
```

## State Machine

Implements a stack-based state machine with transparent/non-blocking states optional.

<details>
<summary>Code</summary>

```lua
State = {}
State.__index = State

function State.new(init, update, draw, transparent_update, transparent_draw)
	return setmetatable(
		{
			init = init,
			update = update,
			draw = draw,
			transparent_update = transparent_update,
			transparent_draw = transparent_draw
		}, State
	)
end

_sm_state = {}

function sm_push(state, params, on_pop)
	add(_sm_state, { state = state, on_pop = on_pop })
	state:init(params)
end

function sm_pop(result)
	local entry = _sm_state[#_sm_state]
	del(_sm_state, entry)
	if entry.on_pop then
		entry.on_pop(result)
	end
end

function sm_reset(state, params)
	_sm_state = {}
	sm_push(state, params)
end

function sm_update()
	for i = #_sm_state, 1, -1 do
		if i == 1 or not _sm_state[i].state.transparent_update then
			for j = i, #_sm_state do
				_sm_state[j].state:update()
			end
			break
		end
	end
end

function sm_draw()
	for i = #_sm_state, 1, -1 do
		if i == 1 or not _sm_state[i].state.transparent_draw then
			for j = i, #_sm_state do
				_sm_state[j].state:draw()
			end
			break
		end
	end
end
```

</details>

### Installation

1. Add `sm_update()` to your `_update()` or `_update60()` function.
2. Add `sm_draw()` to your `_draw()` function.

### Usage

Create states like:

```lua
STATE_PLAY_IDLE = State.new(
	function(self, params) -- init function, called whenever this state is PUSHED to the stack
		self.t = 0
		self.ipy = grid_py
		sfx(7)
        -- ...
	end,
	function(self) -- update function, called unless blocked
		if self.t < 1 then
			-- ...
        else
            sm_pop("optional parameter: message")
        end
	end,
	function() -- draw function, called unless blocked
		map(0, 0, 0, 0, 16, 16)
		-- ...
	end,
    false, -- optional, whether this state will block update calls for states below it on the stack
    false -- optional, whether this state will block draw calls for states below it on the stack
)
```

Interact with states:

```lua
sm_push(STATE_PLAY_IDLE) -- simple
sm_push(STATE_PLAY_IDLE, {death = true}) -- with init params
sm_push(
    STATE_PLAY_IDLE,
    {death = true},
    function(pop_return_message)
        -- do stuff immediately when this state is popped...
    end
) -- with init params and callback

sm_reset(GAME_OVER) -- clear state stack and set this as the only state
```

It's recommended to implement all your update/render logic within states and just push an initial state when the cart loads.
