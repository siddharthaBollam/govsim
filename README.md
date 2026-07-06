# GovSim: Governance of the Commons Simulation

This project is a reproduction and extension of the NeurIPS 2024 paper
**"Cooperate or Collapse: Emergence of Sustainable Cooperation in a Society of LLM Agents"**
by Piatti, Jin, Kleiman-Weiner, Schölkopf, Sachan, and Mihalcea.

The project runs five LLM-driven agents through repeated rounds of resource-sharing dilemmas
modelled on the tragedy of the commons. Agents must decide how much of a shared resource to
take each month. If they act greedily, the resource collapses. If they cooperate, it survives.
The simulation tests whether language models can reason about fairness, sustainability, and
collective consequences — with and without a universalization prompt that asks agents to
consider what would happen if everyone acted the same way.

---

## What This Project Does

Three scenarios are implemented, each sharing the same agent architecture and simulation loop:

- **Fishing** — five agents harvest fish from a shared lake each month. The lake replenishes
  based on what remains. Overfishing depletes it permanently.
- **Sheep grazing** — five shepherds graze flocks on shared pastureland. Overgrazing degrades
  the grass available the following month.
- **Pollution** — five factory operators share a river's pollution capacity. Exceeding it
  causes cumulative environmental damage.

Each agent follows a cognitive loop every round: it perceives the current resource state,
retrieves relevant memories from past rounds, decides how much to take, participates in a
group conversation with other agents, and reflects on the outcome. The simulation runs for
up to 12 months per seed. A run either survives all 12 months or collapses when the
resource hits zero.

Two conditions are tested per model:

- **Baseline** — agents reason normally with no additional prompting strategy
- **Universalization** — agents are prompted to ask themselves: *"What if everyone did this?"*
  before making a decision, forcing consideration of collective consequences

---

## Results

59 valid runs were completed across two models, two conditions, three scenarios, and five
seeds.

| Model | Condition | Survival Rate |
|---|---|---|
| GPT-4o (`gpt-4o-2024-05-13`) | Baseline | ~60% |
| GPT-4o (`gpt-4o-2024-05-13`) | Universalization | 100% |
| Llama-3-70B (`meta-llama/llama-3-70b-instruct`) | Baseline | 0% |
| Llama-3-70B (`meta-llama/llama-3-70b-instruct`) | Universalization | ~86% |

Universalization produced a large, consistent improvement for both models. Llama-3-70B
collapsed in every single baseline run and recovered to 86% survival with the universalization
prompt — the largest observed jump across both models. GPT-4o went from 60% to a perfect
survival rate.

These results extend the paper's original universalization findings to two models the
original authors did not test.

---

## Metrics

Six metrics are computed automatically from raw trajectory logs after each run:

| Metric | What it captures |
|---|---|
| **Survival Rate** | Percentage of runs where the resource lasted all 12 months |
| **Survival Time** | Average months survived before collapse (if it collapsed) |
| **Efficiency** | How close total harvest came to the maximum sustainable yield |
| **Equality** | How evenly the harvest was distributed across all five agents |
| **Over-usage** | Rate of individual agents exceeding their per-agent fair-share threshold |
| **Mean Gain** | Average total resource collected per agent across the run |

---

## Tech Stack

| Layer | Tools |
|---|---|
| Language | Python 3.11 |
| Config management | Hydra 1.3 |
| LLM backends | OpenAI, Anthropic, OpenRouter, HuggingFace Transformers, vLLM |
| Structured generation | pathfinder (vendored) |
| Experiment tracking | Weights & Biases (offline mode) |
| Analysis dashboard | Dash, Plotly, Flask |
| Prompt templating | Jinja2 |
| Environment | conda + pip |

---

## Baseline Installation

Requires conda and Python 3.11.

```bash
git clone --recurse-submodules https://github.com/giorgiopiatti/GovSim.git
cd GovSim
bash ./setup.sh
```

For local GPU inference via vLLM:

```bash
bash ./setup_vllm.sh
```

---

## Configuration

Create a `.env` file in the repo root:

