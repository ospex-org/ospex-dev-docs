# Milestone 021: Automation + Leaderboard Integration

*Created: November 19, 2025*  
*Completed: November 21, 2025*  
*Status: ✅ Complete*

## Problem

Agent requires manual intervention to run analysis, match bets, and participate in leaderboards. Need full automation so competing against the agent does not require babysitting.

## What You're Building

Automated agent scheduler that discovers active leaderboards, registers once, analyzes games periodically, and matches user bets - all without manual commands.

## Success Criteria

- [X] Agent discovers active leaderboards by name pattern
- [X] Agent auto-registers for leaderboard (once, cached in memory)
- [X] Agent runs analysis automatically at scheduled intervals
- [X] Agent checks for matches every 5 minutes (it runs on a cadence every 5 minutes)
- [X] Agent doesn't re-analyze games it already has interests for
- [X] All agent positions tagged to active leaderboard
- [X] Single command starts everything: `yarn agent:start`

## Implementation

### 1. Leaderboard Discovery & Registration

**Pseudocode:**

```
function findActiveLeaderboard():
  - Query Firebase for leaderboards where:
    - leaderboard has started
    - endTime > now
    - name contains "agent" or "league" or "testing" (case insensitive)
  - Return first match or null

function ensureLeaderboardRegistration(agentId):
  - Load agent memory
  - If memory has activeLeaderboard AND endTime hasn't passed:
    - Return cached leaderboard (skip all checks)
  
  - Find active leaderboard from Firebase
  - If none found:
    - Clear memory.activeLeaderboard
    - Return null
  
  - Check if agent is already registered on-chain for this leaderboard
  - If not registered:
    - Get bankroll + entryFee from agent config
    - Call existing registerForLeaderboard() function
  
  - Store leaderboard info in memory:
    - memory.activeLeaderboard = { id, name, startTime, endTime }
  - Return leaderboard
```

**Key point:** Once registered and stored in memory, we never check again until that leaderboard ends.

Note: we may need to alter leaderboard id in config, as over time this will change

### 2. Smart Analysis (Prevent Over-Picking)

**Config Addition:**

```json
{
  "strategy": {
    "maxActiveBets": 20,
    "maxActiveInterests": 10,
    "maxHighConviction": 2,
    "maxMediumConviction": 3
  }
}
```

**Pseudocode:**

```
function shouldRunAnalysis(agentId):
  - Load agent memory
  - Count existing medium-conviction interests with status="active"
  - Get maxActiveInterests from config (default 10)
  
  - If activeInterestCount >= maxActiveInterests:
    - Log: "Already have {count} active interests, skipping analysis"
    - Return false
  
  - Return true

function runAgentAnalysis(agentId):
  - Check if should run analysis
  - If not, skip LLM call entirely
  
  - Ensure leaderboard registration (uses cached memory)
  
  - Run LLM analysis for new games only
  - Create high-conviction positions (tagged with leaderboardId)
  - Store medium-conviction interests (tagged with leaderboardId)
```

**Key point:** Agent only asks LLM for new picks if it has < 10 active interests. This prevents accumulating 100+ interests over time.

### 3. Scheduler Setup

**Pseudocode:**

```
function startAgentScheduler(agentId):
  - Schedule cron job: "0 8,20 * * *" (8am and 8pm daily)
    - Run: ensureLeaderboardRegistration()
    - Run: runAgentAnalysis() (which internally checks shouldRunAnalysis)
    - Log results
  
  - Schedule cron job: "*/5 * * * *" (every 5 minutes)
    - Run: runMatch() (matches user positions)
    - Log results
  
  - Keep process running
```

**Package.json Addition:**

```json
{
  "scripts": {
    "agent:start": "tsx src/scheduler.ts",
    "agent:run": "tsx src/agent.ts",
    "agent:match": "tsx src/match.ts"
  }
}
```

### 4. Leaderboard Tagging

**Agent Side (Update Existing):**

```
When creating contests:
  - Pass leaderboardId from memory.activeLeaderboard.id
  
When creating positions:
  - Pass leaderboardId from memory.activeLeaderboard.id

When matching positions:
  - Pass leaderboardId from memory.activeLeaderboard.id
```

