# Task Orchestration Subsystem

## Task inventory and ordering

- Task script root is /lib/rdk.
- Primary task launcher script: /lib/rdk/Start_MaintenanceTasks.sh.
- Task arguments/order:
  1. RFC
  2. SWUPDATE
  3. LOGUPLOAD
- Critical path helper script: /lib/rdk/xconfImageCheck.sh (via startCriticalTasks).

## Execution strategy

- Each task command is executed by system() with trailing background ampersand.
- m_task_map tracks whether each task is active/pending completion event; guarded by m_taskMapMutex (read/written from the worker, IARM-event, and timer threads).
- currentTask holds the name of the in-flight task; guarded by m_currentTaskMutex (written by the worker thread, read by timer_handler() on the timer thread).
- task_status_map maps each task command to corresponding completion bit index.

- The worker does not directly inspect child process result after launch success; it relies mainly on IARM events and timeout path.

## Retry and timeout

- One retry is attempted for task invocation failure (TASK_RETRY_COUNT=1).
- Retry delay is TASK_RETRY_DELAY seconds (5). The worker drops m_callMutex for that sleep.
- Timeout per task defaults to TASK_TIMEOUT=3600 unless compile-time override in CMake.
- Timeout handler marks the timed-out task's complete bit only, clears m_task_map for that task, and notifies the worker. It does not publish maintenance status.
- A timeout of RFC or SWUPDATE is finalized by a later IARM event. A timeout of LOGUPLOAD, the last task, is finalized by the worker: publishWorkerTerminalStatus() publishes MAINTENANCE_ERROR when the cycle is still STARTED and TASKS_COMPLETED is set without ALL_TASKS_SUCCESS.

## Abort model

- Before system(), the worker and stopMaintenanceTasks share m_abortFlagMutex. Stop takes m_statusMutex first, then the abort mutex. The worker does the same, marks the task active only when the abort flag is clear, and holds the abort mutex until system() returns. If the flag is already set, the worker clears the map entry and does not launch.
- stopMaintenanceTasks inspects /proc cmdline to locate task PIDs.
- abort signaling policy:
  - rfcMgr: SIGUSR1
  - rdkvfwupgrader: SIGUSR1
  - otherwise: caller/default signal (default SIGABRT)

- SIGUSR1 is interpreted by task binaries as graceful stop path.

## Success and terminal outcome derivation

- g_task_status bitmask fields are updated for success/error/complete/reboot/skipped markers.
- Terminal determination when the IARM handler sees TASKS_COMPLETED:
  - ALL_TASKS_SUCCESS bits set -> MAINTENANCE_COMPLETE, and LastSuccessfulCompletionTime is persisted
  - otherwise -> MAINTENANCE_ERROR
- The same ERROR outcome is published by the worker when it finishes a STARTED cycle whose complete bits are all set and whose success bits are not, and neither stop nor IARM is joining it. That is the LOGUPLOAD timeout path.
- A firmware-download abort sets TASK_SKIPPED and SWUPDATE_COMPLETE without SWUPDATE_SUCCESS. The finished cycle is MAINTENANCE_ERROR. MAINTENANCE_INCOMPLETE stays in the enum and is not selected for that abort or for a task timeout.

- Exact contractual mapping between task scripts and emitted IARM statuses is external to this plugin code.
