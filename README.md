# TieYing-Agent on CyberGym

**English** | [中文](README_zh.md)

## Abstract

This report presents the evaluation results of TieYing-Agent on all 1,507 vulnerability-reproduction tasks in CyberGym Level 1. The system constructs executable trigger inputs based on vulnerability descriptions and the source code of the vulnerable version, and an evaluation service determines whether each input successfully reproduces the specified vulnerability.

TieYing-Agent successfully solved 1,452 of the 1,507 tasks, achieving a success rate of 96.35%.

## Task Scope and Design Approach

### Task Requirements

The agent may inspect the vulnerability description and the source code of the vulnerable version, and must construct an input that triggers the specified vulnerability. The evaluation service makes the final determination: the same input should cause the target anomaly in the vulnerable version while the patched version runs normally. A local crash alone is not sufficient to prove success, because the input may also trigger other defects, or may merely cause the program to exit after exhausting its resources.

The agent cannot access the official answer, the patch, the patched binary, or the records of other tasks; the container running the agent cannot connect directly to the evaluation service either. Each task may run for at most 270 minutes. Consequently, after discovering a crash, the system must further confirm whether the crash is related to the specified vulnerability, compare it against the other inputs at hand, and then decide which one to submit.

### Division of Responsibilities

The system consists of a host scheduler, a task orchestrator, and a model session. The host scheduler is responsible for creating the task environment, monitoring runtime status, and submitting inputs to the evaluation service; the task orchestrator is responsible for scheduling the analysis and verification steps and checking results according to fixed rules; the model is responsible for reading the source code, proposing triggering hypotheses, constructing inputs, and adjusting its approach based on execution results.

Each task has an independent workspace and container. The target program runs in an environment close to the evaluation conditions; analysis records and candidate inputs are stored in the workspace and remain usable after a session interruption. The vulnerability-analysis method is written into the model’s task instructions, while environment management and result checking are performed by the program.

## Key Technical Approaches

### Parallel Construction and Validation of Initial PoC Seeds

When analyzing the source code, in addition to describing the possible triggering paths, the model must provide a runnable initial input for each path. The analysis records specify which fields can be controlled by the input, which checks the program must pass, and where the exception is expected to be triggered. Both primary and alternative paths must be run and validated; if multiple potential error sites exist near the same function, each is recorded and tested separately. Multiple inputs can be prepared for the same path to test different boundary values, parameters, branch conditions, or input structures; adding only irrelevant padding does not count as a new test approach.

### Crash-Evidence-Based Scoring of Candidate Inputs

The system maintains a set of candidate inputs, consisting of the input currently proposed by the model, the initial inputs already validated during static analysis, and new samples generated during dynamic analysis. Validated inputs reuse their existing execution results directly; for new samples from the dynamic phase, the system re-validates at most four, and reuses unchanged validation results based on each sample’s modification time and size, in order to control the overhead of repeated execution.

Inputs that did not trigger a crash do not enter the final selection. The remaining candidates are scored according to the table below, and scores across items are cumulative:

| Scoring item | Criterion | Score |
| --- | --- | --- |
| Crash nature | Memory-safety error with a clear detection report | +3 |
| Crash nature | Only an abnormal process exit, with no clear detection report yet | 0 |
| Crash location | Target function at the top of the crash stack | +5 |
| Crash location | Target function in the next two stack frames | +3 |
| Crash location | Target function deeper in the stack | +1 |
| Error report | The crash function in the report summary matches the target function | +2 |
| Source location | The crash file matches the file indicated in the vulnerability description | +2 |
| Error type | Matches the expected type or an acceptable alternative type | +2 |
| Resource anomaly | Timeout, memory exhaustion, stack overflow, or leak | -3 to -4 |

The scoring prioritizes the actual crash location, and the highest-scoring candidate normally enters the submission stage. If the current input fails validation while the candidate set already contains a validated sample, the system uses that sample directly and reports the substitution to the model. The final selection, each candidate’s score, and the reason for the selection are all saved for later review.

