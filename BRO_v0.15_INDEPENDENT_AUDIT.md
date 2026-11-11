# Bro v0.15 — Independent Deep Code & System Audit

**Աուդիտի օբյեկտ:** `Bro_Server_30_Audit_Fix_Round7_v0.15.zip` (SHA-256 `2374bd07…11515895`)
**Ամսաթիվ:** 2026-11-12 · **Ռեժիմ:** AUDIT ONLY — բնօրինակը չի փոփոխվել (բոլոր փորձերը՝ մեկանգամյա պատճեններում)
**Միջավայր:** Linux, Python 3.13.18, 2 vCPU / 7 GB RAM, Docker CLI 29.9.3 (**daemon չկա**), Claude Code CLI 2.1.322 (**ձեր subscription-ը չկա**), Vault չկա, uid=0 (root)

> Կարևոր նշում ապացույցների մասին։ Աուդիտի ընթացքում `claude` CLI-ի argv parsing-ը ստուգելիս երկու *վերահսկիչ* տարբերակ (Bro-ի կոդից դուրս՝ prompt-ը `--tools`-ից առաջ կամ `--`-ից հետո) անսպասելիորեն իրական փոքր model call արեցին այս աուդիտի միջավայրի սեփական հասանելիությամբ։ Դա **Bro-ի իրական աշխատանքի ապացույց չէ**, ձեր subscription-ը չի օգտագործվել։ Bro-ի ճշգրիտ argv-ն ձախողվել է parse-ի փուլում, մինչ model call-ը։ Bro-ի ոչ մի իրական Claude call չի կատարվել։

---

## A. Executive Verdict

## **FAIL**

Հիմնական պատճառը **AUD-001**-ն է (Reproduced)։ Bro-ն prompt-ը փոխանցում է variadic `--tools <tools...>` flag-ից հետո, և Claude Code CLI-ն prompt-ը վերցնում է որպես tool-ի անուն։ Արդյունքում **յուրաքանչյուր** Claude call ավարտվում է `exit 1`-ով, ու ոչ մի job չի անցնում planning-ից այն կողմ։ Սա վերարտադրվել է Claude Code 2.1.322-ով։

Եթե AUD-001-ը ուղղվի, համակարգը կմնա **CONDITIONAL**․ orchestration շերտը աշխատում է (30 զուգահեռ subprocess-ով ստուգված է fake binary-ներով), բայց կան High մակարդակի բացեր․
- մեծ prompt-ներ (E2BIG),
- rootless Docker-ում uid mapping (Unverified),
- կեղծ «completed»,
- agent-ների միջև տվյալների փոխանցման բացակայություն,
- continuation-ը չի տեսնում նախորդ artifact-ները,
- host CLI-ի config-ից (MCP/hooks) մեկուսացման բացակայություն։

Նախորդ 6 ռաունդների «66 tests OK»-ն ճիշտ է որպես փաստ (ես էլ եմ ստացել 65 pass / 1 skip), բայց թեստերից ոչ մեկը չի կանչում իրական CLI-ն ու չի ստուգում argv-ն։ Mutation-ի 14 փորձից 7-ը **survived**։

---

## B. Coverage Matrix

### B.1 Inventory (83 / 83 ֆայլ, archive integrity OK)

| Դաս | Ֆայլեր | Քանակ | Ստուգման մակարդակ |
|---|---|---|---|
| Runtime code | `bro/__init__.py, __main__.py, core.py, sandbox.py, server.py, artifacts.py, router.py, skills.py, dependencies.py, vault.py` | 10 | Ամբողջությամբ, տող առ տող (exception path-երով) |
| Tests | `tests/*.py` | 13 | Ամբողջությամբ + կատարված + mutation |
| Tools | `tools/http_smoke.py` | 1 | Ամբողջությամբ + կատարված |
| Config | `agents.json, agent_skills.json, skill_routes.json, skill_dependency_status.json, os_packs_registry.json` | 5 | Ամբողջությամբ (os_packs-ը՝ կառուցվածքով, 52 pack) |
| Skills | 42 `SKILL.md` + `skills/index.json` | 43 | 42-ի բոլոր domain բաժինները կարդացված են։ Adaptation բաժինը՝ 42-ում նույնական (md5-ով ստուգված), բացի «mapped agents» տողից |
| Docs / audit | `README.md`, 7 `AUDIT_REPORT_*.md`, `TEST_RESULTS*.txt` (2), `SKILL_DEPENDENCIES_ROUND2.json` | 11 | Ամբողջությամբ (ROUND2 JSON-ը՝ կառուցվածքով) |
| **Ընդամենը** | | **83** | **83/83 inspected** |

Արտաքին dependency-ներ՝ միայն stdlib։ Արտաքին ծառայություններ՝ `claude` CLI, `docker` CLI/daemon, Vault (optional)։

### B.2 Module / call map

```
python -m bro ask|doctor|jobs|status  →  core.Bro / core.Store / sandbox.DockerSandbox
python -m bro.server (HTTP 127.0.0.1)  →  core.Bro, artifacts.open_artifact
core.Bro.__init__ → Store(recover=True) → SandboxClaudeCLI(DockerSandbox) → SkillLibrary → SkillRouter → SkillDependencies
core.Bro.submit → Store.create → job_pool(2)._process
   _process → _plan → provider.call(tools=False) → ClaudeCLI.call  [chief]
            → worker_pool(N)._work → router.instructions → SandboxClaudeCLI.call(tools=True)
                 loop ≤12: ClaudeCLI.call(tools=False) → JSON → DockerSandbox.run (docker run --network none)
            → artifacts.manifest → Store.run_state
            → summary: ClaudeCLI.call(tools=False)
vault.VaultKV2 → ՉԻ կանչվում ոչ մի runtime ճանապարհից (միայն tests)
os_packs_registry.json → միայն tests/test_bro.py
```

Չօգտագործվող / dead code․ `vault.py` (runtime-ում), `ClaudeCLI.permission_mode` (հայտարարված, բայց չի կիրառվում), `DockerSandbox.build_command(extra_mounts=…)` (ոչ ոք չի փոխանցում), `SandboxClaudeCLI(claude_image=…)`, `SkillDependencies.available_agents/external_action` (միայն tests), `agents.json: max_concurrent_workers, worker_slots, slot`։

### B.3 Evidence level ըստ component-ի

| Component | Static | Unit | Integration (fake binary) | Live |
|---|---|---|---|---|
| Store / SQLite / recovery | ✔ | ✔ | ✔ (kill/restart) | — |
| Chief planning | ✔ | ✔ (mock) | ✔ | ✘ **AUD-001** |
| Worker tool loop | ✔ | ✔ (mock) | ✔ (fake claude + fake docker, իրական subprocess) | ✘ |
| ClaudeCLI argv | ✔ | ✘ (թեստ չկա) | ✔ **իրական CLI 2.1.322-ով** | parse-ի մակարդակ |
| DockerSandbox flags | ✔ | ✔ | ✔ (fake docker) | ✘ (daemon չկա) |
| Rootless uid mapping | ✔ | ✘ | ✘ | ✘ → Unverified |
| HTTP API | ✔ | ✘ (unittest-ում չէ) | ✔ (19 negative/boundary case) | loopback |
| Artifacts download | ✔ | ✔ | ✔ | — |
| Router | ✔ | ✔ | ✔ (20 EN/HY task) | — |
| Skills (42) | ✔ | ✔ hash | ✔ 1260 load | ✘ |
| Vault | ✔ | ✔ (mock) | ✘ | ✘ |
| Concurrency 1/5/10/20/30 | ✔ | ✔ | ✔ իրական subprocess-ով | ✘ |

---

## C. Complete Findings Register

Դասավորված է ըստ severity-ի։ Ամփոփում՝ **Critical 1 · High 6 · Medium 14 · Low 6 = 27**։

---

