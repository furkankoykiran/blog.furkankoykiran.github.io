---
title: "My Keploy Contribution: Resource Management in Go and Open Source Journey"
description: "Two bug fix PRs I made to the Keploy project, and what I learned about Go's defer pattern and HTTP connection pooling."
date: 2026-03-20 18:00:00 +0300
categories: [Open Source, Go]
tags: [open-source, go, keploy, defer, resource-management]
image:
  path: /assets/img/posts/2026-03-20-keploy/how-keploy-works.png
  alt: "Keploy Architecture Diagram"
---

While tracking popular repositories on GitHub trending with my [awesome-trending-repos](https://github.com/furkankoykiran/awesome-trending-repos) project, I came across [Keploy](https://github.com/keploy/keploy), a modern API testing tool written in Go. While exploring the codebase, I found a couple of resource management bugs that I could fix.

![Keploy Architecture](/assets/img/posts/2026-03-20-keploy/how-keploy-works.png)
*Keploy is a modern tool for automated testing and mock generation for APIs, Integration, and E2E tests.*

---

## What is Keploy?

Keploy is an open-source tool for API and integration testing. It provides features like mock creation and test generation, and has over 8k stars on GitHub. It integrates with VSCode through an extension.

![Keploy VSCode Extension](/assets/img/posts/2026-03-20-keploy/keploy-vscode-extension.png)
*The VSCode extension allows Keploy to manage tests and mocks directly from the IDE.*

While examining the codebase, I noticed some common resource management mistakes in Go. I fixed these issues with two separate PRs.

---

## PR #3927: The Use-After-Close Bug

### Discovering the Problem

While browsing the project's GitHub issues, I came across [#3821](https://github.com/keploy/keploy/issues/3821), which reported a bug in the `isGoBinary` function in `utils/utils.go`. The file was being opened and immediately closed, then operations were performed on the closed file.

```go
// BUGGY CODE
f, err := elf.Open(filePath)
if err != nil {
    logger.Debug(fmt.Sprintf("failed to open file %s", filePath), zap.Error(err))
    return false
}
if err := f.Close(); err != nil {  // ❌ Closed immediately
    LogError(logger, err, "failed to close file", zap.String("file", filePath))
}
// Then operating on the closed file
sections := []string{".go.buildinfo", ".gopclntab"}
for _, section := range sections {
    if sect := f.Section(section); sect != nil {  // ❌ Use-after-close
        fmt.Println(section)
        return true
    }
}
```

This causes what's known as a "use-after-close" error in Go. After the file is closed, the file descriptor is no longer valid.

### The Solution: Defer Pattern

In Go, the `defer` pattern is used to solve this problem. A statement prefixed with `defer` is executed just before the function returns. This ensures that the resource is cleaned up regardless of how the function exits (early return, panic, or normal return).

```go
// FIXED CODE
f, err := elf.Open(filePath)
if err != nil {
    logger.Debug(fmt.Sprintf("failed to open file %s", filePath), zap.Error(err))
    return false
}
defer func() {
    if err := f.Close(); err != nil {
        LogError(logger, err, "failed to close file", zap.String("file", filePath))
    }
}()
// File is now open, can read sections
sections := []string{".go.buildinfo", ".gopclntab"}
for _, section := range sections {
    if sect := f.Section(section); sect != nil {
        return true
    }
}
```

### Code Review Feedback

I received feedback from the Copilot code reviewer. Using `defer f.Close()` directly would ignore the error returned by `Close()`. This didn't match the pattern used elsewhere in the repository.

I addressed the feedback by wrapping the defer in a closure and adding error handling:

```go
defer func() {
    if err := f.Close(); err != nil {
        LogError(logger, err, "failed to close file", zap.String("file", filePath))
    }
}()
```

[PR #3927](https://github.com/keploy/keploy/pull/3927) was successfully merged.

---

## PR #3932: HTTP Response Body Leak

### Discovering the Problem

I found issue [#3854](https://github.com/keploy/keploy/issues/3854) on the project's tracker, which reported that in `pkg/platform/http/agent.go`, the `MockOutgoing` and `UpdateMockParams` functions were not closing HTTP response bodies after making requests.

```go
// BUGGY CODE
res, err := a.client.Do(req)
if err != nil {
    return fmt.Errorf("failed to send request: %s", err.Error())
}
// Body not closed! ❌

var mockResp models.AgentResp
err = json.NewDecoder(res.Body).Decode(&mockResp)
```

This leads to several problems:

1. **Resource Leak**: Unclosed bodies cause memory leaks
2. **Connection Pool Exhaustion**: The HTTP client's connection pool gets exhausted
3. **Port Exhaustion**: Many open connections can consume system ports

### The Solution: Defer Close + Drain Pattern

Go's `http` package documentation states that the caller is responsible for closing the response body. If the body is not closed, the underlying TCP connection cannot be returned to the connection pool, and a new connection must be created for each request.

```go
// FIXED CODE
res, err := a.client.Do(req)
if err != nil {
    return fmt.Errorf("failed to send request: %s", err.Error())
}
defer func() {
    io.Copy(io.Discard, res.Body)  // Drain body to EOF
    if err := res.Body.Close(); err != nil {
        utils.LogError(a.logger, err, "failed to close response body")
    }
}()

var mockResp models.AgentResp
err = json.NewDecoder(res.Body).Decode(&mockResp)
```

Here we do two important things:

1. **`io.Copy(io.Discard, res.Body)`**: We read and discard the body until EOF. This allows the connection to be reused because the JSON decoder might not have read the entire body.
2. **`res.Body.Close()`**: We close the body and log any errors.

### Why Drain the Body?

An important detail I learned from the Copilot reviewer: `json.Decoder.Decode()` doesn't guarantee reading the entire body. If there are unread bytes in the body and we close it directly, the HTTP client cannot reuse the connection (keep-alive won't work).

By draining the body with `io.Copy(io.Discard, res.Body)` to EOF, the connection can be returned to the pool for reuse.

[PR #3932](https://github.com/keploy/keploy/pull/3932) was successfully merged.

---

## Go Resource Management Best Practices

These two PRs taught me important lessons about resource management in Go:

### 1. Using the Defer Pattern

```go
// File operations
file, err := os.Open("file.txt")
if err != nil {
    return err
}
defer file.Close()

// Mutex locking
mu.Lock()
defer mu.Unlock()

// Database transaction
tx, err := db.Begin()
if err != nil {
    return err
}
defer tx.Rollback()  // Rollback if not committed
```

### 2. Error Handling in Defer

```go
// ❌ Error is ignored
defer file.Close()

// ✅ Error is handled
defer func() {
    if err := file.Close(); err != nil {
        log.Printf("close error: %v", err)
    }
}()
```

### 3. HTTP Connection Reuse

```go
// ❌ Connection cannot be reused
defer res.Body.Close()

// ✅ Connection returns to pool
defer func() {
    io.Copy(io.Discard, res.Body)
    res.Body.Close()
}()
```

### 4. Defer Execution Order

Defer statements execute in LIFO (Last In, First Out) order:

```go
defer fmt.Println("1")  // Last: "1"
defer fmt.Println("2")  // Second: "2"
defer fmt.Println("3")  // First: "3"
// Output: 3, 2, 1
```

---

## What I Learned

From this open source contribution process, I learned:

| Topic | Lesson Learned |
|-------|----------------|
| **Go Defer Pattern** | Using defer for resource cleanup, error handling |
| **HTTP Connection Pooling** | Draining response body, connection reuse |
| **Code Review** | Improving code quality with Copilot feedback |
| **Open Source Communication** | Writing PR descriptions, communicating with maintainers |

The maintainers' code reviews were very educational. I received detailed feedback on code style, edge cases, and error handling.

---

## Conclusion

My two small contributions to the Keploy project gave me practical knowledge about resource management in Go. Subtle bugs like use-after-close and HTTP response leaks can cause major problems in production.

![Keploy Logo](/assets/img/posts/2026-03-20-keploy/keploy-logo.svg)
*Keploy is an open-source project open to contributions.*

Contributing to open source is not just about writing code, it's also about interacting with the community and learning. Every PR adds value to both the project and yourself.

If you want to learn more about resource management in Go, I recommend checking out [Effective Go](https://go.dev/doc/effective_go) and [Common Mistakes in Go](https://github.com/golang/go/wiki/CommonMistakes).

---

**Related Posts:**
- [awesome-trending-repos: GitHub Trending Tracker](/posts/awesome-trending-repos-en/)
- [GitHub MCP Server Project Contribution](/posts/github-mcp-server-acik-kaynak-katki/)
- [My MCP Ecosystem Contributions](/posts/mcp-ai-contributions/)

![Furkan Köykıran](/assets/img/avatar.png)
*Furkan Köykıran - Senior Software Engineer*