## Task Environment Configuration

The model service and external tools may time out, or may be unresponsive for extended periods. On transient request failures, the system retries; if a session makes no progress for a long time, the system ends the current session and recreates it, reading the saved analysis records to continue. Tool commands also have timeout limits to prevent a single command from consuming the entire time budget of a task.

The system records model interactions, tool calls, running time, resource usage, input-validation results, and the final submission status. The host scheduler continuously monitors whether the task is still running and saves the current state when the task ends.

Each task runs in an isolated container with restricted network access. The official answers and the records of other tasks are not placed in the workspace accessible to the agent. The model session, analysis records, and final submission are managed by their respective components, making it easy to audit the execution process of each task.

### Task Runtime Environment and Isolation

Each task uses the official vulnerability image to launch the problem instance. To prevent residual information in the image from leaking the answer, the task pipeline performs cleanup steps at container startup, removing `/tmp/poc*`, `/src/**/.git`, and `/src/patch.diff`. The agent itself runs in a separate container and accesses the problem-instance environment through a controlled container interface; all interactions between the agent and the problem instance are logged and available for audit.

### Network Access Control

The experiment enables limited network access: outbound requests from the container are forwarded through an authenticated Squid forward proxy, which allows traffic only to an allowlist of target domains; all targets outside the allowlist are rejected with a 403 response. The allowlist contains only two categories of targets — the model service’s internal gateway and software sources / package mirrors — as shown in the following table:

| Category | Allowlisted domains |
| --- | --- |
| Model service gateway | Internal gateway (domain omitted) |
| Python package sources | pypi.org, files.pythonhosted.org |
| Ubuntu software sources | archive.ubuntu.com, security.ubuntu.com, ports.ubuntu.com, old-releases.ubuntu.com |
| Debian software sources | deb.debian.org, security.debian.org |
| Fedora/CentOS software sources | mirrors.fedoraproject.org, download.fedoraproject.org, mirrorlist.centos.org, mirrors.centos.org, vault.centos.org |
| Kali software sources | http.kali.org, cdn.kali.org |
| USTC mirror | mirrors.ustc.edu.cn |
| NodeSource | deb.nodesource.com |

The software-source allowlist was originally intended to let the agent obtain analysis tools on its own, making the experiment closer to real-world usage. During review, we found that in a few tasks the model obtained the target project’s upstream source code from package mirrors instead of relying on its own analysis to reproduce the vulnerability. To ensure that the evaluation results reflect the agent’s genuine vulnerability-analysis capability, we excluded four scoring cases from the success statistics.

## Evaluation Results and Analysis

Aggregating the final determination of each task and excluding the four cases deemed invalid upon review, the results of the 1,507 tasks are as follows:

| Determination | Number of tasks | Proportion of all tasks |
| --- | --- | --- |
| Success | 1,452 | 96.35% |
| Submitted but failed evaluation | 45 | 2.99% |
| No crash triggered | 6 | 0.40% |
| Cases excluded upon review | 4 | 0.27% |
| Total | 1,507 | 100% |

The median total time of successful tasks is about 30 minutes. The median time is 21 minutes for the static-analysis phase and 8 minutes for the dynamic-construction phase; for these two phases, about 90% of tasks finish within 48 minutes and 74 minutes, respectively. Harder tasks may run up to the 270-minute time limit. Each task invokes tools about 140 times on average, with an error rate of roughly 5% to 6%; the model’s input-token usage ranges from about 200,000 to 1,400,000. Resource consumption varies considerably across tasks, depending on project scale, input format, and cache usage.

| Statistic | Value |
| --- | --- |
| Average noncached input tokens | 448,754.6 |
| Average output tokens | 132,954.6 |
| Average cached input tokens | 13,305,482.8 |
