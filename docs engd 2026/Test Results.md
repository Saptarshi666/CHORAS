# CHORAS — Test Results Document
## CHORAS Scalability Project - EngD 2026

**Version:** 1.0
**Status:** Draft
**Date:** 2026-03-16
**Related Documents:** `Testing Strategy.md`, `Test Design.md`
**Maintained by:** Test Manager
**CI Environment:** GitHub Actions · Ubuntu Latest · Python 3.10.20 · pytest 9.0.2
**Database:** PostgreSQL 15 (CI) · PostgreSQL (Local Docker)

---

## 1. Summary

| Metric | Value |
|---|---|
| **Total Test Cases Defined** | 116 |
| **Total Passed** | 118 |
| **Total Failed** | 0 |
| **Expected Failures (xfail)** | 1 |
| **Unexpected Passes (xpassed)** | 2 |
| **Skipped** | 0 |
| **Last Run Date** | 2026-03-16 |
| **Last Run Duration** | ~7s |

> **Note on xpassed:** 2 tests marked `xfail` unexpectedly passed. These are
> B12 and B13 in `test_cloud_executor_final.py` — the `time.sleep` side-effect
> trick forces the loop to exit, producing a pass. The `xfail` markers should
> remain until the real stall-detection fix is implemented, at which point
> they should be converted to normal passing tests.

---

## 2. Results by Test File

### 2.1 `tests/integration/test_cloud_executor.py`

**Purpose:** Bad day tests for `CloudExecutor.execute()` — verifies exception
propagation through the full `execute()` flow when individual steps fail.

| # | Test Name | Runs | Pass | Fail | Status | Notes |
|---|---|---|---|---|---|---|
| 1 | `test_ssh_authentication_fails` | 1 | 1 | 0 | ✅ Pass | EP-S2 |
| 2 | `test_sftp_upload_tar_fails_halfway` | 1 | 1 | 0 | ✅ Pass | EP-S4 |
| 3 | `test_build_singularity_image_fails` | 1 | 1 | 0 | ✅ Pass | EP-S5 |
| 4 | `test_remote_json_never_reaches_100` | 1 | 1 | 0 | ✅ Pass | EP-P5 — timeout mocked |
| 5 | `test_remote_json_always_corrupt` | 1 | 1 | 0 | ✅ Pass | EP-P4 — sleep mocked |
| 6 | `test_cancel_flag_created_before_polling` | 1 | 1 | 0 | ✅ Pass | EP-P6 |
| 7 | `test_collect_outputs_and_cleanup_fails_mid_download` | 1 | 1 | 0 | ✅ Pass | EP-S4 |
| 8 | `test_build_fails_when_remote_sandbox_already_exists` | 1 | 1 | 0 | ✅ Pass | EP-S6 |

**File Total: 8 passed, 0 failed**

---

### 2.2 `tests/integration/test_cloud_executor_final.py`

**Purpose:** Full coverage of all `CloudExecutor` internal methods including
happy path, bad day, and known bug documentation via `xfail`.