### AUD-001 — Claude CLI-ն prompt-ը վերցնում է որպես `--tools` արժեք → բոլոր model call-երը ձախողվում են
- **Severity:** Critical · **Status:** Reproduced
- **File/lines:** `bro/core.py:136-140`
- **Problem:** argv-ն կառուցվում է `[claude, -p, --output-format, json, --permission-mode, dontAsk, (--append-system-prompt, S), --tools, '', PROMPT]` հերթականությամբ։ Claude Code-ում `--tools <tools...>`-ը variadic է (`claude --help`, 2.1.322), ու այն «կուլ է տալիս» PROMPT-ը։
- **Root cause:** Positional prompt-ը դրված է variadic option-ից հետո, առանց `--` տարանջատիչի։
- **Real-world impact:** Chief planning-ը, worker-ի բոլոր քայլերը և summary-ն ստանում են `exit 1`։ Յուրաքանչյուր job ավարտվում է `failed`-ով՝ «Claude CLI failed (exit 1)»։ Համակարգն ընդհանրապես չի աշխատում իրական CLI-ով։
- **Minimal safe reproduction:** `env -i PATH=… HOME=$(mktemp -d) claude -p --output-format json --permission-mode dontAsk --append-system-prompt sys --tools "" "say hello" </dev/null` → `Error: Input must be provided either through stdin or as a prompt argument when using --print`, exit 1, առանց model call-ի։ Նույնը՝ Bro-ի կոդով․ `ClaudeCLI().call('…', tmp/'chief')` → `RuntimeError: Claude CLI failed (exit 1)`։
- **Expected:** CLI-ն ստանում է prompt-ը և վերադարձնում JSON result։
- **Actual:** Parse error, exit 1։
- **Smallest fix:** `cmd += ['--tools', '', '--', prompt]` (կամ prompt-ը փոխանցել stdin-ով, տես AUD-002)։
- **Related:** AUD-002, AUD-021
- **Regression test:** Integration test, որ իրական `claude --help`-ի դեմ ստուգում է argv-ի դասավորությունը, կամ fake CLI, որ commander.js-ի նման variadic parsing է անում և ձախողվում, եթե prompt-ը `--`-ից հետո չէ։ Hetzner-ում՝ մեկ իրական smoke call։

### AUD-002 — Prompt-ը փոխանցվում է մեկ argv տարրով → Linux-ի 128 KiB սահմանը (E2BIG) կոտրում է summary-ն և երկար worker loop-երը
- **Severity:** High · **Status:** Reproduced
- **File/lines:** `bro/core.py:140, 149-151` (Popen-ը `try`-ից դուրս է), `core.py:320-322` (summary prompt), `bro/sandbox.py:219-221, 248-250` (conversation-ը աճում է)
- **Problem:** Linux-ում մեկ argument-ի սահմանը `MAX_ARG_STRLEN` = 128 KiB է։ Summary prompt-ը ներառում է բոլոր worker result-ները (յուրաքանչյուրը մինչև 64 KiB, 30 հատ)։ Worker conversation-ը աճում է մինչև 12×12000 նիշ + argv JSON։ Հայերեն տեքստը UTF-8-ում 2 byte/նիշ է։
- **Root cause:** Prompt-ի transport-ը command line-ով է, ոչ թե stdin-ով։
- **Real-world impact:** Բոլոր run-երը `completed` են, բայց job-ը `failed` է, ու summary-ն կորչում է։ Worker-ը մի քանի «շատախոս» քայլից հետո ձախողվում է։ Error-ը պարունակում է host binary-ի path-ը։
- **Reproduction:** fake `claude` (scratchpad `bin/claude`)․ (1) 70 000 հայերեն նիշ → `OSError [Errno 7] Argument list too long`, (2) 3 worker, յուրաքանչյուրը 60 KB result → `job state: failed`, `runs: completed×3`, (3) tool output-ը 12000 հայերեն նիշ → run-ը ձախողվում է 7-րդ քայլում։
- **Expected:** Մեծ prompt-ը կա՛մ փոխանցվում է, կա՛մ վերահսկելիորեն կրճատվում։
- **Actual:** E2BIG, job failed։
- **Smallest fix:** `Popen(..., stdin=PIPE)` և prompt-ը գրել stdin-ով (`claude -p`-ն կարդում է stdin), կամ summary-ի համար կրճատել յուրաքանչյուր result-ը (օր.՝ 4 KB) և ընդհանուր prompt-ը սահմանափակել bytes-ով։
- **Related:** AUD-001, AUD-014
- **Regression test:** fake CLI + 3×60 KB result → job-ը `completed` է, summary-ն պահված է, հայերեն 70k նիշը անցնում է։

### AUD-003 — Rootless Docker + `--user <host uid>` → container-ը, ամենայն հավանականությամբ, չի կարող մուտք գործել 0700 workspace
- **Severity:** High · **Status:** Unverified (Docker daemon չկա)
- **File/lines:** `bro/sandbox.py:86-87`, `sandbox.py:199-203` (0700 պահանջ), `sandbox.py:52-61` (rootless պարտադիր)
- **Problem:** Rootless Docker-ում container-ի uid 0-ն map է արվում host user-ին, իսկ container-ի uid N>0-ն՝ subuid range-ին (օր.՝ 100999)։ Bro-ն container-ը գործարկում է `--user 1000:1000`-ով, իսկ workspace-ը 0700 է և պատկանում է host uid 1000-ին։
- **Root cause:** Rootful Docker-ի uid մոդելը կիրառված է rootless daemon-ի վրա։
- **Real-world impact (եթե հաստատվի):** `--workdir /workspace`-ը կամ ֆայլ գրելը կստանա permission denied, և ամեն tool step կավարտվի `sandbox exit 126/1`-ով։ Ոչ մի agent չի կարողանա ֆայլ ստեղծել։
- **Minimal safe reproduction (Hetzner-ում):** `python -c "from bro.sandbox import *; print(DockerSandbox().run('/path/0700dir',['python3','-c','open(\"x\",\"w\").write(\"1\")']))"` rootless daemon-ով, ապա `ls -ln` host-ում։
- **Expected:** Ֆայլը ստեղծվում է և host-ում պատկանում է Bro user-ին։
- **Actual:** Չի ստուգվել։
- **Smallest fix:** Rootless ռեժիմում `--user 0:0` (= host-ի unprivileged user) + պահպանել `--cap-drop ALL`, `no-new-privileges`, `--read-only`։ Կամ ստուգել `docker info` → rootless և ըստ այդմ ընտրել uid։
- **Related:** AUD-021 (`test_real_docker_offline`-ը skip է անում ցանկացած `SandboxError`-ի դեպքում և կթաքցնի հենց այս ձախողումը), AUD-025
- **Regression test:** Իրական rootless daemon-ով integration test՝ գրել ֆայլ և host-ում ստուգել owner-ը։ Docker-ի առկայության դեպքում test-ը պետք է **fail** լինի, ոչ թե skip։

### AUD-004 — Կեղծ կամ չստուգված արդյունքն ընդունվում է որպես `completed`
- **Severity:** High · **Status:** Reproduced
- **File/lines:** `bro/sandbox.py:233-239`, `bro/core.py:301-305, 323`
- **Problem:** Worker-ը կարող է առաջին քայլում վերադարձնել `{"action":"finish","result":"I built and tested the full website…"}` առանց որևէ tool action-ի։ Run-ը ստանում է `completed`, job-ը՝ `completed`։
- **Root cause:** Run-ի state-ը որոշվում է միայն model-ի հայտարարությամբ։ Tool step-երի քանակը, exit code-երը և manifest-ը չեն ազդում state-ի վրա և չեն փոխանցվում summary-ին։
- **Real-world impact:** Բացահայտ խախտում է requirement 13-ը («failed tasks must never silently appear successful»)։ Owner-ը տեսնում է «completed», իսկ workspace-ը դատարկ է։
- **Reproduction:** `FAKE_MODE=fabricate` → `[A] job: completed | runs: [('completed', []), ('completed', [])]`
- **Expected:** `completed_unverified` / `no_evidence` կամ առնվազն summary-ում «0 tool steps, 0 files»։
- **Actual:** `completed`։
- **Smallest fix:** `SandboxClaudeCLI.call`-ը վերադարձնի `(result, steps_executed)`։ `runs` աղյուսակում պահել `tool_steps`։ Job-ը `completed` լինի միայն այն դեպքում, երբ յուրաքանչյուր run ունի ≥1 հաջող tool step կամ դատարկ չէ manifest-ը։ Հակառակ դեպքում՝ `unverified`։ Summary-ին փոխանցել steps և manifest։
- **Related:** AUD-013
- **Regression test:** fabricate provider → job state ≠ `completed`։

