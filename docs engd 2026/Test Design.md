# CHORAS — Test Design Document
## cloud_executor.py · local_executor.py
### CHORAS Scalability Project - EngD 2026

**Version:** 2.0
**Status:** Draft
**Date:** 2026-03-13
**Related Documents:** `Testing Strategy.md`, `Test Results.md`
**Maintained by:** Test Manager

---

## 1. Purpose

This document defines how each test case in `test_cloud_executor.py` and
`test_local_executor.py` is designed. The test files are treated as the
**specification** — they define the intended API and behaviour of
`CloudExecutor` and `LocalExecutor`. Where the current implementation
differs from what the tests expect, the implementation must be updated.

For every test this document specifies:

- **What is being tested** — the component and behaviour under scrutiny
- **Inputs** — the exact data, configuration, and system state passed in
- **Process** — the steps taken to execute the test
- **Expected Output** — the observable result that determines pass or fail
- **Equivalence Partition(s) covered** — tracing each test back to the
  partitions defined in the Testing Strategy

---

## 2. Required Implementation Changes

The following changes to the production code are required before these
tests can pass. They are derived directly from the test files and
represent the intended API of each component.

### 2.1 `CloudExecutor` — Required Changes

| Change | Current | Required |
|---|---|---|
| SSH connection management | `_ssh_session()` context manager, no persistent client | Add `ssh_client = None` attribute · Add `_connect()` and `_disconnect()` methods |
| Upload method visibility | `_upload_file_via_sftp()` (private) | Rename to `upload_file_via_sftp()` (public) |
| Build method visibility | `_build_singularity_image()` (private) | Rename to `build_singularity_image()` (public) |
| Execute Singularity method | `_execute_singularity_image()` (private) | Rename to `execute_singularity_image()` (public) |
| Poll method visibility | `_poll_until_complete()` (private) | Rename to `poll_until_complete()` (public) |
| `execute()` return value | Returns `_CompletedJob()` only | Return `(job_id, _CompletedJob())` tuple |
| `get_filenames()` missing keys | Raises `KeyError` when `msh_path` or `geo_path` absent | Handle gracefully — return `None` for absent keys |
| `get_remote_file_path()` signature | `(remote_work_dir, image_name, task_id, filename)` — 4 args | `(image_name, task_id, filename)` — 3 args, construct path internally |

### 2.2 `LocalExecutor` — Required Changes

| Change | Current | Required |
|---|---|---|
| `execute()` return value | Returns `container` only | Return `(job_id, container)` tuple |
| Job tracking | No `_jobs` attribute | Add `_jobs: dict` — stores containers keyed by `job_id` |
| Container naming | Derived via `_get_container_name(method_config)` | Use `method_config["container_name"]` directly |
| Job ID generation | None | Generate UUID4 per `execute()` call and return as first element of tuple |

---

## 3. Test Case Naming Convention

| Prefix | Type |
|---|---|
| `U` | Unit test — pure logic, no external dependencies |
| `I` | Integration test — happy path |
| `B` | Bad day — failure and edge case scenarios |

---

## 4. `test_cloud_executor.py` — Test Design

### 4.1 Helper Function Tests

---

#### U1 — `get_filenames`: extracts msh and geo filenames

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → get_filenames()` |
| **Partitions** | EP-C1 |
| **What is being tested** | Full absolute paths for `msh_path` and `geo_path` are stripped to filenames only and returned as a tuple |
| **Preconditions** | A valid JSON file exists at `json_path` with full paths for both keys |
| **Input** | `json_path` pointing to `{"msh_path": "/full/path/to/mesh.msh", "geo_path": "/full/path/to/geo.geo"}` |
| **Process** | 1. Write JSON to temp file · 2. Call `get_filenames(str(json_path))` · 3. Capture returned tuple · 4. Assert filenames |
| **Expected Output** | Returns `("mesh.msh", "geo.geo")` |
| **Pass Criteria** | `msh == "mesh.msh"` and `geo == "geo.geo"` |

---

#### U2 — `get_filenames`: updates JSON in place

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → get_filenames()` |
| **Partitions** | EP-C1 |
| **What is being tested** | The JSON file itself is overwritten with filename-only values after the call, so downstream components reading the file also get the stripped paths |
| **Preconditions** | JSON file exists with full paths |
| **Input** | `{"msh_path": "/full/path/mesh.msh", "geo_path": "/full/path/geo.geo"}` |
| **Process** | 1. Write JSON to temp file · 2. Call `get_filenames(str(json_path))` · 3. Re-read the JSON file · 4. Assert updated values |
| **Expected Output** | Re-read JSON: `data["msh_path"] == "mesh.msh"` and `data["geo_path"] == "geo.geo"` |
| **Pass Criteria** | Both values in the re-read JSON are filenames without path separators |

---

#### U3 — `get_filenames`: handles missing `msh_path` gracefully

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → get_filenames()` |
| **Partitions** | EP-C5 |
| **What is being tested** | When `msh_path` is absent from the JSON, the function does not raise — `msh_path` is treated as optional |
| **Preconditions** | JSON file exists with only `geo_path` |
| **Input** | `{"geo_path": "/path/geo.geo"}` |
| **Process** | 1. Write JSON · 2. Call `get_filenames(str(json_path))` · 3. Assert `geo` return value |
| **Expected Output** | No exception raised · `geo == "geo.geo"` |
| **Pass Criteria** | No `KeyError` · `geo == "geo.geo"` |
| **Implementation Note** | Requires `get_filenames()` to handle absent `msh_path` — return `None` for that value |

---

#### U4 — `get_filenames`: handles missing `geo_path` gracefully

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → get_filenames()` |
| **Partitions** | EP-C5 |
| **What is being tested** | When `geo_path` is absent from the JSON, the function does not raise |
| **Preconditions** | JSON file exists with only `msh_path` |
| **Input** | `{"msh_path": "/path/mesh.msh"}` |
| **Process** | 1. Write JSON · 2. Call `get_filenames(str(json_path))` · 3. Assert `msh` return value |
| **Expected Output** | No exception raised · `msh == "mesh.msh"` |
| **Pass Criteria** | No `KeyError` · `msh == "mesh.msh"` |
| **Implementation Note** | Requires `get_filenames()` to handle absent `geo_path` — return `None` for that value |

---

