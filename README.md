# Case Closed Competition — Voronoi Territorial Player · Placed 3rd @ TAMU Datathon (303 participants)

An autonomous Python agent for **Case Closed**, a Tron-style grid duel where every move leaves a permanent trail. This repository ships a production-ready Flask server (`agent.py`) that plays head-to-head matches via the official Judge Engine API.

---

## How It Works

### Match architecture

Each match is a client–server loop between two agent (this vs competitor) processes and a central judge:


1. **Move request** — The agent must respond within **1.5 s** or forfeit the turn (5 random-move fallbacks, then forfeit).
2. **Simultaneous resolution** — Both moves are applied on an **18 × 20 torus** board. Trails are permanent walls; hitting any trail (including your own) is fatal. Head-on head collisions are a draw.
3. **Termination** — Game ends on crash, draw, 500 judge turns, or 200 in-game turns (longer trail wins).


## Our Strategy

### 1. Voronoi territorial scoring

For each candidate move, we simulate occupying the path cells and compute BFS distance maps from **our new head** and the **opponent head** on the torus grid. Every empty cell is assigned to whichever agent reaches it first (ties are neutral):

\[
\text{territory\_score} = w_T \cdot \bigl(|\mathcal{V}_{\text{me}}| - |\mathcal{V}_{\text{opp}}|\bigr)
\]

where \(\mathcal{V}_{\text{me}}\) and \(\mathcal{V}_{\text{opp}}\) are the Voronoi cell sets under graph shortest-path distance.

### 2. Local safety features (Go-inspired)

| Feature | Meaning |
|---------|---------|
| **Liberties** | Count of empty orthogonal neighbors at the landing cell |
| **Component size** | Size of the connected empty region reachable from our head |
| **Choke penalty** | Heavy penalty if component shrinks below 25% or 50% of current size |
| **Dead-pocket penalty** | \(-10{,}000\) if liberties \(= 0\) or component \(\leq 2\) |

### 3. Opponent modeling

The last 12 opponent head positions are tracked. Two modes are inferred:

- **Serpentine** — Dominant horizontal movement with periodic vertical steps (cycle-following opponents).
- **Aggressive** — Manhattan distance to us is decreasing over recent history.

Weights adapt: higher risk aversion vs. aggressive opponents; slightly higher territorial weight vs. serpentine opponents.

### 4. Phase-dependent weights

Board fill ratio \(\rho = \frac{\text{empty cells}}{H \times W}\) selects opening, midgame, or endgame weight tables:

| Phase | Condition | Emphasis |
|-------|-----------|----------|
| Opening | \(\rho > 0.6\) | Territory (\(w_T = 1.0\)) |
| Midgame | \(0.3 < \rho \leq 0.6\) | Balanced territory + liberties |
| Endgame | \(\rho \leq 0.3\) | Liberties + component survival (\(w_{\text{lib}} = 0.55\)) |

### 5. Risk and collision avoidance

- Precompute opponent 1-step and 2-step (boost) reachable cells.
- Penalize landing on or passing through those cells.
- Penalize close proximity (\(d_{\text{opp}} \leq 2\)) at the landing cell.

### 6. Graph-theoretic extras

- **Articulation heuristic** — Detect whether occupying a cell would split empty space into multiple components (bridge detection); slight bonus when splitting may isolate opponent territory.
- **Sealing bonus** — Reward moves adjacent to two or more of our own trail cells (loop-enclosure potential).

### 7. Lookahead layers

| Method | Depth | Purpose |
|--------|-------|---------|
| **Beam search** | 2 plies (our move → our follow-up) | Avoid greedy one-step traps |
| **Monte Carlo rollouts** | 3 random safe steps × 6–10 trials | Estimate post-move mobility stability |

Top-3 candidates from the heuristic pass are refined; the final move maximizes the combined score.

### 8. Boost policy

Each agent has **3 boosts** (move twice in one direction). Boost candidates are generated only when both steps are safe. A phase-dependent penalty discourages wasteful boost use in the opening; endgame allows more aggressive spending.

---

## Testing

### API compliance (`local-tester.py`)

Run with `agent.py` already listening on port 5008:

```bash
python local-tester.py
```

Checks: `GET /`, `POST /send-state`, `GET /send-move` (including `:BOOST` format), `POST /end`.

### Head-to-head simulation

Terminal 1 — your agent (port 5008):

```bash
python agent.py
```

Terminal 2 — baseline opponent (port 5009):

```bash
python sample_agent.py
```

Terminal 3 — judge:

```bash
python judge_engine.py
```

`sample_agent.py` implements a **simpler Voronoi baseline** (single-pass scoring, no opponent modeling, beam search, or rollouts). Matches against it validate that the full agent outperforms the baseline under the same rules.

### Docker smoke test

```bash
docker build -t case-closed-agent .
docker run -p 5008:5008 case-closed-agent
```

Then re-run `local-tester.py` against the container.

### Operational constraints tested against

- **1.5 s move timeout** per judge request (two retries, then random fallback).
- **No 180° reversals** — invalid opposite-direction moves are rejected by the judge.
- **Torus wrap-around** — all coordinate math uses modular arithmetic.

