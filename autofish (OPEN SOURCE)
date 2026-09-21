-- AutoFish for Matcha (external Roblox LuaVM)  cast / shake / reel state machine.
-- F1 toggles it. GUI state is read via memory offsets; the game is driven with input only.

local Players = game:GetService("Players");
local RunService = game:GetService("RunService");

-- Current LocalPlayer (re-fetched each call so it survives respawns/rejoins).
local function getLocalPlayer()
	return Players.LocalPlayer;
end;

-- Let Matcha's injected input reach the game, and start from a clean mouse state.
if type(setrobloxinput) == "function" then
	setrobloxinput(true);
end;
pcall(function()
	mouse1release();
end);

-- The GUI offset math below needs memory reads.
if type(memory_read) ~= "function" then
	notify("Enable Unsafe LuaU in Matcha \226\128\148 memory reads required.", "AutoFish", 6);
end;

-- User tunables.
local CONFIG = {
		castMode = "short",           -- "long" | "short" (timed hold) | "custom" cast power
		castShortHoldMs = 300,       -- short mode: release after this hold time (power reads ignored)
		castPowerCustom = 96.0,      -- power % used when castMode == "custom"
		castTimeoutMs = 12000,       -- recast/stop if a phase stalls this long
		postCastDelayMs = 150,       -- settle after releasing a cast before shaking
		postCatchDelayMs = 400,      -- wait after a catch before the next cast
		castOnTimeout = true,        -- recast on timeout (vs. stop)
		shakeIntervalMs = 15,        -- ms between shake key taps
		noShakeTimeoutMs = 10000,    -- no shake prompt within this after casting -> re-equip + recast
		completionThreshold = 99.0,  -- reel progress % that counts as a catch
		equipSlot = 1,               -- hotbar slot the rod lives in
	};

-- Where to fetch a fresh offset set from, if available.
local OFFSETS_URL = "https://offsets.imtheo.lol/offsets.hpp";

-- GUI memory offsets (bytes from an instance address). Overwritten by loadOffsets().
local OFFSETS = {
		FramePositionX = 1296,   -- GuiObject.Position
		FrameSizeX = 1328,       -- GuiObject.Size
		ScreenGuiEnabled = 1220, -- ScreenGui.Enabled
		FrameVisible = 1453,     -- GuiObject.Visible
	};

-- Pull one `field = 0x..` (or decimal) value out of a `namespace <ns> { ... }` block.
local function parseOffset(source, namespace, field)
	local block = source:match("namespace%s+" .. namespace .. "%s*{(.-)\n%s*}");
	if not block then
		return nil;
	end;
	local value = block:match(field .. "%s*=%s*(0x%x+)") or block:match(field .. "%s*=%s*(%d+)");
	return value and tonumber(value) or nil;
end;

-- Fetch the offsets file and overwrite any defaults we can parse from it.
local function loadOffsets()
	if type(game.HttpGet) ~= "function" then
		return;
	end;
	local ok, body = pcall(function()
			return game:HttpGet(OFFSETS_URL);
		end);
	if not ok or type(body) ~= "string" or #body == 0 then
		warn("[AutoFish] Failed to fetch offsets, using built-in defaults.");
		return;
	end;
	-- Map each offset we care about to its namespace/field name in the file.
	local parsed = {
			FramePositionX = parseOffset(body, "GuiObject", "Position"),
			FrameSizeX = parseOffset(body, "GuiObject", "Size"),
			ScreenGuiEnabled = parseOffset(body, "GuiObject", "ScreenGui_Enabled"),
			FrameVisible = parseOffset(body, "GuiObject", "Visible"),
		};
	local count = 0;
	for key, value in pairs(parsed) do
		if value then
			OFFSETS[key] = value;
			count = count + 1;
		end;
	end;
	if count > 0 then
		print("[AutoFish] Loaded " .. count .. " offset(s) from " .. OFFSETS_URL);
	else
		warn("[AutoFish] Offsets fetched but none matched, using built-in defaults.");
	end;
end;
pcall(loadOffsets);

-- Read a float at an address (0 on failure / bad pointer). memory_read can
-- hand back an error STRING on a bad read (pcall still succeeds) tonumber()
-- everything so a string can never leak into arithmetic.
local function readFloat(addr)
	if not addr or addr <= 4096 then
		return .0;
	end;
	local ok, value = pcall(memory_read, "float", addr);
	return (ok and tonumber(value)) or .0;