#### U5 — `get_local_file_path`: joins dirname and filename

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → get_local_file_path()` |
| **Partitions** | EP-C1 |
| **What is being tested** | The function constructs a local path by joining the directory of the JSON path with a given filename |
| **Preconditions** | None |
| **Input** | `json_path = "/app/uploads/input.json"` · `filename = "mesh.msh"` |
| **Process** | 1. Call `get_local_file_path("/app/uploads/input.json", "mesh.msh")` · 2. Assert result |
| **Expected Output** | Returns `"/app/uploads/mesh.msh"` |
| **Pass Criteria** | Result `== "/app/uploads/mesh.msh"` |

---

#### U6 — `get_local_file_path`: works with nested directory

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → get_local_file_path()` |
| **Partitions** | EP-C1 |
| **What is being tested** | Correctly handles a deeply nested directory path |
| **Preconditions** | None |
| **Input** | `json_path = "/a/b/c/file.json"` · `filename = "out.csv"` |
| **Process** | 1. Call `get_local_file_path("/a/b/c/file.json", "out.csv")` · 2. Assert result |
| **Expected Output** | Returns `"/a/b/c/out.csv"` |
| **Pass Criteria** | Result `== "/a/b/c/out.csv"` |

---

#### U7 — `get_remote_file_path`: constructs correct remote path

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → get_remote_file_path()` |
| **Partitions** | EP-C1 |
| **What is being tested** | The function builds the correct remote sandbox path for a given file |
| **Preconditions** | None |
| **Input** | `image_name = "dg_image"` · `task_id = "abc-123"` · `filename = "input.json"` |
| **Process** | 1. Call `get_remote_file_path("dg_image", "abc-123", "input.json")` · 2. Assert result |
| **Expected Output** | Returns `"dg_image_sif_abc-123/app/input.json"` |
| **Pass Criteria** | Result `== "dg_image_sif_abc-123/app/input.json"` |
| **Implementation Note** | Requires signature change from 4 args to 3 — remove `remote_work_dir` parameter |

---

### 4.2 `_CompletedJob` Tests

---

#### U8 — `_CompletedJob.wait()`: returns zero status code

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _CompletedJob.wait()` |
| **Partitions** | EP-O1 |
| **What is being tested** | `wait()` returns `{"StatusCode": 0}` so `simulation_service.py` can call it without error after a cloud job completes |
| **Preconditions** | None |
| **Input** | `_CompletedJob()` instance |
| **Process** | 1. Instantiate `_CompletedJob()` · 2. Call `.wait()` · 3. Assert return value |
| **Expected Output** | `{"StatusCode": 0}` |
| **Pass Criteria** | Return value `== {"StatusCode": 0}` |

---

#### U9 — `_CompletedJob.logs()`: returns bytes

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _CompletedJob.logs()` |
| **Partitions** | EP-O1 |
| **What is being tested** | `logs()` returns a `bytes` object so callers can call `.decode()` on it without error |
| **Preconditions** | None |
| **Input** | `_CompletedJob()` instance |
| **Process** | 1. Instantiate `_CompletedJob()` · 2. Call `.logs()` · 3. Assert type |
| **Expected Output** | Returns an instance of `bytes` |
| **Pass Criteria** | `isinstance(result, bytes)` is `True` |

---

### 4.3 `CloudExecutor.__init__` Tests

---

#### U10 — Stores all constructor parameters as attributes

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor.__init__()` |
| **Partitions** | EP-S1 |
| **What is being tested** | All constructor arguments are stored as instance attributes with correct values |
| **Preconditions** | None |
| **Input** | `CloudExecutor("host", "user", "pass", "/key", "entry.py", "/work")` |
| **Process** | 1. Instantiate with all params · 2. Assert each attribute value |
| **Expected Output** | `hostname == "host"` · `username == "user"` · `password == "pass"` · `key_path == "/key"` · `entry_file == "entry.py"` · `remote_work_dir == "/work"` |
| **Pass Criteria** | All six assertions pass |

---

#### U11 — `ssh_client` initialised to `None`

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor.__init__()` |
| **Partitions** | EP-S1 |
| **What is being tested** | `ssh_client` is `None` on construction — no connection is opened until `_connect()` is called |
| **Preconditions** | None |
| **Input** | `CloudExecutor("host", "user")` |
| **Process** | 1. Instantiate · 2. Assert `ssh_client` attribute |
| **Expected Output** | `executor.ssh_client is None` |
| **Pass Criteria** | Assertion passes |
| **Implementation Note** | Requires `ssh_client = None` to be added to `__init__` |

---

### 4.4 `_connect` / `_disconnect` Tests

---

#### I1 — `_connect`: uses password when provided

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._connect()` |
| **Partitions** | EP-S1 |
| **What is being tested** | When `password` is set, `paramiko.SSHClient.connect` is called with `password`, `hostname`, and `username` |
| **Preconditions** | `paramiko.SSHClient` mocked |
| **Input** | Executor with `hostname="remote.host.com"`, `username="user"`, `password="secret"` |
| **Process** | 1. Patch `paramiko.SSHClient` · 2. Call `executor._connect()` · 3. Inspect `connect()` kwargs |
| **Expected Output** | `connect` called with `password="secret"`, `hostname="remote.host.com"`, `username="user"` |
| **Pass Criteria** | All three kwargs assertions pass |
| **Implementation Note** | Requires `_connect()` method to be added to `CloudExecutor` |

---

#### I2 — `_connect`: uses key path when provided

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._connect()` |
| **Partitions** | EP-S1 |
| **What is being tested** | When `key_path` is set, `connect` is called with `key_filename` and agent/key lookup disabled |
| **Preconditions** | `paramiko.SSHClient` mocked |
| **Input** | Executor with `key_path="/home/user/.ssh/id_rsa"`, no password |
| **Process** | 1. Patch `paramiko.SSHClient` · 2. Call `executor._connect()` · 3. Inspect `connect()` kwargs |
| **Expected Output** | `key_filename == "/home/user/.ssh/id_rsa"` · `allow_agent is False` · `look_for_keys is False` |
| **Pass Criteria** | All three kwargs assertions pass |

---

#### I3 — `_connect`: closes stale connection before reconnecting

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._connect()` |
| **Partitions** | EP-S1 |
| **What is being tested** | If `ssh_client` is already set when `_connect()` is called, the old client is closed before a new one is created |
| **Preconditions** | `executor.ssh_client` is pre-set to a mock |
| **Input** | Existing mock SSH client on `executor.ssh_client` |
| **Process** | 1. Attach old mock client · 2. Patch `paramiko.SSHClient` · 3. Call `_connect()` · 4. Assert old client was closed |
| **Expected Output** | `old_client.close()` called once |
| **Pass Criteria** | `old_client.close.assert_called_once()` passes |