### AUD-005 — Ենթախնդիրների միջև dependency և տվյալների փոխանցում չկա․ «մեկ project, մի քանի agent» սցենարը իրագործելի չէ
- **Severity:** High · **Status:** Confirmed (static + integration)
- **File/lines:** `bro/core.py:269-274` («independent tasks»), `core.py:281-288` (plan schema-ում `depends_on` չկա), `core.py:292` (`workspaces/jid/rid`), `core.py:314-318` (բոլորը միանգամից submit)
- **Problem:** Plan-ը հարթ ցուցակ է։ Բոլոր task-երը գործարկվում են զուգահեռ, ամեն մեկը իր դատարկ workspace-ում։ Agent B-ն չի տեսնում agent A-ի output-ը, իսկ Test Engineer-ը չունի կոդ, որը պետք է թեստավորի։
- **Root cause:** Shared project workspace կամ dependency մոդել չկա։ Միակ «միաձուլումը» տեքստային summary-ն է։
- **Real-world impact:** «Multi-file website + backend + tests» ստացվում է որպես 3 անկախ թղթապանակ։ Վերջնական արդյունքը ինտեգրված չէ և անմիջապես օգտագործելի չէ։ Requirement 4.5 (decomposition + dependencies) բացակայում է։
- **Reproduction:** `r_conc.py`․ 30 run → 30 առանձին workspace, ոչ մի ընդհանուր ֆայլ։ Trace-ը տես H բաժնում։
- **Expected:** Կախյալ task-ը սկսվում է նախորդից հետո և տեսնում է նրա artifact-ները։
- **Actual:** Ամեն ինչ զուգահեռ և մեկուսացված է։
- **Smallest fix:** Plan item-ում ավելացնել optional `depends_on: [index]`։ Կատարել wave-երով (topological)։ Կախյալ run-ի workspace-ում **պատճենել** upstream run-երի artifact-ները (`upstream/<agent_id>/…`)։ Նոր subsystem պետք չէ։
- **Related:** AUD-006
- **Regression test:** fake plan՝ B depends_on A → B-ի workspace-ում կա A-ի ֆայլը, և B-ն սկսվում է A-ից հետո։

### AUD-006 — `continue_job`-ը չի տեսնում նախորդ artifact-ները (նոր, դատարկ workspace)
- **Severity:** High · **Status:** Reproduced
- **File/lines:** `bro/core.py:236-265`, `core.py:292`
- **Problem:** Continuation-ը ստեղծում է նոր job և նոր run-եր՝ նոր `workspaces/<new_jid>/<new_rid>` դատարկ թղթապանակներով։ Prompt-ը ասում է «existing artifact paths…», բայց այդ ֆայլերը container-ում հասանելի չեն։
- **Root cause:** Continuation-ը փոխանցում է միայն metadata, ոչ թե workspace։
- **Real-world impact:** «Continue unfinished work»-ը փաստացի զրոյից վերսկսում է։ Model-ը կարող է «խմբագրել» գոյություն չունեցող ֆայլ կամ կեղծ հաջողություն հայտարարել (AUD-004)։ Requirement 12-ը (continuation) չի կատարվում։
- **Reproduction:** `[B] first job: completed artifacts [index.html]` → `continuation job: completed | new workspace files: [] | prompt mentions index.html: True`
- **Expected:** Նոր run-ը տեսնում է նախորդ `index.html`-ը (read-only կամ պատճեն)։
- **Actual:** Դատարկ workspace։
- **Smallest fix:** `continue_job`-ում պահել `previous_job_id`-ը։ `_work`-ում, եթե job-ը continuation է, run-ի workspace-ը սկզբում լրացնել նախորդ նույն `agent_id` run-ի workspace-ի պատճենով (`shutil.copytree(..., symlinks=True)`)։ Նախորդ workspace-ը մնում է անփոփոխ։
- **Related:** AUD-005, AUD-004
- **Regression test:** Repro B-ն՝ ակնկալելով `index.html` նոր workspace-ում։

### AUD-007 — Host Claude CLI-ն մեկուսացված չէ user/project config-ից (MCP, hooks, plugins, settings, CLAUDE.md)
- **Severity:** High · **Status:** Unverified (config-ից կախված)
- **File/lines:** `bro/core.py:136-151`, `bro/sandbox.py:207` (`tempfile.mkdtemp()` → `/tmp/bro-model-*`)
- **Problem:** `--tools ''`-ը ըստ `--help`-ի անջատում է միայն **built-in** tool-երը։ Չեն փոխանցվում `--strict-mcp-config`, `--setting-sources`, `--disable-slash-commands`, `--no-session-persistence`։ Host-ի `~/.claude` / `~/.claude.json`-ի MCP server-ները, hooks-ը (host shell command-ներ ամեն call-ի ժամանակ), plugin/skill-երը և allow rule-երը կարող են ակտիվ մնալ։ Worker-ի cwd-ը `/tmp`-ի տակ է, իսկ Claude Code-ը CLAUDE.md-ը կարդում է cwd-ից դեպի վեր, ուստի `/tmp/CLAUDE.md`-ը կարող է prompt injection դառնալ։
- **Root cause:** «Model-ը host tool չունի» ենթադրությունը հիմնված է միայն `--tools ''`-ի վրա։
- **Real-world impact:** Եթե Hetzner-ի user-ի մոտ կա MCP server (օր.՝ filesystem/ssh) կամ hook, AI-ի առաջարկած գործողությունը կարող է կատարվել **sandbox-ից դուրս**՝ host-ում։
- **Reproduction (Hetzner, անվնաս):** `~/.claude/settings.json`-ում ավելացնել `SessionStart` hook, որ գրում է `/tmp/hook_ran`։ Գործարկել մեկ Bro job և ստուգել ֆայլը։ Կամ փորձնական MCP server, որ log է անում կանչերը։
- **Expected:** Bro-ի CLI call-ը չի բեռնում ոչ մի MCP/hook/plugin։
- **Actual:** Չի ստուգվել։
- **Smallest fix:** argv-ին ավելացնել `--strict-mcp-config --disable-slash-commands --no-session-persistence` և `--setting-sources`-ը սահմանափակել (CLI 2.1.322-ում կա նաև `--restricted`)։ Bro-ն գործարկել առանձին OS user-ով՝ մաքուր `~/.claude`-ով։ `model_dir`-ը տեղափոխել `BRO_HOME/model/…` (0700), ոչ թե `/tmp`։
- **Related:** AUD-020, AUD-026
- **Regression test:** Hetzner-ի վրա hook/MCP canary test։ Unit test, որ argv-ն պարունակում է այս flag-երը։

### AUD-008 — SIGTERM/crash-ից հետո մնում են orphan Claude CLI պրոցեսներ (և, հավանաբար, container-ներ)
- **Severity:** Medium · **Status:** Reproduced (CLI) / Unverified (container)
- **File/lines:** `bro/core.py:149-151` (`start_new_session=True`), `bro/sandbox.py:117-118`, `bro/server.py:110-111` (signal handler չկա)
- **Problem:** SIGTERM-ի դեպքում Python-ը մահանում է առանց `finally`-ի։ Child-երը առանձին session-ում են և մնում են կենդանի։ Restart-ը նշում է `interrupted`, բայց պրոցեսները չի մաքրում։ Rootless dockerd-ի container-ները Bro-ի cgroup-ում չեն, իսկ container-ի ներսում timeout չկա։
- **Real-world impact:** Restart-ից հետո ծախսվում են subscription/CPU/RAM։ «Unknown outcome»-ով պրոցեսը կարող է շարունակել գրել workspace-ում։
- **Reproduction:** `r_kill.py`․ `fake claude procs before SIGTERM: 3 → after SIGTERM: 3 → orphans still running after restart: 3`
- **Expected:** Shutdown/restart-ը մաքրում է child process group-երը և Bro-ի container-ները։
- **Actual:** Orphan-ներ։
- **Smallest fix:** `signal.signal(SIGTERM, …)` → `server.shutdown()` + kill tracked pgid-ներ։ `docker run --label bro.run=<rid>`, իսկ startup-ին՝ `docker rm -f $(docker ps -q --filter label=bro.run)`։ Container-ում `timeout <sec>` wrapper։ systemd unit՝ `KillMode=control-group`։
- **Related:** AUD-009
- **Regression test:** Repro-ի նման՝ SIGTERM-ից հետո 0 child։

