# Milestone 003: Complete Contract Redeployment
*Created: September 5, 2025*  
*Target Completion: September 6, 2025*  
*Status: ✅ Complete*

## Problem Statement
LeaderboardModule deployed with dependency on core contract function that doesn't exist in current deployment. Need complete redeployment to sync all modules with core contract changes, including leaderboard entry fee fix and updated event signatures.

## Context
After discovering dependency mismatch between new LeaderboardModule and old OspexCore contract, incremental deployment approach proved insufficient. Must redeploy entire contract suite to ensure all modules reference correct core functions.

## Scope & Boundaries

### In Scope
- Complete contract redeployment on Polygon Amoy testnet
- Oracle consumer setup with new contract addresses
- IPFS secrets file update and Pinata pinning
- Update all project repositories with new addresses/ABIs
- Verify cross-module dependencies work correctly
- Test basic functionality (position creation, leaderboard registration)

### Out of Scope
- Further leaderboard testing (that's milestone 4)
- Frontend UI improvements (unless minor)
- New feature development
- Mainnet considerations

## Prerequisite: Testnet POL Acquisition

### Current Status
- **Current Balance**: 0.09 POL on Amoy
- **Required**: ~1 POL (based on last deployment: 0.88 ETH)
- **Sources Attempted**: 
  - Standard faucet: Errored, 24hr cooldown
  - Chainlink faucet: Rate limited, 70hr wait
  - Alchemy: Only 0.1 POL available

### Action Plan
- [X] **Polygon Form Response**: Wait for manual token request response (POL acquired)
- [ ] **Alternative Sources**: Attempt to acquire testnet tokens with other accounts/wallets

## Deployment Plan

### Phase 1: Pre-Deployment Preparation (30 mins)
- [X] **Verify Local Environment**
  - Confirm all contract changes are committed to git
  - Run full test suite: `forge test`
  - Verify deployment script includes all modules
  - Check current testnet POL balance ≥ 1 POL

- [X] **Backup Current State**
  - Document current contract addresses

### Phase 2: Contract Deployment (45 mins)
- [X] **Execute Full Deployment**
  - Ensure script is up-to-date before executing
  ```bash
  forge script script/DeployAmoy.s.sol:DeployAmoy --rpc-url $AMOY_RPC_URL --broadcast --via-ir --optimize --account my-deployer
  ```
  - Record all new contract addresses
  - Verify deployment transaction success
  - Confirm gas usage within expected range

### Phase 3: Oracle Integration (60 mins)
- [X] **Create New Chainlink Consumer**
  - Add new OracleModule address to Chainlink subscription
  - Fund subscription with sufficient LINK

- [X] **Update Secrets Configuration**
  - Generate new encrypted secrets file
  - Pin to Pinata IPFS
  - Update contract with new secrets URL

### Phase 4: Repository Updates (30 mins)
- [X] **Update Contract Addresses**
  - API server, .env file
  - Frontend configuration files
  - Firebase environment variables
  - MCP server configuration
  - Documentation

- [X] **Update ABIs**
  - Copy new ABIs to frontend
  - Verify MCP server ABI compatibility

- [X] **Update Firebase**
  - Update event handler to reference v2.3 collections
  - Update webhook (Alchemy)

### Phase 5: Smoke Testing (30 mins)
- [X] **Basic Functionality Tests**
  - Create new contest
  - Create new speculation
  - Create new leaderboard
  - Register user for leaderboard ✅ **FIXED: USDC approval, event processing bugs, frontend listener updated**
  - Verify Firebase events are captured
  - Check frontend displays correctly

## Success Criteria

### Technical Acceptance
- [X] All contracts deployed successfully with cross-module dependencies working
- [X] Oracle integration functional with new consumer setup
- [X] Firebase receives events from all new contract addresses
- [X] Frontend connects to new contracts without errors
- [X] User can register for leaderboard (blocking issue resolved)

### Process Acceptance
- [X] All repositories updated with new addresses/ABIs
- [X] Deployment process documented for future reference
- [X] Gas costs tracked and within expected range
- [X] Migration completed without data loss

## Risk Mitigation

### High Risk
- **Insufficient testnet POL**: Deployment fails mid-process
  - *Mitigation*: Secure 1.5+ POL before starting
- **Oracle setup errors**: Complex integration process
  - *Mitigation*: Test oracle separately before main deployment

### Medium Risk
- **ABI compatibility issues**: Frontend/MCP server integration
  - *Mitigation*: Update ABIs immediately after deployment
- **Firebase configuration**: New addresses not reflected
  - *Mitigation*: Test event capture before full testing

### Low Risk
- **Gas price fluctuations**: Deployment costs higher than expected

## Post-Deployment Checklist

### Immediate (Same Day)
- [X] Update system-overview.md with new contract addresses
- [X] Verify leaderboard user registration works

### Next Session
- [X] Return to milestone 2 objectives with fixed contracts
- [X] Begin comprehensive leaderboard validation testing
- [X] Document any deployment issues encountered

## Files to Update Post-Deployment
- `frontend/src/config/contracts.ts` - Contract addresses
- `frontend/src/abis/` - All ABI files
- `ospex-firebase/functions/src/config.ts` - Contract addresses
- `ospex-mcp/config.json` - Contract addresses and ABIs
- `ospex-dev-docs/architecture/system-overview.md` - Deployment status

## Expected Timeline
- **POL Acquisition**: 1-7 days (external dependency)
- **Deployment Process**: 3-4 hours total
- **Testing & Validation**: 1-2 hours
- **Repository Updates**: 1 hour

## Success Definition
This milestone is complete when all contracts are redeployed, oracle integration works, repositories are updated, and user registration for leaderboards functions correctly - enabling completion of milestone 2 objectives.

*This unblocks leaderboard testing and establishes stable foundation for continued development.*