---


### Torus (wraparound) geometry

The board is a flat torus \(T^2 = \mathbb{Z}_W \times \mathbb{Z}_H\). Position normalization:

\[
x' = x \bmod W, \quad y' = y \bmod H
\]

**Torus Manhattan distance** between cells \((x_1,y_1)\) and \((x_2,y_2)\):

\[
d_{\text{torus}}(p_1, p_2) = \min(|x_1 - x_2|, W - |x_1 - x_2|) + \min(|y_1 - y_2|, H - |y_1 - y_2|)
\]

BFS on the 4-connected grid with torus edges produces the **graph shortest-path metric** used for Voronoi partitioning (not Euclidean Voronoi).

### BFS distance map

Given occupied set \(O\) and start cell \(s\), the distance map \(\delta_s : V \to \mathbb{N} \cup \{-1\}\) satisfies:

\[
\delta_s(v) = \min\{ k \mid \exists \text{ path of length } k \text{ from } s \text{ to } v \text{ through cells in } V \setminus O \}
\]

with \(\delta_s(v) = -1\) if unreachable. Computed in \(O(WH)\) per call; opponent distances are cached and recomputed only when our candidate path blocks previously reachable cells.

### Voronoi partition on a grid

For empty cell \(c\):

\[
c \in \mathcal{V}_{\text{me}} \iff \delta_{\text{me}}(c) < \delta_{\text{opp}}(c) \quad \text{(strict inequality)}
\]

Ties are ignored (neutral), which avoids over-counting contested frontier cells.

### Full scoring function

For candidate move \(m\) with landing cell \(\ell\):

\[
S(m) = w_T \Delta\mathcal{V} + w_L \cdot \text{lib}(\ell) + w_C \cdot |C(\ell)| - P_{\text{choke}} - P_{\text{risk}} - P_{\text{boost}} - P_{\text{dead}} + B_{\text{seal}} + B_{\text{straight}}
\]

where \(\Delta\mathcal{V} = |\mathcal{V}_{\text{me}}| - |\mathcal{V}_{\text{opp}}|\), \(\text{lib}(\ell)\) is liberties, \(|C(\ell)|\) is connected empty component size, and penalty terms encode collision risk, choke detection, and dead-end avoidance.

Beam and rollout refinements add:

\[
S_{\text{final}}(m) = S(m) + S_{\text{beam}}(m) + 0.05 \cdot \overline{\text{lib}}_{\text{rollout}}(m)
\]

### Connected components and articulation

The empty cells form a subgraph of the grid graph. **Liberties** count the degree of the landing vertex in that subgraph. **Component size** is the order of the connected component containing the head after the move.

An **articulation point** (cut vertex) in graph theory is a vertex whose removal increases the number of connected components. Our `is_articulation` heuristic approximates this: if occupying \((x,y)\) would disconnect multiple empty neighbor regions, the cell acts as a bridge — valuable for territory sealing.

---

## References & Related Work

| Topic | Reference |
|-------|-----------|
| **Hamiltonian cycle strategies** | Applegate et al., *The Traveling Salesman Problem* (1998) — Hamiltonian paths underpin serpentine full-board coverage strategies common in Tron AI |
| **Voronoi / territory evaluation** | Berlekamp, Conway & Guy, *Winning Ways for Your Mathematical Plays* (1982) — territorial decomposition in combinatorial games |
| **Go liberties & life-and-death** | Benson's algorithm for unconditional life (1976); liberty counting as mobility heuristic |
| **Articulation points** | Tarjan (1972), depth-first search for biconnected components |
| **Monte Carlo rollouts** | Browne et al., "A Survey of Monte Carlo Tree Search Methods" (2012) — rollout-based policy evaluation (we use a lightweight 3-step variant, not full MCTS) |
| **Multi-agent grid games** | Surakarta / Tron bot competitions — Voronoi and flood-fill heuristics are standard baselines when full game-tree search is intractable (\(O(b^d)\) with \(b \approx 4\), \(d \leq 500\)) |

---

### Strengths
- **Principled territory model** — Voronoi scoring directly optimizes space advantage, aligned with win condition (survive + outlast).
- **Multi-layer safety** — Component size, liberties, choke detection, and opponent reachability reduce self-trap deaths.
- **Adaptive play** — Phase weights and opponent modeling adjust without retraining.
- **Low latency** — Pure Python heuristics with BFS caching; no GPU or heavy ML dependencies.
- **Torus-correct** — All distance and neighbor logic respects wrap-around.

### Weaknesses
- **No full opponent lookahead** — Beam search assumes opponent does not interfere on ply 2; rollouts use random moves, not adversarial ones.
- **Greedy Voronoi** — One-step territorial gain can miss long-horizon enclosure setups.
- **Heuristic articulation** — Bridge detection is local, not a full Tarjan pass each move.

### Opportunities
- Full **minimax / alpha-beta** on Voronoi score for 2–3 plies with opponent modeling.
- **Opening book** — Precomputed Hamiltonian serpentine for uncontested early game.
- **Stronger MCTS** — Replace random rollouts with playout policies biased toward territory.

