# Process Automation Analyzer

<a href="https://colab.research.google.com/github/vladislava-c/process-automation-analyzer/blob/main/Process_to_automation_ideas.ipynb" target="_parent"><img src="https://colab.research.google.com/assets/colab-badge.svg" alt="Open In Colab"/></a>

An LLM tool that takes a business process described in plain words, splits it into steps, labels each step **AUTOMATE / ASSIST / MANUAL**, and suggests which Microsoft tool to use. I built it to learn how to test an AI tool properly, so the evaluation is the main part of this project.

<img width="1487" height="345" alt="image" src="https://github.com/user-attachments/assets/367f4373-5fd9-45ec-8b07-51ecd90edde9" />
<img width="1493" height="316" alt="image" src="https://github.com/user-attachments/assets/0a4fa47a-edb4-46da-88aa-e91e0a0015f9" />


## What it does

Paste a process description. The tool returns a table:

| Column | Meaning |
|---|---|
| Step | one step of the process |
| Label | `AUTOMATE` (rule-based, no judgment), `ASSIST` (AI or software prepares, a human reviews), `MANUAL` (needs human judgment, negotiation or accountability) |
| Tool | one of: Power Automate, Power Apps, Copilot, SharePoint, Outlook, Teams, Microsoft Forms, Power BI (`-` for MANUAL) |
| Reason | one sentence from the model explaining the label |

A summary line counts the steps per label, and the table can be downloaded as CSV.

## How it works

1. A Gradio page takes the process text.
2. **Stage 1, split:** the model breaks the text into 3 to 8 steps. The steps appear in an editable box, so the user can fix them before labelling.
3. **Stage 2, label:** each step goes to the model together with my instructions: label definitions, explicit tool-selection rules written from my experience with banking process automation, and two worked examples.
4. The model answers with **structured outputs** (a strict JSON schema). Labels and tools can only be values from fixed lists, and the model writes its reason *before* the label.
5. Stage 2 is the same function that the evaluation scores, so the evaluated code is the code the interface runs.

## Evaluation

**Question:** do the tool's labels and tool choices match an expert's?

### Method

- I created **20 invented processes (104 steps)** with an answer key: a label and a tool for every step. No employer data was used.
- The answer keys were drafted with AI assistance and then reviewed and adapted by me.
- The processes were split into three sets:
  - **Tuning set:** 10 processes, 50 steps. Used to improve the instructions (v1 to v3).
  - **Held-out set:** 5 processes, 25 steps. Written before the tuning and not used while tuning.
  - **Hard set:** 5 processes, 29 steps, with ambiguous steps, approvals and informal wording. Scored once with v3. **In v4 I used its errors to change the instructions, so from v4 on it is a tuning set, not an unseen set** (see below).
- In the evaluation, the steps were already split, so the scores measure labelling and tool choice, not step splitting.
- **Label accuracy:** the model's label equals my label.
- **Tool accuracy:** for steps my key marks AUTOMATE or ASSIST.
  - v1 to v3: the model's tool equals my one tool.
  - v4: the model's tool is in my list of acceptable tools. Where I had noted a second defensible tool (for example Outlook or Teams for sending a list), both count. v4 tool scores are therefore not directly comparable with v1 to v3.

### Results (v1 to v3)

| Set | Steps | Label accuracy | Tool accuracy |
|---|---|---|---|
| Tuning set, baseline instructions | 50 | 76% (38/50) | 32% (13/40) |
| Tuning set, after tuning (v3) | 50 | 84% (42/50) | 60% (24/40) |
| Held-out, clean | 25 | 100% (25/25) | 58% (11/19) |
| Hard set | 29 | 83% (24/29) | 57% (12/21) |
| **Both unseen sets combined** | 54 | **91% (49/54)** | **58% (23/40)** |

### Results (v4, hard set)

Average of 3 runs at temperature 0 (29 steps each, 87 step-runs).

| Metric | v3 (1 run) | v4 (3 runs) |
|---|---|---|
| Label accuracy | 83% | 85% (per run: 86%, 83%, 86%) |
| ASSIST steps labelled ASSIST | 2/6 (33%) | 14/18 (78%) |
| Tool accuracy, all AUTOMATE/ASSIST steps | 57% (strict) | 71% (acceptable list) |
| Tool accuracy, only steps with the correct label | not measured | 85% |

### Confusion matrices (rows: my label, columns: model's label)

Tuning set, after tuning (v3):

| | ASSIST | AUTOMATE | MANUAL |
|---|---|---|---|
| **ASSIST** | 7 | 1 | 3 |
| **AUTOMATE** | 1 | 25 | 3 |
| **MANUAL** | 0 | 0 | 10 |

Held-out set (v3):

| | ASSIST | AUTOMATE | MANUAL |
|---|---|---|---|
| **ASSIST** | 6 | 0 | 0 |
| **AUTOMATE** | 0 | 13 | 0 |
| **MANUAL** | 0 | 0 | 6 |

