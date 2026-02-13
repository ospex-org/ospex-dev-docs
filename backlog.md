# Ospex Backlog

*Living document for features, improvements, and ideas not yet scheduled into milestones.*

*Last updated: February 13, 2026*

---

## 🎯 Near-Term (Next 1-3 Milestones)

### Frontend / UX

| Item | Notes | Complexity |
|------|-------|------------|
| **Agent profile page enhancements** | Different from user profiles; show benchmark metrics, model info, tool access | Medium |
| **Human profile enhancements** | Optional bio/description for non-agents; paid feature, store on-chain? | Medium |
| **Position event history** | Full audit trail: created, adjusted (+/-), matched, claimed. Requires new positions events collection in indexer. Currently Firebase only stores final state. | Medium |
| **Agent vs human identification** | No way to distinguish agent wallets from human wallets in the UI. Need a system - on-chain registration, backend registry, or visual indicator. Affects trust and leaderboard credibility. | Medium |
| **Documentation / "How it works"** | New users have no onboarding. Site complexity is growing (orderbook, agents, leaderboards, odds deviation checks). Doesn't need to be a priority until external users arrive, but should be planned. | Large |

### Agent Infrastructure

| Item | Notes | Complexity |
|------|-------|------------|
| **Agent registration** | On-chain or backend? Need to decide architecture | Medium |
| **Model switching** | Michelle uses Sonnet; how to switch to Opus/GPT? Ideally configurable per-agent | Medium |
| **Agent config as user-facing settings** | Current agent configs (edge thresholds, max active bets, risk parameters) are hardcoded magic numbers. These all need to become configurable when users create their own agents. | Large |
| **Agent rating/tracking UI** | Surface benchmark metrics on frontend; let users run benchmarks (paid feature?) | Large |
| **Log agent rejection reasoning** | When scanning order book and passing, store WHY (cost: more LLM calls) | Medium |

### Sports Intelligence Expansion

Building on M33 foundation + M38 breadth work. Depth items for after breadth is covered.

| Tool | Data Needed | Source Ideas |
|------|-------------|--------------|
| **ELO system** | Custom calculation based on game results | Build ourselves from schedule/results data |
| **Power rankings** | Composite strength metric | Derive from ELO + stats |
| **Weather** | For outdoor games (NFL, MLB) | Weather API |
| **Historical matchups** | H2H record, margin trends | ESPN or custom tracking |
| **Advanced stats** | Basketball Reference, Hockey Reference | Scraping or API - legal gray area |
| **KenPom / NCAAB analytics** | Efficiency metrics, tempo, strength of schedule | KenPom API or scraping - investigate before March Madness. ESPN/Supabase only provides basic standings for NCAAB (no last10, limited stats). KenPom would give Michelle much richer analytical depth for college basketball. Priority: before March 20 tournament start. |

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
| **Add agent IDs to scheduler logs** | Partially done - `[Michelle Matching]` and `[degen_dan]` prefixes added. Review remaining log statements for consistency. | Small |
| **Firebase vs Supabase strategy** | Keep both: Firebase for real-time, Supabase for queries/analytics | Low |
| **Document gating strategy** | How do we prevent unauthorized endpoint access? | Medium |
| **Model configuration system** | Per-agent model selection, easy switching | Medium |
| **Protocol documentation (internal)** | The protocol has many baked-in mechanics (odds deviation checks, expiry defaults, leaderboard configurations) that aren't documented anywhere except code. Growing complexity risk. | Medium |
| **Sport ID mapping consolidation** | Multiple sport-to-ID mappings scattered across codebase: JSONOdds IDs (canonical, stored in Firebase), Rundown API IDs (different numbering), smart contract LeagueId enum, and at least one ad-hoc mapping that may have been implemented from an incomplete milestone doc comment. Need a single canonical sport resolver service (similar to teamResolver pattern from M40 Track 5) with adapter functions for each external API. Currently a source of subtle bugs and catch-all mismatches. | Medium |

---

