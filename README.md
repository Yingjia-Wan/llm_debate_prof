# LLM Debate

## Overview

Code release for "Debating with More Persuasive LLMs Leads to More Truthful Answers"

## Setup

### Prerequisites

- Python 3.11
- Virtual environment tool (e.g., virtualenv)

### Installation

1. **Create and Activate a Virtual Environment:**
    ```bash
    virtualenv --python python3.11 .venv
    source .venv/bin/activate
    ```
2. Install Required Packages:
    ```bash
    pip install -r requirements.txt
    ```
3. Install Pre-Commit Hooks:
    ```bash
    make hooks
    ```
4. Create a SECRETS file
    ```txt
    API_KEY=<KEY>
    ANTHROPIC_API_KEY=<KEY>
    DEFAULT_ORG=
    ```
5. If you want to use the frontend also add the following
    ```txt
    DB_USER=psqluser
    DB_PASSWORD=password
    DB_HOST=localhost
    DB_PORT=5432
    DB_NAME=debate
    ```

## Usage
### Minimal Script
We recommend our minimal script to run everything end-to-end: `./scripts/reproduce_minimal.sh`. This will run all protocols (blind, consultancy, debate, interactive debate, expert judge) with GPT-4-Turbo expert and non-experts on 10 questions from QuALITY. Then run notebook `scripts/plot_minimal.ipynb` to generate a minimal plot similar to Figure 1 in the paper.
Running `./scripts/run_figure1.sh` will reproduce the LLM results in Figure 1 but uses best-of-16 so watch your OpenAI usage limit.

### Running Custom Experiments

```bash
# Run a debate and judgement on one question from the train-set of QuALITY
python -m core.main exp_dir='./exp/test1' +experiment='debate' +index=0 +swap=False

# Use a Claude 2.1 debater using best-of-8 and only 2 rounds of debate
python3 -m core.main exp_dir='./exp/test2' +experiment='debate' +index=0 +swap=False \
    ++correct_debater.BoN=8 \
    ++incorrect_debater.BoN=8 \
    ++correct_debater.language_model.model='claude-2.1' \
    ++incorrect_debater.language_model.model='claude-2.1' \
    ++rollout.num_steps=2 \
    ++anthropic_num_threads=10

# Debate with GPT-4-Turbo models and best-of-4 followed by judging and scoring on the train split of QuALITY
python -m core.debate \
    exp_dir='./exp/test3' \
    +experiment='debate'\
    ++correct_debater.language_model.model='gpt-4-1106-preview' \
    ++incorrect_debater.language_model.model='gpt-4-1106-preview' \
    ++correct_debater.BoN=4 \
    ++incorrect_debater.BoN=4 \
    ++max_num_from_same_story=5 \
    ++split=train
python -m core.judge \
    exp_dir='./exp/test3' \
    +experiment='debate'\
    ++judge.language_model.model='gpt-4-1106-preview' \
    ++judge_name='gpt-4-turbo'
python -m core.scoring.accuracy \
    exp_dir='./exp/test3' \
    +experiment='debate'\
    ++judge_name='gpt-4-turbo'

# Correct Consultantacy
python -m core.debate  \
    exp_dir='./exp/test3' \
    +experiment='consultancy' \
    method_type='correct'

# Incorrect Consultantacy
python -m core.debate  \
    exp_dir='./exp/test3' \
    +experiment='consultancy' \
    method_type='incorrect'
```

We can also call methods within other python scripts:

```python
from core.debate import main as debate
from hydra import compose, initialize

with initialize(version_base=None,
    config_path="../core/config/quality",
    job_name="test4"):

    cfg = compose(
        config_name="config",
        overrides=["method=debate","exp_dir=./exp/test4"])
    debate(cfg)
```

### Datasplits
* `T_h`: Testset used in the human trial with 47 questions. Use options: `++max_num_from_same_story=5 ++split=both ++human_experiments=8`
* `T_l`: Testset used by LLM experiments with 400 questions. Use options `++max_num_from_same_story=5 ++split=train`
* `D_l`: Development set used by LLM experiments with 291 questions. Use options `++max_num_from_same_story=5 ++split=dev`


### Data format

Each csv requires has following fields. This csv file is generated by load/quality.py and it is then populated with debate transcripts by `debate.py` and judge answers by `judge.py`.

| Field          | Type   | Description                                       |
| -------------- | ------ | ------------------------------------------------- |
| id             | `str`  | uuid from dataset                                 |
| question       | `str`  | question (does not include answers)               |
| correct answer | `str`  | the answer to the question                        |
| negative answer| `str`  | misleading or incorrect answer                    |
| complete       | `bool` | used by pipelines to check stage is complete      |
| transcript     | `str`  | entire debate transcript, generated by debate     |
| answer         | `str`  | predicted answer generated by judge               |

## Web Interface

### Running the frontend

```bash
cd web/frontend && npm install && npm run dev
```

### Running the backend

```bash
uvicorn web.backend.main:app --reload
```

## Repository Structure

- `core/main.py`: Script for testing debate and consultancy protocols on one question.
- `core/debate.py`: Script for running debate protocol on a set of questions.
- `core/judge.py`: Script for running a judge on the output transcripts of debate.py.
- `core/scoring/accuracy.py`: Script for computing accuracy of judge answers.
- `core/tournament.py`: Script for running a cross-play tournament with many debaters.
- `core/scoring/rating.py`: Script for computing aggregate Elo rating debaters.
- `core/llm_api`: Directory containing modules for async LLM inference
- `scripts`: Contains the scripts to reproduce the main figures in the paper and otehr minimal examples.
- `scripts/human_trial_example`: Contains the script and config to reproduce a human experiment.
- `data`: zip files containing LLM debates from the human trial
- `web`: Contains the code for the web frontend and backend to host a human trial or view debate transcripts.


## Contributing

Contributions to this repository are welcome. Please follow the standard procedures for submitting issues and pull requests.

---
