# MaintenanceManager Plugin Lifecycle

## Lifecycle Walkthrough

## 1) Build and registration

- Shared library target built from MaintenanceManager.cpp and Module.cpp.
- CMake sets MODULE_NAME compile definition and installs plugin shared object to lib/<namespace-lower>/plugins.
- Thunder registration macro is present: SERVICE_REGISTRATION(MaintenanceManager, 1, 0, 24).

## 2) Activation and constructor path

- Constructor registers all JSON-RPC methods.
- Constructor initializes task map and device initialization context mapping keys:
  - partnerId -> TR181_PARTNER_ID
  - osClass -> TR181_TARGET_OS_CLASS
  - regionalConfigService -> TR181_XCONFURL

- Constructor performs static capability registration and data map pre-wiring before plugin activation callbacks.

## 3) Initialize()

- Stores and AddRef() IShell pointer.
- Reads WHOAMI_SUPPORT from /etc/device.properties.
- If WhoAmI enabled, subscribes to SecManager device context update event.
- Calls InitializeIARM() when IARM support macros are enabled.
- Registers SIGALRM handler for task timeout.
- Returns empty string on success, error text on signal-handler registration failure.

- ASSERT(timerid != nullptr) is intended as runtime guard but may rely on platform/compiler behavior for timer_t representation.

## 4) InitializeIARM() and boot trigger

- Calls Utils::IARM::init().
- Registers IARM event handler for maintenance manager update event.
- Immediately triggers maintenanceManagerOnBootup().

### Bootup Initialization Behavior

- Sets initial mode and trigger fields.
- Sets maintenance type UNSOLICITED_MAINTENANCE.
- Validates persisted softwareoptout and normalizes invalid/empty to NONE.
- Resets runtime flags and posts initial onMaintenanceStatusChange(MAINTENANCE_IDLE).
- Spawns worker thread for maintenance execution.

- Boot-time maintenance is designed as default unsolicited orchestration path once plugin initializes and IARM registration succeeds.

## 5) Steady-state execution resources

- Runtime uses:
  - worker std::thread m_thread
  - task timer created by POSIX timer_create()/timer_settime()/timer_delete()
  - condition variable task_thread for worker/event coordination
  - multiple mutexes: m_callMutex, m_waiMutex, m_statusMutex

## 6) Deinitialize()

- Attempts timer deletion.
- Calls stopMaintenanceTasks() before IARM deinit (under IARM build).
- Removes IARM event handler and nulls singleton instance in deinit path.
- Releases IShell and IAuthService interface references.

- Deinitialize is intended to be defensive cleanup even during in-progress maintenance.

## Failure and recovery behavior

- Thread creation failures in boot and startMaintenance paths are caught and converted to MAINTENANCE_ERROR or failed RPC response.
- Signal handler registration failure fails Initialize().
- Timer operation failures are logged and can degrade timeout enforcement.

- There is no explicit health watchdog for permanently blocked thread waits if no IARM completion/error event arrives and timer path is disabled/failing.
