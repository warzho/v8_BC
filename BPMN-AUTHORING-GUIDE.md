# BPMN Authoring & Git Workflow Guide for IBM BAMOE v8 Business Central

Lessons learned from getting the `banking-migration-processes` project to open in the BC Stunner designer. Use this as a reference when authoring new BPMN files (or generating them via LLM) for BAMOE v8 / RHPAM 7.13 Business Central.

---

## 1. What actually went wrong with the original BPMN files

The three original files were **valid BPMN 2.0 per the OMG schema** and had a complete `<bpmndi:BPMNDiagram>` section — so they imported fine, but the BC Stunner designer threw `NullPointerException` the moment we tried to open them, surfacing as the generic *"Unexpected events occurred... ID: -1738241191"* error.

Stunner is **not a generic BPMN renderer**. It's a jBPM-specific editor that reads Drools extension attributes off the `<process>` element, user tasks, data inputs, and script formats. When those are absent, its property readers dereference `null` and abort.

### The five fatal gaps

| # | Issue | Symptom | Fix |
|---|---|---|---|
| 1 | Declared `xmlns:tns="http://www.jboss.org/drools"` instead of `xmlns:drools=...` | Every `drools:*` attribute is silently undefined | Use the `drools:` prefix |
| 2 | `<bpmn2:process>` had no `drools:packageName`, `drools:version`, `drools:adHoc` | NPE in `ProcessPropertyReader.getPackageName` | Add all three |
| 3 | `<bpmn2:itemDefinition structureRef="String"/>` | jBPM can't resolve bare type names | Use FQCNs: `java.lang.String`, `java.lang.Integer`, etc. |
| 4 | `<bpmn2:property id="x" itemSubjectRef="..."/>` with no `name` | Stunner renders variables by `name`, not `id` | Add `name="x"` on every property |
| 5 | `scriptFormat="javascript"` with bodies that mixed `java.lang.*` calls and `.length()` on strings | Script task editor rejects unknown format; bodies weren't real JS anyway | `scriptFormat="http://www.java.com/java"` + rewrite as real Java |

### Secondary issues that would have bitten us next

- User tasks had no `drools:taskName` attribute — Stunner falls back reasonably but some builds NPE.
- Data inputs/outputs had no `drools:dtype` — defaults to `Object` but strict builds complain.
- `<conditionExpression>` had no `language` attribute — defaults vary between jBPM versions.
- Script bodies had raw `<` characters (`creditScore < 650`) which is **invalid XML**. The file parsed only because the XML parser's lenient mode tolerated it during import, but Stunner's stricter parser choked on open.

---

## 2. BPMN authoring checklist (the "always do this" list)

Apply this checklist to every new BPMN file, hand-written or LLM-generated. Omitting any one of these is a near-certain way to hit *"Unexpected events occurred"* in BC.

### 2.1 Definitions element

```xml
<bpmn2:definitions xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xmlns:bpmn2="http://www.omg.org/spec/BPMN/20100524/MODEL"
                   xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
                   xmlns:dc="http://www.omg.org/spec/DD/20100524/DC"
                   xmlns:di="http://www.omg.org/spec/DD/20100524/DI"
                   xmlns:drools="http://www.jboss.org/drools"
                   id="Definition"
                   targetNamespace="http://www.jboss.org/drools">
```

- **Always** declare the `drools:` prefix. Never use `tns:` or any other alias.
- `targetNamespace` must be `http://www.jboss.org/drools` (not `http://www.omg.org/bpmn20` — Stunner filters on this).

### 2.2 Process element

```xml
<bpmn2:process id="my-process"
               drools:packageName="com.ibm.bamoe.demo"
               drools:version="1.0.0"
               drools:adHoc="false"
               name="My Process"
               isExecutable="true"
               processType="Public">
```

Required attributes on every process:

| Attribute | Required | Notes |
|---|---|---|
| `id` | yes | NCName. Avoid hyphens if you can — some jBPM code paths use this for generated class names. Prefer `camelCase` or `snake_case`. |
| `name` | yes | Human label |
| `isExecutable` | yes | Must be `true` for deployable processes |
| `drools:packageName` | **yes** | Java package. Use the same value for every process in the project. |
| `drools:version` | **yes** | Semver string like `1.0.0` |
| `drools:adHoc` | **yes** | `false` unless you actually want ad-hoc |
| `processType` | recommended | `Public` |