### Threats
- Opponents designed to **bait into narrow corridors** early.
- **Timeout forfeit** if BFS + rollouts exceed 1.5 s on congested endgame boards.
- Adversaries that **break Voronoi assumptions** via unpredictable boosts.

---

# Case Closed Agent Template

### Explanation of Files

This template provides a few key files to get you started. Here's what each one does:

#### `agent.py`
**This is the most important file. This is your starter code, where you will write your agent's logic.**

*   DO NOT RENAME THIS FILE! Our pipeline will only recognize your agent as `agent.py`.
*   It contains a fully functional, Flask-based web server that is already compatible with the Judge Engine's API.
*   It has all the required endpoints (`/`, `/send-state`, `/send-move`, `/end`). You do not need to change the structure of these.
*   Look for the `send_move` function. Inside, you will find a section marked with comments: `# --- YOUR CODE GOES HERE ---`. This is where you should add your code to decide which move to make based on the current game state.
*   Your agent can return moves in the format `"DIRECTION"` (e.g., `"UP"`, `"DOWN"`, `"LEFT"`, `"RIGHT"`) or `"DIRECTION:BOOST"` (e.g., `"UP:BOOST"`) to use a speed boost.

#### `requirements.txt`
**This file lists your agent's Python dependencies.**

*   Don't rename this file either.
*   It comes pre-populated with `Flask` and `requests`.
*   If your agent's logic requires other libraries (like `numpy`, `scipy`, or any other package from PyPI), you **must** add them to this file.
*   When you submit, our build pipeline will run `pip install -r requirements.txt` to install these libraries for your agent.

#### `judge_engine.py`
**A copy of the runner of matches.**

*   The judge engine is the heart of a match in Case Closed. It can be used to simulate a match.
*   The judge engine can be run only when two agents are running on ports `5008` and `5009`.
*   We provide a sample agent that can be used to train your agent and evaluate its performance.

#### `case_closed_game.py`
**A copy of the official game state logic.**

*   Don't rename this file either.
*   This file contains the complete state of the match played, including the `Game`, `GameBoard`, and `Agent` classes.
*   While your agent will receive the game state as a JSON object, you can read this file to understand the exact mechanics of the game: how collisions are detected, how trails work, how boosts function, and what ends a match. This is the "source of truth" for the game rules.
*   Key mechanics:
    - Agents leave permanent trails behind them
    - Hitting any trail (including your own) causes death
    - Head-on collisions: both agents die (draw)
    - Each agent has 3 speed boosts (moves twice instead of once)
    - The board has torus (wraparound) topology
    - Game ends after 500 turns or when one/both agents die

#### `sample_agent.py`
**A simple agent that you can play against.**

*   The sample agent is provided to help you evaluate your own agent's performance. 
*   In conjunction with `judge_engine.py`, you should be able to simulate a match against this agent.
*   It implements a simplified Voronoi territorial baseline (single-pass scoring without opponent modeling or lookahead).

#### `local-tester.py`
**A local tester to verify your agent's API compliance.**

*   This script tests whether your agent correctly implements all required endpoints.
*   Run this to ensure your agent can communicate with the judge engine before submitting.

#### `Dockerfile`
**A copy of the Dockerfile your agent will be containerized with.**

*   This is a copy of a Dockerfile. This same Dockerfile will be used to containerize your agent so we can run it on our evaluation platform.
*   It is **HIGHLY** recommended that you try Dockerizing your agent once you're done. We can't run your agent if it can't be containerized.
*   There are a lot of resources at your disposal to help you with this. We recommend you recruit a teammate that doesn't run Windows for this. 

#### `.dockerignore`
**A .dockerignore file doesn't include its contents into the Docker image**

*   This `.dockerignore` file will be useful for ensuring unwanted files do not get bundled in your Docker image.
*   You have a 5GB image size restriction, so you are given this file to help reduce image size and avoid unnecessary files in the image.

#### `.gitignore`
*   A standard configuration file that tells Git which files and folders (like the `venv` virtual environment directory) to ignore. You shouldn't need to change this.


### Testing your agent:
**Both `agent.py` and `sample_agent.py` come ready to run out of the box!**

*   To test your agent, you will likely need to create a `venv`. Look up how to do this. 
*   Next, you'll need to `pip install` any required libraries. `Flask` is one of these.
*   Finally, in separate terminals, run both `agent.py` and `sample_agent.py`, and only then can you run `judge_engine.py`.
*   You can also run `local-tester.py` to verify your agent's API compliance before testing against another agent.


### Disclaimers:
* There is a 5GB limit on Docker image size, to keep competition fair and timely.
* Due to platform and build-time constraints, participants are limited to **CPU-only PyTorch**; GPU-enabled versions, including CUDA builds, are disallowed. Any other heavy-duty GPU or large ML frameworks (like Tensorflow, JAX) will not be allowed.
* Ensure your agent's `requirements.txt` is complete before pushing changes.
* If you run into any issues, take a look at your own agent first before asking for help.