---

#### I4 — `_disconnect`: closes client and sets to `None`

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._disconnect()` |
| **Partitions** | EP-S1 |
| **What is being tested** | `_disconnect()` closes the SSH client and sets `ssh_client` to `None` |
| **Preconditions** | `executor.ssh_client` is a mock |
| **Input** | Mock SSH client attached to `executor.ssh_client` |
| **Process** | 1. Attach mock · 2. Call `executor._disconnect()` · 3. Assert close called · 4. Assert `ssh_client is None` |
| **Expected Output** | `mock_ssh.close()` called · `executor.ssh_client is None` |
| **Pass Criteria** | Both assertions pass |
| **Implementation Note** | Requires `_disconnect()` method to be added to `CloudExecutor` |

---

#### I5 — `_disconnect`: safe when `ssh_client` is already `None`

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._disconnect()` |
| **Partitions** | EP-S1 |
| **What is being tested** | Calling `_disconnect()` when no connection exists does not raise |
| **Preconditions** | `executor.ssh_client = None` |
| **Input** | None |
| **Process** | 1. Ensure `ssh_client is None` · 2. Call `_disconnect()` · 3. Assert no exception |
| **Expected Output** | No exception raised |
| **Pass Criteria** | Call completes without error |

---

### 4.5 SFTP Operation Tests

---

#### I6 — `upload_file_via_sftp`: opens SFTP and calls `put`

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → upload_file_via_sftp()` |
| **Partitions** | EP-S1 |
| **What is being tested** | A successful upload calls `sftp.put` with the correct local and remote paths, then closes the SFTP connection |
| **Preconditions** | `executor.ssh_client` is a mock · local file exists |
| **Input** | `local_path = tmp_path / "test.json"` · `remote_path = "remote/test.json"` |
| **Process** | 1. Create local temp file · 2. Attach mock SSH client · 3. Call `upload_file_via_sftp(local, remote)` · 4. Assert `sftp.put` called with correct args · 5. Assert `sftp.close` called |
| **Expected Output** | `sftp.put.assert_called_once_with(str(local_file), "remote/test.json")` · `sftp.close.assert_called_once()` |
| **Pass Criteria** | Both mock assertions pass |

---

#### B1 — `upload_file_via_sftp`: raises when not connected

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → upload_file_via_sftp()` |
| **Partitions** | EP-S2 |
| **What is being tested** | Calling upload before `_connect()` raises `RuntimeError("not connected")` |
| **Preconditions** | `executor.ssh_client = None` |
| **Input** | `local = "local.json"` · `remote = "remote.json"` |
| **Process** | 1. Ensure `ssh_client is None` · 2. Call `upload_file_via_sftp` · 3. Observe exception |
| **Expected Output** | `RuntimeError` raised matching `"not connected"` |
| **Pass Criteria** | `pytest.raises(RuntimeError, match="not connected")` passes |
| **Implementation Note** | Requires `upload_file_via_sftp()` to check `ssh_client` is not `None` before proceeding |

---

#### I7 — `_download_file_via_sftp`: calls `sftp.get` with correct paths

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _download_file_via_sftp()` |
| **Partitions** | EP-S1 |
| **What is being tested** | A successful download calls `sftp.get` with the remote and local paths and closes SFTP |
| **Preconditions** | `executor.ssh_client` is a mock |
| **Input** | `remote = "remote/file.json"` · `local = "/local/file.json"` |
| **Process** | 1. Attach mock SSH client · 2. Call `_download_file_via_sftp(remote, local)` · 3. Assert mock calls |
| **Expected Output** | `sftp.get.assert_called_once_with("remote/file.json", "/local/file.json")` · `sftp.close.assert_called_once()` |
| **Pass Criteria** | Both mock assertions pass |

---

#### B2 — `_download_file_via_sftp`: raises when not connected

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _download_file_via_sftp()` |
| **Partitions** | EP-S2 |
| **What is being tested** | Calling download before `_connect()` raises `RuntimeError("not connected")` |
| **Preconditions** | `executor.ssh_client = None` |
| **Input** | `remote = "remote.json"` · `local = "local.json"` |
| **Process** | 1. Ensure `ssh_client is None` · 2. Call `_download_file_via_sftp` · 3. Observe exception |
| **Expected Output** | `RuntimeError` raised matching `"not connected"` |
| **Pass Criteria** | `pytest.raises(RuntimeError, match="not connected")` passes |

---

#### I8 — `_list_remote_files`: returns full paths, excludes hidden files

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _list_remote_files()` |
| **Partitions** | EP-O1 |
| **What is being tested** | Returns full remote paths for all non-hidden files and excludes files starting with `.` |
| **Preconditions** | `executor.ssh_client` mocked · `sftp.listdir_attr` returns three entries including one hidden |
| **Input** | Directory entries: `output.json`, `data.csv`, `.hidden` |
| **Process** | 1. Mock `listdir_attr` · 2. Call `_list_remote_files("sandbox/app")` · 3. Assert contents |
| **Expected Output** | Result contains `"sandbox/app/output.json"` and `"sandbox/app/data.csv"` · does not contain `"sandbox/app/.hidden"` |
| **Pass Criteria** | Both presence and absence assertions pass |

---

#### B3 — `_list_remote_files`: raises when not connected

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _list_remote_files()` |
| **Partitions** | EP-S2 |
| **What is being tested** | Calling list before `_connect()` raises `RuntimeError("not connected")` |
| **Preconditions** | `executor.ssh_client = None` |
| **Input** | `remote_dir = "remote/dir"` |
| **Process** | 1. Ensure `ssh_client is None` · 2. Call `_list_remote_files` · 3. Observe exception |
| **Expected Output** | `RuntimeError` raised matching `"not connected"` |
| **Pass Criteria** | `pytest.raises(RuntimeError, match="not connected")` passes |

---