### 2.3 Item definitions — use FQCNs

```xml
<bpmn2:itemDefinition id="_stringItem"  structureRef="java.lang.String"/>
<bpmn2:itemDefinition id="_integerItem" structureRef="java.lang.Integer"/>
<bpmn2:itemDefinition id="_booleanItem" structureRef="java.lang.Boolean"/>
<bpmn2:itemDefinition id="_longItem"    structureRef="java.lang.Long"/>
<bpmn2:itemDefinition id="_customerItem" structureRef="com.ibm.bamoe.demo.model.Customer"/>
```

- **Never** use bare names (`String`, `Integer`). Stunner cannot resolve them.
- For custom POJOs, use the full classpath; make sure the class actually exists in the kjar (under `src/main/java/`).
- Convention: one item definition per property (`_fooItem`), not one shared across many. Shared definitions work but confuse the designer's variable panel.

### 2.4 Properties (process variables)

```xml
<bpmn2:property id="customerId" itemSubjectRef="_stringItem" name="customerId"/>
```

- **Always include `name`** — the designer displays and references variables by `name`.
- `id` and `name` should match for simplicity.
- Use one itemDefinition per variable in larger processes.

### 2.5 Script tasks

```xml
<bpmn2:scriptTask id="ScriptTask_1" name="Validate" scriptFormat="http://www.java.com/java">
  <bpmn2:incoming>SequenceFlow_1</bpmn2:incoming>
  <bpmn2:outgoing>SequenceFlow_2</bpmn2:outgoing>
  <bpmn2:script>boolean valid = customerId != null &amp;&amp; customerId.length() > 0;
kcontext.setVariable("statusCode", valid ? "VALID" : "INVALID");</bpmn2:script>
</bpmn2:scriptTask>
```

- `scriptFormat` must be a URI. Accepted values:
  - `http://www.java.com/java` (recommended)
  - `http://www.mvel.org/2.0`
  - `http://www.javascript.com/javascript` (supported at runtime but the editor form is limited)
- **XML-escape script bodies**: `&&` → `&amp;&amp;`, `<` → `&lt;`, `&` → `&amp;`. `>` is legal as-is but escaping it is also fine.
- Alternative: wrap the script in `<![CDATA[ ... ]]>` to avoid escaping entirely. BC reads CDATA correctly, though it re-emits as entity-escaped on save.
- Process variables are injected as local Java variables of the FQCN type you declared. `customerId` inside the script is a `java.lang.String`.
- Use `kcontext.setVariable("name", value)` to write back.
- Avoid mixing JavaScript and Java idioms. Pick Java and stick with it.

### 2.6 User tasks

```xml
<bpmn2:userTask id="UserTask_1" name="Manager Approval" drools:taskName="Manager Approval">
  <bpmn2:ioSpecification>
    <bpmn2:dataInput id="DataInput_1" name="TaskName" drools:dtype="Object"/>
    <bpmn2:dataInput id="DataInput_2" name="GroupId"  drools:dtype="Object"/>
    <bpmn2:dataInput id="DataInput_3" name="Priority" drools:dtype="Object"/>
    <bpmn2:dataOutput id="DataOutput_1" name="approved" drools:dtype="Object"/>
    <bpmn2:inputSet>
      <bpmn2:dataInputRefs>DataInput_1</bpmn2:dataInputRefs>
      <bpmn2:dataInputRefs>DataInput_2</bpmn2:dataInputRefs>
      <bpmn2:dataInputRefs>DataInput_3</bpmn2:dataInputRefs>
    </bpmn2:inputSet>
    <bpmn2:outputSet>
      <bpmn2:dataOutputRefs>DataOutput_1</bpmn2:dataOutputRefs>
    </bpmn2:outputSet>
  </bpmn2:ioSpecification>
  <!-- assignments ... -->
  <bpmn2:potentialOwner>
    <bpmn2:resourceAssignmentExpression>
      <bpmn2:formalExpression>managers</bpmn2:formalExpression>
    </bpmn2:resourceAssignmentExpression>
  </bpmn2:potentialOwner>
</bpmn2:userTask>
```