Hard set (v3, 1 run):

| | ASSIST | AUTOMATE | MANUAL |
|---|---|---|---|
| **ASSIST** | 2 | 0 | 4 |
| **AUTOMATE** | 0 | 14 | 1 |
| **MANUAL** | 0 | 0 | 8 |

Hard set (v4, 3 runs added together):

| | ASSIST | AUTOMATE | MANUAL |
|---|---|---|---|
| **ASSIST** | 14 | 0 | 4 |
| **AUTOMATE** | 0 | 39 | 6 |
| **MANUAL** | 0 | 3 | 21 |

### What changed between versions

- **v1 (baseline):** label definitions, free choice of Microsoft tools. The model rarely used ASSIST (3 of 11 correct) and often named tools differently from my key.
- **v2:** exact tool names, tool-selection rules, and a definition of ASSIST ("AI prepares, a human reviews"). Labels 82%, tools 52%.
- **v3:** improved the instructions. Labels 84%, tools 60%.
- **v4:**
  - Removed a contradiction in the instructions. One rule said "extracting content from documents is done by a person (MANUAL)", while others said reading and summarising text is Copilot work. The rule now covers only copying fields out of scanned or attached files.
  - Added the rule "use ASSIST whenever AI could do the first pass, including reading, reviewing or summarising text", and "a check against a fixed threshold is AUTOMATE, a judgment about what looks unusual is ASSIST".
  - Added two worked examples (not taken from any evaluation set).
  - Structured outputs with a strict schema, and a reason written before each label.
  - Split the interface into two stages (split, then label) so the interface uses the evaluated function.
  - Evaluation: lists of acceptable tools, 3 runs at temperature 0, and a table of every miss with the model's reason.

### Findings

- Up to v3, label accuracy was about the same on the tuning set (84%) and the hard set (83%), so the tuning did not obviously overfit. The clean held-out set (100%) was easy.
- Up to v3, the model never labelled a MANUAL step as automatable. In v4 it did so for one step in all 3 runs ("Collect data from the different systems"), which my key had already marked as borderline.
- The main weakness up to v3: **11 of the 13 label errors** in the tuned and hard sets were the model choosing MANUAL where I chose ASSIST or AUTOMATE. It treats "a human is involved" as "a human must do this", and confuses a human *reviewing* AI output (ASSIST) with a human *doing* the work.
- v4 fixed most of the ASSIST problem (2/6 to 14/18 on the hard set) but did not remove the MANUAL bias. It moved: AUTOMATE steps labelled MANUAL rose from 1 to 2 per run. The model's reasons show the pattern, for example "requires human oversight" for removing unconfirmed accounts, and "involves human interaction" for telling sales a decision that was already made.
- About a third of the v4 label errors are on steps my own key marks as borderline. The answer key is part of the measurement, not ground truth.
- Tool errors in v4 are mostly about intake. The model picks Power Automate for "receive the request" steps, where I chose the channel the request arrives on (Forms, Outlook, Teams).

### Limitations

- One person's judgment defines the answer key, and the keys were drafted with AI assistance.
- Small sample (104 steps in total). One step moves a set's accuracy by 2 to 4 points, and the 100% result covers only 25 steps.
- **v4 was tuned on the hard set, so its hard-set score is optimistic.** v4 has not yet been scored on the held-out set.
- All processes are invented and fairly short, so real documents may be harder.
- Step splitting (stage 1) is not evaluated. The evaluation uses pre-split steps, and different step boundaries could change the results.
- The tool list is limited to eight Microsoft products. Tool suggestions are a first draft, not a verified design.
- The output is a plan, not a working automation. A person still has to build and test it.
- The labels test how well the model reproduces my judgment, not whether automating a step is a good idea.

## Run it yourself (Google Colab)

1. Open the notebook with the **Open in Colab** badge at the top of this page.
2. Add your OpenAI API key under Secrets (key icon), named `API`, with notebook access switched on.
3. Run the cells in order. The evaluation prints its scores and a table of misses, and the interface appears under the last cell.

The first cell installs the dependencies:

```
pip install openai gradio
```

The full evaluation (3 runs) costs a few francs of API usage.

## Repository contents

- `Process_to_automation_ideas.ipynb`: the tool, the answer key, the evaluation and the interface
- `README.md`: this file

## Possible next steps

- Score v4 on the held-out set, and write a new unseen set for future versions
- Accept more than one label for steps that are borderline in practice
- Add rules for "passing on a decision that was already made is AUTOMATE" and "intake uses the channel the request arrives on"
- Evaluate step splitting by comparing the model's steps with mine
- Read Word or PDF process documents instead of pasted text
- Compare models on cost vs. accuracy
- Host the interface permanently, for example on Hugging Face Spaces

Done in v4: lists of acceptable tools, a rule for "human reviews AI output", structured outputs, two-stage interface.

## Author

Vladislava_C