#### I9 — `_delete_remote_path`: runs `rm -rf` command

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _delete_remote_path()` |
| **Partitions** | EP-S1 |
| **What is being tested** | `_delete_remote_path` executes `rm -rf {path}` on the remote via `exec_command` |
| **Preconditions** | `executor.ssh_client` is a mock · `exec_command` returns exit status 0 |
| **Input** | `remote_path = "sandbox/path"` |
| **Process** | 1. Mock SSH client · 2. Call `_delete_remote_path("sandbox/path")` · 3. Assert `exec_command` call |
| **Expected Output** | `exec_command` called with `"rm -rf sandbox/path"` |
| **Pass Criteria** | `mock_ssh.exec_command.assert_called_once_with("rm -rf sandbox/path")` passes |

---

### 4.6 Singularity Image Tests

---

#### I10 — `build_singularity_image`: runs correct build command

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → build_singularity_image()` |
| **Partitions** | EP-S1 |
| **What is being tested** | The correct `singularity build` command is issued, referencing the sandbox name and Docker archive |
| **Preconditions** | `executor.ssh_client` mocked · `exec_command` returns exit 0 |
| **Input** | `sandbox_name = "my_sandbox"` · `image_tar_name = "my_image.tar"` |
| **Process** | 1. Attach mock SSH client · 2. Call `build_singularity_image("my_sandbox", "my_image.tar")` · 3. Inspect command string |
| **Expected Output** | Command contains `"singularity build --sandbox my_sandbox"` and `"docker-archive://my_image.tar"` |
| **Pass Criteria** | Both substrings present in `exec_command` call argument |

---

#### B4 — `build_singularity_image`: raises on SSH error

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → build_singularity_image()` |
| **Partitions** | EP-S5 |
| **What is being tested** | When `exec_command` raises (e.g. SSH broken), the exception propagates out |
| **Preconditions** | `executor.ssh_client` mocked · `exec_command` raises `Exception("SSH broken")` |
| **Input** | `sandbox_name = "sandbox"` · `image_tar_name = "image.tar"` |
| **Process** | 1. Set `exec_command` to raise · 2. Call `build_singularity_image(...)` · 3. Observe exception |
| **Expected Output** | `Exception` raised matching `"SSH broken"` |
| **Pass Criteria** | `pytest.raises(Exception, match="SSH broken")` passes |

---

#### I11 — `execute_singularity_image`: runs `nohup` command in background

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → execute_singularity_image()` |
| **Partitions** | EP-S1 |
| **What is being tested** | The Singularity execution command is launched with `nohup`, uses `singularity exec`, passes the input JSON, and ends with `&` to run in background |
| **Preconditions** | `executor.ssh_client` mocked · `exec_command` returns exit 0 |
| **Input** | `sandbox_name = "sandbox"` · `input_json = "input.json"` |
| **Process** | 1. Attach mock · 2. Call `execute_singularity_image("sandbox", "input.json")` · 3. Inspect command string |
| **Expected Output** | Command contains `"nohup"` · contains `"singularity exec"` · contains `"input.json"` · ends with `"&"` |
| **Pass Criteria** | All four string assertions pass |

---

#### B5 — `execute_singularity_image`: raises on error

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → execute_singularity_image()` |
| **Partitions** | EP-S5 |
| **What is being tested** | When `exec_command` raises, the exception propagates |
| **Preconditions** | `executor.ssh_client` mocked · `exec_command` raises `Exception("exec failed")` |
| **Input** | `sandbox_name = "sandbox"` · `input_json = "input.json"` |
| **Process** | 1. Set `exec_command` to raise · 2. Call `execute_singularity_image(...)` · 3. Observe exception |
| **Expected Output** | `Exception` raised matching `"exec failed"` |
| **Pass Criteria** | `pytest.raises(Exception, match="exec failed")` passes |

---

### 4.7 `_parse_overall_progress` Tests

---

#### U12 — Returns minimum percentage across multiple results

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._parse_overall_progress()` |
| **Partitions** | EP-P1 |
| **What is being tested** | Returns the minimum value across all result entries so overall progress only reaches 100 when all sources complete |
| **Input** | `{"results": [{"percentage": 80}, {"percentage": 60}, {"percentage": 100}]}` |
| **Process** | Call `_parse_overall_progress(data)` · Assert return value |
| **Expected Output** | `60` |
| **Pass Criteria** | Return value `== 60` |

---

#### U13 — Returns `None` when `results` key absent

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._parse_overall_progress()` |
| **Partitions** | EP-C5 |
| **What is being tested** | Missing `results` key returns `None` without raising |
| **Input** | `{}` |
| **Expected Output** | `None` |
| **Pass Criteria** | Return value is `None` |

---

#### U14 — Returns `None` when results is empty list

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._parse_overall_progress()` |
| **Partitions** | EP-C5 |
| **What is being tested** | Empty results list returns `None` |
| **Input** | `{"results": []}` |
| **Expected Output** | `None` |
| **Pass Criteria** | Return value is `None` |

---

#### U15 — Defaults missing `percentage` to `0`

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._parse_overall_progress()` |
| **Partitions** | EP-C5 |
| **What is being tested** | An entry missing `percentage` key defaults to `0` via `.get("percentage", 0)` and is included in the minimum calculation |
| **Input** | `{"results": [{"percentage": 50}, {}]}` |
| **Expected Output** | `0` (minimum of 50 and 0) |
| **Pass Criteria** | Return value `== 0` |

---

#### U16 — Returns `100` when all results complete

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._parse_overall_progress()` |
| **Partitions** | EP-P2 |
| **What is being tested** | Returns `100` only when every entry is at 100% |
| **Input** | `{"results": [{"percentage": 100}, {"percentage": 100}]}` |
| **Expected Output** | `100` |
| **Pass Criteria** | Return value `== 100` |

---