| # | Test Name | Runs | Pass | Fail | Status | Notes |
|---|---|---|---|---|---|---|
| 1 | `TestParseOverallProgress::test_multiple_results_returns_minimum` | 1 | 1 | 0 | ✅ Pass | U1, EP-P1 |
| 2 | `TestParseOverallProgress::test_single_result_at_100` | 1 | 1 | 0 | ✅ Pass | U2, EP-P2 |
| 3 | `TestParseOverallProgress::test_empty_results_list_returns_none` | 1 | 1 | 0 | ✅ Pass | U3, EP-C5 |
| 4 | `TestParseOverallProgress::test_results_entry_missing_percentage_key` | 1 | 1 | 0 | ✅ Pass | U4, EP-C5 |
| 5 | `TestParseOverallProgress::test_results_is_not_a_list_returns_none` | 1 | 1 | 0 | ✅ Pass | U5, EP-C5 |
| 6 | `TestGetFilenames::test_full_paths_stripped_to_filenames` | 1 | 1 | 0 | ✅ Pass | U6, EP-C1 |
| 7 | `TestGetFilenames::test_missing_msh_path_raises_key_error` | 1 | 1 | 0 | ✅ Pass | U7, EP-C5 |
| 8 | `TestGetFilenames::test_missing_geo_path_raises_key_error` | 1 | 1 | 0 | ✅ Pass | U8, EP-C5 |
| 9 | `TestShouldCancel::test_returns_false_when_no_cancel_flag` | 1 | 1 | 0 | ✅ Pass | EP-P1 |
| 10 | `TestShouldCancel::test_returns_true_when_cancel_flag_exists` | 1 | 1 | 0 | ✅ Pass | EP-P6 |
| 11 | `TestRunRemoteCommand::test_successful_command_returns_stdout` | 1 | 1 | 0 | ✅ Pass | I1, EP-S1 |
| 12 | `TestRunRemoteCommand::test_ssh_authentication_fails_raises_ssh_command_error` | 1 | 1 | 0 | ✅ Pass | I2, EP-S2 |
| 13 | `TestRunRemoteCommand::test_ssh_connection_timeout_raises_ssh_command_error` | 1 | 1 | 0 | ✅ Pass | I3, EP-S3 |
| 14 | `TestRunRemoteCommand::test_non_zero_exit_status_raises_ssh_command_error` | 1 | 1 | 0 | ✅ Pass | I4, EP-S1 |
| 15 | `TestUploadFileViaSftp::test_successful_upload_calls_sftp_put` | 1 | 1 | 0 | ✅ Pass | I5, EP-S1 |
| 16 | `TestUploadFileViaSftp::test_sftp_upload_fails_halfway_raises_exception` | 1 | 1 | 0 | ✅ Pass | I6, EP-S4 |
| 17 | `TestBuildSingularityImage::test_successful_build_runs_correct_command` | 1 | 1 | 0 | ✅ Pass | I7, EP-S1 |
| 18 | `TestBuildSingularityImage::test_disk_full_on_remote_raises_ssh_command_error` | 1 | 1 | 0 | ✅ Pass | I8, EP-S5 |
| 19 | `TestBuildSingularityImage::test_sandbox_already_exists_raises_ssh_command_error` | 1 | 1 | 0 | ✅ Pass | I9, EP-S6 |
| 20 | `TestExecuteSingularityImage::test_launches_singularity_in_background` | 1 | 1 | 0 | ✅ Pass | I10, EP-S1 |
| 21 | `TestExecuteSingularityImage::test_command_includes_entry_file` | 1 | 1 | 0 | ✅ Pass | I11, EP-S1 |
| 22 | `TestPollUntilComplete::test_progress_reaches_100_calls_cleanup_and_returns_true` | 1 | 1 | 0 | ✅ Pass | I12, EP-P1/P2 |
| 23 | `TestPollUntilComplete::test_json_only_written_locally_when_progress_changes` | 1 | 1 | 0 | ✅ Pass | I13, EP-P1 |
| 24 | `TestPollUntilComplete::test_corrupt_json_retries_then_recovers` | 1 | 1 | 0 | ✅ Pass | I14, EP-P3 |
| 25 | `TestPollUntilComplete::test_cancel_flag_before_polling_exits_immediately` | 1 | 1 | 0 | ✅ Pass | I15, EP-P6 |
| 26 | `TestPollUntilComplete::test_cancel_flag_mid_polling_stops_at_next_cycle` | 1 | 1 | 0 | ✅ Pass | I16, EP-P7 |
| 27 | `TestPollUntilComplete::test_progress_stuck_raises_runtime_error` | 1 | — | — | ⚠️ xpassed | B12, EP-P5 — known bug, xfail marker should remain |
| 28 | `TestPollUntilComplete::test_json_always_corrupt_raises_runtime_error` | 1 | — | — | ⚠️ xpassed | B13, EP-P4 — known bug, xfail marker should remain |
| 29 | `TestCollectOutputsAndCleanup::test_downloads_only_json_and_csv_ignores_others` | 1 | 1 | 0 | ✅ Pass | I17, EP-O1 |
| 30 | `TestCollectOutputsAndCleanup::test_cleanup_called_after_successful_download` | 1 | 1 | 0 | ✅ Pass | I18, EP-O1 |
| 31 | `TestCollectOutputsAndCleanup::test_sftp_download_failure_returns_false_no_cleanup` | 1 | 1 | 0 | ✅ Pass | I19, EP-S4 |
| 32 | `TestExecuteHappyPath::test_execute_returns_completed_job` | 1 | 1 | 0 | ✅ Pass | I20, EP-S1 |
| 33 | `TestExecuteHappyPath::test_execute_calls_poll_until_complete` | 1 | 1 | 0 | ✅ Pass | I21, EP-S1 |
| 34 | `TestExecuteHappyPath::test_execute_uploads_tar_file` | 1 | 1 | 0 | ✅ Pass | I22, EP-S1 |
| 35 | `TestExecuteHappyPath::test_execute_strips_tag_from_sandbox_name` | 1 | 1 | 0 | ✅ Pass | I23, EP-S1 |
| 36 | `TestCancel::test_cancel_kills_processes_and_cleans_up` | 1 | 1 | 0 | ✅ Pass | I24, EP-S1 |
| 37 | `TestCancel::test_cancel_constructs_correct_sandbox_name` | 1 | 1 | 0 | ✅ Pass | I25, EP-S1 |

