### Link to Issue or Description of Change

**1. Link to an existing issue (if applicable):**

- Closes: #4127

**Problem:**
The `A2aAgentExecutor.cancel()` method was raising `NotImplementedError`, making the CancelTask A2A protocol method completely non-functional in ADK. Any attempt to cancel a running task would fail immediately, preventing clients from canceling tasks through the A2A protocol.

**Solution:**
Implemented the `cancel()` method with the following changes:

1. **Task Tracking**: Added `_active_tasks` dictionary to track running tasks by `task_id` for cancellation support.
2. **Race Condition Protection**: Added `asyncio.Lock` (`_tasks_lock`) to protect concurrent access to `_active_tasks`, following ADK patterns used in other components (e.g., `mcp_session_manager.py`, `local_storage.py`).
3. **Cancellation Logic**: 
   - Modified `_handle_request()` to wrap async generator iteration in an `asyncio.Task` and store it in `_active_tasks`.
   - Implemented `cancel()` to lookup the task, cancel it gracefully, and publish a `TaskStatusUpdateEvent` with `TaskState.failed` and "Task was cancelled" message.
   - Added proper cleanup in `finally` blocks to remove tasks from tracking.
4. **Edge Case Handling**: Gracefully handles missing `task_id`, non-existent tasks, and already-completed tasks with appropriate logging.

The implementation follows existing ADK patterns for async task management and ensures thread-safe access to shared state.

### Testing Plan

**Unit Tests:**

- [x] I have added or updated unit tests for my change.
- [x] All unit tests pass locally.

**pytest Results Summary:**

All 27 tests in `test_a2a_agent_executor.py` pass, including 6 new cancellation tests:

```
tests/unittests/a2a/executor/test_a2a_agent_executor.py::TestA2aAgentExecutor::test_cancel_with_task_id PASSED
tests/unittests/a2a/executor/test_a2a_agent_executor.py::TestA2aAgentExecutor::test_cancel_without_task_id PASSED
tests/unittests/a2a/executor/test_a2a_agent_executor.py::TestA2aAgentExecutor::test_cancel_running_task PASSED
tests/unittests/a2a/executor/test_a2a_agent_executor.py::TestA2aAgentExecutor::test_cancel_nonexistent_task PASSED
tests/unittests/a2a/executor/test_a2a_agent_executor.py::TestA2aAgentExecutor::test_cancel_completed_task PASSED
tests/unittests/a2a/executor/test_a2a_agent_executor.py::TestA2aAgentExecutor::test_cancel_race_condition_task_completes_before_cancel PASSED
```

**Test Coverage:**
- `test_cancel_with_task_id`: Verifies successful cancellation when task_id is provided
- `test_cancel_without_task_id`: Verifies graceful handling when task_id is missing
- `test_cancel_running_task`: Verifies cancellation of an actively running task and proper event publishing
- `test_cancel_nonexistent_task`: Verifies graceful handling when task doesn't exist
- `test_cancel_completed_task`: Verifies graceful handling when task is already completed
- `test_cancel_race_condition_task_completes_before_cancel`: Verifies race condition where task completes before cancel() can execute (prevents duplicate final events)

**Manual End-to-End (E2E) Tests:**

**Setup:**
1. Ensure you have an A2A-compatible agent configured
2. Start an A2A server with `A2aAgentExecutor`

**Test Steps:**
1. Start a long-running task via A2A protocol
2. While the task is running, send a CancelTask request with the task's `task_id`
3. Verify that:
   - The task is cancelled and execution stops
   - A `TaskStatusUpdateEvent` with `TaskState.failed` and message "Task was cancelled" is published
   - The event has `final=True` to indicate task completion
   - No duplicate cancellation events are published

**Expected Logs:**
```
INFO: Cancelling task <task_id>
INFO: Task <task_id> was cancelled
```

**Verification:**
- Check event queue for `TaskStatusUpdateEvent` with `state=TaskState.failed` and `message="Task was cancelled"`
- Verify task execution stops immediately after cancellation
- Verify no errors are raised when cancelling non-existent or completed tasks

### Screenshots

**Unit Test Results - All Cancellation Tests Passing:**
<img width="1209" height="336" alt="Screenshot 2026-01-14 160554" src="https://github.com/user-attachments/assets/a422acf4-1cd6-4fda-afcd-c9af95397f19" />

**Implementation - cancel() Method:**
<img width="1228" height="987" alt="Screenshot 2026-01-14 163417" src="https://github.com/user-attachments/assets/4ada39c5-33e2-460a-b218-adf0a5a81f8c" />

**E2E Test Results - Cancellation Verification:**
<img width="845" height="204" alt="Screenshot 2026-01-14 163215" src="https://github.com/user-attachments/assets/dd8cbf53-91bb-4236-b69e-ba079c28abca" />