#### U17 — Returns `None` on malformed data

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor._parse_overall_progress()` |
| **Partitions** | EP-C5 |
| **What is being tested** | When `results` is a string instead of a list, returns `None` without crashing |
| **Input** | `{"results": "bad"}` |
| **Expected Output** | `None` |
| **Pass Criteria** | Return value is `None` · no exception raised |

---

### 4.8 `_collect_outputs_and_cleanup` Tests

---

#### I12 — Downloads only `.json` and `.csv`, returns `True`

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _collect_outputs_and_cleanup()` |
| **Partitions** | EP-O1 |
| **What is being tested** | From a mixed remote directory, only files with extensions in `_OUTPUT_EXTENSIONS` are downloaded. Returns `True` on success |
| **Preconditions** | `_list_remote_files` mocked · SFTP mocked · `_delete_remote_path` mocked |
| **Input** | Remote files: `output.json`, `data.csv`, `script.py`, `mesh.msh` |
| **Process** | 1. Mock `_list_remote_files` to return all four · 2. Call `_collect_outputs_and_cleanup(...)` · 3. Inspect `sftp.get` call arguments · 4. Assert return value |
| **Expected Output** | `sftp.get` called for `output.json` and `data.csv` only · returns `True` |
| **Pass Criteria** | `script.py` and `mesh.msh` absent from call args · `result is True` |

---

#### I13 — Deletes sandbox and tar after download

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _collect_outputs_and_cleanup()` |
| **Partitions** | EP-O1 |
| **What is being tested** | After outputs are downloaded, both the sandbox directory and tar file are deleted from the remote host |
| **Preconditions** | `_list_remote_files` returns empty list · `_delete_remote_path` mocked |
| **Input** | `remote_sandbox_path = "sandbox"` · `remote_tar_path = "image.tar"` |
| **Process** | 1. Mock dependencies · 2. Call `_collect_outputs_and_cleanup(...)` · 3. Assert delete calls |
| **Expected Output** | `_delete_remote_path` called with `"sandbox"` and with `"image.tar"` |
| **Pass Criteria** | Both `assert_any_call` assertions pass |

---

#### I14 — Skips tar deletion when `remote_tar_path` is `None`

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _collect_outputs_and_cleanup()` |
| **Partitions** | EP-O1 |
| **What is being tested** | When no tar path is provided, only the sandbox is deleted — no attempt to delete `None` |
| **Input** | `remote_tar_path = None` |
| **Process** | 1. Mock `_delete_remote_path` · 2. Call with `remote_tar_path=None` · 3. Inspect delete call arguments |
| **Expected Output** | `"image.tar"` never passed to `_delete_remote_path` |
| **Pass Criteria** | `"image.tar"` absent from all delete call arguments |

---

#### B6 — Returns `False` on error

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → _collect_outputs_and_cleanup()` |
| **Partitions** | EP-S4 |
| **What is being tested** | When an exception is raised (e.g. SFTP failure), the method catches it and returns `False` instead of propagating — sandbox may not be cleaned up |
| **Preconditions** | `_list_remote_files` mocked to raise `Exception("SFTP error")` |
| **Input** | `_list_remote_files` side effect: `Exception("SFTP error")` |
| **Process** | 1. Mock to raise · 2. Call `_collect_outputs_and_cleanup(...)` · 3. Assert return value |
| **Expected Output** | Returns `False` |
| **Pass Criteria** | `result is False` |

---

### 4.9 `poll_until_complete` Tests

---

#### I15 — Returns `True` when job completes on first poll

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → poll_until_complete()` |
| **Partitions** | EP-P2 |
| **What is being tested** | When `percentage == 100` on the first download, `_collect_outputs_and_cleanup` is called and `True` is returned |
| **Preconditions** | `_download_file_via_sftp` writes JSON with `percentage: 100` · `_collect_outputs_and_cleanup` mocked to return `True` |
| **Input** | Remote JSON: `{"results": [{"percentage": 100}]}` |
| **Process** | 1. Patch `time.sleep` · 2. Mock `_connect`, `_disconnect`, `_download_file_via_sftp`, `_collect_outputs_and_cleanup` · 3. Call `poll_until_complete(...)` · 4. Assert result and cleanup call |
| **Expected Output** | Returns `True` · `_collect_outputs_and_cleanup` called once |
| **Pass Criteria** | `result is True` · `mock_cleanup.assert_called_once()` |
| **Implementation Note** | Requires `poll_until_complete()` public method — rename from `_poll_until_complete()` |

---

#### I16 — Polls multiple cycles before completion

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → poll_until_complete()` |
| **Partitions** | EP-P1 |
| **What is being tested** | The polling loop iterates multiple cycles and calls `time.sleep` between them |
| **Preconditions** | Progress sequence: `30% → 60% → 100%` |
| **Input** | Cycles 1–2: increasing progress · Cycle 3+: `100%` |
| **Process** | 1. Patch `time.sleep` · 2. Define `fake_download` with increasing percentages · 3. Run `poll_until_complete` · 4. Assert sleep call count |
| **Expected Output** | `time.sleep` called at least twice |
| **Pass Criteria** | `mock_sleep.call_count >= 2` |

---

#### I17 — Retries polling on SSH connect failure

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → poll_until_complete()` |
| **Partitions** | EP-S2 |
| **What is being tested** | If `_connect()` raises on the first attempt, polling retries and succeeds on the next cycle |
| **Preconditions** | `_connect` raises once then succeeds · final poll returns 100% |
| **Input** | `_connect` side effects: `[Exception("SSH timeout"), None]` |
| **Process** | 1. Mock flaky `_connect` · 2. Run `poll_until_complete` · 3. Assert connect was called at least twice and result is `True` |
| **Expected Output** | `result is True` · `connect_calls >= 2` |
| **Pass Criteria** | Both assertions pass |

---

#### I18 — Applies backoff after fast phase cycles

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → poll_until_complete()` |
| **Partitions** | EP-P1 |
| **What is being tested** | After `POLL_FAST_PHASE_CYCLES` cycles, the sleep interval grows beyond `POLL_INTERVAL_MIN` |
| **Preconditions** | Progress stuck at `50%` for first `POLL_FAST_PHASE_CYCLES + 1` cycles, then `100%` |
| **Input** | Cycles 1–6: `percentage = 50` · Cycle 7+: `percentage = 100` |
| **Process** | 1. Patch `time.sleep` · 2. Run polling · 3. Inspect all sleep call values |
| **Expected Output** | At least one sleep value greater than `POLL_INTERVAL_MIN` |
| **Pass Criteria** | `any(s > POLL_INTERVAL_MIN for s in sleep_intervals)` |

---

#### I19 — Local JSON written only when progress changes

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → poll_until_complete()` |
| **Partitions** | EP-P1 |
| **What is being tested** | The local JSON file is only written to disk when the percentage changes — no unnecessary writes when progress is unchanged |
| **Preconditions** | Cycle 1: `50%` · Cycle 2: `50%` (no write) · Cycle 3: `100%` |
| **Process** | 1. Run `poll_until_complete` · 2. Assert local JSON exists after completion |
| **Expected Output** | Local JSON file exists |
| **Pass Criteria** | `local_json.exists() is True` |