### AUD-009 — Running job/run-ը չեղարկելու (cancel) մեխանիզմ չկա
- **Severity:** Medium · **Status:** Confirmed
- **File/lines:** `bro/server.py:71-106` (cancel endpoint չկա), `bro/core.py` (cancel state չկա)
- **Problem / impact:** Սխալ կամ վտանգավոր job-ը (30 agent × 12 քայլ, մինչև ~7 ժամ/run) կարելի է կանգնեցնել միայն ամբողջ Bro-ն սպանելով, ինչը հանգեցնում է AUD-008-ին։
- **Smallest fix:** `POST /jobs/{id}/cancel` → `cancel_requested` դրոշ DB-ում։ Worker loop-ը այն ստուգում է ամեն քայլից առաջ, իսկ ընթացիկ pgid-ը kill է արվում։
- **Regression test:** Slow fake provider, cancel → run `cancelled`, 0 child process։

### AUD-010 — Skill router-ը keyword-ային է․ 16/42 skill գործնականում անհասանելի է, հայերենի հոլովված ձևերը չեն ճանաչվում, կան false positive-ներ
- **Severity:** Medium · **Status:** Reproduced
- **File/lines:** `bro/router.py:281-298`, `config/skill_routes.json`
- **Problem:** 16 skill-ի միակ tag-ը նրա slug-ն է (`cybersecurity operations soc`, `accounting bookkeeping tax`, …)։ Համընկնումը միայն ամբողջական բառով է, ուստի «կայքը», «թեստեր», «tests»-ը չեն համընկնում։ `router` բառը ընտրում է telecom-ը նաև «React router»-ի դեպքում։ «Review this NDA»-ն ընտրում է auditing-ը։
- **Real-world impact:** Չհամընկնելու դեպքում ընտրվում է agent-ի primary skill-ը, այսինքն՝ ոչ համապատասխան հրահանգներ։ `analysis-primary`-ն, որը նախատեսված է որպես «first-pass», երբեք չի բեռնվում։
- **Reproduction:** `r_router.py`, agent `agt-p04-r02`, 20 ներկայացուցչական task (EN/HY) → **4/20** ճիշտ։ Օրինակներ․ `"Ստեղծիր կայքը մեր բաժանորդների համար" → software-systems-architecture`, `"Write unit tests…" → software-systems-architecture`, `"Fix the React router navigation bug" → [frontend, telecom-isp-network-ops]`։ (Task-երը ընտրել եմ ես, դրանք benchmark չեն։ Telecom agent-ի համար GPON-ը կընկնի fallback-ի վրա և պատահաբար ճիշտ կլինի։)
- **Expected:** Համապատասխան skill-ը ընտրվում է նաև հայերեն հոլովված ձևերով և սովորական բառերով։
- **Smallest fix:** (1) Chief-ի plan item-ում թույլատրել optional `"skills": [id,…]` (≤2, validated)։ Chief-ն արդեն տեսնում է catalog-ը, ուստի սա semantic ընտրություն է առանց նոր component-ի։ Router-ը մնում է fallback։ (2) `skill_routes.json`-ում ավելացնել բնական tag-եր 16 skill-ի համար և prefix match ≥4 նիշանոց հայերեն tag-երի համար։
- **Regression test:** 20+ fixture (EN/HY), որոնց ակնկալվող skill-ը hardcoded է։

### AUD-011 — SKILL.md-երը հակասում են runtime-ի access մոդելին
- **Severity:** Medium · **Status:** Confirmed
- **File/lines:** `skills/*/SKILL.md` (adaptation բաժին, օր.՝ `legal-compliance-contracts/SKILL.md:46`), `config/skill_dependency_status.json` (`version: v0.14`, բոլոր 42-ում `primary_agents: []`)
- **Problem:** 16 skill գրում է «Current mapped agents: NONE — no dedicated active agent», իսկ 26-ը թվարկում է ենթաբազմություն և «Mapping comes from config/agent_skills.json»։ v0.14-ից ի վեր, սակայն, բոլոր 30 agent-ը մուտք ունեն բոլոր 42-ին։ Այս տեքստը բառացի մտնում է worker-ի system prompt։
- **Impact:** Model-ը կարող է հրաժարվել աշխատանքից կամ գրել «ownership unresolved»։
- **Smallest fix:** Փոխարինել տողը չեզոքով («Any registered Bro worker may apply this skill when assigned»), վերահաշվել `index.json`-ի hash-երը, թարմացնել `primary_agents`-ը։
- **Regression test:** Test, որ ոչ մի SKILL.md չի պարունակում «Current mapped agents»։

### AUD-012 — Legacy governance բառապաշար առանց runtime մեխանիզմի (grant, receipts, verifier, GREEN/RED)
- **Severity:** Medium · **Status:** Confirmed
- **File/lines:** `skills/*/SKILL.md`․ «exact governing grant and approval boundary»՝ 42/42, «independent verifier»՝ 42/42, «mode grant»՝ 21, «stale receipts»՝ 22, «exact-head evidence»՝ 10, GREEN/RED՝ ~41
- **Problem:** Workflow-ի 1-ին քայլը հաճախ «Confirm identity, mode grant… refuse if…» է։ Bro-ում grant, receipt կամ verifier մեխանիզմ չկա։ Adaptation բաժինը մասամբ բացատրում է, բայց «grant»-ը չի սահմանում։
- **Impact:** Անհամապատասխան մերժումներ («missing grant») կամ հակառակը՝ grant-ի «հորինում»։
- **Smallest fix:** Adaptation բաժնում ավելացնել մեկ տող․ «grant = Owner-ի task text-ը. independent verifier ավտոմատ չկա → գրիր "verification not performed"»։ Domain բովանդակությունը չփոխել։
- **Regression test:** Static test՝ adaptation-ում կա grant-ի սահմանումը։

### AUD-013 — Worker-ի execution միջավայրը չի համապատասխանում agent-ների հայտարարված պարտականություններին
- **Severity:** Medium · **Status:** Confirmed
- **File/lines:** `bro/sandbox.py:26` (`python:3.12-slim`), `sandbox.py:177, 210-218` (12 քայլ, 64 KiB սահմանները model-ին չեն ասվում), `core.py:269-274` (chief-ին սահմանները չեն ասվում)
- **Problem:** Container-ը offline է և ունի միայն Python stdlib․ չկան node/npm, git, compiler, DB engine, browser։ Model-ը web search չունի։ Researcher-ի skill-ը պահանջում է «Search systematically across independent sources», Mobile Engineer-ը՝ device matrix, Frontend-ը՝ cross-browser/screenshots, DB-ը՝ EXPLAIN։
- **Impact:** 30 agent-ը **registered** է, protocol-ի մակարդակով **runnable**, բայց **functionally verified** չէ։ Մի քանի agent ֆիզիկապես չի կարող կատարել հիմնական գործառույթը (տես E)։
- **Smallest fix:** Chief/worker prompt-ում հստակ նշել սահմանները (offline, միայն Python stdlib, ≤12 քայլ, finish ≤64 KiB)։ Per-agent `image` դաշտ `agents.json`-ում (`node:22-slim` frontend-ի համար)՝ կիրառելով արդեն գոյություն ունեցող `claude_image` պարամետրը։
- **Regression test:** Plan prompt-ը պարունակում է սահմանները։ Per-agent image-ը փոխանցվում է `build_command`-ին։

