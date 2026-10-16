# Process Automation Analyzer

An LLM tool that breaks a business process into steps, labels each step **AUTOMATE / ASSIST / MANUAL**, and suggests which Microsoft tool to use, with a short build outline and risk note.

## What it does

Paste a process description in plain words. The tool returns a table with:

| Column | Meaning |
|---|---|
| step | one step of the process |
| label | `AUTOMATE` (rule-based, no judgment), `ASSIST` (AI prepares, a human reviews), `MANUAL` (needs human judgment or accountability) |
| tool | one of: Power Automate, Power Apps, Copilot, SharePoint, Outlook, Teams, Microsoft Forms, Power BI |
| how / risk | a short build outline and the main risk |

## How it works

1. A Gradio page takes the process text.
2. The text and my instructions go to an OpenAI model ([model name]) through the API, requesting JSON output.
3. The JSON is shown as a table.
4. The instructions contain the label definitions and explicit tool-selection rules, written from my experience with banking process automation.

## Evaluation

The question was whether the tool's labels and tool choices match an expert's. A tool that sounds confident can still be wrong, so I measured it.

### Method

- I wrote **20 invented processes (104 steps)** with my own answer key: a label and a tool for each step. No employer data was used.
- Processes were split into three sets:
  - **Tuning set:** 10 processes, 50 steps. Used to improve the instructions.
  - **Held-out set:** 5 processes, 25 steps. Written before the instructions were tuned and not used during tuning.
  - **Hard set:** 5 processes, 29 steps. Ambiguous steps, approvals, and informal, messy wording. Scored once, with no changes afterwards.
- The model receives the steps already split, so answers can be compared row by row.
- **Label accuracy:** the model's label matches my key.
- **Tool accuracy:** the model's tool matches my key, counted only for steps the key marks AUTOMATE or ASSIST.

### Results

| Set | Steps | Label accuracy | Tool accuracy |
|---|---|---|---|
| Tuning set, baseline instructions | 50 | 76% (38/50) | 32% (13/40) |
| Tuning set, after tuning | 50 | 84% (42/50) | 60% (24/40) |
| Held-out, clean | 25 | 100% (25/25) | 58% (11/19) |
| Hard set | 29 | 83% (24/29) | 57% (12/21) |
| **Both unseen sets combined** | 54 | **91% (49/54)** | **58% (23/40)** |

### Confusion matrices (rows: my label, columns: model's label)

Tuning set, after tuning:

| | ASSIST | AUTOMATE | MANUAL |
|---|---|---|---|
| **ASSIST** | 7 | 1 | 3 |
| **AUTOMATE** | 1 | 25 | 3 |
| **MANUAL** | 0 | 0 | 10 |

Held-out set:

| | ASSIST | AUTOMATE | MANUAL |
|---|---|---|---|
| **ASSIST** | 6 | 0 | 0 |
| **AUTOMATE** | 0 | 13 | 0 |
| **MANUAL** | 0 | 0 | 6 |

Hard set:

| | ASSIST | AUTOMATE | MANUAL |
|---|---|---|---|
| **ASSIST** | 2 | 0 | 4 |
| **AUTOMATE** | 0 | 14 | 1 |
| **MANUAL** | 0 | 0 | 8 |

### What changed between versions

- **v1 (baseline):** label definitions and a free choice of Microsoft tools. The model rarely used ASSIST (3 of 11 correct) and often named tools in a different form than the key.
- **v2:** added exact tool names, tool-selection rules, and a definition of ASSIST ("AI prepares, a human reviews"). Label accuracy 82%, tool accuracy 52%.
- **v3:** [describe the further changes you made]. Label accuracy 84%, tool accuracy 60%.

### Findings

- Label accuracy was about the same on the tuning set (84%) and the hard set (83%), so the tuning did not obviously overfit. The clean held-out set (100%) was easy.
- **The model never labelled a human-judgment step (MANUAL) as automatable**, in any of the 104 steps.
- The consistent weakness is the opposite direction: the model treats "a human is involved" as MANUAL. It confuses a human *reviewing* AI output (ASSIST) with a human *doing* the work. 9 of the 13 label errors in the tuned and hard sets are of this kind (ASSIST or AUTOMATE called MANUAL).
- Tool accuracy plateaued at about 58% across all sets. Many steps have two defensible tools (for example, Outlook vs. Power Automate for sending an email), so a one-answer key understates real quality. [Optional: add a second figure counting my listed acceptable alternatives.]

### Limitations

- One author wrote the processes and the key, so the key reflects one person's judgment.
- Small sample: 104 steps in total. A single step moves accuracy by 2 to 4 points on any one set, and the 100% result covers only 25 steps.
- All processes are invented and written in a similar style, so real-world documents may be harder.
- The tool list is restricted to eight Microsoft products, and the tool suggestions are a first draft, not a verified design.
- The output is a plan, not a working flow. A person still has to build and test the automation.
- The label tests how well the model reproduces my judgment, not whether the automation is a good idea.

## Run it yourself (Google Colab)

1. Open the notebook in Google Colab: `[link]`
2. Add your OpenAI API key under Secrets (key icon), named `API`, with notebook access on.
3. Run the cells in order. The Gradio interface appears under the last cell.

```
pip install openai gradio
```

The evaluation cells use the answer keys in the notebook. Expect a cost of a few francs for the full evaluation.

## Repository contents

- `[notebook name].ipynb`: the tool, the answer keys, and the evaluation
- `README.md`: this file

## Possible next steps

- Count alternative acceptable tools as correct, defined in advance
- Add a rule or example for "human reviews AI output" to reduce the MANUAL bias
- Read process documents (Word, PDF) instead of pasted text
- Compare models on cost vs. accuracy
- Host the interface permanently, for example on Hugging Face Spaces

## Author

Vladislava_C