---

### 4.10 `execute` Tests

---

#### I20 — Returns job ID and `_CompletedJob`

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor.execute()` |
| **Partitions** | EP-S1, EP-M2 |
| **What is being tested** | `execute()` completes the full workflow and returns a valid UUID job ID and a `_CompletedJob` instance |
| **Preconditions** | All SSH methods mocked · `get_filenames` mocked |
| **Input** | `method_config["container_image"] = "dg_image:latest"` · valid `sim_config` |
| **Process** | 1. Mock all internal methods · 2. Call `execute(method_config, sim_config)` · 3. Unpack `(job_id, completed)` · 4. Assert types |
| **Expected Output** | `job_id` is a 36-character UUID string · `completed` is a `_CompletedJob` instance |
| **Pass Criteria** | `len(job_id) == 36` · `isinstance(completed, _CompletedJob)` |
| **Implementation Note** | Requires `execute()` to return `(job_id, _CompletedJob())` tuple |

---

#### I21 — Uploads tar and JSON files

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor.execute()` |
| **Partitions** | EP-S1 |
| **What is being tested** | `execute()` uploads the Docker tar and input JSON to the remote host |
| **Input** | `method_config["container_image"] = "dg_image:latest"` |
| **Process** | 1. Mock all SSH methods · 2. Call `execute()` · 3. Inspect `upload_file_via_sftp` call arguments |
| **Expected Output** | Upload calls include `"dg_image.tar"` and `"input.json"` |
| **Pass Criteria** | Both substrings present across upload call args |

---

#### I22 — Calls `poll_until_complete`

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor.execute()` |
| **Partitions** | EP-S1 |
| **What is being tested** | `execute()` calls `poll_until_complete` exactly once to wait for the job to finish |
| **Process** | 1. Mock all SSH methods · 2. Call `execute()` · 3. Assert `poll_until_complete` called |
| **Expected Output** | `poll_until_complete.assert_called_once()` |
| **Pass Criteria** | Assertion passes |

---

#### I23 — Image tag stripped from sandbox name

| Field | Detail |
|---|---|
| **Component** | `cloud_executor.py → CloudExecutor.execute()` |
| **Partitions** | EP-S1 |
| **What is being tested** | The `:latest` tag is removed from the image name when constructing the sandbox name — sandbox names cannot contain colons |
| **Input** | `method_config["container_image"] = "dg_image:latest"` |
| **Process** | 1. Mock all SSH methods · 2. Call `execute()` · 3. Inspect first argument to `build_singularity_image` |
| **Expected Output** | Sandbox name contains `"dg_image"` but not `":latest"` |
| **Pass Criteria** | `":latest" not in sandbox_arg` and `"dg_image" in sandbox_arg` |

---

## 5. `test_local_executor.py` — Test Design

### 5.1 `get_host_path_for_container_path` Tests

---

#### U18 — Resolves exact mount destination to host path

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → get_host_path_for_container_path()` |
| **Partitions** | EP-D1 |
| **What is being tested** | A container path that exactly matches a mount destination is resolved to the corresponding host source path |
| **Preconditions** | Docker client mocked · container mount: source `/host/uploads`, destination `/app/uploads` |
| **Input** | `container_path = "/app/uploads"` · `socket.gethostname` mocked to `"my-container-id"` |
| **Process** | 1. Mock `docker.from_env()` and `containers.get()` · 2. Patch `socket.gethostname` · 3. Call `get_host_path_for_container_path("/app/uploads")` · 4. Assert result |
| **Expected Output** | Returns `"/host/uploads"` |
| **Pass Criteria** | Result `== "/host/uploads"` |

---

#### U19 — Resolves subdirectory of mount

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → get_host_path_for_container_path()` |
| **Partitions** | EP-D1 |
| **What is being tested** | A path under a mount destination is resolved by computing the relative suffix and appending to host source |
| **Input** | `container_path = "/app/uploads/subdir"` · same mount as U18 |
| **Process** | Same as U18 with subdirectory path |
| **Expected Output** | Returns `"/host/uploads/subdir"` |
| **Pass Criteria** | Result `== "/host/uploads/subdir"` |

---

#### B7 — Raises `RuntimeError` when no mount covers path

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → get_host_path_for_container_path()` |
| **Partitions** | EP-D4 |
| **What is being tested** | When no mount covers the requested container path, `RuntimeError` is raised with `"No mount found covering container path"` |
| **Input** | `container_path = "/some/unmounted/path"` · mount only covers `/app/uploads` |
| **Process** | 1. Mock Docker with unrelated mount · 2. Call with unmounted path · 3. Observe exception |
| **Expected Output** | `RuntimeError` raised matching `"No mount found covering container path"` |
| **Pass Criteria** | `pytest.raises(RuntimeError, match="No mount found covering container path")` passes |

---

#### B8 — Raises when Docker client fails

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → get_host_path_for_container_path()` |
| **Partitions** | EP-D2 |
| **What is being tested** | If `containers.get()` raises (e.g. Docker daemon down), the exception propagates out |
| **Input** | `containers.get` side effect: `Exception("Docker socket error")` |
| **Process** | 1. Mock to raise · 2. Call function · 3. Observe exception |
| **Expected Output** | `Exception` raised matching `"Docker socket error"` |
| **Pass Criteria** | `pytest.raises(Exception, match="Docker socket error")` passes |

---

#### U20 — Uses hostname to identify container

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → get_host_path_for_container_path()` |
| **Partitions** | EP-D1 |
| **What is being tested** | `containers.get()` is called with the current machine hostname which identifies the container |
| **Input** | `socket.gethostname` mocked to return `"abc123"` |
| **Process** | 1. Mock Docker · 2. Patch hostname · 3. Call function · 4. Assert `containers.get` argument |
| **Expected Output** | `containers.get` called with `"abc123"` |
| **Pass Criteria** | `mock_docker_client.containers.get.assert_called_once_with("abc123")` |

---