- `drools:taskName` on `<bpmn2:userTask>` — Stunner displays this in the property panel. Must match the assigned value of the `TaskName` dataInput.
- `drools:dtype="Object"` on each dataInput/dataOutput.
- Include `<bpmn2:potentialOwner>` for any task that a human actually performs. Without it the task runs but no one can claim it.
- The "well-known" dataInput names jBPM recognizes: `TaskName`, `GroupId`, `ActorId`, `Priority`, `Description`, `Skippable`, `Content`, `CreatedBy`, `Comment`, `Locale`.

### 2.7 Sequence flow conditions

```xml
<bpmn2:sequenceFlow id="SequenceFlow_3" sourceRef="Gateway_1" targetRef="UserTask_1">
  <bpmn2:conditionExpression xsi:type="bpmn2:tFormalExpression"
                             language="http://www.java.com/java">return requiresApproval;</bpmn2:conditionExpression>
</bpmn2:sequenceFlow>
```

- Always specify `language` on `<conditionExpression>`.
- Use `return someBoolean;` — don't write `return x == true;`. Simpler and avoids boxing edge cases.
- For `Boolean` wrapper types (the default when you use `java.lang.Boolean`), use `return Boolean.TRUE.equals(x);` to avoid `NullPointerException` when the variable is unset.

### 2.8 BPMNDI (diagram interchange)

- **Required** for the editor to open. A BPMN file with no `<bpmndi:BPMNDiagram>` section will fail to render even if everything else is perfect.
- Every flow-node element (`startEvent`, `task`, `gateway`, `endEvent`, `subProcess`, etc.) needs a matching `<bpmndi:BPMNShape>` with `<dc:Bounds>`.
- Every `<sequenceFlow>` needs a matching `<bpmndi:BPMNEdge>` with at least two `<di:waypoint>` entries.
- Coordinates don't need to be pretty — BC will let you re-layout on open. Just make sure every element is accounted for.

### 2.9 XML hygiene

- Element IDs must be valid NCName (start with letter or `_`; no spaces; hyphens, digits, dots OK after the first char).
- **Never emit raw `<` or `&` inside script bodies** — the file will parse inconsistently.
- UTF-8 encoding. Always declare `<?xml version="1.0" encoding="UTF-8"?>`.
- Don't leave trailing comments outside `<bpmn2:definitions>` — some parsers handle it, some don't. Keep all content inside the root element.

---

## 3. Scaling to more complex BPMN

The five fixes above are the minimum for a process to *open*. These additional rules apply once processes get bigger.

### 3.1 One process per file

- One `<bpmn2:process>` per `.bpmn` file. BC Stunner supports multiple in a single file in theory but the designer only shows the first one, and deployment behavior is inconsistent.
- Filename should match the process `id`: `wire-transfer.bpmn` contains `<process id="wire-transfer">`.

### 3.2 Packages and folder layout

- Set `drools:packageName` to match the folder the BPMN sits in under `src/main/resources/`. Example: a process with `drools:packageName="com.ibm.bamoe.demo.banking"` should live at `src/main/resources/com/ibm/bamoe/demo/banking/wire-transfer.bpmn`.
- For small demos, you can keep everything at the root of `src/main/resources/` and use a single flat package. Just be consistent — mixing layouts confuses BC's project navigator.

### 3.3 Subprocesses and call activities

- Reusable subprocess (`<bpmn2:callActivity>`): the `calledElement` attribute references the child process by its `id`, not by file path. The child process must be in the same kjar.
- Embedded subprocess (`<bpmn2:subProcess>`): needs its own `<bpmndi:BPMNShape>` + nested shapes for every child flow-node. Missing child shapes is the #1 cause of "Unexpected events" with subprocesses.
- Event subprocesses require `triggeredByEvent="true"` and their own start event.

### 3.4 Boundary events

