---
name: test-driven-development
description: Drives development with tests. Use when implementing any logic, fixing any bug, or changing any behavior. Use when you need to prove that code works, when a bug report arrives, or when you're about to modify existing functionality.
x-provenance:
  origin: addyosmani/agent-skills
  upstream_path: skills/test-driven-development/SKILL.md
  upstream_commit: 19e49a094d79540e635b107cb3490926ddeac7a3
  status: modified
  last_synced: 2026-05-01
---

# Test-Driven Development

## Overview

Write a failing test before writing the code that makes it pass. For bug fixes, reproduce the bug with a test before attempting a fix. Tests are proof — "seems right" is not done. A codebase with good tests is an AI agent's superpower; a codebase without tests is a liability.

## When to Use

- Implementing any new logic or behavior
- Fixing any bug (the Prove-It Pattern)
- Modifying existing functionality
- Adding edge case handling
- Any change that could break existing behavior

**When NOT to use:** Pure configuration changes, documentation updates, or static content changes that have no behavioral impact.

## The TDD Cycle

```
    RED                GREEN              REFACTOR
 Write a test    Write minimal code    Clean up the
 that fails  ──→  to make it pass  ──→  implementation  ──→  (repeat)
      │                  │                    │
      ▼                  ▼                    ▼
   Test FAILS        Test PASSES         Tests still PASS
```

### Step 1: RED — Write a Failing Test

Write the test first. It must fail. A test that passes immediately proves nothing.

```go
// RED: This test fails because CreateTask doesn't exist yet
func TestTaskService_CreateTask(t *testing.T) {
    svc := NewTaskService(...)

    task, err := svc.CreateTask(ctx, CreateTaskInput{Title: "Buy groceries"})
    if err != nil {
        t.Fatalf("CreateTask returned error: %v", err)
    }

    if task.ID == "" {
        t.Errorf("task.ID is empty")
    }
    if task.Title != "Buy groceries" {
        t.Errorf("task.Title = %q, want %q", task.Title, "Buy groceries")
    }
    if task.Status != StatusPending {
        t.Errorf("task.Status = %v, want %v", task.Status, StatusPending)
    }
    if task.CreatedAt.IsZero() {
        t.Errorf("task.CreatedAt is zero")
    }
}
```

### Step 2: GREEN — Make It Pass

Write the minimum code to make the test pass. Don't over-engineer:

```go
// GREEN: Minimal implementation
func (s *TaskService) CreateTask(ctx context.Context, in CreateTaskInput) (*Task, error) {
    task := &Task{
        ID:        generateID(),
        Title:     in.Title,
        Status:    StatusPending,
        CreatedAt: time.Now(),
    }
    if err := s.db.InsertTask(ctx, task); err != nil {
        return nil, fmt.Errorf("insert task: %w", err)
    }
    return task, nil
}
```

### Step 3: REFACTOR — Clean Up

With tests green, improve the code without changing behavior:

- Extract shared logic
- Improve naming
- Remove duplication
- Optimize if necessary

Run tests after every refactor step to confirm nothing broke.

## The Prove-It Pattern (Bug Fixes)

When a bug is reported, **do not start by trying to fix it.** Start by writing a test that reproduces it.

```
Bug report arrives
       │
       ▼
  Write a test that demonstrates the bug
       │
       ▼
  Test FAILS (confirming the bug exists)
       │
       ▼
  Implement the fix
       │
       ▼
  Test PASSES (proving the fix works)
       │
       ▼
  Run full test suite (no regressions)
```

**Example:**