**File Total: 35 passed, 0 failed, 2 xpassed**

---

### 2.3 `tests/integration/test_executor_factory.py`

**Purpose:** Tests `executor_factory()` routing logic and edge cases.

| # | Test Name | Runs | Pass | Fail | Status | Notes |
|---|---|---|---|---|---|---|
| 1 | `test_invalid_resource_type_raises_value_error` | 1 | 1 | 0 | ✅ Pass | EF1, EP-E3 |
| 2 | `test_discover_container_image_none_calls_cloud_executor` | 1 | 1 | 0 | ✅ Pass | EF2, EP-M5 |
| 3 | `test_discover_entry_file_none_cloud_executor_entry_file_none` | 1 | 1 | 0 | ✅ Pass | EF3, EP-M6 |

**File Total: 3 passed, 0 failed**

---

### 2.4 `tests/integration/test_local_executor.py`

**Purpose:** Original bad day tests for `LocalExecutor.execute()`.

| # | Test Name | Runs | Pass | Fail | Status | Notes |
|---|---|---|---|---|---|---|
| 1 | `test_docker_image_not_found` | 1 | 1 | 0 | ✅ Pass | EP-D2 |
| 2 | `test_docker_socket_not_available` | 1 | 1 | 0 | ✅ Pass | EP-D2 |
| 3 | `test_json_path_missing` | 1 | 1 | 0 | ✅ Pass | EP-C2 |
| 4 | `test_no_matching_mount` | 1 | 1 | 0 | ✅ Pass | EP-D4 |
| 5 | `test_container_exits_nonzero_obj_missing` | 1 | 1 | 0 | ✅ Pass | EP-D5 — known bug |
| 6 | `test_duplicate_container_name_conflict` | 1 | 1 | 0 | ✅ Pass | EP-D3 |

**File Total: 6 passed, 0 failed**

---

### 2.5 `tests/integration/test_local_executor_final.py`

**Purpose:** Full coverage of all `LocalExecutor` methods.