end;

-- Read a 32-bit int at an address (0 on failure).
local function readInt(addr)
	if not addr or addr <= 4096 then
		return 0;
	end;
	local ok, value = pcall(memory_read, "int32", addr);
	if not ok then
		ok, value = pcall(memory_read, "int", addr);
	end;
	return (ok and tonumber(value)) or 0;
end;

-- Read a single byte at an address (0 on failure).
local function readByte(addr)
	if not addr or addr <= 4096 then
		return 0;
	end;
	local ok, value = pcall(memory_read, "byte", addr);
	return (ok and tonumber(value)) or 0;
end;

-- Memory address of an instance, or nil if it can't be resolved.
local function getAddress(inst)
	if not inst then
		return nil;
	end;
	local ok, addr = pcall(function()
			return inst.Address;
		end);
	addr = (ok and addr) and tonumber(addr) or nil;
	return (addr and addr > 4096) and addr or nil;
end;

-- GuiObject.Position as (xScale, xOffset, yScale, yOffset).
local function readFramePos(frame)
	local addr = getAddress(frame);
	if not addr then
		return 0, 0, 0, 0;
	end;
	local base = addr + OFFSETS.FramePositionX;
	return readFloat(base + 0), readInt(base + 4), readFloat(base + 8), readInt(base + 12);
end;

-- GuiObject.Size as (xScale, xOffset, yScale, yOffset).
local function readFrameSize(frame)
	local addr = getAddress(frame);
	if not addr then
		return 0, 0, 0, 0;
	end;
	local base = addr + OFFSETS.FrameSizeX;
	return readFloat(base + 0), readInt(base + 4), readFloat(base + 8), readInt(base + 12);
end;

-- ScreenGui.Enabled try the property, fall back to a memory read.
local function isEnabled(gui)
	if not gui then
		return false;
	end;
	local ok, value = pcall(function()
			return gui.Enabled;
		end);
	if ok and type(value) == "boolean" then
		return value;
	end;
	local addr = getAddress(gui);
	if not addr then
		return true;
	end;
	return readByte(addr + OFFSETS.ScreenGuiEnabled) ~= 0;
end;

-- GuiObject.Visible try the property, fall back to a memory read.
local function isVisible(gui)
	if not gui then
		return false;
	end;
	local ok, value = pcall(function()
			return gui.Visible;
		end);
	if ok and type(value) == "boolean" then
		return value;
	end;
	local addr = getAddress(gui);
	if not addr then
		return true;
	end;
	return readByte(addr + OFFSETS.FrameVisible) ~= 0;
end;

-- True for a normal finite number (rejects non-numbers, NaN and ±inf).
local function isFinite(n)
	return type(n) == "number" and n == n and (n ~= math.huge and n ~= -math.huge);
end;

-- Safe FindFirstChild.
local function findChild(parent, name)
	if not parent then
		return nil;
	end;
	local ok, child = pcall(parent.FindFirstChild, parent, name);
	return ok and child or nil;
end;

-- Safe GetChildren (empty table on failure).
local function getChildren(inst)
	if not inst then
		return {};
	end;
	local ok, kids = pcall(inst.GetChildren, inst);
	return (ok and kids) or {};
end;

-- The local player's PlayerGui.
local function getPlayerGui()
	local player = getLocalPlayer();
	if not player then
		return nil;
	end;
	return player:FindFirstChildOfClass("PlayerGui") or findChild(player, "PlayerGui");
end;

-- The local player's character model (falls back to the Workspace-named model).
local function getCharacter()
	local player = getLocalPlayer();
	if not player then
		return nil;
	end;
	return player.Character or (workspace and findChild(workspace, player.Name));
end;

-- The backpack hotbar frame (helper, currently unused).
local function getHotbar()
	local playerGui = getPlayerGui();
	if not playerGui then
		return nil;
	end;
	local backpack = findChild(playerGui, "backpack");
	if not backpack then
		return nil;
	end;
	return findChild(backpack, "hotbar");
end;