#### U21 — Normalises backslashes to forward slashes

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → get_host_path_for_container_path()` |
| **Partitions** | EP-D1 |
| **What is being tested** | Windows-style backslashes in the host source path are normalised to forward slashes |
| **Input** | Mount source: `"C:\\Users\\host\\uploads"` · destination: `"/app/uploads"` |
| **Process** | 1. Mock Docker with Windows-style source path · 2. Call function · 3. Assert no backslashes |
| **Expected Output** | Result contains no `"\\"` characters |
| **Pass Criteria** | `"\\" not in result` |

---

### 5.2 `LocalExecutor.__init__` Tests

---

#### U22 — Uses `DOCKER_WORK_DIR` environment variable

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.__init__()` |
| **Partitions** | EP-D1 |
| **What is being tested** | When `work_dir` is not provided, `DOCKER_WORK_DIR` env var is used as the working directory |
| **Input** | `DOCKER_WORK_DIR = "/custom/workdir"` set in environment · `LocalExecutor()` with no args |
| **Process** | 1. Set env var · 2. Instantiate · 3. Assert `work_dir` |
| **Expected Output** | `executor.work_dir == "/custom/workdir"` |
| **Pass Criteria** | Assertion passes |

---

#### U23 — Falls back to `/app` when env var absent

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.__init__()` |
| **Partitions** | EP-D1 |
| **What is being tested** | When neither `work_dir` argument nor `DOCKER_WORK_DIR` env var is set, defaults to `/app` |
| **Input** | No `DOCKER_WORK_DIR` · `LocalExecutor()` with no args |
| **Process** | 1. Clear env var · 2. Instantiate · 3. Assert `work_dir` |
| **Expected Output** | `executor.work_dir == "/app"` |
| **Pass Criteria** | Assertion passes |

---

#### U24 — Uses explicit `work_dir` argument

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.__init__()` |
| **Partitions** | EP-D1 |
| **What is being tested** | An explicitly passed `work_dir` overrides the environment variable |
| **Input** | `LocalExecutor(work_dir="/my/dir")` |
| **Process** | 1. Instantiate with explicit arg · 2. Assert `work_dir` |
| **Expected Output** | `executor.work_dir == "/my/dir"` |
| **Pass Criteria** | Assertion passes |

---

#### U25 — `_jobs` dict initialised empty

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.__init__()` |
| **Partitions** | EP-D1 |
| **What is being tested** | The `_jobs` dict starts empty on construction — no stale state from previous runs |
| **Input** | `LocalExecutor()` |
| **Process** | 1. Instantiate · 2. Assert `_jobs` |
| **Expected Output** | `executor._jobs == {}` |
| **Pass Criteria** | Assertion passes |
| **Implementation Note** | Requires `_jobs = {}` to be added to `__init__` |

---

### 5.3 `LocalExecutor.execute` Tests

---

#### I24 — Returns job ID and container

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.execute()` |
| **Partitions** | EP-D1, EP-M1 |
| **What is being tested** | `execute()` returns a tuple of `(job_id, container)` where `job_id` is a UUID and `container` is the Docker container object |
| **Preconditions** | Docker mocked · `get_host_path_for_container_path` mocked |
| **Input** | Valid `method_config` and `sim_config` |
| **Process** | 1. Mock Docker and resolver · 2. Call `execute(method_config, sim_config)` · 3. Unpack `(job_id, container)` · 4. Assert types |
| **Expected Output** | `job_id` is a 36-char UUID · `container` is the mock container |
| **Pass Criteria** | `isinstance(job_id, str)` · `len(job_id) == 36` · `container is fake_container` |
| **Implementation Note** | Requires `execute()` to return `(uuid4_string, container)` |

---

#### I25 — Stores job in `_jobs` dict

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.execute()` |
| **Partitions** | EP-D1 |
| **What is being tested** | After `execute()` returns, the container is stored in `executor._jobs` keyed by the returned `job_id` |
| **Preconditions** | Docker and resolver mocked |
| **Input** | Valid `method_config` and `sim_config` |
| **Process** | 1. Mock dependencies · 2. Call `execute()` · 3. Assert `job_id in executor._jobs` · 4. Assert `executor._jobs[job_id] is fake_container` |
| **Expected Output** | Container stored under `job_id` in `_jobs` |
| **Pass Criteria** | Both key existence and value identity assertions pass |
| **Implementation Note** | Requires `_jobs[job_id] = container` inside `execute()` |

---

#### I26 — Passes correct image and env to `containers.run`

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.execute()` |
| **Partitions** | EP-D1 |
| **What is being tested** | `containers.run` receives the correct `image` from `method_config` and `environment` from `sim_config` |
| **Input** | `method_config["container_image"] = "my-sim-image:latest"` · `sim_config["env"] = {"JSON_PATH": "..."}` |
| **Process** | 1. Mock Docker · 2. Call `execute()` · 3. Inspect `containers.run` kwargs |
| **Expected Output** | `image == "my-sim-image:latest"` · `environment == {"JSON_PATH": "..."}` |
| **Pass Criteria** | Both kwargs assertions pass |

---

#### I27 — Volume mount uses resolved host path

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.execute()` |
| **Partitions** | EP-D1 |
| **What is being tested** | The volume mount uses the host path resolved by `get_host_path_for_container_path`, bound to the container path in read-write mode |
| **Preconditions** | `get_host_path_for_container_path` mocked to return `"/host/uploads"` |
| **Input** | `sim_config["env"]["JSON_PATH"] = "/app/uploads/input.json"` |
| **Process** | 1. Mock resolver · 2. Call `execute()` · 3. Inspect `volumes` kwarg |
| **Expected Output** | `volumes["/host/uploads"]["bind"] == "/app/uploads"` · `mode == "rw"` |
| **Pass Criteria** | Both volume structure assertions pass |

---

#### I28 — Container runs in detached mode

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.execute()` |
| **Partitions** | EP-D1 |
| **What is being tested** | `containers.run` is always called with `detach=True` so `execute()` returns immediately |
| **Process** | 1. Mock Docker · 2. Call `execute()` · 3. Assert `detach` kwarg |
| **Expected Output** | `detach == True` |
| **Pass Criteria** | `call_kwargs["detach"] is True` |

---