**Frontend Side (Update Existing):**

```
When user creates position:
  - Query for active leaderboard
  - If found, call registerPositionForLeaderboards() after position creation
  - Log success/failure
```

Note: we do need admin interaction for the speculation to be added to the leaderboard, so we may need some sort of output so I, as an admin, can actually add the speculations the agent wants on the leaderboard *to* the leaderboard.

### 5. Memory Structure Enhancement

**Add to memory.json:**

```json
{
  "decisions": [],
  "activeLeaderboard": {
    "id": "6",
    "name": "Agent Testing League",
    "startTime": 1234567890,
    "endTime": 1234999999,
    "registeredAt": "2025-11-19T..."
  }
}
```

**Cleanup Logic:**

```
On each agent run:
  - If memory.activeLeaderboard.endTime < now:
    - Clear memory.activeLeaderboard
    - Log: "Leaderboard {name} ended, cleared from memory"
```

Note: Leaderboards also have safety periods and periods where ROI submittal can take place, we need to account for that before removing from memory - we also need to either claim if the agent has won the leaderboard, and then remove if the agent has lost the leaderboard - then we would clear

## File Structure (Note: this is only a suggestion, if the contents of these files fit better in existing files, that's fine too)

**New Files:**

```
ospex-agent-server/src/
├── scheduler.ts (NEW - cron jobs)
└── leaderboard-utils.ts (NEW - discovery and registration logic)
```

**Modified Files:**

```
ospex-agent-server/src/
├── agent.ts (add shouldRunAnalysis check, use cached leaderboard)
├── match.ts (pass leaderboardId when matching)
├── memory.ts (add activeLeaderboard field)
└── agents/degen_dan/config.json (add maxActiveInterests)
```

## Testing Checklist

### Phase 1: Leaderboard Discovery

1. [X] Manually create leaderboard in Firebase with "Testing" in name
2. [X] Run agent once
3. [X] Verify agent registered for leaderboard
4. [X] Check memory.json has activeLeaderboard stored
5. [X] Run agent again, verify it uses cached registration (no duplicate registration attempt)

### Phase 2: Scheduling

1. [X] Start scheduler: `yarn agent:start`
2. [X] Wait for scheduled analysis time (or temporarily change cron for faster testing)
3. [X] Verify agent creates interests
4. [X] Create user position on opposite side
5. [X] Wait up to 5 minutes, verify agent matches automatically
6. [X] Leave running overnight, check logs in morning

### Phase 3: Interest Limiting

1. [X] Set maxActiveInterests: 3 in config
2. [X] Run agent multiple times
3. [X] Verify it stops analyzing after 3 active interests
4. [ ] Wait for games to start (clearing interests from memory) (untested, cleanup to be tackled later)
5. [X] Verify agent resumes analyzing once below limit

### Phase 4: Frontend Integration

1. [X] Create user position against agent interest
2. [X] Check Firebase: position should have leaderboardIds array with active leaderboard
3. [X] Verify on Leaderboard page that both you and agent have positions tracked

## Out of Scope (Deferred to M22)

- ❌ Volatility agent (auto-matching agent's own positions)
- ❌ Sophisticated re-evaluation (Wemby problem)
- ❌ Contest scoring automation
- ❌ Prize claiming automation
- ❌ Multiple simultaneous leaderboards

## Done When

You can:

1. ✅ Create a leaderboard called "Agent Testing December"
2. ✅ Run `yarn agent:start` once in a terminal
3. ✅ Leave it running (or set up as a background service)
4. ✅ Go about your day
5. ✅ Check Agents page whenever you want to bet
6. ✅ Create positions without any manual agent commands
7. ✅ Check leaderboard to see standings
8. ✅ Agent never creates more than 10 active interests
9. ✅ Agent auto-registers for new leaderboards when old ones end

## Notes

- Leaderboard registration is cached in memory and only checked once per leaderboard
- Agent analysis only runs if current active interests are below configured limit
- Analysis runs 2x daily, matching runs every 5 minutes
- All agent actions automatically tagged to active leaderboard