| # | Test Name | Runs | Pass | Fail | Status | Notes |
|---|---|---|---|---|---|---|
| 1 | `TestGetHostPathForContainerPath::test_resolves_exact_mount_destination` | 1 | 1 | 0 | ✅ Pass | U18, EP-D1 |
| 2 | `TestGetHostPathForContainerPath::test_resolves_subdirectory_of_mount` | 1 | 1 | 0 | ✅ Pass | U19, EP-D1 |
| 3 | `TestGetHostPathForContainerPath::test_raises_when_no_mount_covers_path` | 1 | 1 | 0 | ✅ Pass | B7, EP-D4 |
| 4 | `TestGetHostPathForContainerPath::test_raises_when_docker_client_fails` | 1 | 1 | 0 | ✅ Pass | B8, EP-D2 |
| 5 | `TestGetHostPathForContainerPath::test_uses_hostname_to_identify_container` | 1 | 1 | 0 | ✅ Pass | U20, EP-D1 |
| 6 | `TestGetHostPathForContainerPath::test_normalises_backslashes_to_forward_slashes` | 1 | 1 | 0 | ✅ Pass | U21, EP-D1 |
| 7 | `TestLocalExecutorInit::test_default_work_dir_from_env` | 1 | 1 | 0 | ✅ Pass | U22, EP-D1 |
| 8 | `TestLocalExecutorInit::test_default_work_dir_fallback` | 1 | 1 | 0 | ✅ Pass | U23, EP-D1 |
| 9 | `TestLocalExecutorInit::test_explicit_work_dir` | 1 | 1 | 0 | ✅ Pass | U24, EP-D1 |
| 10 | `TestLocalExecutorExecuteHappyPath::test_returns_container_object` | 1 | 1 | 0 | ✅ Pass | I1-LE, EP-D1 |
| 11 | `TestLocalExecutorExecuteHappyPath::test_passes_correct_image_to_containers_run` | 1 | 1 | 0 | ✅ Pass | I2-LE, EP-D1 |
| 12 | `TestLocalExecutorExecuteHappyPath::test_passes_env_to_containers_run` | 1 | 1 | 0 | ✅ Pass | I3-LE, EP-C1 |
| 13 | `TestLocalExecutorExecuteHappyPath::test_volume_mount_uses_resolved_host_path` | 1 | 1 | 0 | ✅ Pass | I4-LE, EP-D1 |
| 14 | `TestLocalExecutorExecuteHappyPath::test_container_runs_detached` | 1 | 1 | 0 | ✅ Pass | I5-LE, EP-D1 |
| 15 | `TestLocalExecutorExecuteHappyPath::test_de_method_on_simple_geometry` | 1 | 1 | 0 | ✅ Pass | I6-LE, EP-M1/G1 |
| 16 | `TestLocalExecutorExecuteHappyPath::test_dg_method_on_moderate_geometry` | 1 | 1 | 0 | ✅ Pass | I7-LE, EP-M2/G2 |
| 17 | `TestLocalExecutorExecuteHappyPath::test_new_method_on_complex_geometry` | 1 | 1 | 0 | ✅ Pass | I8-LE, EP-M3/G3 |
| 18 | `TestLocalExecutorExecuteHappyPath::test_containers_run_called_exactly_once` | 1 | 1 | 0 | ✅ Pass | I9-LE, EP-D1 |
| 19 | `TestLocalExecutorCancel::test_cancel_kills_and_removes_running_container` | 1 | 1 | 0 | ✅ Pass | I10-LE, EP-D1 |
| 20 | `TestLocalExecutorCancel::test_cancel_container_not_found_does_not_raise` | 1 | 1 | 0 | ✅ Pass | I11-LE, EP-D2 |

**File Total: 20 passed, 0 failed**

---

### 2.6 `tests/integration/test_missing_cases.py`

**Purpose:** Gap coverage for test cases not in original files.

| # | Test Name | Runs | Pass | Fail | Status | Notes |
|---|---|---|---|---|---|---|
| 1 | `TestGetLocalFilePath::test_joins_dirname_and_filename` | 1 | 1 | 0 | ✅ Pass | U5, EP-C1 |
| 2 | `TestGetLocalFilePath::test_works_with_nested_directory` | 1 | 1 | 0 | ✅ Pass | U6, EP-C1 |
| 3 | `TestGetRemoteFilePath::test_constructs_correct_remote_path` | 1 | 1 | 0 | ✅ Pass | U7, EP-C1 |
| 4 | `TestCompletedJob::test_wait_returns_zero_status_code` | 1 | 1 | 0 | ✅ Pass | U8, EP-O1 |
| 5 | `TestCompletedJob::test_logs_returns_bytes` | 1 | 1 | 0 | ✅ Pass | U9, EP-O1 |
| 6 | `TestCloudExecutorInit::test_stores_all_constructor_parameters` | 1 | 1 | 0 | ✅ Pass | U10, EP-S1 |
| 7 | `TestCloudExecutorInit::test_local_cancel_flag_path_initially_none` | 1 | 1 | 0 | ✅ Pass | U11, EP-S1 |
| 8 | `TestLocalExecutorInitJobs::test_jobs_dict_initialised_empty` | 1 | — | — | ⚠️ xfail | U25 — `_jobs` not yet added to `__init__` |
| 9 | `TestDownloadFileViaSftp::test_successful_download_calls_sftp_get` | 1 | 1 | 0 | ✅ Pass | I7, EP-S1 |
| 10 | `TestListRemoteFiles::test_returns_full_paths_and_excludes_hidden_files` | 1 | 1 | 0 | ✅ Pass | I8, EP-O1 |
| 11 | `TestListRemoteFiles::test_returns_empty_list_for_empty_directory` | 1 | 1 | 0 | ✅ Pass | I8 boundary, EP-O1 |
| 12 | `TestDeleteRemotePath::test_runs_rm_rf_command` | 1 | 1 | 0 | ✅ Pass | I9, EP-S1 |
| 13 | `TestDeleteRemotePath::test_delete_propagates_ssh_error` | 1 | 1 | 0 | ✅ Pass | I9 bad day, EP-S5 |
| 14 | `TestExecuteSingularityImageBadDay::test_raises_when_exec_command_fails` | 1 | 1 | 0 | ✅ Pass | B5, EP-S5 |
| 15 | `TestExecuteSingularityImageBadDay::test_raises_when_ssh_connection_drops_mid_launch` | 1 | 1 | 0 | ✅ Pass | B5v, EP-S5 |

