# DejaPod

**"It's seen this before."** An AI agent that fixes broken apps in Kubernetes and remembers
how every problem was fixed, so the next time is faster.

When a pod crashes, DejaPod gathers the evidence (status, events, logs), checks its memory for
similar past incidents and your team's runbooks, works out the cause, proposes a fix, waits for a
human to approve it, applies it, checks that it worked, and saves the incident to memory.

The name: *déjà vu* + *pod*. Checked 2026-10-15: no GitHub repo, PyPI or npm package, or
website uses it.

Why it's worth building: tools like K8sGPT explain Kubernetes errors, but teams still lose the
knowledge of *how they fixed it last time*. DejaPod's difference is that memory, plus a safe
approve → apply → verify loop.

---

## How to use this plan

- **Easy to complex.** Each phase builds on the last. Phases 1-3 have no AI at all: you learn
  Docker and Kubernetes by hand first, so you know what the agent is supposed to do.
- **Every phase has:** what you'll learn, the steps, "done when", questions to check yourself,
  and the resume line it unlocks. Only claim a line once the phase works.
- **Learn by doing.** Claude guides you step by step ("now we do this, now that"), explains
  each idea in simple words, and gives you the code in small chunks with an explanation of each
  part. You type or paste it and run every command yourself (rules in `CLAUDE.md`). Interviews
  test understanding, not code.
- **Keep a journal** in `LEARNING.md`: after each phase, 3-5 sentences on what you learned and
  what broke. It becomes your interview prep.
- **Commit after every phase** with a clear message. A steady commit history on GitHub shows the
  work was really yours.

**Amii checkpoint:** apply once Phase 6 works (the minimum is Phase 5). Don't wait for the
whole plan.

---

## The big picture

```
            ┌──────────── your Kubernetes cluster (kind) ─────────────┐
            │  demo apps, broken on purpose: bad image, crash, OOM ... │
            └──────────────▲───────────────────────────┬──────────────┘
                 apply fix │                           │ status, events, logs
                           │                           ▼
 ┌─────────────────────── DejaPod agent (LangGraph) ───────────────────────┐
 │ detect → collect → recall → diagnose → propose → ⏸ approve → apply →    │
 │ verify ──(failed)──► back to diagnose          (worked) ──► remember     │
 └────────────────────────────┬────────────────────────────────────────────┘
                              │
              ┌───────────────┴────────────────┐
              ▼                                ▼
      Qdrant (memory)                  Claude (reasoning)
   - runbooks                         via LangChain
   - past incidents + fixes
                              ▲
                              │ live steps (SSE), Approve button
                      React + TypeScript dashboard
```

**Stack:** Python 3.12 (uv), Docker, kind + kubectl, the official Kubernetes Python client,
LangChain, LangGraph, Qdrant, fastembed, Claude API, FastAPI, React + TypeScript (Vite).

---

## Phase 0: Setup (30 min)

**Learn:** what each tool is for.

**Already installed:** Docker 29, Python 3.14, Node 24, uv, git.

**Steps**
1. Install `kubectl` (the Kubernetes command line) and `kind` (runs a Kubernetes cluster inside
   Docker on your machine). Both are single binaries; follow their official install pages for
   Linux.
