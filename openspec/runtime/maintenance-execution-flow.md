# Maintenance Execution Flow

## Overview

Maintenance execution has two trigger families:

- Unsolicited flow: boot-triggered, starts during plugin initialization path.
- Solicited flow: caller-triggered through startMaintenance JSON-RPC.

Both converge in task_execution_thread() with divergent gating behavior.

## Trigger Paths

## Unsolicited

- Set in maintenanceManagerOnBootup(): g_maintenance_type = UNSOLICITED_MAINTENANCE.
- Boot path evaluates conditional skip before worker creation.
- Skip condition: LastMaintenanceStatus == MAINTENANCE_COMPLETE and maintenance reboot marker is present.
- If skipped, status transitions directly to MAINTENANCE_COMPLETE and g_unsolicited_complete is set true.
- If not skipped, worker starts from bootup path.

- This provides a startup maintenance cycle without explicit northbound request, while avoiding immediate repeat runs after successful maintenance reboot.

## Solicited

- startMaintenance() proceeds only if m_notify_status != MAINTENANCE_STARTED and g_unsolicited_complete is true.
- It resets task status, sets SOLICITED_MAINTENANCE, marks reboot pending true, and starts worker thread.

- Solicited start remains blocked until unsolicited phase is considered complete, including both executed and conditionally skipped unsolicited paths.

## Worker Thread Decision Logic

## Phase 1: pre-start and status handling

- For unsolicited + WhoAmI enabled, MAINTENANCE_STARTED notification may be delayed for power-compliance behavior.
- Otherwise status changes to MAINTENANCE_STARTED immediately.

## Phase 2: network and activation/identity gates

- checkNetwork()/isDeviceOnline() determine internet readiness with retries.
- If WhoAmI disabled and suppress-maintenance enabled, getActivatedStatus() can skip firmware check for activation-connect/ready/not-activated states.
- If WhoAmI enabled and unsolicited, knowWhoAmI() attempts SecManager query and may wait for onDeviceInitializationContextUpdate event before continuing.
- If gating fails with no network, status transitions to MAINTENANCE_ERROR and unsolicited flow may switch to waiting for future network event.

- Control flow is strongly policy-driven around activation and identity readiness, not just raw connectivity.

## Phase 3: task list construction

- Default ordered tasks:
  1. Start_MaintenanceTasks.sh RFC
  2. Start_MaintenanceTasks.sh SWUPDATE
  3. Start_MaintenanceTasks.sh LOGUPLOAD
- If skipFirmwareCheck true in suppress-maintenance path, SWUPDATE is skipped and marked as success+complete bits.

## Phase 4: task execution and waiting

- Tasks are launched via system() with background ampersand.
- Timer starts per task with TASK_TIMEOUT (default 3600 sec unless compile-time override).
- Non-zero task launch status triggers one retry after TASK_RETRY_DELAY, then marks error completion bit for task.
- On successful invocation, worker waits on condition variable for event-handler completion/error signal.

## Phase 5: event-driven completion and finalization

- IARM event handler updates task bits and wakes worker as tasks report complete/error/aborted.
- When TASKS_COMPLETED mask is satisfied inside the IARM handler:
  - ALL_TASKS_SUCCESS => MAINTENANCE_COMPLETE and LastSuccessfulCompletionTime persisted.
  - otherwise => MAINTENANCE_ERROR.
- The IARM handler then joins the worker. The worker does not publish a second status for that cycle.
- MAINTENANCE_INCOMPLETE remains in the status enum. The firmware-download abort path sets TASK_SKIPPED and SWUPDATE_COMPLETE without SWUPDATE_SUCCESS, so a finished cycle that includes that abort is MAINTENANCE_ERROR. MAINTENANCE_TASK_SKIPPED (0x200) is not set by that path.

## Abort and timeout behavior

- stopMaintenance() routes to stopMaintenanceTasks().
- If status is MAINTENANCE_STARTED, plugin sets the join flag and the abort flag, finds the active task process by name in /proc, sends signals (SIGUSR1 for rfcMgr/rdkvfwupgrader, default SIGABRT otherwise), stops the timer, notifies the worker, joins the thread, and sets status MAINTENANCE_ERROR.
- The worker checks the abort flag under m_abortFlagMutex before system(). If stop already won, the task map entry stays clear and system() is not called. If the worker wins, it holds m_abortFlagMutex until system() returns, so stop sees the process when it searches for the PID.
- Timer timeout handler marks the current task complete without its success bit, clears that task in m_task_map, and wakes the worker. It does not change m_notify_status.
- RFC or SWUPDATE timeout is closed by a later task's IARM event, which publishes the terminal status.
- LOGUPLOAD is the last task. Its timeout, or a failed system() on the last task, leaves every complete bit set and at least one success bit clear, with no later IARM event. After the worker releases m_callMutex, publishWorkerTerminalStatus() publishes MAINTENANCE_ERROR and, for an unsolicited cycle, sets g_unsolicited_complete.
- publishWorkerTerminalStatus() does nothing when status is not MAINTENANCE_STARTED, the abort flag is set, stop or IARM has set the join flag, a complete bit is still missing, or every success bit is set. MAINTENANCE_COMPLETE stays on the IARM path so LastSuccessfulCompletionTime is written once.

- Precise semantics of SIGUSR1 handling inside task executables are external and require component-level validation.