- Boundary events attached to a task need `attachedToRef="TaskId"` on the `<boundaryEvent>`.
- Each boundary event needs its own `<bpmndi:BPMNShape>` positioned on the edge of the task's bounds.
- Interrupting vs non-interrupting: set `cancelActivity="true"` / `"false"`. Default is `true`.

### 3.5 Multi-instance

- `<bpmn2:multiInstanceLoopCharacteristics>` needs both a collection (`loopDataInputRef`) and an item variable (`inputDataItem`).
- Reference the collection via a property first: declare `<bpmn2:property id="items" itemSubjectRef="_listItem" name="items"/>` and then a dataInput bound to it.

### 3.6 Data objects vs properties

- Use `<bpmn2:property>` (inside the process) for process variables. Scope: entire process.
- Use `<bpmn2:dataObject>` only when you need BPMN-visible data flow between tasks. Most jBPM processes don't need them.

### 3.7 Error handling

- Define errors at the definitions level: `<bpmn2:error id="Error_1" errorCode="VALIDATION_FAILED" name="ValidationError"/>`.
- Reference from error end events and boundary error events via `errorRef`.
- Script tasks can throw exceptions — wrap in try/catch and use `kcontext.getKieRuntime().signalEvent(...)` to signal boundary handlers.

### 3.8 Variable count and complexity

- Keep process variables under ~30 per process. Beyond that, the BC variables panel gets unwieldy and open times slow noticeably.
- Prefer a single POJO wrapper (`BankingContext`) over 30 scalar variables for complex processes.
- When using POJOs, the class must be on the kjar classpath (`src/main/java/...`) and referenced by FQCN in `structureRef`.

### 3.9 LLM-generation guidance

When asking an LLM to generate BPMN:

1. **Give it this guide.** Paste sections 2 and 3 into the prompt. Don't assume the model knows BAMOE conventions — public BPMN examples on the web are mostly Camunda-flavored, which uses different extension attributes (`camunda:` instead of `drools:`).
2. **Provide a known-good template.** One of our working files (e.g. [account-opening.bpmn](src/main/resources/account-opening.bpmn)) makes an excellent shot-prompt: "generate a new process in this exact style."
3. **Ask for Java scripts, not JavaScript.** Models default to JS when they see `<script>` tags.
4. **Validate before committing.** Run `xmllint --noout file.bpmn` locally to catch malformed XML (raw `<`, unclosed tags, etc.). Then open in BC to confirm Stunner accepts it. Never push straight from LLM output to the repo.
5. **Start simple.** Have the model generate the skeleton (start → task → end with all extension attributes correct), then iterate to add complexity. One round-trip adding 10 features is much harder to debug than ten round-trips adding one each.

---

## 4. Repository structure for BC Git import

BC's "Import Project" feature clones a remote repo and scans for KIE projects (`pom.xml` with `<packaging>kjar</packaging>`). The repo layout matters.

### 4.1 Single-project repo (recommended for demos)

```
repo-root/
├── pom.xml                              ← kjar packaging at root
├── README.md
├── src/
│   └── main/
│       ├── java/                        ← POJOs, work item handlers
│       │   └── com/ibm/bamoe/demo/
│       │       └── model/Customer.java
│       └── resources/
│           ├── META-INF/
│           │   └── kmodule.xml          ← required, even if empty
│           ├── account-opening.bpmn
│           ├── wire-transfer.bpmn
│           └── credit-card-application.bpmn
```

Rules:

- `pom.xml` at **repo root**. BC's clone endpoint treats the repo as one project and will not recurse into subfolders for a second `pom.xml`.
- `src/main/resources/META-INF/kmodule.xml` is mandatory. Even an empty `<kmodule/>` satisfies BC.
- BPMN files directly under `src/main/resources/` land in the default package. Move them into `src/main/resources/com/ibm/bamoe/demo/` once you start using custom POJOs.

### 4.2 Minimum viable `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.ibm.bamoe.demo</groupId>
  <artifactId>banking-processes</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>kjar</packaging>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <version.org.kie>8.0.0.Final</version.org.kie>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.kie</groupId>
      <artifactId>kie-api</artifactId>
      <version>${version.org.kie}</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.kie</groupId>
        <artifactId>kie-maven-plugin</artifactId>
        <version>${version.org.kie}</version>
        <extensions>true</extensions>
      </plugin>
    </plugins>
  </build>