2. Make sure Docker runs without sudo: `docker run hello-world`.
3. Create the project with Python 3.12, not 3.14 (some AI libraries don't support 3.14 yet):
   `uv init --python 3.12` in this folder.
4. Get an Anthropic API key; put it in a `.env` file and add `.env` to `.gitignore` right away.
5. `git init`, create the GitHub repo `dejapod`, push the empty project.

**Done when:** `docker`, `kubectl version --client`, `kind version` and `uv run python --version`
all work, and the repo is on GitHub.

**Ask yourself:** Why does kind need Docker? Why must `.env` never be committed?

---

## Python foundations (from scratch, 2-4 days, alongside Phases 0-1)

DejaPod is written in Python, so Python comes first. Every exercise below uses data the project
needs later, so nothing is wasted. Do each exercise in `learn-python/` as its own small file,
run it with `uv run python <file>`, and don't move on until it works and you can explain it.

| Step | Learn | Exercise (project-flavoured) |
|---|---|---|
| P1. Basics | running a script, variables, `int` / `float` / `str` / `bool`, f-strings, `print`, `input`, comments, indentation | print a pod's name, status and restart count from variables |
| P2. Collections | lists, dicts, tuples, sets, indexing and slicing, `in`, `len` | store 5 pods as a list of dicts; print only the ones with restarts > 3 |
| P3. Control flow | `if` / `elif` / `else`, `for`, `while`, `range`, `break` / `continue`, comprehensions | label each pod `healthy` / `crashing` / `pending` |
| P4. Functions | parameters, defaults, return values, scope, docstrings, type hints | `def classify(pod: dict) -> str` |
| P5. Files and JSON | `open` / `with`, reading and writing text, the `json` module, `pathlib` | run `kubectl get pods -A -o json > pods.json` (after Phase 2, or use a sample) and print each pod's status from the file |
| P6. Errors | exceptions, `try` / `except` / `finally`, raising your own | handle a missing file and a pod with no `status` key without crashing |
| P7. Modules and packages | `import`, your own modules, `__name__ == "__main__"`, uv projects, `uv add`, virtual environments | split P4-P6 into `classify.py` and `main.py` |
| P8. Classes | classes, `__init__`, methods, `@dataclass`, then **Pydantic** models | an `Evidence` Pydantic model built from a pod dict (used for real in Phase 3) |
| P9. Testing | `pytest`, asserts, test files, fixtures | tests for `classify` covering every status |
| P10. Working with the outside world | environment variables, `.env` with `python-dotenv`, HTTP requests with `httpx`, logging | call your Phase 1 shop app's `/health` endpoint from Python |
| P11. Async (just enough) | `async` / `await`, why web servers use it, `asyncio.run` | call `/health` 10 times concurrently and time it |

**Done when:** you can write a small script from a blank file without help: read a JSON file,
loop over it, make decisions, handle errors, and test it.

Python keeps growing with the project: the Kubernetes client (Phase 3), typed models and
structured output (Phase 4), generators and streaming (Phases 6 and 9), and FastAPI (Phase 9).

**For Amii:** your BIS chatbot server was already Python, so you don't need to finish this
track before applying. Do P1-P8 before Phase 3.

---

## Phase 1: Docker basics (2-3 hours)

**Learn:** images vs containers, Dockerfile instructions (`FROM`, `WORKDIR`, `COPY`, `RUN`,
`CMD`), layers and caching, ports, environment variables, logs, volumes, `.dockerignore`,
docker-compose, container networking.

**Steps**
1. Write a tiny **demo app** in `demo-apps/shop/`: a FastAPI app with `/health` and `/items`.
   This is the app you'll break later. Give it settings read from environment variables
   (e.g. `DB_URL`) so you can break it by leaving one out.
2. Write its `Dockerfile`. Build it (`docker build -t shop:v1 .`), run it
   (`docker run -p 8000:8000 shop:v1`), open it in the browser.
3. Experiment: change code and rebuild (watch which layers are cached), pass an env var with
   `-e`, read logs with `docker logs`, open a shell inside with `docker exec -it`.
4. Run **Qdrant** in Docker with a volume so its data survives restarts. Open its dashboard on
   port 6333.
5. Write a `docker-compose.yml` that starts the shop app and Qdrant together. Notice the shop
   can reach Qdrant by the name `qdrant`: that's Docker networking.

**Done when:** `docker compose up` starts both, and you can explain each line of the Dockerfile.

**Ask yourself:** Image vs container? Why copy `requirements` before the code? What's lost when
a container is removed, and how does a volume prevent that?

---

## Phase 2: Kubernetes by hand (4-6 hours, the most important phase)

**Learn:** cluster, node, pod, Deployment, ReplicaSet, Service, namespace, ConfigMap, Secret,
resource requests and limits, liveness and readiness probes, labels and selectors, rollouts.
And the errors: `ImagePullBackOff`, `ErrImagePull`, `CrashLoopBackOff`, `OOMKilled`,
`CreateContainerConfigError`, readiness failures, `Pending`.

**Steps**
1. `kind create cluster --name dejapod`. Look around: `kubectl get nodes`, `kubectl get pods -A`.
2. Load your image into the cluster: `kind load docker-image shop:v1 --name dejapod`.
3. Write `k8s/shop.yaml`: a Deployment (2 replicas) and a Service. Apply it, then reach it with
   `kubectl port-forward`.
4. Move the app's settings into a **ConfigMap** and a **Secret**. Add memory limits and
   liveness/readiness probes.
5. **Break it on purpose, then fix it by hand.** One scenario at a time, each as its own YAML
   in `scenarios/`:

   | # | Break it by | You'll see | Fix |
   |---|---|---|---|
   | 1 | Image tag that doesn't exist (`shop:v9`) | `ImagePullBackOff` | correct the tag |
   | 2 | Remove the required `DB_URL` env var | `CrashLoopBackOff` | add it back |
   | 3 | Add an endpoint that eats memory, set a 64Mi limit | `OOMKilled` | raise the limit (or fix the code) |
   | 4 | Point a ConfigMap reference at a name that doesn't exist | `CreateContainerConfigError` | fix the name |
   | 5 | Readiness probe on the wrong path | running but 0/1 ready, no traffic | fix the path |
   | 6 | Request more CPU than the node has | `Pending` | lower the request |

   For each one, use `kubectl get pods`, `kubectl describe pod`, `kubectl logs` (and
   `--previous`), `kubectl get events`.
6. **Write a runbook for each scenario** in `runbooks/` (one markdown file each: symptoms, how to
   confirm, cause, fix). These become DejaPod's first memory in Phase 5.

**Done when:** you can break and fix all 6 without looking anything up, and the 6 runbooks are
written.

**Ask yourself:** Pod vs Deployment? Why did Kubernetes keep restarting the crashing pod? What's
the difference between a liveness and a readiness probe? Where do you look first for each error?

---

## Phase 3: Collect evidence with Python (2-3 hours, still no AI)

**Learn:** the official Kubernetes Python client, kubeconfig, reading pod status and container
states, events, logs, and how to shape messy data into a clean structure (Pydantic models).

**Steps**
1. `uv add kubernetes pydantic`.
2. Write `dejapod/collect.py`: find unhealthy pods in a namespace, and for each one build an
   **Evidence** object: pod name, phase, container state and reason (e.g. `waiting:
   CrashLoopBackOff`), restart count, the last 50 log lines (and the previous container's
   logs), recent events, and the relevant part of the Deployment spec (image, env, limits,
   probes).
3. Write the 6 detection rules yourself as plain Python (no AI): a function that labels the
   evidence `image_pull`, `crash_loop`, `oom`, `config_error`, `not_ready` or `pending`.
4. Run it against each broken scenario and print the evidence as JSON.

**Done when:** `uv run python -m dejapod.collect` correctly detects and labels all 6 scenarios.

**Ask yourself:** Why gather evidence in code before asking an AI? (Cost, accuracy, and the AI
can only reason about what you give it.)

---

## Phase 4: LangChain + Claude: the first diagnosis (2-3 hours)

**Learn:** chat models, messages, prompt templates, **structured output** (the model returns a
Pydantic object, not free text), token usage and latency, temperature.

**Steps**
1. `uv add langchain langchain-anthropic python-dotenv`.
2. Define a `Diagnosis` model: `cause`, `evidence_used`, `confidence` (0-1), `proposed_fix`
   (a structured patch, e.g. "set image to shop:v1"), `risk`.
3. Write `dejapod/diagnose.py`: a prompt template that gives Claude the Evidence JSON and asks
   for a `Diagnosis`. Use `with_structured_output`.
4. Log time and tokens for every call.

**Done when:** it diagnoses all 6 scenarios with a sensible cause and fix, and prints cost and
time per diagnosis.

**Ask yourself:** Why structured output instead of text? What happens to accuracy if you send
500 log lines instead of 50?

**Unlocks:** LangChain and the Claude API on the resume.

---

## Phase 5: Qdrant + RAG: memory, part 1 (2-3 hours)

**Learn:** embeddings, chunking (size and overlap), collections, cosine similarity, top-k,
payloads and filters, how retrieved text gets into the prompt.

**Steps**
1. `uv add qdrant-client fastembed langchain-qdrant`.
2. Write `dejapod/memory.py`: split the 6 runbooks into chunks, embed them with fastembed, and
   store them in a Qdrant collection `runbooks` with payloads (`scenario`, `title`).
3. Before diagnosing, search `runbooks` with the evidence summary and add the top 3 chunks to the
   prompt.
4. Compare: diagnosis with and without the runbooks. Write the difference in `LEARNING.md`.

**Done when:** the right runbook comes back first for each scenario, and diagnoses cite it.

**Ask yourself:** What is an embedding? Why might the wrong runbook come back, and how would you
fix that (better chunks, filters, a better query)?

**Unlocks:** RAG with Qdrant on the resume (Amii names Qdrant). **Minimum point to apply to
Amii.**

---

## Phase 6: LangGraph: make it an agent (4-5 hours)

**Learn:** graph state (a typed dict), nodes, edges, **conditional edges** (decisions),
**loops** with a limit, streaming the steps, visualizing the graph.

**Steps**
1. `uv add langgraph`.
2. Turn Phases 3-5 into a graph in `dejapod/graph.py`:
   `detect → collect → recall → diagnose → propose`.
3. Add the first decision: if `confidence < 0.7`, go to `dig_deeper` (fetch more logs, the
   previous container's logs, the Deployment's rollout history) and diagnose again. At most
   2 loops.
4. Stream each step as it runs, so you can watch the agent think.
5. Draw the graph (LangGraph can export a Mermaid diagram) and put it in the README.

**Done when:** `uv run dejapod diagnose --namespace demo` runs the whole graph for any broken
scenario and you can explain why each edge exists.

**Ask yourself:** What's in the state, and why? Why limit the loop? How is this different from
one big prompt?

**Unlocks:** LangGraph and multi-step agentic workflows on the resume. **Apply to Amii here.**

> Resume line once Phases 1-6 work (fill in your real numbers):
> "Built DejaPod, an AI agent in LangGraph that diagnoses failing Kubernetes pods from their
> status, events and logs, using RAG over runbooks in Qdrant. Python, LangChain, Claude, Docker."

---

## Phase 7: Approve → apply → verify (3-4 hours)

**Learn:** **human-in-the-loop** (LangGraph `interrupt`), **checkpointers** (saving graph state
so it can pause and resume, e.g. `SqliteSaver`), patching Kubernetes objects, server-side dry
run, watching a rollout.

**Steps**
1. Add an `approve` node that pauses the graph and shows the proposed fix. Resume with
   approve / reject.
2. Add `apply`: turn the structured fix into a Kubernetes patch. **Safety first:** only allow a
   fixed list of fix types (image tag, env var, memory limit, probe path, ConfigMap ref), always
   run a server-side dry run before the real patch, and never touch anything outside the
   chosen namespace.
3. Add `verify`: wait for the rollout, check the pod is ready and stays up for 60 seconds.
4. If verify fails, loop back to `diagnose` with the new evidence (at most 2 tries), then give
   up and report.

**Done when:** you break a scenario, approve the fix in the terminal, and DejaPod fixes it.

**Ask yourself:** Why a checkpointer? What could go wrong if the agent could patch anything?

---

## Phase 8: Memory, part 2: the "déjà vu" (2-3 hours)

**Learn:** writing to a vector store while the app runs, what to store, deduplication,
ranking by similarity and recency.

**Steps**
1. After a verified fix, add a `remember` node: save an **incident** (symptoms, evidence summary,
   cause, the fix that worked, how long it took) to a Qdrant collection `incidents`.
2. In `recall`, search `incidents` first, then runbooks. If a close match exists, show
   **"Seen this before: N times. Last fixed by: ..."** and give it to the diagnosis.
3. Measure: time and tokens for a first-time incident vs a repeat. That's your headline number.

**Done when:** the second time you break the same thing, DejaPod recognizes it and fixes it
faster. Record the real numbers in `LEARNING.md`.

---

## Phase 9: API + dashboard (5-6 hours)

**Learn:** FastAPI, **Server-Sent Events** (streaming), React + TypeScript with Vite, consuming
a stream, simple state management.

**Steps**
1. `dejapod/api.py`: endpoints to list unhealthy pods, start a diagnosis, stream the graph's
   steps (SSE), and approve or reject.
2. `web/`: a React + TypeScript dashboard: broken pods, the agent's steps appearing live, the
   diagnosis and evidence, a "Seen before" badge, and Approve / Reject buttons.

**Done when:** you can do the whole break → diagnose → approve → fixed loop from the browser.

---

## Phase 10: Run DejaPod itself in Kubernetes (4-5 hours)

**Learn:** packaging an app as several images, **ServiceAccounts and RBAC** (least privilege:
DejaPod may read pods, events and logs and patch Deployments in one namespace, and nothing
else), running Qdrant in the cluster with storage (a PersistentVolumeClaim), watching for
problems continuously instead of on demand.

**Steps**
1. Dockerfiles for the agent/API and the web app; a docker-compose for local runs.
2. Kubernetes manifests in `deploy/`: the agent, the web app, Qdrant with a PVC, a
   ServiceAccount, a Role and a RoleBinding.
3. Watch mode: DejaPod notices new failures by itself (Kubernetes watch API).

**Done when:** `kubectl apply -f deploy/` runs all of DejaPod inside the cluster, and it catches
a newly broken app on its own.

**Unlocks:** Kubernetes RBAC and running services in Kubernetes, honestly on the resume.

---

## Phase 11: Measure it (2-3 hours)

**Learn:** evaluating an agent, tracing, cost tracking.

**Steps**
1. An eval script: run all 6 scenarios N times; record correct diagnosis (yes/no), fixed
   (yes/no), time, tokens and cost.
2. Optional: LangSmith tracing to see every step of every run.
3. Put the results table in the README.

**Done when:** the README says something like "correct diagnosis in X of Y runs, average Z
seconds, $W per incident", with your real numbers.

---

## Later (build on it as you like)

- More failure types: bad Secret, wrong Service selector, failing init container, DNS.
- GitHub Actions: tests and image builds on every push.
- A Helm chart.
- Run it on **AWS EKS** (ties into the AWS certification).
- Slack or email notifications with an approve link.
- Share memory across clusters and teams.
- Prometheus metrics and a Grafana dashboard.

---

## Suggested repo layout

```
dejapod/
├── PLAN.md  LEARNING.md  README.md  .env (never committed)
├── learn-python/           Python exercises P1-P11
├── demo-apps/shop/         the app you break (FastAPI + Dockerfile)
├── k8s/                    healthy manifests for the demo app
├── scenarios/              one broken manifest per failure
├── runbooks/               one markdown runbook per failure
├── dejapod/                the agent: collect, diagnose, memory, graph, api
├── web/                    React + TypeScript dashboard
├── deploy/                 manifests to run DejaPod in the cluster
└── evals/                  the evaluation script and results
```

## Progress

- [x] Phase 0: Setup
- [ ] Python foundations (P1-P11)
- [ ] Phase 1: Docker basics
- [ ] Phase 2: Kubernetes by hand
- [ ] Phase 3: Collect evidence
- [ ] Phase 4: First diagnosis (LangChain)
- [ ] Phase 5: RAG with Qdrant ← minimum for Amii
- [ ] Phase 6: LangGraph agent ← apply to Amii
- [ ] Phase 7: Approve, apply, verify
- [ ] Phase 8: Incident memory
- [ ] Phase 9: API + dashboard
- [ ] Phase 10: DejaPod in Kubernetes
- [ ] Phase 11: Measure it
