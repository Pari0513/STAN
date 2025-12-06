# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

STAN (Stigmergic A* Navigation) is a production-ready graph pathfinding system that combines classical A* algorithms with stigmergic coordination principles inspired by ant colony optimization. The system demonstrates emergent intelligence through pheromone-based collective optimization, achieving 95% cost reduction through multi-agent coordination.

**Key Characteristics:**
- Implements pheromone-weighted pathfinding using formula: `cost_eff = w/(1 + τ · α)`
- Integrates with Memgraph graph database for persistent pheromone storage
- Exposes REST API (FastAPI) and MCP (Model Context Protocol) server
- Tested with 40,000+ routes and 10+ hours continuous operation
- Achieves sub-100ms query performance in production

## Technology Stack

**Core Technologies:**
- **Database**: Memgraph 2.x (graph database with Cypher query language)
- **Backend**: Python 3.9+, FastAPI framework
- **Graph ORM**: GQLAlchemy 1.8.0+
- **Deployment**: Docker Compose, Kubernetes manifests available
- **OS**: Linux 6.14.0-1016-aws

**Key Dependencies:**
- Memgraph for graph storage and query modules
- FastAPI for REST API endpoints
- GQLAlchemy for Python-Memgraph integration
- WebSocket for real-time visualization

## System Architecture

### Three-Layer Architecture

1. **Data Layer**: Memgraph stores 185 concepts (nodes) and 3,570 directed edges with pheromone properties
2. **Service Layer**: Pathfinding, pattern mining, analytics services
3. **Application Layer**: REST API (port 8000), MCP server (9 AI tools), WebSocket interface

### Core Algorithm: Stigmergic A*

The pathfinding algorithm modifies classical A* by incorporating pheromone-weighted edge costs:

```
costeff(e) = w / (1 + τ · α)
```

Where:
- `w` = base edge weight (static cost)
- `τ` = pheromone level (dynamic, 0 to ∞)
- `α` = pheromone influence parameter (0.7 default)

**Pheromone Mechanics:**
- **Deposition**: `τ_new = min(τ_old + Δτ, τ_max)` where Δτ = 1.0 per traversal
- **Decay**: `τ(t + Δt) = τ(t) · (1 - ρ)` where ρ = 0.1 (10% evaporation)
- **Decay interval**: Every 50 iterations
- **Max pheromone**: τ_max = 100.0 (prevents infinite accumulation)

## Development Commands

### Running the System

**Start Memgraph database:**
```bash
docker-compose up -d memgraph
```

**Start STAN API server:**
```bash
# From project root
python -m uvicorn src.main:app --reload --port 8000
```

**Run all tests:**
```bash
pytest tests/ -v
```

**Run specific test suite:**
```bash
pytest tests/test_stigmergy.py -v
```

### Database Operations

**Connect to Memgraph:**
```bash
# Via Docker
docker exec -it memgraph mgconsole

# Via Python
python -c "from gqlalchemy import Memgraph; db = Memgraph()"
```

**Load Hyperseed-1 ontology:**
```bash
# Import concepts and relationships
python scripts/load_hyperseed.py
```

**Reset pheromones:**
```cypher
MATCH ()-[r:INFLUENCES]->()
SET r.pheromone = 0.0;
```

**Query top pheromone edges:**
```cypher
MATCH (a:Concept)-[r:INFLUENCES]->(b:Concept)
WHERE r.pheromone > 0
RETURN a.name, b.name, r.pheromone
ORDER BY r.pheromone DESC
LIMIT 15;
```

### Testing Commands

**Unit tests (9 procedures):**
```bash
pytest tests/test_procedures.py -v
# Tests: connection, procedures loaded, A* pathfinding,
# pheromone deposition/decay, statistics, reset
```

**Integration test (10-iteration convergence):**
```bash
pytest tests/test_convergence.py -v
# Expected: 95% cost reduction (2790.0 → 140.2)
```

