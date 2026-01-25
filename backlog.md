# Ospex Backlog

*Living document for features, improvements, and ideas not yet scheduled into milestones.*

*Last updated: January 19, 2025*

---

## 🎯 Near-Term (Next 1-3 Milestones)

### Frontend / UX

| Item | Notes | Complexity |
|------|-------|------------|
| ~~**Confetti on instant match**~~ | ✅ Done M34 - team colors, directional (over/under), logo SVGs | ~~Small~~ |
| **Agent profile page enhancements** | Different from user profiles; show benchmark metrics, model info, tool access | Medium |
| **Insights tab** | Agent's WHY for their positions; query `agentQuotes` for that agent | Medium |
| **Position evaluation history** | WHAT happened - who looked, who passed, who matched; display on position detail page | Medium |
| **Number line visualization** | For betting interfaces (may already be in progress?) | Medium |
| **Human profile enhancements** | Optional bio/description for non-agents; paid feature, store on-chain? | Medium |
| **Position event history** | Full audit trail: created, adjusted (+/-), matched, claimed. Requires new `amoyPositionEventsv2.3` collection in indexer (Option A from investigation). Currently Firebase only stores final state. | Medium |

### Agent Infrastructure

| Item | Notes | Complexity |
|------|-------|------------|
| **Instant match staleness check** | Re-evaluate if cached offer is stale (>30min) or market drifted (>0.05) | Small |
| **Proactive offer maintenance** | Scheduled job: agents review own offers, update if market moved | Medium |
| **Agent registration** | On-chain or backend? Need to decide architecture | Medium |
| **Model switching** | Michelle uses Sonnet; how to switch to Opus/GPT? Ideally configurable per-agent | Medium |
| **Agent rating/tracking** | Surface benchmark metrics on frontend; let users run benchmarks (paid feature?) | Large |
| **More tools for Michelle** | See "Sports Intelligence Expansion" below | Large |
| **Log agent rejection reasoning** | When scanning order book and passing, store WHY (cost: more LLM calls) | Medium |

### Sports Intelligence Expansion

Building on M33's foundation. Goal: Michelle can create lines without looking at sportsbooks.

| Tool | Data Needed | Source Ideas |
|------|-------------|--------------|
| **Standings** | Current W-L, conference rank, playoff position | ESPN API |
| **ELO system** | Custom calculation based on game results | Build ourselves |
| **Team strength metrics** | Point differential, offensive/defensive ratings | ESPN or Basketball Reference |
| **Recent form/momentum** | Last 10 games, home/away splits | ESPN API |
| **Weather** | For outdoor games (NFL, MLB) | Weather API |
| **Rest days/travel** | Back-to-backs, miles traveled | Calculate from schedule |
| **Historical matchups** | H2H record, margin trends | ESPN or custom tracking |

### Revenue / Gating

| Item | Notes | Complexity |
|------|-------|------------|
| **Endpoint protection audit** | Can users hit endpoints without paying? Document gating strategy | Small |
| **Token cost management** | As we add tools, costs increase. Need budget/monitoring | Medium |
| **Benchmark-as-a-service** | Users pay to run benchmarks on any agent | Medium |
| **Pick selling foundation** | Agents with good track records can monetize | Large |

---

## 🔧 Technical Debt / Architecture

| Item | Notes | Priority |
|------|-------|----------|
| ~~**Firebase collection consolidation**~~ | ✅ Done M34 - `michelle*` → `agent*` with `agentId` field | ~~Medium~~ |
| **Add agent IDs to scheduler logs** | `[match]` logs don't identify which agent; hard to debug | Small |
| **Debug Michelle scanning** | "No enriched candidates" despite finding 4 pre-filter; why filtering out? | Medium |
| **Review maxActiveBets limit** | Some agent at 3/3 slots - might be too conservative (unclear if Dan or Michelle) | Small |
| **Firebase vs Supabase strategy** | Keep both: Firebase for real-time, Supabase for queries/analytics | Low |
| **Document gating strategy** | How do we prevent unauthorized endpoint access? | Medium |
| **Model configuration system** | Per-agent model selection, easy switching | Medium |

---

## 📅 Medium-Term (Next Quarter)

### Agent Ecosystem

- [ ] Base agent toolkit that any user can build on
- [ ] Agent marketplace / discovery
- [ ] Agent vs agent competitions
- [ ] Personality-driven agents (Degen Dan as template)

### Sports Expansion

- [ ] NCAAB support (March Madness prep - target February)
- [ ] MLB support (season starts April)
- [ ] NFL/NCAAF (revisit August)

### User Features

- [ ] Leaderboard enhancements
- [ ] Social features (following, notifications)
- [ ] Mobile optimization pass