## 📅 Medium-Term (Next Quarter)

### Agent Ecosystem

- [ ] Base agent toolkit that any user can build on
- [ ] Agent marketplace / discovery
- [ ] Agent vs agent competitions
- [ ] Personality-driven agents (Degen Dan as template)
- [ ] **Agent archetype radar chart** — visual identity system for agents based on five strategy axes: **Pundit** (tailing/fading expert content), **Team** (deep knowledge on specific teams), **Situational** (travel, fatigue, weather, scheduling spots), **Analyst** (ELO, statistical systems, quantitative models), **Sentiment** (public betting %, sharp money signals, line movements). Each axis represents tool usage and prompt weighting. Radar chart on agent cards immediately communicates what kind of bettor the agent is. Natural tradeoffs — can't max all five without hitting token budgets and conflicting signals, which creates interesting build decisions. Target: before 2026 NFL season. M40 Track 5 (pundit pipeline) is proof-of-concept for the Pundit axis.

### Sports Expansion

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
- Automated contest creation based on agent preferences
- Collaborative betting (pools, groups)
- Live betting support (fundamentally different product - expiry assumptions don't hold)
- Contract verification (when traction justifies it)

---

## Notes

### On Agent Registration

Need to decide:
- On-chain: More "crypto-native", auditable, but gas costs and complexity
- Backend only: Simpler, but less transparent
- Hybrid: Register on-chain, store metadata in Supabase?

### On Degen Dan

Keep him exactly as-is — he's the "baseline" agent that shows users can build simple prompt-only agents if they want. No tools, just vibes. His consistent losing is actually valuable — users could fade him profitably.

### On Agent Archetype System (Radar Chart)

Five strategy axes that define an agent's identity:
1. **Pundit** — Ingests content from sports pundits/experts. Tails, fades, or blends. Tools: Firecrawl content ingestion, pundit picks pipeline.
2. **Team** — Deep knowledge on specific teams. Bets on or against based on roster depth, coaching tendencies, historical patterns. Tools: roster data, injury reports, team-specific stats.
3. **Situational** — Weighs contextual factors: travel distance, back-to-backs, rest days, altitude, revenge games, scheduling spots. Bets on the belief that these factors are over/underrated by the market. Tools: schedule data, travel calculations, rest day tracking.
4. **Analyst** — Quantitative/statistical approach. ELO ratings, power rankings, advanced metrics, model-driven predictions. Tools: custom ELO system, statistical APIs, historical data.
5. **Sentiment** — Market-derived signals. Public betting percentages, sharp money indicators, line/price movements, steam moves. Tools: odds APIs, line movement tracking, betting percentage data.

Each axis is 0-100 representing how heavily that strategy influences the agent's final decision. The radar chart visualizes this on agent cards in the directory. An agent's archetype is determined by its tool access and prompt weighting — not self-reported.

Key insight: natural tradeoffs exist. Maxing all five axes would require massive token budgets and produce conflicting signals. The constraint is what makes agent building interesting.

Milestone mapping:
- M40 Track 5 (pundit pipeline) = Pundit axis proof of concept
- M33/M38 sports intelligence = Situational + Analyst axis foundation
- Future odds/line movement tracking = Sentiment axis
- Agent creation UI (M41+) = where users configure their mix

### On Agent Config as User Settings

Current hardcoded configs that will need UI:
- Edge threshold (minimum edge to take a bet)
- Max active bets
- Risk parameters (kelly fraction, bankroll %)
- Model selection
- Tool access (which data sources)
- Strategy/personality prompts
- Token budget per evaluation

These are all working as magic numbers for Michelle/Dan today. When users create agents, every one of these becomes a settings field.

### On Agent vs Human Identification

Current state: no way to tell. Options:
1. Separate wallets by policy (current approach)
2. On-chain agent registry
3. Backend-maintained agent list with UI indicator
4. Agent bets must come through API, not UI

Recommendation: Start with #3 (simplest), graduate to #2 when agent ecosystem grows.