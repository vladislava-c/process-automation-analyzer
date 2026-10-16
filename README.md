# Process Automation Analyzer

An LLM tool that takes a business process described in plain words, splits it into steps, labels each step **AUTOMATE / ASSIST / MANUAL**, and suggests which Microsoft tool to use. I built it to learn how to test an AI tool properly, so the evaluation is the main part of this project.

<img width="1487" height="345" alt="image" src="https://github.com/user-attachments/assets/367f4373-5fd9-45ec-8b07-51ecd90edde9" />
<img width="1493" height="316" alt="image" src="https://github.com/user-attachments/assets/0a4fa47a-edb4-46da-88aa-e91e0a0015f9" />


## What it does

Paste a process description. The tool returns a table:

| Column | Meaning |
|---|---|
| Process | short title of the process |
| Step | one step of the process |
| Label | `AUTOMATE` (rule-based, no judgment), `ASSIST` (AI or software prepares, a human reviews), `MANUAL` (needs human judgment, negotiation or accountability) |
| Tool | one of: Power Automate, Power Apps, Copilot, SharePoint, Outlook, Teams, Microsoft Forms, Power BI (`-` for MANUAL) |

## How it works

1. A Gradio page takes the process text.
2. The text goes to an OpenAI model ([model name]) together with my instructions: label definitions and explicit tool-selection rules written from my experience with banking process automation.
3. The model returns JSON, which is shown as a table.
4. Label and tool names are checked and normalised (for example, capitalisation of tool names).

## Evaluation

**Question:** do the tool's labels and tool choices match an expert's?

### Method

- I created **20 invented processes (104 steps)** with an answer key: a label and a tool for every step. No employer data was used.
- The answer keys were drafted with AI assistance and then reviewed and adapted by me.
- The processes were split into three sets:
  - **Tuning set:** 10 processes, 50 steps. Used to improve the instructions.
  - **Held-out set:** 5 processes, 25 steps. Written before the tuning and not used while tuning.
  - **Hard set:** 5 processes, 29 steps, with ambiguous steps, approvals and informal wording. Scored once, with no changes afterwards.
- In the evaluation, the steps were already split, so the scores measure labelling and tool choice, not step splitting.
- **Label accuracy:** the model's label equals my label.
- **Tool accuracy:** the model's tool equals my tool, counted only for steps my key marks AUTOMATE or ASSIST.

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

- **v1 (baseline):** label definitions, free choice of Microsoft tools. The model rarely used ASSIST (3 of 11 correct) and often named tools differently from my key.
- **v2:** exact tool names, tool-selection rules, and a definition of ASSIST ("AI prepares, a human reviews"). Labels 82%, tools 52%.
- **v3:** improved the instructions. Labels 84%, tools 60%.

### Findings

- Label accuracy was about the same on the tuning set (84%) and the hard set (83%), so the tuning does not obviously overfit. The clean held-out set (100%) was easy.
- **The model never labelled a MANUAL step as automatable**, in any of the 104 steps.
- The main weakness: **11 of the 13 label errors** in the tuned and hard sets are the model choosing MANUAL where I chose ASSIST or AUTOMATE. It tends to treat "a human is involved" as "a human must do this", and confuses a human *reviewing* AI output (ASSIST) with a human *doing* the work. ASSIST was the hardest label (7/11 on the tuning set, 2/6 on the hard set).
- Tool accuracy stopped improving at about 58%. Many steps have two defensible tools (for example, Outlook or Power Automate for sending an email), so a one-answer key understates the tool's real usefulness. 

### Limitations

- One person's judgment defines the answer key, and the keys were drafted with AI assistance.
- Small sample (104 steps in total). One step moves a set's accuracy by 2 to 4 points, and the 100% result covers only 25 steps.
- All processes are invented and fairly short, so real documents may be harder.
- In the interface, the model splits the process into steps itself. This was not evaluated, because the evaluation used pre-split steps. Different step boundaries could change the results.
- The tool list is limited to eight Microsoft products. Tool suggestions are a first draft, not a verified design.
- The output is a plan, not a working automation. A person still has to build and test it.
- The labels test how well the model reproduces my judgment, not whether automating a step is a good idea.

## Run it yourself (Google Colab)

1. Open the notebook: `https://colab.research.google.com/drive/1u15198Nbh5zj5f27VOTAR3w_af8e6sVY?authuser=1#scrollTo=1v9fBBESnnK5`
2. Add your OpenAI API key under Secrets (key icon), named `API`, with notebook access switched on.
3. Run the cells in order. The interface appears under the last cell.

```
pip install openai gradio
```

The full evaluation costs a few francs of API usage.

## Repository contents

- `Process_to_automations_ideas.ipynb`: the tool, the answer keys, the evaluation and the interface
- `README.md`: this file

## Possible next steps

- Count a defined list of acceptable tools as correct
- Add a rule or example for "human reviews AI output" to reduce the MANUAL bias
- Evaluate step splitting by comparing the model's steps with mine
- Read Word or PDF process documents instead of pasted text
- Compare models on cost vs. accuracy
- Host the interface permanently, for example on Hugging Face Spaces

## Author

Vladislava_C
