# In-Lab Programming Examination Infrastructure

This repository contains the examination notebooks and infrastructure documentation for the in-lab programming examination used in the *Experiment Design and Execution* course. The setup is designed to assess programming and data science skills at scale while preventing the use of generative AI tools and external resources during the exam.

---

## Repository Structure

```
.
├── notebooks/
│   └── Cyclists.ipynb          # Example examination notebook (with sample solutions)
├── data/
│   └── cyclists/
│       ├── weather/            # Tri-daily weather CSV files (2009–2023)
│       └── cyclists/           # Daily cyclist count CSV files (2021–2022)
└── README.md
```

---

## Course Context

The course targets master's students in Data Science with heterogeneous programming backgrounds. Examination notebooks cover the following core competencies:

- **Data wrangling** — loading, reshaping, and cleaning datasets using `pandas` (e.g. wide-to-long transforms, multi-index construction, type coercion)
- **Data understanding** — descriptive statistics, identifying and handling outliers and missing/non-numeric values, temporal alignment of multiple datasets
- **Experiment design** — aggregation strategies, merging datasets, reasoning about appropriate dependent variables and model evaluation criteria
- **Predictive modelling** — building and evaluating models using `scikit-learn`
- **Conceptual understanding** — multiple-choice questions testing statistical reasoning (e.g. overfitting vs. underfitting, classification vs. regression targets)

---

## Infrastructure Overview

```
┌────────────────────┐         ┌─────────────────────────┐
│   Student PC       │────────▶│  Network Access          │
│  (lab workstation) │         │  Controller (whitelist)  │
└────────────────────┘         └────────────┬────────────┘
                                            │ Whitelisted IPs only
                                            ▼
                                ┌─────────────────────────┐
                                │   Examination Server     │
                                │   (JupyterHub on K8s)    │
                                │   — accepts connections  │
                                │     from exam IP ranges  │
                                │     only                 │
                                └─────────────────────────┘
```

### Student PC Configuration

Student PCs are university-managed lab workstations running **Firefox** as the only open application during the exam — no other browser or local tool is started. Students interact with the examination notebook entirely through the browser.

**Network restrictions** are enforced at the network level via **iptables rules** on the Network Access Controller. Before the exam, the controller performs an automatic DNS lookup to resolve whitelisted domain names to their current IP addresses, which are then used to populate the iptables rules. This means that only traffic to those resolved IPs is permitted — everything else, including LLM interfaces (e.g. ChatGPT, Copilot) and code hosting platforms (e.g. GitHub), is blocked.

> **Security note:** The DNS-lookup-based approach of pre-resolving domain names to IPs carries an inherent risk: if a whitelisted service's IP changes between the lookup and the exam, students may lose access to permitted resources. Conversely, if a blocked service shares IP infrastructure with a whitelisted one (e.g. via a CDN), it could theoretically become reachable. This trade-off should be kept in mind when selecting which domains to whitelist.

Whitelisted resources include the examination server itself and the official documentation sites for the permitted libraries. **Exam data is stored in a read-only shared folder on the examination server**, so no external data URLs are needed.

**Proctoring** is carried out by physical presence in the examination room. In addition, instructors can observe student activity remotely via the JupyterHub admin interface, which provides visibility into submitted work.

**Identity verification:** While authentication uses the university's SSO system, student IDs are additionally checked in person at the start of the exam to confirm the authenticated user matches the registered student.

### Examination Server Configuration

The examination server runs **JupyterHub deployed on a Kubernetes cluster**. It is configured to:

- **Accept incoming connections only from the IP ranges of the examination rooms.** Connections originating outside these ranges are rejected at the server level.
- **Block all outbound internet access** from the server itself, ensuring that code executing inside student notebooks cannot retrieve external resources at runtime.
- Allocate **2 CPUs and 6 GB of RAM per student** to ensure sufficient resources for data processing tasks under exam load.
- Run on **dedicated infrastructure**, isolated from the regular teaching JupyterHub used for take-home assignments, to prevent students from accessing previously created training notebooks during the exam.

#### Notebook Distribution

Examination notebooks are distributed to students via the JupyterHub interface. When a student logs in, the **master notebook** (containing sample solutions and hidden tests) is automatically pulled into their personal JupyterHub repository. Before the notebook becomes visible to the student, the server processes it as follows:

- All content between `### BEGIN SOLUTION` and `### END SOLUTION` markers is **replaced with a `# Your solution here` placeholder**.
- All cells between `#### BEGIN HIDDEN TESTS` and `#### END HIDDEN TESTS` are **completely removed**.

This ensures students receive a clean notebook with only the task descriptions and visible test cells, while the master version with solutions and hidden tests remains on the server for grading.

---

## Automatic Grading

Grading uses a customised variant of [nbgrader](https://nbgrader.readthedocs.io/) adapted for JupyterHub. Key properties:

- **Graded sections** are marked with `### BEGIN SOLUTION` / `### END SOLUTION` delimiters within answer cells. Students implement their solutions between these markers.
- **Test cells** contain `assert` statements that automatically verify the correctness of student implementations. These are divided into:
  - *Visible tests* — shown to students during the exam to give basic correctness feedback (e.g. checking output shape or data types).
  - *Hidden tests* — marked with `#### BEGIN HIDDEN TESTS` / `#### END HIDDEN TESTS`. These cells are **injected server-side at submission time** and are never visible to students during the exam, making the grading process tamper-resistant.
- Because grading logic is added after submission, instructors can also **adjust tests post-exam** if a discrepancy between the exercise description and the grading tests is discovered.
- In our setup the notebook combines **programming tasks** (approx. 2/3 of total points) with **multiple-choice questions** (approx. 1/3 of total points) in a single `.ipynb` file.

### Sample Solutions

The notebooks in this repository include full **sample solutions** (within `### BEGIN SOLUTION` / `### END SOLUTION` blocks) and all hidden tests. These are intended for instructors to:

1. Verify that the entire notebook runs end-to-end without errors before the exam.
2. Check that grading tests are consistent with the exercise descriptions.
3. Serve as a reference when reviewing borderline student submissions.

---

## Example Notebook: Cyclists

`notebooks/Cyclists.ipynb` is a training notebook that models the relationship between weather observations and daily cyclist counts in Vienna, using:

- Tri-daily weather data from 2009–2023 (`data/cyclists/weather/`)
- Daily cyclist counts from 2021–2022 (`data/cyclists/cyclists/`)

Tasks progress through the full data science pipeline: loading and reshaping raw CSVs, cleaning outliers and non-numeric values, daily aggregation, dataset merging, exploratory visualisation, and multiple-choice conceptual questions.