```go
// Bug: "Completing a task doesn't update the CompletedAt timestamp"

// Step 1: Write the reproduction test (it should FAIL)
func TestTaskService_CompleteTask_SetsCompletedAt(t *testing.T) {
    svc := NewTaskService(...)
    ctx := context.Background()

    task, err := svc.CreateTask(ctx, CreateTaskInput{Title: "Test"})
    if err != nil {
        t.Fatalf("CreateTask returned error: %v", err)
    }

    completed, err := svc.CompleteTask(ctx, task.ID)
    if err != nil {
        t.Fatalf("CompleteTask returned error: %v", err)
    }

    if completed.Status != StatusCompleted {
        t.Errorf("Status = %v, want %v", completed.Status, StatusCompleted)
    }
    if completed.CompletedAt.IsZero() {
        t.Errorf("CompletedAt is zero (bug confirmed)")
    }
}

// Step 2: Fix the bug
func (s *TaskService) CompleteTask(ctx context.Context, id string) (*Task, error) {
    return s.db.UpdateTask(ctx, id, TaskUpdate{
        Status:      StatusCompleted,
        CompletedAt: time.Now(), // This was missing
    })
}

// Step 3: Test passes → bug fixed, regression guarded
```

## The Test Pyramid

Invest testing effort according to the pyramid — most tests should be small and fast, with progressively fewer tests at higher levels:

```
          ╱╲
         ╱  ╲         E2E Tests (~5%)
        ╱    ╲        Full user flows, real environment
       ╱──────╲
      ╱        ╲      Integration Tests (~15%)
     ╱          ╲     Component interactions, API boundaries
    ╱────────────╲
   ╱              ╲   Unit Tests (~80%)
  ╱                ╲  Pure logic, isolated, milliseconds each
 ╱──────────────────╲
```

**The Beyonce Rule:** If you liked it, you should have put a test on it. Infrastructure changes, refactoring, and migrations are not responsible for catching your bugs — your tests are. If a change breaks your code and you didn't have a test for it, that's on you.

### Test Sizes (Resource Model)

Beyond the pyramid levels, classify tests by what resources they consume:

| Size | Constraints | Speed | Example |
|------|------------|-------|---------|
| **Small** | Single process, no I/O, no network, no database | Milliseconds | Pure function tests, data transforms |
| **Medium** | Multi-process OK, localhost only, no external services | Seconds | API tests with test DB, component tests |
| **Large** | Multi-machine OK, external services allowed | Minutes | E2E tests, performance benchmarks, staging integration |

Small tests should make up the vast majority of your suite. They're fast, reliable, and easy to debug when they fail.

**Mark medium and large tests with build tags so they don't run by default.** A common pattern is `//go:build integration` at the top of integration test files, with `go test -tags=integration ./...` as the explicit invocation. This keeps the small-test suite fast enough to run frequently during development.

### Decision Guide

```
Is it pure logic with no side effects?
  → Unit test (small)

Does it cross a boundary (API, database, file system)?
  → Integration test (medium)

Is it a critical user flow that must work end-to-end?
  → E2E test (large) — limit these to critical paths
```

## Writing Good Tests

### Test State, Not Interactions

Assert on the *outcome* of an operation, not on which methods were called internally. Tests that verify method call sequences break when you refactor, even if the behavior is unchanged.

```go
// Good: Tests what the function does (state-based)
func TestListTasks_SortedByCreationDateDescending(t *testing.T) {
    tasks, err := listTasks(ctx, ListOptions{SortBy: "createdAt", SortOrder: "desc"})
    if err != nil {
        t.Fatalf("listTasks returned error: %v", err)
    }
    if !tasks[0].CreatedAt.After(tasks[1].CreatedAt) {
        t.Errorf("expected tasks sorted newest first, got %v then %v",
            tasks[0].CreatedAt, tasks[1].CreatedAt)
    }
}

// Bad: Tests how the function works internally (interaction-based)
func TestListTasks_CallsDBQueryWithOrderByClause(t *testing.T) {
    fakeDB := &fakeDB{}
    listTasksWithDB(ctx, fakeDB, ListOptions{SortBy: "createdAt", SortOrder: "desc"})

    if !strings.Contains(fakeDB.LastQuery, "ORDER BY created_at DESC") {
        t.Errorf("expected query to contain ORDER BY clause, got %q", fakeDB.LastQuery)
    }
}
```

### DAMP Over DRY in Tests