-- Breadth-first search for a descendant Frame with the given name.
local function findFrameNamed(root, name)
	if not root then
		return nil;
	end;
	local queue, index = { root }, 1;
	while index <= #queue do
		local node = queue[index];
		index = index + 1;
		for _, child in ipairs(getChildren(node)) do
			if child.Name == name and child.ClassName == "Frame" then
				return child;
			end;
			queue[#queue + 1] = child;
		end;
		-- Safety cap so a pathological tree can't spin forever.
		if index > 8192 then
			return nil;
		end;
	end;
	return nil;
end;

-- Virtual-key codes we use.
local KEYS = { Enter = 13, F1 = 112 };

-- Press a key and release it after `holdMs` (default 25) on a background thread.
local function tapKey(vk, holdMs)
	keypress(vk);
	task.spawn(function()
		wait((holdMs or 25) / 1000);
		keyrelease(vk);
	end);
end;

-- Tap Enter (the shake input)
local function tapEnter()
	tapKey(KEYS.Enter, 20);
end;

-- Map a hotbar slot (0-9 or a letter) to its virtual-key code.
local function slotToVk(slot)
	local text = tostring(slot or "");
	local num = tonumber(text);
	if num and (num >= 0 and num <= 9) then
		return 48 + num;   -- '0'..'9'
	end;
	if #text == 1 then
		local byte = text:upper():byte();
		if byte >= 65 and byte <= 90 then
			return byte;   -- 'A'..'Z'
		end;
	end;
	return nil;
end;

-- Tap the key for a hotbar slot.
local function pressSlot(slot)
	local vk = slotToVk(slot);
	if not vk then
		return;
	end;
	tapKey(vk, 25);
end;

-- Is the Roblox window focused? (Assume yes if the API is missing.)
local function isRobloxActive()
	if type(isrbxactive) ~= "function" then
		return true;
	end;
	local ok, active = pcall(isrbxactive);
	return (not ok) or (active ~= false);
end;

-- Track whether we're holding left mouse, so press/release stay idempotent.
local mouseHeld = false;

-- Hold left mouse down (charge a cast / drive the reel bar up).
local function holdMouse()
	if mouseHeld then
		return;
	end;
	mouse1press();
	mouseHeld = true;
end;

-- Release left mouse.
local function releaseMouse()
	if not mouseHeld then
		return;
	end;
	mouse1release();
	mouseHeld = false;
end;

-- True if any Tool is in the character (i.e. the rod is out).
local function hasToolEquipped()
	local character = getCharacter();
	if not character then
		return false;
	end;
	for _, child in ipairs(getChildren(character)) do
		if child.ClassName == "Tool" then
			return true;
		end;
	end;
	return false;
end;

-- Equip the rod by tapping its hotbar slot if nothing is equipped.
local function equipRod()
	if hasToolEquipped() then
		return;
	end;
	if CONFIG.equipSlot and CONFIG.equipSlot > 0 then
		pressSlot(CONFIG.equipSlot);
	end;
end;

-- The reel minigame ScreenGui (PlayerGui.reel).
local function getReelGui()
	local playerGui = getPlayerGui();
	if not playerGui then
		return nil;
	end;
	return findChild(playerGui, "reel");
end;

-- Is the reel GUI enabled? (helper, currently unused).
local function isReelEnabled()
	local reel = getReelGui();
	return reel and isEnabled(reel) or false;
end;

-- Live reel context { bar, fish, playerbar } while the minigame is up, else nil.
local function getReelContext()
	local reel = getReelGui();
	if not reel or not isEnabled(reel) then
		return nil;
	end;
	local bar = findChild(reel, "bar");
	if not bar then
		return nil;
	end;
	local fish = findChild(bar, "fish");
	local playerbar = findChild(bar, "playerbar");
	if not (fish and playerbar) then
		return nil;
	end;
	return { bar = bar, fish = fish, playerbar = playerbar };
end;

-- True when the reel minigame is actually active (context present).
local function isReeling(ctx)
	ctx = ctx or getReelContext();
	return ctx and (ctx.fish and (ctx.playerbar and true)) or false;
end;

-- Shake prompt is up only when its button is actually visible (Fisch leaves the
-- shakeui ScreenGui enabled even when idle, so gate on the button, not Enabled).
local function shakeUp()
	local playerGui = getPlayerGui();
	local shakeGui = playerGui and findChild(playerGui, "shakeui");
	if not shakeGui or not isEnabled(shakeGui) then
		return false;
	end;
	local safezone = findChild(shakeGui, "safezone");
	local button = safezone and findChild(safezone, "button");
	if not button then
		return false;
	end;
	return isVisible(safezone) and isVisible(button);
end;

-- A fill-bar frame's width (Size.X.Scale) as a 0-100 percent, or nil if implausible.
local function readFillPercent(frame)
	local xScale = readFrameSize(frame);
	if not isFinite(xScale) or xScale < -0.05 or xScale > 1.5 then
		return nil;
	end;
	return math.max(.0, math.min(100.0, xScale * 100.0));
end;

-- Current reel progress percent (reel.bar.progress.bar width), or nil.
local function getReelProgress()
	local reel = getReelGui();
	if not reel then
		return nil;
	end;
	local bar = findChild(reel, "bar");
	if not bar then
		return nil;
	end;
	local progress = findChild(bar, "progress");
	if not progress then
		return nil;
	end;
	local progressBar = findChild(progress, "bar");
	if not progressBar then
		return nil;
	end;
	return readFillPercent(progressBar);
end;

-- The cast power bar frame (character.HumanoidRootPart.power → "bar" Frame), or nil.
local function getPowerBar()
	local character = getCharacter();
	if not character then
		return nil;
	end;
	local hrp = findChild(character, "HumanoidRootPart");
	if not hrp then
		return nil;
	end;
	local power = findChild(hrp, "power");
	if not power then
		return nil;
	end;
	return findFrameNamed(power, "bar");
end;

-- Cast power as a 0-100 percent, read from the bar's Size.Y.Scale, or nil.
local function readPowerPercent(barFrame)
	local addr = getAddress(barFrame);
	if not addr then
		return nil;
	end;
	local yScale = readFloat((addr + OFFSETS.FrameSizeX) + 8);
	if not isFinite(yScale) or yScale < -0.05 or yScale > 1.5 then
		return nil;
	end;
	return math.max(.0, math.min(100.0, yScale * 100.0));
end;

-- Power % at which we release a cast, based on castMode ("short" is time-based, not power-based).
local function getCastThreshold()
	if CONFIG.castMode == "custom" then
		return math.max(1.0, math.min(100.0, CONFIG.castPowerCustom));
	end;
	return 90.0;
end;

-- Reel bar-tracking controller tuning (prediction + PWM duty-cycle mix).
local REEL_TUNING = {
		CloseThreshold = .01,
		DerivativeGain = .55,
		EdgeBoundary = .1,
		NeutralDutyCycle = .5,
		PredictionStrength = 7.5,
		ProportionalGain = .42,
		Resilience = .0,
		VelocityDamping = 38,
	};

-- Reel controller: nudges the player bar toward the fish by holding/releasing mouse.
local ReelController = {};
ReelController.__index = ReelController;

-- Fresh controller with cleared velocity/PWM memory.
function ReelController.new()
	return setmetatable({ lastPlayerbarPos = nil, lastFishPos = nil, pwmAccumulator = .0 }, ReelController);
end;

-- Clear per-reel state (call when a reel starts/ends).
function ReelController.Reset(self)
	self.lastPlayerbarPos = nil;
	self.lastFishPos = nil;
	self.pwmAccumulator = .0;
end;

-- Fish bar center position (0-1 across the bar).
function ReelController.GetFishPosition(self, ctx)
	if not ctx or not ctx.fish then
		return nil;
	end;
	local pos = readFramePos(ctx.fish);
	local size = readFrameSize(ctx.fish);
	return pos + (size / 2);
end;

-- Player bar position (0-1 across the bar).
function ReelController.GetPlayerbarPosition(self, ctx)
	if not ctx or not ctx.playerbar then
		return nil;
	end;
	return (readFramePos(ctx.playerbar));
end;

-- Is the fish currently within the player bar's span? nil if context is bad.
local function isFishOnBar(ctx)
	if not (ctx and (ctx.playerbar and ctx.fish)) then
		return nil;
	end;
	local playerbarPos = readFramePos(ctx.playerbar);
	local playerbarSize = readFrameSize(ctx.playerbar);
	local fishPos = readFramePos(ctx.fish);
	local fishSize = readFrameSize(ctx.fish);
	local fishCenter = fishPos + (fishSize / 2);
	local halfBar = playerbarSize / 2;
	return fishCenter >= playerbarPos - halfBar and fishCenter <= playerbarPos + halfBar;
end;

-- One reel tick: decide hold vs. release to track the fish with the player bar.
function ReelController.Update(self, ctx)
	-- No overlap info -> let go.
	if isFishOnBar(ctx) == nil then
		releaseMouse();
		return;
	end;
	local fishPos = self:GetFishPosition(ctx);
	local playerbarPos = self:GetPlayerbarPosition(ctx);
	if not (fishPos and playerbarPos) then
		return;
	end;
	-- Seed velocity memory on the first tick.
	if self.lastPlayerbarPos == nil then
		self.lastPlayerbarPos = playerbarPos;
	end;
	if self.lastFishPos == nil then
		self.lastFishPos = fishPos;
	end;
	-- Per-tick velocities.
	local playerbarVel = playerbarPos - self.lastPlayerbarPos;
	self.lastPlayerbarPos = playerbarPos;
	local fishVel = fishPos - self.lastFishPos;
	self.lastFishPos = fishPos;
	-- Signed error: where the fish is relative to the bar.
	local err = fishPos - playerbarPos;
	local edge = REEL_TUNING.EdgeBoundary;
	-- Hard clamps near the bar's travel limits.
	if playerbarPos < edge then
		holdMouse();
		return;
	end;
	if playerbarPos > 1 - edge then
		releaseMouse();
		return;
	end;
	-- Predict where the bar is heading, then re-measure the error against that.
	local predicted = playerbarPos + (playerbarVel * (REEL_TUNING.PredictionStrength * (1.0 - REEL_TUNING.Resilience)));
	local predErr = fishPos - predicted;
	local closeThresh = REEL_TUNING.CloseThreshold;
	local errAgrees = (err * predErr) > 0;          -- prediction still on the same side
	local movingToward = (err * playerbarVel) > 0;  -- bar already heading at the fish
	local errBeyond = math.max(.0, math.abs(err) - closeThresh);
	local velReach = math.abs(playerbarVel) * 8;
	local willArrive = movingToward and (velReach >= errBeyond);
	-- Far from target and not already coasting in -> push hard toward the fish.
	if math.abs(err) > closeThresh and (errAgrees and not willArrive) then
		if err > 0 then
			holdMouse();
		else
			releaseMouse();
		end;
		return;
	end;
	-- Otherwise steer with a PWM duty cycle around the neutral hold ratio.
	local neutral = REEL_TUNING.NeutralDutyCycle;
	local duty;
	if willArrive and velReach > 0 then
		-- Ease off as we close in so we don't overshoot.
		local closeness = 1.0 - math.min(1.0, errBeyond / velReach);
		if err > 0 then
			duty = neutral * (1.0 - closeness);
		else
			duty = neutral + ((1.0 - neutral) * closeness);
		end;
	else
		-- PD term (+ velocity damping) folded into the duty cycle.
		local adjust = ((REEL_TUNING.ProportionalGain * err) + (REEL_TUNING.DerivativeGain * fishVel)) - (REEL_TUNING.VelocityDamping * playerbarVel);
		duty = math.max(.0, math.min(1.0, neutral + adjust));
	end;
	-- Accumulate duty; when it crosses 1 we spend a "hold" frame, else release.
	self.pwmAccumulator = self.pwmAccumulator + duty;
	if self.pwmAccumulator >= 1.0 then
		self.pwmAccumulator = self.pwmAccumulator - 1.0;
		holdMouse();
	else
		releaseMouse();
	end;
end;

-- The single reel controller instance.
local reelCtrl = ReelController.new();

-- Live macro state.
local STATE = {
		phase = "OFF",             -- OFF | CASTING | CASTED | SHAKE | REEQUIP | FISHING | DONE
		castStartedAt = 0,         -- ms tick when the current cast began charging
		castReleasedAt = 0,        -- ms tick when the cast was released
		castBarSeen = false,       -- have we seen the power bar this cast?
		castThreshold = 90.0,      -- power % to release at (from castMode)
		castWaitTimeoutMs = 15000, -- per-cast stall timeout
		castChargeLastPct = nil,   -- last power reading (to detect a frozen bar)
		castChargeMotionAt = 0,    -- ms tick the power last moved
		lastShakedAt = 0,          -- ms tick of the last shake tap
		shakeCount = 0,            -- Enter taps this cast (for the console prints)
		shakingIntervalMs = 25,    -- ms between shake taps
		shakeSeen = false,         -- has the shake prompt appeared this cast?
		reequipTapsLeft = 0,       -- slot taps remaining in the REEQUIP sequence
		reequipLastTapAt = 0,      -- ms tick of the last REEQUIP slot tap
		fishingLostAt = 0,         -- reel-loss timestamp (reserved)
		completionReached = false, -- did reel progress hit the catch threshold?
		doneAt = 0,                -- ms tick we entered DONE
		powerPercent = "",         -- debug: current power reading
		progressPercent = "",      -- debug: current reel progress
		caught = 0,                -- catch counter
		lost = 0,                  -- loss counter
		timeouts = 0,              -- timeout counter
	};

-- Begin a fresh cast cycle: clean slate, equip rod, pick the starting phase from the UI.
local function startCycle()
	releaseMouse();
	reelCtrl:Reset();
	equipRod();
	STATE.castStartedAt = tick() * 1000;
	STATE.castReleasedAt = 0;
	STATE.castBarSeen = false;
	STATE.castChargeLastPct = nil;
	STATE.castChargeMotionAt = 0;
	STATE.lastShakedAt = 0;
	STATE.shakeCount = 0;
	STATE.shakeSeen = false;
	STATE.reequipTapsLeft = 0;
	STATE.reequipLastTapAt = 0;
	STATE.fishingLostAt = 0;
	STATE.completionReached = false;
	STATE.doneAt = 0;
	STATE.powerPercent = "";
	STATE.progressPercent = "";
	STATE.castThreshold = getCastThreshold();
	STATE.castWaitTimeoutMs = math.max(5000, CONFIG.castTimeoutMs);
	STATE.shakingIntervalMs = CONFIG.shakeIntervalMs;
	-- UI check: reel up -> reel, shake up -> shake, otherwise cast.
	STATE.phase = isReeling() and "FISHING" or (shakeUp() and "SHAKE" or "CASTING");
	STATE.shakeSeen = STATE.phase == "SHAKE";
end;

-- Drop to an idle phase (default OFF), releasing input and clearing the reel.
local function resetToPhase(phase)
	releaseMouse();
	reelCtrl:Reset();
	STATE.powerPercent = "";
	STATE.progressPercent = "";
	STATE.phase = phase or "OFF";
end;

-- A phase stalled: recast (or stop, per config).
local function onCastTimeout()
	STATE.timeouts = STATE.timeouts + 1;
	if CONFIG.castOnTimeout then
		startCycle();
	else
		resetToPhase("OFF");
	end;
end;

-- CASTING: charge the power bar and release at the threshold; bail to reel/shake if seen.
local function updateCasting()
	STATE.progressPercent = "";
	-- Reel already up (fast bite) -> reel.
	if isReeling() then
		releaseMouse();
		STATE.fishingLostAt = 0;
		STATE.phase = "FISHING";
		print("[AutoFish] Detected reel");
		return;
	end;
	-- Shake prompt already up -> shake.
	if shakeUp() then
		releaseMouse();
		STATE.lastShakedAt = 0;
		STATE.shakeSeen = true;
		if STATE.castReleasedAt == 0 then
			STATE.castReleasedAt = tick() * 1000;
		end;
		STATE.phase = "SHAKE";
		return;
	end;
	holdMouse();
	if STATE.castStartedAt == 0 then
		STATE.castStartedAt = tick() * 1000;
	end;
	-- Short mode: power reads are unreliable, so release purely on hold time.
	if CONFIG.castMode == "short" then
		local nowMs = tick() * 1000;
		STATE.powerPercent = "---";
		if (nowMs - STATE.castStartedAt) >= CONFIG.castShortHoldMs then
			releaseMouse();
			STATE.castReleasedAt = nowMs;
			STATE.phase = "CASTED";
			print(string.format("[AutoFish] Casted (short, %dms hold)", CONFIG.castShortHoldMs));
		end;
		return;
	end;
	local powerBar = getPowerBar();
	-- No power bar yet: wait briefly, then recast if it never appears.
	if not powerBar then
		STATE.powerPercent = "---";
		local elapsed = tick() * 1000 - STATE.castStartedAt;
		if not STATE.castBarSeen and elapsed >= 2000 then
			onCastTimeout();
			return;
		end;
		if elapsed >= STATE.castWaitTimeoutMs then
			onCastTimeout();
		end;
		return;
	end;
	STATE.castBarSeen = true;
	local power = readPowerPercent(powerBar);
	local now = tick() * 1000;
	if power then
		STATE.powerPercent = string.format("%.1f", power);
		-- Charged enough -> release the cast.
		if power >= STATE.castThreshold then
			releaseMouse();
			STATE.castReleasedAt = now;
			STATE.phase = "CASTED";
			print(string.format("[AutoFish] Casted at %.1f%%", power));
			return;
		end;
	else
		STATE.powerPercent = "---";
	end;
	-- Track whether the bar is still moving (a frozen leftover bar never reaches threshold).
	local moving = (power ~= nil) and (STATE.castChargeLastPct == nil or math.abs(power - STATE.castChargeLastPct) >= .5);
	if moving then
		STATE.castChargeMotionAt = now;
	end;
	STATE.castChargeLastPct = power;
	if STATE.castChargeMotionAt == 0 then
		STATE.castChargeMotionAt = now;
	end;
	-- Bar stuck for >1.2s -> recast.
	if STATE.castBarSeen and (now - STATE.castChargeMotionAt) >= 1200 then
		onCastTimeout();
		return;
	end;
	if (now - STATE.castStartedAt) >= STATE.castWaitTimeoutMs then
		onCastTimeout();
	end;
end;

-- CASTED: brief settle after release, then move to shaking.
local function updateCasted()
	STATE.powerPercent = "";
	releaseMouse();
	if STATE.castReleasedAt == 0 then
		STATE.castReleasedAt = tick() * 1000;
	end;
	if (tick() * 1000 - STATE.castReleasedAt) < CONFIG.postCastDelayMs then
		return;
	end;
	STATE.lastShakedAt = 0;
	STATE.phase = "SHAKE";
end;

-- No shake prompt appeared after the cast: switch to the REEQUIP phase, which
-- taps the rod slot twice (unequip, then re-equip) before restarting the loop.
local function beginReequip()
	releaseMouse();
	reelCtrl:Reset();
	STATE.timeouts = STATE.timeouts + 1;
	STATE.reequipTapsLeft = 2;
	STATE.reequipLastTapAt = 0;
	STATE.powerPercent = "";
	STATE.progressPercent = "";
	STATE.phase = "REEQUIP";
end;

-- REEQUIP: tap the rod slot twice with a gap between taps, let the re-equip
-- register, then start a fresh cast cycle.
local function updateReequip()
	releaseMouse();
	local now = tick() * 1000;
	if STATE.reequipTapsLeft > 0 then
		if STATE.reequipLastTapAt == 0 or (now - STATE.reequipLastTapAt) >= 400 then
			pressSlot(CONFIG.equipSlot);
			STATE.reequipTapsLeft = STATE.reequipTapsLeft - 1;
			STATE.reequipLastTapAt = now;
		end;
		return;
	end;
	-- Give the re-equip a moment to land before recasting.
	if (now - STATE.reequipLastTapAt) >= 600 then
		startCycle();
	end;
end;

-- SHAKE: tap Enter on an interval until the reel starts (or time out and recast).
local function updateShake()
	STATE.powerPercent = "";
	STATE.progressPercent = "";
	releaseMouse();
	-- Reel started -> reel.
	if isReeling() then
		STATE.fishingLostAt = 0;
		STATE.phase = "FISHING";
		print("[AutoFish] Shook " .. STATE.shakeCount .. " times");
		print("[AutoFish] Detected reel");
		return;
	end;
	local now = tick() * 1000;
	if shakeUp() then
		STATE.shakeSeen = true;
	end;
	if STATE.lastShakedAt == 0 or (now - STATE.lastShakedAt) >= STATE.shakingIntervalMs then
		tapEnter();
		STATE.shakeCount = STATE.shakeCount + 1;
		STATE.lastShakedAt = now;
	end;
	-- No shake prompt within the window after casting -> re-equip the rod and restart.
	if not STATE.shakeSeen and STATE.castReleasedAt > 0 and (now - STATE.castReleasedAt) >= CONFIG.noShakeTimeoutMs then
		beginReequip();
		return;
	end;
	-- Waited too long without a reel -> recast.
	if STATE.castReleasedAt > 0 and (now - STATE.castReleasedAt) >= STATE.castWaitTimeoutMs then
		startCycle();
	end;
end;

-- FISHING: run the reel controller each frame; when the reel ends, score it.
local function updateFishing()
	STATE.powerPercent = "";
	local ctx = getReelContext();
	local progress = getReelProgress();
	STATE.progressPercent = progress and string.format("%.1f", progress) or "";
	if progress and progress >= CONFIG.completionThreshold then
		STATE.completionReached = true;
	end;
	-- Still reeling -> keep tracking the fish.
	if ctx and isReeling(ctx) then
		STATE.fishingLostAt = 0;
		reelCtrl:Update(ctx);
		return;
	end;
	-- Reel gone: release, tally catch vs. loss, finish the cycle.
	releaseMouse();
	reelCtrl:Reset();
	if STATE.completionReached or (progress and progress >= CONFIG.completionThreshold) then
		STATE.caught = STATE.caught + 1;
		print("[AutoFish] Catch successful! (" .. STATE.caught .. " total)");
	else
		STATE.lost = STATE.lost + 1;
	end;
	resetToPhase("DONE");
end;

-- DONE: wait out the post-catch delay, then start the next cast.
local function updateDone()
	-- A new reel somehow appeared -> reel it.
	if isReeling() then
		STATE.doneAt = 0;
		STATE.phase = "FISHING";
		return;
	end;
	local now = tick() * 1000;
	if STATE.doneAt == 0 then
		STATE.doneAt = now;
	end;
	if (now - STATE.doneAt) < CONFIG.postCatchDelayMs then
		return;
	end;
	startCycle();
end;

-- Phase -> handler dispatch.
local PHASE_HANDLERS = {
		CASTING = updateCasting,
		CASTED = updateCasted,
		SHAKE = updateShake,
		REEQUIP = updateReequip,
		FISHING = updateFishing,
		DONE = updateDone,
	};

-- Master on/off.
local enabled = false;

-- Toggle the macro; resets to a clean OFF state and notifies.
local function setEnabled(on)
	on = on and true or false;
	if on == enabled then
		return;
	end;
	enabled = on;
	if on then
		STATE.phase = "OFF";
	else
		releaseMouse();
		STATE.phase = "OFF";
	end;
	notify(on and "AutoFish ON" or ("AutoFish OFF (" .. (STATE.caught .. " caught)")), "AutoFish", 2);
end;

-- F1 edge state, so a held key toggles only once.
local f1WasDown = false;

RunService.Heartbeat:Connect(function()
	-- F1 toggles the macro on a key-down edge.
	if type(iskeypressed) == "function" then
		local ok, down = pcall(iskeypressed, KEYS.F1);
		if ok and (down and not f1WasDown) then
			f1WasDown = true;
			setEnabled(not enabled);
		elseif ok and not down then
			f1WasDown = false;
		end;
	end;
	if not enabled then
		return;
	end;
	-- Pause (and drop the mouse) while Roblox isn't focused.
	if not isRobloxActive() then
		releaseMouse();
		return;
	end;
	-- From idle, decide the starting phase from the live UI.
	if STATE.phase == "OFF" then
		startCycle();
	end;
	-- Run the current phase handler, guarding against a nil read killing the loop.
	local handler = PHASE_HANDLERS[STATE.phase];
	if handler then
		local ok, err = pcall(handler);
		if not ok then
			releaseMouse();
			warn("[AutoFish] " .. (tostring(STATE.phase) .. (": " .. tostring(err))));
		end;
	end;
end);

notify("AutoFish loaded, press F1 to start.", "AutoFish", 4);
print("[AutoFish] loaded. F1 toggles cast/shake/reel. Cast mode: " .. CONFIG.castMode);