</project>
```

- **`<packaging>kjar</packaging>` is the signal** BC's importer looks for. Plain `jar` projects are silently skipped and surface as *"There are no projects available to import."*
- `kie-maven-plugin` with `<extensions>true</extensions>` is mandatory. Without it, BC clones fine but the build step fails.

### 4.3 Branch requirements — `master`, not `main`

BAMOE v8 Business Central's importer checks out the branch named `master`. GitHub creates new repos with `main` as the default. If your repo has only `main`, the import fails with *"There are no projects available to import. Check the URL, the credentials, and if there's a master branch."*

Fix on the remote:

```bash
git branch -m main master
git push -u origin master
git push origin --delete main
```

Then on GitHub: **Settings → Branches → Default branch → `master`**.

This is not documented prominently anywhere and is the single biggest time-waster with BC Git import.

### 4.4 Multi-project repos (advanced)

If you need multiple kjar projects in one GitHub repo (e.g. shared library + two process projects), BC's import won't help you — it expects one project per clone target. Two options:

1. **Separate repos.** One GitHub repo per BC project. Cleanest.
2. **Single repo, multiple clones.** Point BC's import at the same URL multiple times, once per subfolder — but this requires the `pom.xml` to be at the URL root, which means you can't have a monorepo via a single clone. The only workaround is Git submodules or scripted per-project clones, neither of which are pleasant.

Stick with one repo per BC project for BAMOE v8.

---

## 5. Git workflow: local ↔ GitHub ↔ Business Central

BC maintains its **own internal Git filesystem** (NIO2 Git) for every project. When you import from GitHub, BC clones the remote into this internal store. After that point BC's internal copy and GitHub are two independent repositories — they do not auto-sync.

Understanding this is critical. There are three ways to get new commits into BC after the initial import, and they all have trade-offs.

### 5.1 The three sync patterns

**Pattern A — Delete & re-import** (what we did during debugging)

1. Make changes locally → commit → push to GitHub
2. In BC: delete the project from the `banking-demo` space
3. Import again from the same GitHub URL

Pros: dead simple, always works. Cons: destroys any changes made in BC since the last import; slow for large projects.

**Pattern B — BC as downstream, GitHub as source of truth**

Use GitHub's Web UI or your local clone as the only place edits happen. Use Pattern A to refresh BC whenever you need the latest in the designer.

This is the right pattern for a **migration demo** or any scenario where BPMN is versioned in Git and BC is just a viewer/runner.

**Pattern C — BC as source of truth, push to GitHub**

BC exposes its internal Git over HTTP on the same port as the UI:

```
http://bamoeAdmin@localhost:8180/git/banking-demo/banking-processes
```

From your local clone, add BC as a remote:

```bash
git remote add bc http://bamoeAdmin@localhost:8180/git/banking-demo/banking-processes
git fetch bc
git pull bc master     # pull BC's changes into your local
git push bc master     # push your local changes into BC
```

BC will prompt for the password (`bamoeAdmin1!`) on each request, or you can embed it in the URL:

```bash
git remote set-url bc http://bamoeAdmin:bamoeAdmin1!@localhost:8180/git/banking-demo/banking-processes
```

Pros: bidirectional, real sync. Cons: you now have **three remotes** (local, GitHub, BC) to keep in step, and BC's internal Git is not backed up outside the BAMOE server.

### 5.2 Recommended workflow for this demo

For the banking migration demo, use **Pattern B** with one exception: after making BC-specific edits in the designer (laying out the diagram, adjusting properties via the UI), push them back to GitHub via Pattern C's `git pull bc master` locally, then `git push origin master` to GitHub.

```
┌──────────────┐      edits         ┌─────────────────┐
│    Local     │ ─────────────────> │     GitHub      │
│  (IDE/CLI)   │ <───────────────── │   (warzho/...)  │
└──────────────┘      git pull      └─────────────────┘
       │                                     │
       │ git pull bc master                  │ Import Project
       │ (occasional, after BC edits)        │ (initial + refreshes)
       v                                     v