```env
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
OPENROUTER_API_KEY=...
MISTRAL_API_KEY=...
HF_TOKEN=...
```

Primary config at `simulation/conf/config.yaml`:

```yaml
seed: 42
llm:
  path: gpt-4o-2024-05-13
  backend: openai
  is_api: true
  temperature: 0.0
  top_p: 1.0
defaults:
  - experiment: basic
```

Any value can be overridden via CLI without touching the file.

---

## Usage

```bash
python3 -m simulation.main experiment=<id> llm.path=<model>
```

**Experiment IDs:**

| Scenario | Baseline | + Universalization | No-language ablation |
|---|---|---|---|
| Fishing | `fish_baseline_concurrent` | `fish_baseline_concurrent_universalization` | `fish_perturbation_no_language` |
| Sheep | `sheep_baseline_concurrent` | `sheep_baseline_concurrent_universalization` | `sheep_perturbation_no_language` |
| Pollution | `pollution_baseline_concurrent` | `pollution_baseline_concurrent_universalization` | `pollution_perturbation_no_language` |

**Examples:**

```bash
# GPT-4o fishing baseline, seed 1
python3 -m simulation.main \
  experiment=fish_baseline_concurrent \
  llm.path=gpt-4o-2024-05-13 \
  seed=1

# Llama-3-70B universalization via OpenRouter
python3 -m simulation.main \
  experiment=fish_baseline_concurrent_universalization \
  llm.path=meta-llama/llama-3-70b-instruct \
  llm.backend=openai \
  llm.is_api=true

# Mixed-LLM mode (different model per agent)
python3 -m simulation.main \
  --config-name=multiple_llm \
  experiment=fish_baseline_concurrent
```

**Subskills — isolated single-shot reasoning tests (no memory or conversation):**

```bash
python3 -m subskills.fishing llm.path=gpt-4o-2024-05-13
python3 -m subskills.sheep llm.path=gpt-4o-2024-05-13
python3 -m subskills.pollution llm.path=gpt-4o-2024-05-13
```

**Analysis dashboard:**

```bash
python3 -m simulation.analysis.app
```

---

## Project Structure

```
govsim/
├── simulation/
│   ├── main.py                          # Entry point — builds models, dispatches to scenario
│   ├── conf/                            # Hydra base configs
│   ├── scenarios/
│   │   ├── fishing/                     # Fishing scenario
│   │   ├── sheep/                       # Sheep grazing scenario
│   │   └── pollution/                   # Pollution scenario
│   │       ├── run.py                   # Monthly simulation loop
│   │       ├── environment/             # Resource math, state transitions, pool updates
│   │       ├── agents/persona_v3/       # Full cognitive architecture per agent
│   │       └── conf/                    # Scenario-specific Hydra configs
│   ├── analysis/
│   │   ├── app.py                       # Dash dashboard
│   │   ├── preprocessing.py             # Metric computation from log_env.json
│   │   └── plots.py                     # Chart generation
│   └── utils/                           # WandbLogger, ModelWandbWrapper
├── subskills/
│   ├── fishing/ sheep/ pollution/       # Isolated reasoning tests, no agent memory
│   └── analysis/                        # Subskill metric aggregation
├── pathfinder/                          # Vendored LLM client library
│   ├── model.py                         # get_model() — main entry, called by main.py
│   ├── loader.py                        # Backend routing and initialization
│   ├── _gen.py / _select.py / _find.py  # Free-form, constrained, structured generation
│   └── trie.py                          # Token trie for constrained decoding
├── utils/charts.py                      # Standalone plotting helpers
├── generate_random_group.py             # Random persona group generator
├── setup.sh / setup_vllm.sh            # Environment setup scripts
└── requirements.txt
```

---

## Acknowledgements

- Original authors: Giorgio Piatti, Zhijing Jin, Max Kleiman-Weiner,
  Bernhard Schölkopf, Mrinmaya Sachan, Rada Mihalcea
- [Paper — NeurIPS 2024](https://arxiv.org/abs/2404.16698)
- [pathfinder](https://github.com/giorgiopiatti/pathfinder) — vendored LLM client by the same authors