---

## 💡 Ideas / Someday

- RSS feeds for breaking injury news (RotoWire)
- Real-time lineup changes (not just pre-game)
- Advanced stats integration (Basketball Reference, Hockey Reference)
- Automated contest creation based on agent preferences
- Agent "personalities" that affect risk tolerance and communication style
- Collaborative betting (pools, groups)

---

## ✅ Recently Completed

| Item | Milestone | Date |
|------|-----------|------|
| Confetti on instant match (team colors, directional, logos) | M34 | Jan 19, 2025 |
| Firebase collection consolidation (`michelle*` → `agent*`) | M34 | Jan 19, 2025 |
| Firebase collections documentation | M34 | Jan 19, 2025 |
| Sports intelligence tools (injuries, rosters) | M33 | Jan 2025 |
| Line creation benchmark | M33 | Jan 2025 |
| Supabase integration | M33 | Jan 2025 |
| LangChain agent architecture | M32 | Jan 2025 |
| Firebase read optimization | M31 | Jan 2025 |

---

## Notes

### On Agent Registration

Need to decide:
- On-chain: More "crypto-native", auditable, but gas costs and complexity
- Backend only: Simpler, but less transparent
- Hybrid: Register on-chain, store metadata in Supabase?

### On Degen Dan

Keep him exactly as-is — he's the "baseline" agent that shows users can build simple prompt-only agents if they want. No tools, just vibes. His consistent losing is actually valuable — users could fade him profitably.

### On Insights vs Position History

**Insight** = WHY (the thesis)
- "I'm taking Celtics -6.5 because Tatum is averaging 35 in his last 5 and Heat are missing Butler"
- Agent-generated: Required for all agent positions
- Human-generated: Optional
- UI Surface: Profile page → Insights tab
- Data source: `agentQuotes`

**Position History** = WHAT (the audit trail)
- "Agent X looked at this position → evaluated edge at 2.3% → PASS"
- "Agent Y looked at this position → evaluated edge at 4.1% → MATCH"
- Only meaningful for agents (humans don't leave a trail unless they act)
- UI Surface: Position detail page → "Evaluation History" section
- Data source: `agentCalculations` + potentially new `positionEvaluations` collection

| | Created by Human | Created by Agent |
|---|---|---|
| **Insight (why)** | Optional | Required |
| **Position History (what)** | Only shows if they matched | Every evaluation, even rejections |

### On Proactive Offer Maintenance

Agents should update their own offers when market moves, not just when asked. Considerations:
- **Token cost:** Could be expensive if updating entire slate frequently
- **Tiered staleness:** Check more often close to game time

| Time to game | Check frequency |
|--------------|-----------------|
| > 24 hours | Every 60 min |
| 2-24 hours | Every 30 min |
| < 2 hours | Every 10 min |
| < 30 minutes | Every evaluation |

### On Agent vs Human Betting

If an agent wallet is also used by a human, how do we distinguish? Options:
1. **Separate wallets** - Require agents to use dedicated wallets (simplest)
2. **API-only** - Agent bets must come through API, not UI
3. **Honor system** - Trust that agent wallets are agent-only

Recommendation: Start with separate wallets by policy. Add technical enforcement later if needed.

### On Agent Settings Page Organization

The Settings tab needs to handle a lot:
- Model selection (Sonnet, Opus, GPT, etc.)
- Tool access (which tools is this agent using?)
- Prompt/personality configuration
- Benchmark controls (run benchmark, view history)

**Proposed structure:**
```
Settings
├── Model & Tools
│   ├── Model: [dropdown - Claude Sonnet 4.5]
│   ├── Tools: [checkboxes - injuries, rosters, standings, etc.]
│   └── Token budget: [input]
├── Personality
│   ├── Strategy prompt
│   └── Personality prompt
├── Benchmarks
│   ├── Run benchmark [button]
│   └── Benchmark history [list]
└── Advanced
    ├── Raw prompt [textarea, collapsed by default]
    └── API access [if we ever expose this]
```

Alternatively, split into multiple tabs: Settings | Tools | Benchmarks

### On Human Profile Enhancements

Humans currently just show wallet address + stats. Could add optional:
- Display name
- Bio/description
- Avatar

**Revenue angle:** Charge a small fee (stored on-chain or hashed) for profile customization. Makes it a premium feature without requiring traditional signup.

The benchmark tool (spread accuracy, etc.) is valuable. Users should be able to:
1. See an agent's historical benchmark scores (free)
2. Run a fresh benchmark against current games (paid)
3. Compare agents head-to-head (paid)

This creates a "prove your agent is good" dynamic that could drive engagement and revenue.