In production code, DRY (Don't Repeat Yourself) is usually right. In tests, **DAMP (Descriptive And Meaningful Phrases)** is better. A test should read like a specification — each test should tell a complete story without requiring the reader to trace through shared helpers.

```go
// DAMP: Each test is self-contained and readable
func TestCreateTask_RejectsEmptyTitle(t *testing.T) {
    _, err := CreateTask(CreateTaskInput{Title: "", Assignee: "user-1"})
    if err == nil {
        t.Fatal("expected error for empty title, got nil")
    }
    if !errors.Is(err, ErrTitleRequired) {
        t.Errorf("err = %v, want %v", err, ErrTitleRequired)
    }
}

func TestCreateTask_TrimsWhitespaceFromTitle(t *testing.T) {
    task, err := CreateTask(CreateTaskInput{Title: "  Buy groceries  ", Assignee: "user-1"})
    if err != nil {
        t.Fatalf("CreateTask returned error: %v", err)
    }
    if task.Title != "Buy groceries" {
        t.Errorf("task.Title = %q, want %q", task.Title, "Buy groceries")
    }
}

// Over-DRY: Shared setup obscures what each test actually verifies
// (Don't extract the input shape into a helper just to avoid repetition)
```

Duplication in tests is acceptable when it makes each test independently understandable.

### Prefer Real Implementations Over Mocks

Use the simplest test double that gets the job done. The more your tests use real code, the more confidence they provide.

```
Preference order (most to least preferred):
1. Real implementation  → Highest confidence, catches real bugs
2. Fake                 → In-memory version of a dependency (e.g., fake DB)
3. Stub                 → Returns canned data, no behavior
4. Mock (interaction)   → Verifies method calls — use sparingly
```

**Use mocks only when:** the real implementation is too slow,
non-deterministic, or has side effects you can't control (external APIs, email
sending). Over-mocking creates tests that pass while production breaks.

**For external dependencies, prefer ephemeral real instances over mocks.**
Tests that need a database, message queue, or third-party service should spin
up a real instance per test run rather than mock the dependency. Tools like
testcontainers (available for Go and Python) and docker-compose make this
practical. Real dependencies catch bugs that mocks miss: schema mismatches,
connection-pooling edge cases, SQL dialect quirks, serialization issues. The
cost is slower tests; the benefit is tests that actually verify the contract
with the real system.

### Use the Arrange-Act-Assert Pattern

```go
func TestCheckOverdue_MarksOverdueWhenDeadlinePassed(t *testing.T) {
    // Arrange: Set up the test scenario
    deadline := time.Date(2025, 1, 1, 0, 0, 0, 0, time.UTC)
    now := time.Date(2025, 1, 2, 0, 0, 0, 0, time.UTC)
    task := Task{Title: "Test", Deadline: deadline}

    // Act: Perform the action being tested
    result := CheckOverdue(task, now)

    // Assert: Verify the outcome
    if !result.IsOverdue {
        t.Errorf("IsOverdue = false, want true")
    }
}
```

### One Assertion Per Concept

```go
// Good: Each case verifies one behavior
func TestCreateTask_TitleValidation(t *testing.T) {
    tests := []struct {
        name    string
        title   string
        want    string
        wantErr error
    }{
        {
            name:    "rejects_empty_title",
            title:   "",
            wantErr: ErrTitleRequired,
        },
        {
            name:  "trims_whitespace",
            title: "  hello  ",
            want:  "hello",
        },
        {
            name:    "rejects_too_long_title",
            title:   strings.Repeat("a", 256),
            wantErr: ErrTitleTooLong,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            task, err := CreateTask(CreateTaskInput{Title: tt.title})

            if tt.wantErr != nil {
                if !errors.Is(err, tt.wantErr) {
                    t.Errorf("err = %v, want %v", err, tt.wantErr)
                }
                return
            }
            if err != nil {
                t.Fatalf("CreateTask returned error: %v", err)
            }
            if task.Title != tt.want {
                t.Errorf("task.Title = %q, want %q", task.Title, tt.want)
            }
        })
    }
}

// Bad: Everything in one test, multiple concepts entangled
func TestCreateTask_ValidatesTitlesCorrectly(t *testing.T) {
    if _, err := CreateTask(CreateTaskInput{Title: ""}); err == nil {
        t.Errorf("expected error for empty title")
    }
    if task, _ := CreateTask(CreateTaskInput{Title: "  hello  "}); task.Title != "hello" {
        t.Errorf("expected trimmed title")
    }
    if _, err := CreateTask(CreateTaskInput{Title: strings.Repeat("a", 256)}); err == nil {
        t.Errorf("expected error for too-long title")
    }
}
```