### AUD-014 — JSON tool protocol-ը խոցելի է և token-ների առումով անարդյունավետ (stateless re-send)
- **Severity:** Medium · **Status:** Confirmed
- **File/lines:** `bro/sandbox.py:220-232, 248-250`
- **Problem:** Ցանկացած պատասխան, որ մաքուր JSON չէ (prose + JSON), անմիջապես ձախողում է ամբողջ run-ը, առանց retry-ի։ Ամեն քայլը նոր `claude -p` պրոցես է՝ ամբողջ conversation-ով։ N task-ի համար սա 2+N×steps CLI պրոցես է (30 task, 2 քայլ → 62 պրոցես, չափված), իսկ token-ների ծախսը աճում է քառակուսային։
- **Answer to §8:** Սա **ոչ թե ինքնավար Claude Code agent է**, այլ custom ReAct loop մեկանգամյա tool-less պատասխանների վրա։ Session-ը չի պահպանվում (`--resume` չի օգտագործվում)։
- **Smallest fix:** Արտահանել առաջին JSON object-ը և անել մեկ corrective retry։ Session-ը պահել `--session-id`/`--resume`-ով և ամեն քայլին ուղարկել միայն նոր observation-ը։
- **Regression test:** Provider-ը վերադարձնում է `"Sure! {...}"` → run-ը շարունակվում է։

### AUD-015 — Diagnostics չկա․ stderr-ը կորչում է, logging չկա, quota/auth սխալները չեն տարբերակվում
- **Severity:** Medium · **Status:** Confirmed
- **File/lines:** `bro/core.py:150, 165, 178`, `bro/sandbox.py:118, 142`, `bro/server.py:107` (`log_message`-ը անջատված է). Ամբողջ codebase-ում `logging` չկա։
- **Problem:** Error-ները գրում են «inspect broker-side diagnostics», բայց այդպիսի diagnostics գոյություն չունի։ Subscription-ի սպառումը, login-ի ժամկետի ավարտը և AUD-001-ի parse error-ը տալիս են նույն «exit 1»-ը։ Circuit breaker չկա, ուստի 30 run-ը ձախողվում են հերթով։
- **Impact:** Hetzner-ում խնդիրների ախտորոշումը գործնականում անհնար է։ Հենց այդ պատճառով AUD-001-ը վեց ռաունդ աննկատ մնաց։
- **Smallest fix:** Bounded (օր.՝ 8 KB) stderr-ը գրել `BRO_HOME/logs/broker.log` (0600) ֆայլում, ոչ DB-ում։ Դասակարգել հայտնի ձևանմուշները (auth / usage limit / parse)։ N հաջորդական auth/quota սխալից հետո դադարեցնել նոր run-երի dispatch-ը։
- **Regression test:** fake CLI-ն stderr-ում գրում է «usage limit» → job error-ը `quota_exhausted` է, log-ում կա bounded հատված, DB-ում՝ ոչ։

### AUD-016 — Agent ավելացնելը/անջատելը modular չէ (3 config ֆայլ, 42 entry, `enabled:false`-ը կոտրում է startup-ը)
- **Severity:** Medium · **Status:** Reproduced
- **File/lines:** `bro/core.py:26, 35, 208-210`, `bro/dependencies.py:318-322`, `config/skill_dependency_status.json`
- **Reproduction (պատճենում):**
  - միայն `agents.json` → `ValueError agent skills registry mismatch`
  - + `agent_skills.json` → `skill agent ownership drift`
  - + 42 entry `skill_dependency_status.json`-ում → OK (31 agent)
  - `enabled:false` → `registry mismatch`
  - `agent_id: "network-planner"` → `invalid agent IDs`
  - 31 agent-ի դեպքում 6 test ձախողվում է (hardcoded 30)։
- **Impact:** Requirement 2-ը («expandable without restructuring») մասամբ է կատարվում։ Core կոդ փոխել պետք չէ, բայց պահանջվում է եռակի ձեռքով bookkeeping։ `enabled` դրոշը անօգտագործելի է։
- **Smallest fix:** `skill_dependency_status.json`-ից հեռացնել `agents` (հաշվարկել runtime-ում)։ `agent_skills.json`-ում բացակայող agent-ը համարել «առանց primary hint»։ Disabled agent-ները զտել mismatch ստուգումից առաջ։ Tests-ում 30-ի փոխարեն օգտագործել `len(registry())`։
- **Regression test:** Repro-ի 5 սցենարը։

### AUD-017 — Artifact manifest-ը սահմանափակված է 200 ֆայլով․ մնացածը հնարավոր չէ ներբեռնել, archive download չկա
- **Severity:** Medium · **Status:** Reproduced
- **File/lines:** `bro/artifacts.py:4, 12-14`, `bro/server.py:48-51, 58`
- **Reproduction:** 250 ֆայլ → `manifest recorded 200 of 250; truncated=True`։ `GET …/f240.txt` → `404 artifact not recorded`։
- **Impact:** «Hundreds of files» project-ը ամբողջությամբ չի ներբեռնվում։ Ֆայլերը ներբեռնվում են միայն մեկ առ մեկ, և բոլորը կոչվում են `filename="artifact"`։
- **Smallest fix:** Ամբողջական ցուցակը գրել workspace-ից դուրս (`BRO_HOME/manifests/<rid>.json`, 0600), իսկ DB-ում պահել միայն ամփոփում։ `GET …/runs/{rid}/archive` endpoint՝ `tarfile` stream, O_NOFOLLOW։ `Content-Disposition`-ում՝ basename։
- **Regression test:** 250 ֆայլ → f240-ը ներբեռնվում է։ Archive-ը պարունակում է 250 ֆայլ։

### AUD-018 — Workspace-ում ցանկացած symlink (որը AI-ն ստեղծել է container-ում) ձախողում է բոլոր հաջորդ քայլերը
- **Severity:** Medium · **Status:** Reproduced
- **File/lines:** `bro/sandbox.py:73-74`
- **Reproduction:** fake docker-ը առաջին քայլում ստեղծում է `node_modules_link -> /etc` → `[C] job: partial | run error: workspace contains symlinks`
- **Impact:** npm (`node_modules/.bin`), venv և git-ի որոշ կառուցվածքներ օրինական symlink-եր են։ Run-ը մշտապես կոտրվում է։ Անվտանգության տեսանկյունից սա պարտադիր չէ․ container-ը host symlink-երով չի կարող դուրս գալ, իսկ host-ի կողմում կարդալը (artifacts) արդեն O_NOFOLLOW է։
- **Smallest fix:** Մերժել միայն workspace root-ը, եթե այն symlink է (արդեն արվում է)։ Ներքին symlink-երը թույլատրել։ `manifest`-ը արդեն skip է անում symlink-երը։
- **Regression test:** Repro C-ն → երկրորդ քայլը հաջողվում է։ Download-ը `linked/passwd`-ով մերժվում է (արդեն կա)։

### AUD-019 — Vault adapter-ը միացված չէ ոչ մի runtime ճանապարհի
- **Severity:** Medium · **Status:** Confirmed
- **File/lines:** `bro/vault.py` (ամբողջը)։ `VaultKV2`-ը import է արվում միայն `tests/test_vault_boundary.py`-ում։
- **Problem:** Requirement 9-ը (կենտրոնացված credentials, Vault) runtime-ում չկա։ Կա միայն adapter։ `read()`-ը token file-ի owner-ը չի ստուգում (`server.py`-ն ստուգում է), և `is_symlink()`/`read_text()` միջև կա TOCTOU։
- **Impact:** Credentials-ի կառավարման պահանջը **Missing** է։ Ներկայում connector-ներ չկան, ուստի ակտիվ խոցելիություն չկա։
- **Smallest fix:** Մինչև առաջին connector-ը նշել README-ում որպես «not wired»։ `read()`-ում ավելացնել `st_uid == getuid()` և `os.open(O_NOFOLLOW)`։
- **Regression test:** Owner mismatch → `VaultError`։