┌────────────────────────────────────────────────────┐
│        Business Central internal Git (NIO2)       │
│    http://localhost:8180/git/banking-demo/...      │
└────────────────────────────────────────────────────┘
```

Rules of thumb:

1. **Make structural edits locally**, commit and push to GitHub. Refresh BC via delete-and-reimport.
2. **Make cosmetic edits in BC** (diagram layout, form customization, property tweaks) directly in the designer. When done, `git pull bc master` into your local, then push to GitHub.
3. **Never edit the same file in both places concurrently.** BC's merge behavior is unforgiving.
4. **Commit early, commit often locally.** Before any `Pattern A` delete-and-reimport, make sure everything in BC is pushed back to origin (or you accept losing it).

### 5.3 After the initial import — the "refresh cycle"

This is what we just did successfully:

```bash
# 1. Edit BPMN files locally
cd <local clone of v8_BC>
# ... make changes ...

# 2. Commit and push
git add src/main/resources/*.bpmn
git commit -m "Fix BPMN for BC Stunner compatibility"
git push origin master

# 3. In BC UI:
#    banking-demo space → banking-processes project
#    → Delete project
#    → Import Project → https://github.com/warzho/v8_BC.git
#    → Open any .bpmn — should render in Stunner
```

### 5.4 Verifying before you push

Before pushing BPMN changes to GitHub, run these locally:

```bash
# Syntax check
xmllint --noout src/main/resources/*.bpmn

# Maven build (catches kjar compilation errors)
mvn clean package

# Optional: show what's going up
git diff --stat origin/master
```

If `xmllint` complains, fix the XML before anything else. If `mvn package` fails with a kie-maven-plugin error, the BPMN is malformed in a way Stunner won't like either.

### 5.5 Common Git gotchas specific to BC

- **Line endings.** On Windows, `core.autocrlf=true` rewrites `\n` to `\r\n` on checkout. BC tolerates both, but `git diff` will show false positives after a BC round-trip. Set `core.autocrlf=input` in your local clone for a cleaner experience.
- **File mode changes.** BC's internal Git always stores mode `100644`. If your local has `100755` on any file, every push will show spurious mode-change diffs. Fix with `git config core.fileMode false`.
- **Credentials in URL.** Embedding `bamoeAdmin1!` in the remote URL leaks it into `~/.gitconfig` and shell history. For a local demo on your own machine it's fine. For anything shared, use `git credential-manager` or a credential helper instead.
- **BC's Git is HTTP, not HTTPS.** That's why the URL starts with `http://`. Don't let a browser or tool "upgrade" it.

---

## 6. TL;DR checklist for every new BPMN file

Before committing a new BPMN file to a repo destined for BC import:

- [ ] `xmlns:drools="http://www.jboss.org/drools"` declared (not `tns:`)
- [ ] `<bpmn2:process>` has `drools:packageName`, `drools:version`, `drools:adHoc`
- [ ] All `<bpmn2:itemDefinition>` use FQCNs (`java.lang.String`, not `String`)
- [ ] Every `<bpmn2:property>` has a `name` attribute
- [ ] Script tasks use `scriptFormat="http://www.java.com/java"` with Java bodies
- [ ] Script bodies XML-escape `&&` → `&amp;&amp;` and `<` → `&lt;`
- [ ] User tasks have `drools:taskName` and `<bpmn2:potentialOwner>`
- [ ] DataInputs/DataOutputs have `drools:dtype="Object"`
- [ ] Sequence flow conditions specify `language="http://www.java.com/java"`
- [ ] Complete `<bpmndi:BPMNDiagram>` with shape/edge for every flow-node and flow
- [ ] `xmllint --noout` passes
- [ ] `mvn package` succeeds
- [ ] Repo root has `pom.xml` with `<packaging>kjar</packaging>`
- [ ] `src/main/resources/META-INF/kmodule.xml` exists
- [ ] Default branch is `master` (not `main`)

If all boxes are checked, BC will open the file. If any one is missing, expect *"Unexpected events occurred... ID: -<random>"* when you click to view it.