### Name Tests Descriptively

```go
// Good: Reads like a specification
func TestTaskService_CompleteTask(t *testing.T) {
    t.Run("sets_status_to_completed_and_records_timestamp", func(t *testing.T) {
        // ...
    })
    t.Run("returns_not_found_error_for_nonexistent_task", func(t *testing.T) {
        // ...
    })
    t.Run("is_idempotent_completing_already_completed_task_is_noop", func(t *testing.T) {
        // ...
    })
    t.Run("sends_notification_to_task_assignee", func(t *testing.T) {
        // ...
    })
}

// Bad: Vague names
func TestTaskService(t *testing.T) {
    t.Run("works", func(t *testing.T) { /* ... */ })
    t.Run("handles_errors", func(t *testing.T) { /* ... */ })
    t.Run("test_3", func(t *testing.T) { /* ... */ })
}
```

Subtest names are also addressable via `go test -run TestName/subtest_name`, so descriptive names pay off when investigating failures.

## Test Anti-Patterns to Avoid

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Testing implementation details | Tests break when refactoring even if behavior is unchanged | Test inputs and outputs, not internal structure |
| Flaky tests (timing, order-dependent) | Erode trust in the test suite | Use deterministic assertions, isolate test state |
| Testing framework code | Wastes time testing third-party behavior | Only test YOUR code |
| Snapshot abuse | Large snapshots nobody reviews, break on any change | Use snapshots sparingly and review every change |
| No test isolation | Tests pass individually but fail together | Each test sets up and tears down its own state |
| Mocking everything | Tests pass but production breaks | Prefer real implementations > fakes > stubs > mocks. Mock only at boundaries where real deps are slow or non-deterministic |

## When to Use Subagents for Testing

For complex bug fixes, spawn a subagent to write the reproduction test:

```
Main agent: "Spawn a subagent to write a test that reproduces this bug:
[bug description]. The test should fail with the current code."

Subagent: Writes the reproduction test

Main agent: Verifies the test fails, then implements the fix,
then verifies the test passes.
```

This separation ensures the test is written without knowledge of the fix, making it more robust.

## Common Rationalizations

| Rationalization | Reality |
|---|---|
| "I'll write tests after the code works" | You won't. And tests written after the fact test implementation, not behavior. |
| "This is too simple to test" | Simple code gets complicated. The test documents the expected behavior. |
| "Tests slow me down" | Tests slow you down now. They speed you up every time you change the code later. |
| "I tested it manually" | Manual testing doesn't persist. Tomorrow's change might break it with no way to know. |
| "The code is self-explanatory" | Tests ARE the specification. They document what the code should do, not what it does. |
| "It's just a prototype" | Prototypes become production code. Tests from day one prevent the "test debt" crisis. |

## Red Flags

- Writing code without any corresponding tests
- Tests that pass on the first run (they may not be testing what you think)
- "All tests pass" but no tests were actually run
- Bug fixes without reproduction tests
- Tests that test framework behavior instead of application behavior
- Test names that don't describe the expected behavior
- Skipping tests to make the suite pass

## Verification

After completing any implementation:

- [ ] Every new behavior has a corresponding test
- [ ] All tests pass: `go test ./...`
- [ ] Bug fixes include a reproduction test that failed before the fix
- [ ] Test names describe the behavior being verified
- [ ] No tests were skipped or disabled
- [ ] Coverage hasn't decreased (if tracked)

