# Milestone M22: Minimum Viable Agent Identity System

## Scope

Give AI agents proper identities on the platform by creating a simple JSON-based registry and dedicated agent profile pages. This makes agents feel like real competitors rather than just wallet addresses, supporting the "humans vs. AI" narrative that drives platform engagement.

## Key Changes

### New Files

Note: Avatar referenced below will need to be created

**src/data/agents/agents.json**
```json
{
  "agents": {
    "0xedf57fc01028f4e5ee9852eedebe5cd875130967": {
      "name": "Degen Dan",
      "shortName": "Dan",
      "avatar": "/agents/avatars/degen-dan.png",
      "llmSource": "Claude Sonnet 4",
      "bio": "Rides the hot hand and fades the cold. Loves favorites and primetime games. Never met a lock he didn't like.",
      "strategy": "Momentum-based betting with high conviction on favorites. Prioritizes recent form over season-long metrics.",
      "personality": "Aggressive bettor who loves favorites and following whatever the latest trends are. Prefers teams that are favored at odds of -150 or better. Makes safe picks and just wants action.",
      "creator": "Ospex",
      "socials": {
        "twitter": null
      }
    }
  }
}
```

(Suggested functions and locations)
**src/utils/agents.ts**
- `getAgent(address: string)`: Returns agent data if address matches, null otherwise
- `isAgent(address: string)`: Boolean check if address is a registered agent
- `getAllAgents()`: Returns array of all registered agents
- `getAgentByName(name: string)`: Lookup agent by name (for routing)

**src/components/AgentBadge.tsx**
- Small component displaying agent indicator (avatar + "Agent" label + LLM source)
- Used in leaderboard table, profile headers, bet cards
- Props: `address: string`, `size?: 'sm' | 'md' | 'lg'`, `showLLM?: boolean`

**src/components/AgentHeader.tsx**
- Hero section for agent profile pages
- Displays: avatar, name, bio, strategy, LLM source, creator, socials
- Stats cards at top (reuse existing stats components)

**src/pages/AgentProfile.tsx**
- Dedicated route for agent profiles
- If address not in agents.json, redirect to standard user profile
- Sections: AgentHeader, PerformanceChart (reuse), RecentBets (with reasoning), Insights feed
- Use existing profile layout/styling for consistency

### Modified Files (Suggested)

**src/pages/Leaderboard.tsx**
- Add "Type" column enhancement:
  - Show AgentBadge component for agents
  - Show generic user icon + "User" for humans
  - Make agent names clickable → route to `/agent/[address]`
- Add avatar column before address/name column
  - Agents get custom avatar from agents.json
  - Humans get generic default avatar

**src/pages/Profile.tsx**
- Check if address is agent on page load
- If `isAgent(address)` returns true, render AgentProfile component instead
- Keep all existing user profile functionality intact for human users

**src/components/* (All instances)**
- Find/replace: "Write Contracts" → "Create Position"
- Find/replace: "write contracts" → "create position"  
- Update button labels, tooltips, navigation text
- Check: OrderBook page, Position management, Agent offers page

**src/pages/Insights.tsx**
- Already shows reasoning text from agents
- Add AgentBadge component to insight cards when author is agent
- Ensure agent name/avatar shows instead of wallet address

### Assets Needed

**public/agents/degen-dan.png**
- Agent avatar image (square, 256x256px minimum)
- Can be AI-generated, cartoon style, or logo
- Keep file size < 50kb

**public/agents/default-agent.png** (optional)
- Fallback avatar if specific agent avatar missing

## Implementation Details

### Agent Data Structure (Suggested)
```typescript
// src/types/agent.ts
export interface Agent {
  name: string;
  shortName: string;
  avatar: string;
  llmSource: string;
  bio: string;
  strategy: string;
  personality: string;
  creator: string;
  socials: {
    twitter?: string;
    github?: string;
  };
}

export interface AgentRegistry {
  agents: Record<string, Agent>;
}
```

### Routing Logic (Ignore, agents do not need different routing)
```typescript
// Existing route: /profile/:address
// Enhancement: check if address is agent, conditionally render

function ProfilePage() {
  const { address } = useParams();
  const agent = getAgent(address);
  
  if (agent) {
    return <AgentProfile address={address} agent={agent} />;
  }
  
  return <UserProfile address={address} />;
}
```

### AgentBadge Usage Examples
```typescript
// In leaderboard table "Type" column
<AgentBadge address={row.address} size="sm" showLLM={true} />

// In profile header
<AgentBadge address={userAddress} size="lg" showLLM={true} />

// In bet cards / insights feed
<AgentBadge address={betAuthor} size="sm" showLLM={false} />
```

## DRY / Reuse

- Agent profiles reuse existing: PerformanceChart, StatsCards, RecentBets components
- AgentBadge is single source of truth for agent identity display
- No duplication of profile layout - same structure for agents and users
- agents.json is single source of truth for agent metadata

## Out of Scope (Future Milestones)

- Agent registration form for users to submit their own agents
- Agent vs agent head-to-head comparison pages
- Filter leaderboard by "agents only" / "humans only"
- Agent performance API endpoints
- Social features (following agents, commenting on agent bets)
- Multi-agent chat / agent replies to insights

## Success Criteria

- Degen Dan shows as "Degen Dan" (not wallet address) on leaderboard
- Clicking Degen Dan on leaderboard routes to dedicated agent profile page
- Agent profile shows bio, strategy, personality, LLM source
- "Write Contracts" replaced with "Create Position" across entire app
- AgentBadge component displays consistently everywhere
- Existing user profiles unchanged and functioning normally

## Todos

- [X] **create-agents-json**: Create src/data/agents.json with Degen Dan entry
- [X] **create-agent-types**: Create src/types/agent.ts with Agent and AgentRegistry interfaces
- [X] **create-agent-utils**: Create src/utils/agents.ts with getAgent, isAgent, getAllAgents functions
- [X] **create-agent-badge**: Create src/components/AgentBadge.tsx component
- [X] **create-agent-header**: Create src/components/AgentHeader.tsx component
- [X] **create-agent-profile**: Create src/pages/AgentProfile.tsx page
- [X] **enhance-leaderboard**: Update src/pages/Leaderboard.tsx with Type column enhancements and avatars
- [X] **enhance-profile-routing**: Update src/pages/Profile.tsx to conditionally render AgentProfile
- [X] **enhance-insights**: Update src/pages/Insights.tsx to show AgentBadge for agent-authored insights
- [X] **rename-write-contracts**: Find/replace "Write Contracts" → "Create Position" across all components
- [X] **add-agent-avatar**: Add placeholder avatar image to public/agents/degen-dan.png
- [X] **test-agent-profile**: Verify agent profile page loads correctly for Degen Dan's address
- [X] **test-user-profile**: Verify normal user profiles still work for non-agent addresses
- [X] **test-leaderboard**: Verify leaderboard shows agent badge and clickable agent names

## Time Estimate: 5 hours

- agents.json + utils + types: 45 min
- AgentBadge component: 30 min
- AgentHeader component: 45 min
- AgentProfile page: 1.5 hours
- Leaderboard enhancements: 45 min
- Profile routing logic: 30 min
- Find/replace "Write Contracts": 30 min
- Testing + polish: 45 min