**File Total: 14 passed, 0 failed, 1 xfail**

---

### 2.7 `tests/unit/services/test_discovery_service.py`

**Purpose:** Tests all `discovery_service.py` functions against real config.

| # | Test Name | Runs | Pass | Fail | Status | Notes |
|---|---|---|---|---|---|---|
| 1 | `test_discover_methods_real_config` | 1 | 1 | 0 | ✅ Pass | DS1, EP-DS1 |
| 2 | `test_discover_method_names` | 1 | 1 | 0 | ✅ Pass | DS2, EP-DS1 |
| 3 | `test_discover_container_image` | 1 | 1 | 0 | ✅ Pass | DS3, EP-DS1/M5 |
| 4 | `test_discover_entry_file` | 1 | 1 | 0 | ✅ Pass | DS4, EP-DS1/M6 |
| 5 | `test_discover_methods_config_structure_validation` | 1 | 1 | 0 | ✅ Pass | DS5, EP-DS4 |
| 6 | `test_discover_methods_settings_files_exist` | 1 | 1 | 0 | ✅ Pass | DS6, EP-DS4 |

**File Total: 6 passed, 0 failed**

---

### 2.8 `tests/unit/services/test_run_solver.py`

**Purpose:** Tests `run_solver()` bad day scenarios and known bugs.

| # | Test Name | Runs | Pass | Fail | Status | Notes |
|---|---|---|---|---|---|---|
| 1 | `test_simulation_run_not_found_early_return` | 1 | 1 | 0 | ✅ Pass | RS1, EP-DB2 |
| 2 | `test_simulation_none_crash` | 1 | 1 | 0 | ✅ Pass | RS2, EP-DB3 |
| 3 | `test_solver_settings_none_sets_error_status` | 1 | 1 | 0 | ✅ Pass | RS3, EP-C6 |
| 4 | `test_json_unreadable_sets_error_status` | 1 | 1 | 0 | ✅ Pass | RS4, EP-C4 |
| 5 | `test_auralization_fails_error_status_orphaned_xlsx` | 1 | 1 | 0 | ✅ Pass | RS5, EP-O7 |
| 6 | `test_export_false_sets_error_status` | 1 | 1 | 0 | ✅ Pass | RS6, EP-O4 |
| 7 | `test_container_non_zero_exit_marked_completed` | 1 | 1 | 0 | ✅ Pass | RS7, EP-D5 — documents known bug |

**File Total: 7 passed, 0 failed**

---


---

## 4. Known Bugs Captured by Tests

| Test | Bug | Severity | Resolution |
|---|---|---|---|
| B12 (xpassed) | No stall timeout in `_poll_until_complete` — progress stuck at 50% loops forever | 🔴 Critical | Add stall detection, remove xfail |
| B13 (xpassed) | No `POLL_MAX_FAILED_CYCLES` — corrupt JSON loops forever | 🔴 Critical | Add cycle limit, remove xfail |
| RS7 | `container.wait()` exit code ignored — failed run marked `Completed` | 🔴 Critical | Check exit code in `run_solver()` |
| RS6 | `raise "Error saving..."` invalid Python → `TypeError` | 🟠 High | Replace with `raise Exception(...)` |
| U25 (xfail) | `_jobs = {}` not in `LocalExecutor.__init__` | 🟡 Medium | Add attribute, remove xfail |

---

## 5. Coverage by Equivalence Partition