**Mass navigation test (40,000 routes):**
```bash
pytest tests/test_mass_navigation.py -v
# Tests diverse concept pairs for statistical validation
```

## Important Implementation Details

### Memgraph Query Modules

STAN uses custom Cypher procedures implemented in `stigmergy_astar.py`:

1. **stigmergy_astar.find_path**: Pheromone-weighted A* pathfinding
2. **stigmergy_astar.deposit**: Add pheromones to successful paths
3. **stigmergy_astar.decay**: Apply exponential evaporation
4. **stigmergy_astar.stats**: Aggregate pheromone statistics
5. **stigmergy_astar.reset**: Clear all pheromones

**Deploy query module:**
```bash
# Copy to Memgraph container
docker cp src/query_modules/stigmergy_astar.py memgraph:/usr/lib/memgraph/query_modules/

# Reload modules
docker exec memgraph mgconsole -e "CALL mg.load_all();"
```

### Admissibility Trade-offs

**Critical Algorithm Characteristic:**

Classical A* guarantees optimal solutions when the heuristic is admissible (never overestimates). STAN's pheromone weighting modifies edge costs dynamically, which **violates admissibility**. However, the system compensates through:

1. **Zero heuristic** for semantic graphs (degrades to Dijkstra's algorithm, which is optimal)
2. **Collective optimization objective**: Optimizes pheromone-weighted cost, not base weight
3. **Exploration-exploitation balance**: Non-admissibility introduces beneficial exploration

**Trade-off:** Individual query optimality sacrificed for collective optimization (95% cost reduction over time).

### Power-Law Distribution

After continuous operation, pheromone distribution exhibits **Zipfian characteristics**:
- 94.9% of pheromones concentrated in top 10% of edges
- 91.4% of edges remain in 0-1 pheromone range (long tail)
- 0.4% of edges become super-highways (≥20 pheromones)

This mirrors natural ant colony behavior.

### Hyperseed-1 Ontology Integration

STAN evaluates routes using AGI concepts from Hyperseed-1:

- **Coherence**: Collective consensus measure (low variance = high coherence)
- **Emergence**: Novel properties from collective optimization
- **Intersubjective Reality**: Shared pheromone graph as collective knowledge
- **Morphic Resonance**: Spatial influence fields from high-pheromone routes

## API Endpoints

**Base URL:** `http://localhost:8000`

### Core Endpoints

```
POST   /api/v1/routes/find           # Stigmergic pathfinding
POST   /api/v1/routes/decay          # Pheromone evaporation
GET    /api/v1/analytics/convergence # Convergence metrics
GET    /api/v1/analytics/coherence   # Coherence statistics
POST   /api/v1/patterns/mine         # Pattern discovery
POST   /api/v1/ontology/evaluate     # Route evaluation via Hyperseed
GET    /health                       # System health check
```

### Example API Calls

**Find path with pheromones:**
```bash
curl -X POST http://localhost:8000/api/v1/routes/find \
  -H "Content-Type: application/json" \
  -d '{
    "start": "Process",
    "goal": "Pattern",
    "influence": 0.7
  }'
```

**Apply decay:**
```bash
curl -X POST http://localhost:8000/api/v1/routes/decay \
  -H "Content-Type: application/json" \
  -d '{"decay_rate": 0.1}'
```

## MCP Server Integration

STAN provides 9 AI tools via Model Context Protocol for integration with Claude, ChatGPT, etc.:

1. **find_path**: Stigmergic navigation between concepts
2. **deposit_pheromones**: Reinforce successful paths
3. **apply_decay**: Evaporate pheromones
4. **get_statistics**: Aggregate metrics
5. **mine_patterns**: Discover emergent structures
6. **evaluate_route**: Hyperseed ontological analysis
7. **get_convergence**: Optimization metrics
8. **get_coherence**: Collective consensus
9. **reset_pheromones**: Clear graph state

**Start MCP server:**
```bash
python src/mcp_server.py
```

## Performance Benchmarks

**Latency Requirements (Production):**
- Single pathfinding query: <50ms
- Pheromone deposition: <10ms
- Full graph decay (1,785 edges): ~100ms
- Statistics query: <5ms

**Tested at Scale:**
- 40,000+ routes analyzed
- 10+ hours continuous operation
- 155,700 total iterations
- 1,724 edges enriched (48.3% of graph)

## Known Limitations

### Graph Size
- **Current**: 185 nodes, 1,785 bidirectional edges
- **Target**: 100,000+ nodes, 1,000,000+ edges
- **Mitigation**: Graph partitioning, distributed Memgraph cluster

### Single Dataset
All experiments use Hyperseed-1 ontology. Generalization requires testing on:
- Wikipedia concept graphs
- Academic citation networks (ArXiv)
- Code dependency graphs (GitHub)
- Social networks

### Coherence Remains Zero
Low coherence (high pheromone variance) is expected during exploration but may require higher decay rates (20-30%) for production convergence.

## Research Context

### Experimental Parameters (from Paper)

These values are empirically optimized for the 185-node Hyperseed graph:

| Parameter | Value | Justification |
|-----------|-------|---------------|
| Pheromone influence (α) | 0.7 | Strong bias toward proven paths |
| Decay rate (ρ) | 0.1 (10%) | Moderate memory retention |
| Decay interval | 50 iterations | Balance freshness/stability |
| Max pheromone (τ_max) | 100.0 | Prevent infinite accumulation |
| Deposit amount (Δτ) | 1.0 | Standard unit per traversal |

### Key Findings from Research Paper

1. **95% cost reduction** within 10 iterations (matches classical ACO performance)
2. **Power-law distribution** emerges naturally (94.9% concentration in top 10%)
3. **TEMPORAL ORDERINGS I** emerged as dominant hub (avg 76.58 pheromone/edge)
4. **Statistical robustness**: 95% CI [94.12%, 95.73%] across 1,000 Monte Carlo runs

### Comparison to Classical ACO Variants

| Algorithm | Convergence | Quality | Real-time | Graph DB |
|-----------|-------------|---------|-----------|----------|
| AS (1992) | 10-20 iter | 90-95% | No | No |
| MMAS (2000) | 5-15 iter | 95-98% | No | No |
| ACS (1997) | 3-10 iter | 96-99% | No | No |
| **STAN (2025)** | **10 iter** | **95%** | **Yes** | **Yes** |

STAN's unique contribution: persistent pheromones in graph database enabling continuous learning across system restarts.

## File Structure

```
STAN/
├── src/
│   ├── query_modules/
│   │   └── stigmergy_astar.py      # Memgraph procedures
│   ├── api/
│   │   └── routes.py                # FastAPI endpoints
│   ├── mcp_server.py                # Model Context Protocol server
│   └── main.py                      # Application entry point
├── tests/
│   ├── test_procedures.py           # Unit tests (9/9 passed)
│   ├── test_convergence.py          # Integration test
│   └── test_mass_navigation.py      # 40k routes test
├── data/
│   ├── hyperseed_ontology.json      # 185 concepts graph
│   └── mass_navigation/             # Test results
├── docs/
│   ├── STAN_Research_Paper_Dey_2025.pdf
│   └── API_SPEC.md
├── docker-compose.yml               # Memgraph + API deployment
├── kubernetes/                      # K8s manifests
└── STAN_Research_Paper_Dey_2025.pdf # Full research paper
```

## Citation

If using STAN in research, cite:

```
Dey, R. (2025). STAN: Stigmergic A* Navigation for Emergent Intelligence
in Graph Pathfinding. Vicharatmak Buddhi Research Labs (VBRL.ai).
```

## Contact

- **Author**: Robin Dey
- **Organization**: VBRL.ai (Vicharatmak Buddhi Research Labs)
- **Email**: robin@vbrl.ai
