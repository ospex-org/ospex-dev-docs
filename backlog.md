# Ospex Backlog

*Living document for features, improvements, and ideas not yet scheduled into milestones.*

*Last updated: February 3, 2026*

---

## 🎯 Near-Term (Next 1-3 Milestones)

### Frontend / UX

| Item | Notes | Complexity |
|------|-------|------------|
| **Market odds setter on order book details** | Currently only possible via v2.2 admin panel. Need UI for triggering oracle call to set/update market odds on a speculation. Power-user feature, not prominent. Could also be scripted on a cadence. | Medium |
| **Agent profile page enhancements** | Different from user profiles; show benchmark metrics, model info, tool access | Medium |
| **Insights tab enhancements** | Agent's WHY for their positions; query `agentQuotes` for that agent. ~~Filter out legacy Amoy insights (pre 2/1/2026 or network !== polygon)~~ ✅ DONE. Consider additional filtering/sorting options. | Medium |
| **Position evaluation history** | WHAT happened - who looked, who passed, who matched; display on position detail page | Medium |
| **Human profile enhancements** | Optional bio/description for non-agents; paid feature, store on-chain? | Medium |
| **Position event history** | Full audit trail: created, adjusted (+/-), matched, claimed. Requires new positions events collection in indexer. Currently Firebase only stores final state. | Medium |
| **Agent vs human identification** | No way to distinguish agent wallets from human wallets in the UI. Need a system - on-chain registration, backend registry, or visual indicator. Affects trust and leaderboard credibility. | Medium |
| **Documentation / "How it works"** | New users have no onboarding. Site complexity is growing (orderbook, agents, leaderboards, odds deviation checks). Doesn't need to be a priority until external users arrive, but should be planned. | Large |

### Agent Infrastructure

| Item | Notes | Complexity |
|------|-------|------------|
| **Proactive offer maintenance** | Scheduled job: agents review own offers, update if market moved | Medium |
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
| **Add agent IDs to scheduler logs** | `[match]` logs don't identify which agent; hard to debug | Small |
| **Review maxActiveBets limit** | Some agent at 3/3 slots - might be too conservative | Small |
| **Firebase vs Supabase strategy** | Keep both: Firebase for real-time, Supabase for queries/analytics | Low |
| **Document gating strategy** | How do we prevent unauthorized endpoint access? | Medium |
| **Model configuration system** | Per-agent model selection, easy switching | Medium |
| **Protocol documentation (internal)** | The protocol has many baked-in mechanics (odds deviation checks, expiry defaults, leaderboard configurations) that aren't documented anywhere except code. Growing complexity risk. | Medium |

---

## 📅 Medium-Term (Next Quarter)

### Agent Ecosystem

- [ ] Base agent toolkit that any user can build on
- [ ] Agent marketplace / discovery
- [ ] Agent vs agent competitions
- [ ] Personality-driven agents (Degen Dan as template)

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

| Time to game | Check frequency |
|--------------|-----------------|
| > 24 hours | Every 60 min |
| 2-24 hours | Every 30 min |
| < 2 hours | Every 10 min |
| < 30 minutes | Every evaluation |

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

### On Market Odds & Leaderboard Integrity

Leaderboards are configured with odds deviation checks (currently 10% from market) to prevent gaming via extreme odds on both sides. This requires market odds to be set on speculations via oracle call. Currently only possible through v2.2 admin panel. Options for long-term:
1. Add to order book details page (power-user UI)
2. Automated script on a cadence
3. Both

### On Agent vs Human Identification

Current state: no way to tell. Options:
1. Separate wallets by policy (current approach)
2. On-chain agent registry
3. Backend-maintained agent list with UI indicator
4. Agent bets must come through API, not UI

Recommendation: Start with #3 (simplest), graduate to #2 when agent ecosystem grows.