#### B9 — Raises on `containers.run` failure

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.execute()` |
| **Partitions** | EP-D2 |
| **What is being tested** | When `containers.run` raises (e.g. image not found, daemon down), the exception propagates out of `execute()` |
| **Input** | `containers.run` side effect: `Exception("Image not found")` |
| **Process** | 1. Mock Docker to raise · 2. Call `execute()` · 3. Observe exception |
| **Expected Output** | `Exception` raised matching `"Image not found"` |
| **Pass Criteria** | `pytest.raises(Exception, match="Image not found")` passes |

---

#### I29 — Each call produces a unique job ID

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.execute()` |
| **Partitions** | EP-D1 |
| **What is being tested** | Two successive calls to `execute()` with the same config produce different job IDs |
| **Input** | Two calls with identical `method_config` and `sim_config` |
| **Process** | 1. Mock Docker · 2. Call `execute()` twice · 3. Unpack both job IDs · 4. Assert they differ |
| **Expected Output** | `job_id_1 != job_id_2` |
| **Pass Criteria** | Inequality assertion passes |

---

#### I30 — Uses `container_name` from `method_config`

| Field | Detail |
|---|---|
| **Component** | `local_executor.py → LocalExecutor.execute()` |
| **Partitions** | EP-D1 |
| **What is being tested** | The `name` passed to `containers.run` comes from `method_config["container_name"]` directly |
| **Input** | `method_config["container_name"] = "sim_container"` |
| **Process** | 1. Mock Docker · 2. Call `execute()` · 3. Assert `name` kwarg |
| **Expected Output** | `containers.run` called with `name="sim_container"` |
| **Pass Criteria** | `call_kwargs["name"] == "sim_container"` |
| **Implementation Note** | Requires `execute()` to use `method_config["container_name"]` instead of deriving via `_get_container_name()` |

---

## 6. Traceability Matrix

| Test | Partition(s) | Component | Type |
|---|---|---|---|
| U1 | EP-C1 | `get_filenames` | Unit |
| U2 | EP-C1 | `get_filenames` | Unit |
| U3 | EP-C5 | `get_filenames` | Unit |
| U4 | EP-C5 | `get_filenames` | Unit |
| U5 | EP-C1 | `get_local_file_path` | Unit |
| U6 | EP-C1 | `get_local_file_path` | Unit |
| U7 | EP-C1 | `get_remote_file_path` | Unit |
| U8 | EP-O1 | `_CompletedJob.wait` | Unit |
| U9 | EP-O1 | `_CompletedJob.logs` | Unit |
| U10 | EP-S1 | `CloudExecutor.__init__` | Unit |
| U11 | EP-S1 | `CloudExecutor.__init__` | Unit |
| U12 | EP-P1 | `_parse_overall_progress` | Unit |
| U13 | EP-C5 | `_parse_overall_progress` | Unit |
| U14 | EP-C5 | `_parse_overall_progress` | Unit |
| U15 | EP-C5 | `_parse_overall_progress` | Unit |
| U16 | EP-P2 | `_parse_overall_progress` | Unit |
| U17 | EP-C5 | `_parse_overall_progress` | Unit |
| U18 | EP-D1 | `get_host_path_for_container_path` | Unit |
| U19 | EP-D1 | `get_host_path_for_container_path` | Unit |
| U20 | EP-D1 | `get_host_path_for_container_path` | Unit |
| U21 | EP-D1 | `get_host_path_for_container_path` | Unit |
| U22 | EP-D1 | `LocalExecutor.__init__` | Unit |
| U23 | EP-D1 | `LocalExecutor.__init__` | Unit |
| U24 | EP-D1 | `LocalExecutor.__init__` | Unit |
| U25 | EP-D1 | `LocalExecutor.__init__` | Unit |
| I1 | EP-S1 | `_connect` | Integration |
| I2 | EP-S1 | `_connect` | Integration |
| I3 | EP-S1 | `_connect` | Integration |
| I4 | EP-S1 | `_disconnect` | Integration |
| I5 | EP-S1 | `_disconnect` | Integration |
| I6 | EP-S1 | `upload_file_via_sftp` | Integration |
| I7 | EP-S1 | `_download_file_via_sftp` | Integration |
| I8 | EP-O1 | `_list_remote_files` | Integration |
| I9 | EP-S1 | `_delete_remote_path` | Integration |
| I10 | EP-S1 | `build_singularity_image` | Integration |
| I11 | EP-S1 | `execute_singularity_image` | Integration |
| I12 | EP-O1 | `_collect_outputs_and_cleanup` | Integration |
| I13 | EP-O1 | `_collect_outputs_and_cleanup` | Integration |
| I14 | EP-O1 | `_collect_outputs_and_cleanup` | Integration |
| I15 | EP-P2 | `poll_until_complete` | Integration |
| I16 | EP-P1 | `poll_until_complete` | Integration |
| I17 | EP-S2 | `poll_until_complete` | Integration |
| I18 | EP-P1 | `poll_until_complete` | Integration |
| I19 | EP-P1 | `poll_until_complete` | Integration |
| I20 | EP-S1, EP-M2 | `CloudExecutor.execute` | Integration |
| I21 | EP-S1 | `CloudExecutor.execute` | Integration |
| I22 | EP-S1 | `CloudExecutor.execute` | Integration |
| I23 | EP-S1 | `CloudExecutor.execute` | Integration |
| I24 | EP-D1, EP-M1 | `LocalExecutor.execute` | Integration |
| I25 | EP-D1 | `LocalExecutor.execute` | Integration |
| I26 | EP-D1 | `LocalExecutor.execute` | Integration |
| I27 | EP-D1 | `LocalExecutor.execute` | Integration |
| I28 | EP-D1 | `LocalExecutor.execute` | Integration |
| I29 | EP-D1 | `LocalExecutor.execute` | Integration |
| I30 | EP-D1 | `LocalExecutor.execute` | Integration |
| B1 | EP-S2 | `upload_file_via_sftp` | Bad Day |
| B2 | EP-S2 | `_download_file_via_sftp` | Bad Day |
| B3 | EP-S2 | `_list_remote_files` | Bad Day |
| B4 | EP-S5 | `build_singularity_image` | Bad Day |
| B5 | EP-S5 | `execute_singularity_image` | Bad Day |
| B6 | EP-S4 | `_collect_outputs_and_cleanup` | Bad Day |
| B7 | EP-D4 | `get_host_path_for_container_path` | Bad Day |
| B8 | EP-D2 | `get_host_path_for_container_path` | Bad Day |
| B9 | EP-D2 | `LocalExecutor.execute` | Bad Day |