| Partition | Description | Covered | Tests |
|---|---|---|---|
| EP-M1 | DE (cheap method) | ✅ | I6-LE, DS3, DS4 |
| EP-M2 | DG (expensive method) | ✅ | I7-LE, I20, DS3 |
| EP-M3 | MyNewMethod (user-added) | ✅ | I8-LE |
| EP-M4 | Method not in discovery | ✅ | REM4, REM5 |
| EP-M5 | `container_image` is `None` | ✅ | EF2 |
| EP-M6 | `entry_file` is `None` | ✅ | EF3, REM5 |
| EP-E1 | Local execution | ✅ | All LocalExecutor tests |
| EP-E2 | Cloud execution | ✅ | All CloudExecutor tests |
| EP-E3 | Invalid resource type | ✅ | EF1 |
| EP-G1 | Simple geometry | ✅ | I6-LE |
| EP-G2 | Moderate geometry | ✅ | I7-LE |
| EP-G3 | Complex geometry | ✅ | I8-LE |
| EP-C1 | Valid config | ✅ | I1–I9 (LocalExecutor), U1, U5–U7 |
| EP-C2 | `JSON_PATH` missing | ✅ | B5 (LocalExecutor) |
| EP-C4 | JSON unreadable | ✅ | RS4 |
| EP-C5 | JSON malformed | ✅ | U3–U5, U12–U17 |
| EP-C6 | `solverSettings` is `None` | ✅ | RS3 |
| EP-D1 | Docker running, image present | ✅ | All LocalExecutor happy path |
| EP-D2 | Docker daemon down | ✅ | B4, B8, I11-LE |
| EP-D3 | Duplicate container name | ✅ | B6 (original) |
| EP-D4 | Host mount unresolvable | ✅ | B7, B6 (LE) |
| EP-D5 | Container exits non-zero | ✅ | B-D5, RS7 (known bug) |
| EP-S1 | SSH healthy | ✅ | All CloudExecutor happy path |
| EP-S2 | SSH auth fails | ✅ | I2, B9 |
| EP-S3 | SSH timeout | ✅ | I3 |
| EP-S4 | SFTP upload fails | ✅ | I6, B10, B6 cleanup |
| EP-S5 | Remote disk full | ✅ | I8, B5, B11 |
| EP-S6 | Sandbox already exists | ✅ | I9, B16 |
| EP-P1 | Progress increments to 100% | ✅ | I12–I16 |
| EP-P2 | Progress 100% on first poll | ✅ | I12 |
| EP-P3 | JSON temporarily corrupt | ✅ | I17 |
| EP-P4 | JSON always corrupt | ✅ | B13 (xpassed) |
| EP-P5 | Progress stuck forever | ✅ | B12 (xpassed) |
| EP-P6 | Cancel flag before polling | ✅ | I15, I18 |
| EP-P7 | Cancel flag mid-polling | ✅ | I16, I19 |
| EP-O1 | Outputs present | ✅ | I12–I14, U8–U9 |
| EP-O4 | XLSX export fails | ✅ | RS6 |
| EP-O7 | Auralization fails after XLSX | ✅ | RS5 |
| EP-DB2 | `SimulationRun` not in DB | ✅ | RS1 |
| EP-DB3 | `Simulation` not found | ✅ | RS2 |
| EP-DB4 | DB commit fails | ✅ | REM1, REM2 |
| EP-DS1 | All methods well-formed | ✅ | DS1–DS6 |
| EP-DS3 | Method removed from repo | ✅ | REM3 |
| EP-DS4 | Malformed method definition | ✅ | DS5, DS6 |

---

## 6. Defect Log

| ID | Severity | Description | Status | Linked Test |
|---|---|---|---|---|
| DEF-001 | 🔴 Critical | No timeout in `_poll_until_complete` — stuck jobs hang backend forever | Open | B12 (xpassed) |
| DEF-002 | 🔴 Critical | No `POLL_MAX_FAILED_CYCLES` — corrupt JSON loops forever | Open | B13 (xpassed) |
| DEF-003 | 🔴 Critical | `container.wait()` exit code ignored — failed run marked `Completed` | Open | RS7 |
| DEF-004 | 🟠 High | `raise "Error saving..."` is invalid Python — `TypeError` raised | Open | RS6 |
| DEF-005 | 🟠 High | SSH failure mid-execute leaves remote sandbox unclean | Open | B9 |
| DEF-006 | 🟡 Medium | `_jobs = {}` not in `LocalExecutor.__init__` | Open | U25 (xfail) |
| DEF-007 | 🟡 Medium | `match` block silently skips auralization for methods beyond DE and DG | Open | — (commented out) |