### AUD-020 — Ամբողջ broker environment-ը փոխանցվում է Claude CLI-ին․ `ANTHROPIC_AUTH_TOKEN`/`ANTHROPIC_BASE_URL`-ը չեն հեռացվում
- **Severity:** Medium · **Status:** Confirmed (static), mutation M3-ը survived
- **File/lines:** `bro/core.py:141-144`
- **Problem:** Հեռացվում է միայն 4 փոփոխական։ Service env-ի մնացած բոլոր secret-ները (cloud keys, GitHub token և այլն) հասանելի են CLI-ին և նրա spawn արած hooks/MCP-ներին։ `ANTHROPIC_AUTH_TOKEN` կամ `ANTHROPIC_BASE_URL` դրված լինելու դեպքում CLI-ն կարող է օգտագործել ոչ subscription auth կամ այլ endpoint։
- **Smallest fix:** Env allowlist․ `PATH, HOME, LANG, LC_*, TZ, CLAUDE_CODE_OAUTH_TOKEN` (եթե օգտագործվում է), proxy փոփոխականներ։ Մնացածը՝ drop։
- **Regression test:** fake CLI-ն dump է անում env-ը → չկան `ANTHROPIC_*` (բացի allowlist-ից) և test secret-ը։

### AUD-021 — Test suite-ի որակ․ վակուում թեստեր, skip-ը թաքցնում է ձախողումը, կրիտիկական ճյուղեր չթեստավորված
- **Severity:** Medium · **Status:** Reproduced (mutation)
- **File/lines:**
  - `tests/test_round4.py:64-75`․ patch է անում `subprocess.run`, բայց `DockerSandbox.run`-ը օգտագործում է `Popen`։ Թեստը իրականում գործարկում է host-ի **իրական** `docker run` CLI-ն։ Այստեղ ստացվեց «sandbox exit 1», իսկ daemon ունեցող server-ում իրական container կգործարկվի։
  - `tests/test_sandbox.py:46-55`․ skip է անում **ցանկացած** `SandboxError`-ի դեպքում։
  - `test_skill_dependency_round3.py:27`, `:30`․ relative path-եր, կախված cwd-ից։
  - HTTP-ն unittest-ում չէ։ Argv-ի test չկա։
- **Mutation results (14):**
  - **SURVIVED:** M1 (download-ը առանց run state-ի), M3 (API key-ը չի հեռացվում), M4 (run-երը restart-ից հետո չեն նշվում interrupted), M6 (`--user`-ը հեռացված), M7 (container output-ը error-ում), M11 (MAX_STEPS 1000), M12 (AUD-001-ի ուղղումը՝ թեստերը չեն տարբերակում կոտրվածն ու ուղղվածը)։ M2 (auth-ը անջատված)՝ միայն `http_smoke`-ն է բռնում։
  - **KILLED:** M5, M8, M9, M10, M13, M14։
- **Impact:** «66 tests OK»-ն չի ապացուցում CLI-ի, Docker-ի կամ HTTP-ի ճշտությունը։
- **Smallest fix:** `test_round4`-ում patch անել `Popen`-ը։ Docker test-ը skip անել միայն `not available()`-ի դեպքում։ `http_smoke`-ը դարձնել unittest։ Ավելացնել argv test (AUD-001), restart-ի runs test, env test։ Tests-ում path-երը կառուցել `__file__`-ից։
- **Regression test:** Կրկնել mutation run-ը, որտեղ բոլոր 14-ը պետք է KILLED լինեն։

### AUD-022 — HTTP hardening․ socket timeout չկա, thread-երը անսահմանափակ են, `/health`-ը ստատիկ է, unhandled exception-ներ
- **Severity:** Low · **Status:** Confirmed (static) / Reproduced (մասամբ)
- **File/lines:** `bro/server.py:25-111`
- **Details:**
  - `Content-Length`-ը մեծ է իրական body-ից → `rfile.read` blocking է առանց timeout-ի։
  - `ThreadingHTTPServer`-ը thread-երի սահմանափակում չունի։
  - `/health`-ը միշտ `{"ok":true}` է, անկախ Claude/Docker/DB վիճակից։
  - `submit`-ի ոչ ակնկալվող exception-ը (օր.՝ `sqlite3.OperationalError`) փակում է կապը առանց պատասխանի։
  - Stream-ի ընթացքում `BrokenPipe`-ի դեպքում փորձ է արվում ուղարկել երկրորդ response։
  - `Content-Length: -5`-ը տալիս է «413 body too large»։
  - Error-ները բացահայտում են Python-ի ներքին հաղորդագրությունները։
- **Impact:** Ցածր, քանի որ server-ը loopback է, token-ով և SSH tunnel-ով։
- **Smallest fix:** `Handler.timeout = 30`։ Catch-all `except Exception` → 500։ `/health`-ում DB ping։
- **Regression test:** Slow-body client → թել-ը ազատվում է ≤30 s-ում։

### AUD-023 — Հնացած կամ հակասական փաստաթղթեր և evidence ֆայլեր
- **Severity:** Low · **Status:** Confirmed
- **Details:**
  - `audit/TEST_RESULTS.txt`-ը ցույց է տալիս 34 test և `test_never_load_unassigned_skill`, որն այլևս չկա։
  - `config/agents.json:222`․ «Skills are not bundled» (իրականում 42 skill կա)։
  - `README.md:62`․ «router still loads skills only when assigned» (v0.14-ից հնացած)։
  - `skill_dependency_status.json` → `version: v0.14`։
  - README-ն գրում է «Bro calls the installed claude CLI», ինչը փաստացի չի աշխատում (AUD-001)։
- **Smallest fix:** Թարմացնել կամ հեռացնել։

### AUD-024 — Root-ով գործարկումը չի մերժվում startup-ին
- **Severity:** Low · **Status:** Reproduced (այս միջավայրը root էր)
- **File/lines:** `bro/core.py:181-199`, `bro/sandbox.py:54-55`
- **Problem:** `Bro()`-ն root-ով աշխատում է։ Planning-ը կատարվում է, բայց ամեն tool step ձախողվում է «must not run as host root»-ով, այսինքն՝ ուշ և աղմկոտ։
- **Smallest fix:** `Bro.__init__`-ում `os.getuid()==0` → error (`provider=None`, այսինքն՝ real mode-ի դեպքում)։

### AUD-025 — Rootless Docker-ում resource limit-երը կարող են լուռ անտեսվել
- **Severity:** Low · **Status:** Unverified
- **File/lines:** `bro/sandbox.py:82-83`, `__main__.py:138` (`doctor`)
- **Problem:** Առանց cgroup v2 delegation-ի rootless Docker-ը `--memory/--cpus/--pids-limit`-ի համար տալիս է warning և անտեսում դրանք։ Bro-ն stderr-ը չի տեսնում, իսկ `doctor`-ը սա չի ստուգում։
- **Smallest fix:** `doctor`-ում ստուգել `docker info --format '{{.CgroupVersion}} {{.CgroupDriver}}'` և փորձնական `--memory`-ով run։

### AUD-026 — Claude session transcript-ները պահվում են ամեն քայլի համար (`~/.claude/projects`)
- **Severity:** Low · **Status:** Confirmed (`--help`: session persistence-ը default է)
- **Impact:** Մեկ job-ը կարող է ստեղծել մինչև ~360 transcript ֆայլ՝ prompt-երով և tool output-ով։ Սա անսահմանափակ աճ է և տվյալների պատճեն `BRO_HOME`-ի permission մոդելից դուրս։
- **Fix:** `--no-session-persistence` (մաս է AUD-007-ի ուղղման)։

### AUD-027 — Retention, quota և idempotency բացակայում են
- **Severity:** Low · **Status:** Confirmed
- **Details:** Workspace-երը երբեք չեն մաքրվում, disk quota չկա։ `POST /jobs`-ը չունի idempotency key, ուստի client-ի retry-ը ստեղծում է կրկնակի job (կրկնակի ծախս)։ `GET /jobs`-ը վերադարձնում է միայն վերջին 30-ը, առանց pagination-ի։
- **Fix:** `BRO_RETENTION_DAYS` + startup cleanup։ Optional `Idempotency-Key` header (unique column)։

---

## D. Requirements Traceability Matrix

| # | Requirement | Կարգավիճակ | Ապացույց / finding |
|---|---|---|---|
| 1 | Մեկ main Bro AI | **Partial** | Կոդը կա, mock-ով աշխատում է։ Իրական CLI-ով ձախողվում է (AUD-001) |
| 2 | 30 agent, ընդլայնելի | **Partial** | 30 registered (verified)։ Ընդլայնումը՝ 3 ֆայլ, `enabled:false`-ը կոտրված է (AUD-016) |
| 3 | Agent-ները զուգահեռ | **Implemented, orchestration-level verified** | 30 զուգահեռ subprocess (peak=30, չափված)։ Live Claude/Docker-ով՝ unverified |
| 4 | 42 Skills library | **Implemented & verified** | 42 hash OK, 42/42 load |
| 5 | Ամեն agent → ամեն Skill | **Implemented & verified (access/load)** | 1260/1260 accessible և loadable։ Functional՝ 0/1260 verified |
| 6 | Contextual lazy loading | **Partial** | Lazy՝ verified (1 read)։ Contextual-ը keyword է, 4/20, 16 skill slug-only (AUD-010) |
| 7 | Subscription, ոչ API key | **Partial** | `ANTHROPIC_API_KEY`-ը հեռացվում է։ `ANTHROPIC_AUTH_TOKEN/BASE_URL`-ը՝ ոչ (AUD-020)։ Live-ը blocked է AUD-001-ով |
| 8 | Tool work-ը isolated sandbox-ում | **Implemented, unverified** | Flag-երը ճիշտ են (static + unit)։ Rootless uid-ը՝ AUD-003, host CLI config-ը՝ AUD-007 |
| 9 | Credentials-ը կենտրոնացված (Vault) | **Missing (runtime)** | Կա միայն adapter (AUD-019) |
| 10 | Agent-ները credentials չեն ստանում | **Partial** | Container-ը՝ ոչ մի credential (static verified)։ Host CLI պրոցեսը՝ subscription auth + ամբողջ env (AUD-020, AUD-007) |
| 11 | Մեծ project-ներ և ֆայլեր | **Partial** | Workspace-ը անսահմանափակ է։ Սահմանափակումներ՝ 12 քայլ, 200-ֆայլանոց manifest, E2BIG, archive չկա (AUD-002, -013, -017) |
| 12 | Task/artifact-ները survive interruption | **Partial** | `interrupted` marker-ը verified է (kill + restart)։ Continuation-ը չի տեսնում artifact-ները (AUD-006) |
| 13 | Failure-ը երբեք որպես success | **Partial** | Failure propagation-ը verified է։ Կեղծ completion-ը ընդունվում է (AUD-004) |
| 14 | Desktop app պետք չէ | **Implemented & verified** | CLI + loopback HTTP |
| 15 | Անընդհատ աշխատանք Hetzner-ում | **Partial / Blocked by environment** | Signal handling, log, cancel, retention չկան (AUD-008, -009, -015, -027) |
| 16 | Հնարավորինս պարզ | **Implemented** | Միայն stdlib, ~900 LOC, 10 module |

Լրացուցիչ (§4)․ AI-driven delegation՝ այո (chief LLM), բայց capability description-ը միայն role+pack է։ Decomposition՝ այո։ Dependencies՝ **Missing** (AUD-005)։ Concurrent state corruption՝ չի հայտնաբերվել (առանձին workspace, SQLite busy_timeout)։

---

## E. Agent–Skill Matrix (30 × 42 = 1260)

Ամբողջական matrix-ը՝ `agent_skill_matrix.csv` (1260 տող)։ Սյունակներ․ accessible, loadable_hash_ok, router_reachable, primary_hint, fallback, live_functional_compatibility։

**Ամփոփում**
| Չափանիշ | Արդյունք | Ինչպես է ստուգվել |
|---|---|---|
| Accessible | **1260/1260** | `SkillLibrary.for_agent()` |
| Loadable (hash OK, prompt built) | **1260/1260** | `prompt_for_agent(agent, selected=[skill])` ամեն զույգի համար |
| Router-reachable բնական tag-ով | **780/1260** (26 skill × 30) | Ամեն tag → `route()` |
| Միայն slug phrase-ով | **480/1260** (16 skill × 30) | `skill_routes.json` |
| Functional compatibility (live) | **0/1260 verified** | Իրական model run չկա (AUD-001) |

**Agent-ների դասակարգում (registered / runnable / functionally verified)**
Բոլոր 30-ը **registered** են (unique ID, role, pack, enabled), իսկ protocol-ի մակարդակով **runnable** են fake binary-ներով։ **Functionally verified՝ 0/30։**

Ըստ sandbox-ի հնարավորությունների (offline `python:3.12-slim`, առանց web search-ի)․
- **Հիմնականում իրագործելի** (տեքստ/փաստաթուղթ/Python stdlib)․ Systems Architect, Service Engineer (stdlib Python), API Engineer, Database Engineer (SQLite only), Test Engineer (unittest), Code Auditor, App Security Auditor (static), Agent Builder, Workflow Engineer, Project Manager, Discovery Analyst, Data Scientist (առանց pandas/numpy), Brand Strategist, Account Executive, Support Specialist, Technical Writer, Lifecycle Strategist, Business Analyst, Financial Analyst, Process Analyst, ISP Operations Specialist (MOP/config proposal, առանց device access), UI Designer (HTML/CSS)։
- **Մասնակի** (հիմնական tool-երը բացակայում են)․ UI Engineer (առանց node/npm build), Animation Engineer (առանց browser preview), BI Dashboard Engineer (առանց BI tool), DevOps Engineer և Infrastructure Engineer (առանց docker/terraform/network), Connector Engineer (առանց network)։
- **Հիմնական պարտականությունը անհնար է**․ Researcher (web search չկա), Mobile Engineer (Android/iOS toolchain չկա)։

Ամեն agent-ի prompt-ը կառուցվում է նույն ձևանմուշով (`core.py:294-299`)․ role + id + ≤2 skill (մոտ 9.8 KB)։ Agent-specific tool, image կամ dependency չկա։ Agent-ների միջև տարբերությունը միայն role string-ն ու primary skill hint-ն է։

**Overlap-ներ** (հեռացնել **չեմ** առաջարկում)․ frontend-ux ↔ ui-page-structure, cybersecurity-soc ↔ security-privacy ↔ offensive, data-architecture ↔ data-engineering ↔ databases, analysis-primary ↔ research, finance-unit-economics ↔ pricing, communication-negotiation ↔ procurement-vendor-negotiation, business-strategy ↔ operating-data-roles։

---

## F. Test Results

| # | Command / script | Environment | Արդյունք |
|---|---|---|---|
| 1 | `unzip -tq` + `sha256sum` | — | No errors, 83 ֆայլ |
| 2 | `python3 -m unittest discover -s tests -v` (պատճենում) | Py 3.13.18, root | **66 run, 65 OK, 1 skipped** (`test_real_docker_offline`: «Bro must not run as host root»), 2.3 s |
| 3 | `PYTHONPATH=. python3 tools/http_smoke.py` | loopback | OK |
| 4 | Իրական CLI argv parse (Bro-ի argv) | Claude Code 2.1.322, առանց credentials | **exit 1**, «Input must be provided…» (AUD-001) |
| 5 | `ClaudeCLI().call()` իրական CLI-ով | նույնը | `RuntimeError: Claude CLI failed (exit 1)` |
| 6 | `r_e2big.py` | fake claude | E2BIG ×2, job failed / runs completed (AUD-002) |
| 7 | `r_misc.py` A–D | fake claude + fake docker | A․ կեղծ completed, B․ continuation-ի դատարկ workspace, C․ symlink → partial, D․ E2BIG 7-րդ քայլում |
| 8 | `r_router.py` | — | 4/20, 16 slug-only |
| 9 | Modularity (պատճեն `mod/`) | — | 5 սցենար + 6 test fail 31 agent-ի դեպքում |
| 10 | `r_kill.py` (SIGTERM + restart) | — | 3 orphan, job/runs → interrupted |
| 11 | `r_http.py` (19 request) | loopback server | Բոլոր validation-ները աշխատում են։ 250→200 manifest cap |
| 12 | `r_conc.py` 1/5/10/20/30 | fake binaries, իրական subprocess | Բոլորը completed, peak concurrent CLI = N, wall 1.2–2.3 s, CLI calls = 2+2N։ Նույն agent ×10՝ OK։ Queue cap=5՝ 5 accepted / 3 rejected, միաժամանակ 2 job |
| 13 | `mutate.py` (14 mutation) | — | 6 killed, 7 survived, 1 (auth)՝ միայն http_smoke-ով |
| 14 | Agent–Skill load 1260 | — | 1260/1260 |

**Չի կատարվել (environment)**․ իրական Claude subscription call, Docker daemon (rootless կամ rootful), Vault server, Hetzner-ի ռեսուրսներ, իրական 30 Claude session-ի concurrency, subscription quota։ Բոլոր repro script-երը և fake binary-ները կցված են `repro_scripts.zip`-ում (գործարկում․ `S=<dir> BRO_SRC=<bro_copy> python3 r_*.py`)։

---

## G. Minimal Fix Plan (ըստ dependency-ի և severity-ի)

1. **AUD-001 + AUD-002** (մեկ փոփոխություն `ClaudeCLI.call`-ում)․ prompt-ը փոխանցել stdin-ով, ավելացնել `--`, argv test։ *Առանց սրա մնացածը չի կարելի ստուգել live։*
2. **AUD-007 + AUD-020 + AUD-026** (նույն ֆունկցիան)․ isolation flag-եր, env allowlist, `model_dir`-ը `BRO_HOME`-ում։
3. **AUD-015**․ broker.log + error դասակարգում։ *Անհրաժեշտ է Hetzner-ի test-ի համար։*
4. **Hetzner integration gate**․ AUD-003 (uid), AUD-025 (cgroup), AUD-008 (container cleanup)։ `test_real_docker_offline`-ը դարձնել fail-on-error։
5. **AUD-004**․ tool_steps/evidence և `unverified` state։
6. **AUD-006**, ապա **AUD-005**․ continuation-ի workspace copy, `depends_on` waves + upstream copy։
7. **AUD-018, AUD-017, AUD-013**․ ներքին symlink-երի թույլատրում, ամբողջական manifest + archive, սահմանների բացահայտում prompt-ում, per-agent image։
8. **AUD-010, AUD-011, AUD-012**․ chief-ի `skills` դաշտ, SKILL.md adaptation-ի տողեր (hash-երի վերահաշվարկ)։
9. **AUD-014, AUD-009, AUD-016**․ JSON retry/session resume, cancel, registry-ի պարզեցում։
10. **AUD-021**, ապա Low-երը (AUD-019, 022–027)։

Բոլորը փոքր փոփոխություններ են գոյություն ունեցող ֆունկցիաներում։ Նոր subsystem չկա։

---

## H. Readiness Assessment + Complex Task Trace

**Trace՝ «Build a multi-file website with a backend API and tests» (Owner → HTTP)**
1. `POST /jobs` → `Store.create` (BEGIN IMMEDIATE, cap 100) → `queued` → job_pool (2 slot) → `_process`։
2. `planning` → `ClaudeCLI.call` → **CLI exit 1 (AUD-001) → job `failed`**։ Իրական CLI-ով trace-ը ավարտվում է այստեղ։
3. *(Ենթադրենք՝ ուղղված է)* chief-ը վերադարձնում է 3 անկախ task՝ UI Engineer, Service Engineer, Test Engineer։ Validation-ը ստուգում է agent_id/title/prompt։
4. 3 run զուգահեռ, ամեն մեկը **դատարկ** `workspaces/jid/rid`-ում։ Test Engineer-ը չունի կոդ (AUD-005)։
5. Run loop․ router (keyword, AUD-010) → system prompt (role + ≤2 skill, հակասական տողերով՝ AUD-011/012) → ≤12 × [`claude -p` stateless → JSON → `docker run --network none --user uid` (AUD-003)] → `finish` (կարող է կեղծ լինել, AUD-004)։
6. `manifest` (≤200 ֆայլ) → `completed`։ Summary prompt (AUD-002 ռիսկ) → job `completed`/`partial`։
7. Owner-ը ներբեռնում է ֆայլերը մեկ առ մեկ 3 անկախ թղթապանակից։ Ինտեգրված կայք չի ստացվում։

| Մակարդակ | Կարգավիճակ |
|---|---|
| Ready for local testing | **Միայն fake/mock provider-ով։** Իրական `claude`-ով՝ **ոչ** (AUD-001) |
| Ready for Hetzner integration testing | **Ոչ, քանի դեռ AUD-001/002-ը չեն ուղղվել։** Դրանից հետո՝ այո, որպես AUD-003/007/025/008-ի ստուգման միջավայր, AUD-015-ի log-ով |
| Ready for real multi-agent workloads | **Ոչ** (AUD-004, -005, -006, -013) |
| Ready for production | **Ոչ** |

---

## I. Unverified Items

1. Իրական Claude subscription call-եր, auth-ի ժամկետի ավարտ, usage exhaustion-ի ձևը, rate limit-եր 5–30 զուգահեռ session-ի դեպքում։
2. Rootless Docker uid mapping-ը workspace-ի վրա (AUD-003)։
3. Rootless Docker-ում cgroup limit-երի կիրառումը (AUD-025)։
4. Container cleanup-ը Bro-ի crash-ից հետո (AUD-008-ի container մասը)։
5. Host CLI-ի MCP/hooks/plugins/CLAUDE.md վարքը Hetzner user-ի config-ով (AUD-007)։
6. `python:3.12-slim` image-ի առկայությունը և pull-ի վարքը (`--pull never` չկա)։
7. Docker-ի իրական isolation (seccomp, `--read-only`, `--cap-drop`) կերնելի մակարդակում։
8. Vault server-ի հետ E2E, token renewal, revocation։
9. Model-ի JSON protocol adherence-ը երկար run-երում, chief-ի agent selection-ի որակը։
10. Skill-ների բովանդակության իրական արդյունավետությունը (0/1260 live)։
11. Disk/RAM-ի օգտագործումը իրական 30 container + 30 Node CLI դեպքում (մոտավորապես մինչև 30 GB container memory limit + CLI RSS)։
12. CLI-ի ապագա տարբերակներում `--tools`-ի parsing-ը (AUD-001-ը վերարտադրված է 2.1.322-ով)։

Չեմ պնդում, որ բոլոր defect-ները հայտնաբերված են։

---

## Final Summary

1. **Files inspected:** 83 / 83
2. **Tests executed:** unittest-ի 66 test (65 OK, 1 skip), `http_smoke`, 7 repro script (fake binary-ներով), 14 mutation, 1260 load check, իրական CLI argv parse test (Claude Code 2.1.322)
3. **Findings:** Critical **1** · High **6** · Medium **14** · Low **6** (ընդամենը 27)
4. **Genuinely operational:** SQLite store/admission/recovery marker, instance lock, orchestration-ը մինչև 30 զուգահեռ subprocess, worker loop-ը fake binary-ներով, offline Docker argv-ի կառուցում, bounded output, artifact streaming-ը O_NOFOLLOW-ով, HTTP auth/validation, skill integrity և lazy load, 1260/1260 access
5. **Not operational:** իրական Claude CLI call (AUD-001), մեծ summary-ներ (AUD-002), agent-ների միջև dependency/shared project (AUD-005), continuation-ը artifact-ներով (AUD-006), evidence-ի վրա հիմնված completion (AUD-004), Vault runtime-ում (AUD-019), cancel (AUD-009), semantic skill routing (AUD-010)
6. **Cannot verify without Hetzner:** AUD-003, AUD-007, AUD-008 (container), AUD-025, subscription quota/auth-ի վարքը, 30 live session, Docker isolation
7. **Final verdict:** **FAIL**
