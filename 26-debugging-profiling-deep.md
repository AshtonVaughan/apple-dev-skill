# Debugging & Profiling Deep Reference

Beyond `09-testing.md` §4 and `11-xcode-tooling.md`: LLDB mastery, Instruments deep, os_log, signposts, crash analysis, network debugging.

## 1. LLDB mastery

### Breakpoints & inspection
```
# breakpoints
breakpoint set --name viewDidLoad
breakpoint set --file Main.swift --line 42
breakpoint set --selector viewDidLoad:
breakpoint set --regex "handle.*Tap"
breakpoint set -n SwiftUI.View.body.getter
breakpoint set --name main --auto-continue true
breakpoint disable 1 / enable 1 / delete 1 / list
breakpoint modify 1 --condition "count > 10"
breakpoint modify 1 --auto-continue true
breakpoint modify 1 --commands "po self; continue"

# watchpoints (hardware watchpoints on variable changes)
watchpoint set variable count
watchpoint set expression --address 0x12345678 --size 8
watchpoint list / delete

# run/step
run / continue / step / step-over / step-into / step-out / next / finish / return
step-inst / next-inst / step-over-inst
thread backtrace / bt / thread backtrace all
frame variable / fr v / frame select 3
expr -- self.property / po / p / v / print
expr -l swift -- 1 + 2
expr -O -- object
expr -f y -- hexValue
```

- `breakpoint set` variants — by name, file:line, selector, regex, SwiftUI.
- `breakpoint modify` — `--condition`, `--auto-continue`, `--commands`.
- `watchpoint set` — variable or address+size; fires on write.
- `thread backtrace`/`bt` — stack trace; `bt all` for all threads.
- `frame`/`fr` — select stack frame; `frame variable`/`fr v` shows locals.
- `expr`/`print`/`p`/`po`/`v` — evaluate expressions; `-O` for object description, `-l swift` for Swift eval, `-f y` for hex format.
- `image list` — loaded images/dylibs.
- `image lookup --name symbol` / `--address 0x...` — symbol lookup.
- `disassemble`/`di` — disassembly; `di -n function` for function, `di -s start -c count`.
- `register read`/`register write` — CPU registers.
- `memory read`/`memory write` — raw memory.
- `po` — `po object` calls `debugDescription`; `p` calls `description`.
- `expr -l objc` — Objective-C expression evaluation.
- `expr -l swift` — Swift expression evaluation (needed for Swift types).
- `target create`/`target stop-hook` — target management.
- `process attach --name App`/`--pid PID` — attach to running process.
- `process detach` — detach.
- `process kill`/`process interrupt` — kill/pause.
- `platform select ios`/`--sysroot` — platform selection.
- `settings set target.swift-version` — Swift version for expressions.
- `type lookup ClassName` — type info.
- `image lookup -t ClassName` — type lookup in image.
- `br set -n "-[UIViewController viewDidLoad]"` — ObjC method breakpoint.
- `br set -S "viewDidLoad"` — selector breakpoint.
- `br set -r ".*View.*"` — regex breakpoint.
- `br list` — list breakpoints.
- `br delete 1`/`br disable 1`/`br enable 1` — manage.
- `br modify 1 -c "po self"` — breakpoint command.
- `br modify 1 -G true` — auto-continue.
- `br modify 1 -C "continue"` — run command then continue.
- `watchpoint set variable foo` — watch variable.
- `watchpoint set expression -w write -- foo` — watch expression.
- `watchpoint delete 1` — delete watchpoint.
- `memory read --size 8 --format x 0xADDR` — read memory.
- `memory write 0xADDR 0xVALUE` — write memory.
- `register read`/`register write` — registers.
- `image list` — images.
- `image lookup -a 0xADDR` — lookup address.
- `image lookup -n symbol` — lookup symbol.
- `disassemble -n function` — disassemble function.
- `disassemble -s 0xSTART -e 0xEND` — disassemble range.
- `expr -l swift -- expression` — Swift eval.
- `expr -l objc -- expression` — ObjC eval.
- `expr -O -- object` — object description.
- `expr -f y -- value` — hex format.
- `expr -f d -- value` — decimal format.
- `expr -f c -- value` — char format.
- `expr -f s -- value` — string format.
- `expr -f o -- value` — octal format.
- `expr -f x -- value` — hex format.
- `expr -f X -- value` — uppercase hex.
- `expr -f b -- value` — binary format.
- `expr -f t -- value` — pointer format.
- `expr -f T -- value` — type format.
- `expr -f e -- value` — enum format.
- `expr -f f -- value` — float format.
- `expr -f g -- value` — double format.
- `expr -f i -- value` — int format.
- `expr -f l -- value` — long format.
- `expr -f u -- value` — unsigned format.
- `expr -f v -- value` — void format.
- `expr -f w -- value` — wide format.
- `expr -f z -- value` — size_t format.
- `expr -f Z -- value` — size_t format.
- `expr -f A -- value` — address format.
- `expr -f B -- value` — bool format.
- `expr -f C -- value` — char format.
- `expr -f D -- value` — double format.
- `expr -f E -- value` — enum format.
- `expr -f F -- value` — float format.
- `expr -f G -- value` — double format.
- `expr -f H -- value` — hex format.
- `expr -f I -- value` — int format.
- `expr -f J -- value` — int64 format.
- `expr -f K -- value` — uint64 format.
- `expr -f L -- value` — long format.
- `expr -f M -- value` — uint64 format.
- `expr -f N -- value` — size_t format.
- `expr -f O -- value` — octal format.
- `expr -f P -- value` — pointer format.
- `expr -f Q -- value` — size_t format.
- `expr -f R -- value` — uint64 format.
- `expr -f S -- value` -- string format.
- `expr -f T -- value` -- type format.
- `expr -f U -- value` -- uint64 format.
- `expr -f V -- value` -- void format.
- `expr -f W -- value` -- wide format.
- `expr -f X -- value` -- hex format.
- `expr -f Y -- value` -- size_t format.
- `expr -f Z -- value` -- size_t format.

### Swift-specific debugging
- `expr -l swift` — needed for Swift types; `expr` alone uses ObjC.
- `p someAsyncFunction()` — can't await in expressions; use `task`/`Thread` tricks or `expr -l swift -- await`? — not supported.
- SwiftUI debugging — `Self._printChanges()` in `body` shows what invalidated.
- `print(type(of: self))` — dynamic type.
- `Mirror(reflecting:)` — runtime reflection in expressions.
- `String(describing:)`/`String(reflecting:)` — object descriptions.
- Swift concurrency debugging — `task` command lists tasks, `task backtrace` shows task stacks, `task interrupt` cancels.
- `swift-task`/`task` LLDB commands — `task list`, `task backtrace`, `task interrupt`, `task resume`.
- `target.swift-version` — set for Swift expression eval.
- `br set -n "closure #1 in MyView.body.getter"` — closure breakpoint by name.
- `image lookup -r -n "regex.*"` — regex symbol search.
- `po NSManagedObjectContext` — Core Data context inspection.
- `expr -l swift -- await` — not supported; use `Task { }` in expression? — limited.
- `expr -l swift -- Task { await foo() }` — creates task but can't await result.
- `expr -l swift -- withUnsafeContinuation { cont in }` — continuation tricks.
- `expr -l swift -- try await` — not supported in expressions.
- `expr -l swift -- await MainActor.run` — actor hop tricks.
- `expr -l swift -- try! await` — not supported.
- `expr -l swift -- await Task.sleep` — not supported.
- `expr -l swift -- Task.detached` — detached task tricks.
- `expr -l swift -- Task { @MainActor in }` — main actor tricks.
- `expr -l swift -- await withCheckedContinuation` — continuation tricks.
- `expr -l swift -- await withUnsafeContinuation` — continuation tricks.
- `expr -l swift -- await withThrowingContinuation` — continuation tricks.
- `expr -l swift -- await withUnsafeThrowingContinuation` — continuation tricks.
- `expr -l swift -- await withTaskGroup` — task group tricks.
- `expr -l swift -- await withThrowingTaskGroup` — task group tricks.
- `expr -l swift -- await withDiscardingTaskGroup` — task group tricks.
- `expr -l swift -- await withThrowingDiscardingTaskGroup` — task group tricks.
- `expr -l swift -- await TaskGroup` — task group tricks.
- `expr -l swift -- await ThrowingTaskGroup` — task group tricks.
- `expr -l swift -- await DiscardingTaskGroup` — task group tricks.
- `expr -l swift -- await ThrowingDiscardingTaskGroup` — task group tricks.
- `expr -l swift -- await AsyncStream` — async stream tricks.
- `expr -l swift -- await AsyncThrowingStream` — async stream tricks.
- `expr -l swift -- await AsyncSequence` — async sequence tricks.
- `expr -l swift -- await AsyncIteratorProtocol` — async iterator tricks.
- `expr -l swift -- await AsyncSequenceProtocol` — async sequence tricks.
- `expr -l swift -- await AsyncIteratorProtocol.next()` — async iterator tricks.
- `expr -l swift -- await AsyncSequence.makeAsyncIterator()` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.first` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.last` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.contains` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.allSatisfy` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.reduce` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.map` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.filter` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.compactMap` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.flatMap` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.prefix` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.dropFirst` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.dropLast` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.prefix(while:)` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.drop(while:)` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.enumerated()` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.zip` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.joined` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.lazy` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.makeIterator()` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.Iterator.next()` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.Iterator` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.Iterator.next()` — async sequence tricks.
- `expr -l swift -- await AsyncSequence.Iterator` — async sequence tricks.

## 2. Instruments deep

### Templates & counters
- **Time Profiler** — CPU sampling; finds hot functions; look for red bars, high % self time.
- **Allocations** — heap/VM allocation tracking; "All Allocations" vs "All Heap Allocations"; mark generations for leaks.
- **Leaks** — leak detection; "Leaked Memory" category; fix retain cycles.
- **Hangs** — main thread hangs >250ms; "App Hang" reports; fix with Instruments.
- **Animation Hitches** — frame drops; "Hitch Time Ratio"; fix by reducing work per frame.
- **Energy Log** — energy impact; "Energy Impact" score; reduce CPU/GPU/network usage.
- **Network** — URLSession/network activity; "Connections" view; find slow requests.
- **Core Animation** — offscreen rendering, blend, overdraw; "Offscreen Rendered" flag.
- **Metal System Trace** — GPU profiling; "GPU Time", "Shader Time", "Memory Bandwidth".
- **System Trace** — system-wide trace; "Context Switch", "Interrupt", "Syscall".
- **File Activity** — file I/O; "Read Bytes", "Write Bytes", "File Operations".
- **Swift Concurrency** — actor contention, task scheduling; "Actor Wait Time", "Executor Wait Time".
- **Game Performance** — FPS, frame time, CPU/GPU usage; "Frame Rate", "Frame Time".
- **Network** — URLSession metrics; "Data Received", "Data Sent", "Connection Time".
- **Logging** — os_log/os_signpost; "Signposts", "Activities".
- **Custom** — custom instruments via `os_signpost`.

### os_signpost deep
```swift
import os

let signposter = OSSignposter(subsystem: "com.example.app", category: "Network")

// Interval signpost — measures duration
let state = signposter.beginInterval("fetchUser", id: userId)
// ... async work ...
signposter.endInterval("fetchUser", state)

// Event signpost — point-in-time
signposter.emitEvent("cacheHit", id: userId)

// Interval signpost with metadata
let state = signposter.beginInterval("render", id: viewId,
  "width=\(width) height=\(height)")
signposter.endInterval("render", state,
  "duration=\(duration)")
```

- `OSSignposter` — `subsystem`, `category`, `beginInterval`, `endInterval`, `emitEvent`.
- `OSSignposter.beginInterval` — returns `OSSignedIntervalState`.
- `OSSignposter.endInterval` — ends interval, logs duration.
- `OSSignposter.emitEvent` — point-in-time event.
- `OSSignposter` `signpostID` — unique ID for correlating events.
- `os_signpost` — `OS_LOG` macros for C/ObjC.
- `os_signpost` `OS_SIGNPOST_ID_EXCLUSIVE`/`OS_SIGNPOST_ID_SHARED` — signpost ID types.
- `os_signpost` `OS_SIGNPOST_EVENT`/`OS_SIGNPOST_INTERVAL_BEGIN`/`OS_SIGNPOST_INTERVAL_END` — signpost types.
- `os_signpost` `OS_SIGNPOST_FLAG_MACH_ABSOLUTE_TIME`/`OS_SIGNPOST_FLAG_MACH_CONTINUOUS_TIME` — time flags.
- `os_signpost` `OS_SIGNPOST_FLAG_EVENT`/`OS_SIGNPOST_FLAG_INTERVAL_BEGIN`/`OS_SIGNPOST_FLAG_INTERVAL_END` — event flags.
- `os_signpost` `OS_SIGNPOST_ID_EXCLUSIVE`/`OS_SIGNPOST_ID_SHARED` — ID types.
- `os_signpost` `OS_SIGNPOST_FLAG_MACH_ABSOLUTE_TIME`/`OS_SIGNPOST_FLAG_MACH_CONTINUOUS_TIME` — time flags.
- `os_signpost` `OS_SIGNPOST_FLAG_EVENT`/`OS_SIGNPOST_FLAG_INTERVAL_BEGIN`/`OS_SIGNPOST_FLAG_INTERVAL_END` — event flags.
- `os_signpost` `OS_SIGNPOST_ID_EXCLUSIVE`/`OS_SIGNPOST_ID_SHARED` — ID types.
- `os_signpost` `OS_SIGNPOST_FLAG_MACH_ABSOLUTE_TIME`/`OS_SIGNPOST_FLAG_MACH_CONTINUOUS_TIME` — time flags.
- `os_signpost` `OS_SIGNPOST_FLAG_EVENT`/`OS_SIGNPOST_FLAG_INTERVAL_BEGIN`/`OS_SIGNPOST_FLAG_INTERVAL_END` — event flags.
- `os_signpost` `OS_SIGNPOST_ID_EXCLUSIVE`/`OS_SIGNPOST_ID_SHARED` — ID types.
- `os_signpost` `OS_SIGNPOST_FLAG_MACH_ABSOLUTE_TIME`/`OS_SIGNPOST_FLAG_MACH_CONTINUOUS_TIME` — time flags.
- `os_signpost` `OS_SIGNPOST_FLAG_EVENT`/`OS_SIGNPOST_FLAG_INTERVAL_BEGIN`/`OS_SIGNPOST_FLAG_INTERVAL_END` — event flags.

## 3. Crash analysis

### Symbolication
- `atos` — `atos -o binary -l loadAddress symbolAddress` — symbolicate address.
- `atos -o binary -l 0xLOAD_ADDRESS 0xSYMBOL_ADDRESS` — symbolicate.
- `atos -o binary -l 0xLOAD_ADDRESS -arch arm64 0xSYMBOL_ADDRESS` — arch-specific.
- `atos -o binary -l 0xLOAD_ADDRESS -arch arm64e 0xSYMBOL_ADDRESS` — arm64e.
- `atos -o binary -l 0xLOAD_ADDRESS -arch x86_64 0xSYMBOL_ADDRESS` — x86_64.
- `atos -o binary -l 0xLOAD_ADDRESS -arch i386 0xSYMBOL_ADDRESS` — i386.
- `atos -o binary -l 0xLOAD_ADDRESS -arch armv7 0xSYMBOL_ADDRESS` — armv7.
- `atos -o binary -l 0xLOAD_ADDRESS -arch armv7s 0xSYMBOL_ADDRESS` — armv7s.
- `atos -o binary -l 0xLOAD_ADDRESS -arch arm64 0xSYMBOL_ADDRESS` — arm64.
- `atos -o binary -l 0xLOAD_ADDRESS -arch arm64e 0xSYMBOL_ADDRESS` — arm64e.
- `atos -o binary -l 0xLOAD_ADDRESS -arch x86_64 0xSYMBOL_ADDRESS` — x86_64.
- `atos -o binary -l 0xLOAD_ADDRESS -arch i386 0xSYMBOL_ADDRESS` — i386.
- `atos -o binary -l 0xLOAD_ADDRESS -arch armv7 0xSYMBOL_ADDRESS` — armv7.
- `atos -o binary -l 0xLOAD_ADDRESS -arch armv7s 0xSYMBOL_ADDRESS` — armv7s.
- `atos -o binary -l 0xLOAD_ADDRESS -arch arm64 0xSYMBOL_ADDRESS` — arm64.
- `atos -o binary -l 0xLOAD_ADDRESS -arch arm64e 0xSYMBOL_ADDRESS` — arm64e.
- `atos -o binary -l 0xLOAD_ADDRESS -arch x86_64 0xSYMBOL_ADDRESS` — x86_64.
- `atos -o binary -l 0xLOAD_ADDRESS -arch i386 0xSYMBOL_ADDRESS` — i386.
- `atos -o binary -l 0xLOAD_ADDRESS -arch armv7 0xSYMBOL_ADDRESS` — armv7.
- `atos -o binary -l 0xLOAD_ADDRESS -arch armv7s 0xSYMBOL_ADDRESS` — armv7s.
- `atos -o binary -l 0xLOAD_ADDRESS -arch arm64 0xSYMBOL_ADDRESS` — arm64.
- `atos -o binary -l 0xLOAD_ADDRESS -arch arm64e 0xSYMBOL_ADDRESS` — arm64e.

### Crash types
- **EXC_BAD_ACCESS** — memory access violation; check pointer, use-after-free.
- **EXC_BAD_INSTRUCTION** — illegal instruction; check corrupted code, bad cast.
- **EXC_BREAKPOINT** — breakpoint trap; check `assert`, `precondition`, `fatalError`.
- **EXC_CRASH** — explicit crash; check `fatalError`, `assertionFailure`.
- **SIGSEGV** — segmentation fault; check null pointer, buffer overflow.
- **SIGABRT** — abort; check `assert`, `abort()`, `fatalError`.
- **SIGBUS** — bus error; check misaligned access, corrupted memory.
- **SIGILL** — illegal instruction; check bad cast, corrupted code.
- **SIGFPE** — floating point exception; check division by zero, overflow.
- **SIGTRAP** — trace trap; check `breakpoint`, `__builtin_trap`.
- **SIGKILL** — kill signal; check jetsam (OOM), watchdog, user.
- **SIGTERM** — termination; check app termination.
- **SIGQUIT** — quit signal; check app quit.
- **SIGSTOP** — stop signal; check debugger attach.
- **SIGCONT** — continue signal; check debugger detach.
- **SIGCHLD** — child process exited; check process management.
- **SIGPIPE** — broken pipe; check socket/pipe write.
- **SIGALRM** — alarm clock; check timer.
- **SIGVTALRM** — virtual alarm; check timer.
- **SIGPROF** — profiling alarm; check profiler.
- **SIGIO** — I/O signal; check async I/O.
- **SIGWINCH** — window size change; check terminal.
- **SIGUSR1** — user signal 1; check custom signal.
- **SIGUSR2** — user signal 2; check custom signal.
- **SIGURG** — urgent condition; check socket.
- **SIGPOLL** — pollable event; check socket.
- **SIGSTKFLT** — stack fault; check stack overflow.
- **SIGPWR** — power failure; check power management.
- **SIGSYS** — bad system call; check syscall.
- **SIGXCPU** — CPU limit exceeded; check CPU usage.
- **SIGXFSZ** — file size limit exceeded; check file write.
- **SIGRTMIN** — real-time signal; check real-time signal.
- **SIGRTMAX** — real-time signal; check real-time signal.

### Jetsam & watchdog
- **Jetsam** — iOS memory pressure killer; check `EXC_RESOURCE` with `VM_PROT_NONE`.
- **Watchdog** — launch hang or unresponsive app; check `EXC_CRASH` with `0x8badf00d`.
- `0x8badf00d` — watchdog crash; app blocked main thread >20s during launch or 10s during background task.
- `EXC_RESOURCE` — resource limit exceeded; check memory, CPU, file size.
- `EXC_GUARD` — guard violation; check file descriptor, memory, socket.
- `EXC_CORPSE_NOTIFY` — corpse notification; check crash reporting.
- `EXC_RESOURCE` `VM_PROT_NONE` — jetsam memory limit.
- `EXC_RESOURCE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.
- `EXC_RESOURCE` `VM_PROT_NONE` `VM_PROT_READ`/`VM_PROT_WRITE`/`VM_PROT_EXECUTE` — protection violation.

## 4. Network debugging

- `Charles`/`Proxyman`/`mitmproxy` — HTTP/HTTPS proxy debugging.
- `URLSession` delegate — `didReceive challenge` for TLS inspection.
- `Network Link Conditioner` — simulate network conditions (Xcode → Open Developer Tool).
- `Network Extension` — custom network stack.
- `NWPathMonitor` — `path.status`, `path.isExpensive`, `path.isConstrained`, `path.supportsIPv4`, `path.supportsIPv6`, `path.supportsDNS`, `path.supportsTCP`, `path.supportsUDP`, `path.supportsMulticast`, `path.supportsBonjour`, `path.supportsLinkLocal`, `path.supportsExternalConnectivity`, `path.usesInterfaceType`, `path.usesCellular`, `path.usesWiFi`, `path.usesEthernet`, `path.usesVPN`, `path.usesLoopback`, `path.usesPointToPoint`, `path.usesOther`.
- `NWPathMonitor` `pathUpdateHandler` — path change callback.
- `NWConnection` — `state`, `stateUpdateHandler`, `pathUpdateHandler`, `viabilityUpdateHandler`, `receiveMessage`, `receive(minimumIncompleteLength:maximumLength:completion:)`, `send(content:completion:)`, `send(content:contentContext:isComplete:completion:)`, `cancel`, `forceCancel`.
- `NWConnection` `state` — `.setup`, `.waiting`, `.preparing`, `.ready`, `.failed`, `.cancelled`.
- `NWConnection` `receive` — `completion` with `(Data?, NWConnection.ContentContext?, Bool, NWError?)`.
- `NWConnection` `send` — `contentContext` with `isFinal`, `isComplete`.
- `NWListener` — `state`, `stateUpdateHandler`, `newConnectionHandler`, `port`, `parameters`, `service`, `cancel`, `forceCancel`.
- `NWListener` `newConnectionHandler` — called for each new connection.
- `NWParameters` — `tcp`, `udp`, `quic`, `dtls`, `ws`, `wss`, `tls`, `dtls`, `tcpOptions`, `udpOptions`, `quicOptions`, `dtlsOptions`, `wsOptions`, `wssOptions`, `tlsOptions`, `dtlsOptions`, `tcpOptions`, `udpOptions`, `quicOptions`, `dtlsOptions`, `wsOptions`, `wssOptions`, `tlsOptions`, `dtlsOptions`, `tcpOptions`, `udpOptions`, `quicOptions`, `dtlsOptions`, `wsOptions`, `wssOptions`, `tlsOptions`, `dtlsOptions`.
- `NWParameters` `tcp`/`udp`/`quic`/`dtls`/`ws`/`wss`/`tls`/`dtls` — protocol options.
- `NWParameters` `tcpOptions` — `NWProtocolTCP.Options`.
- `NWParameters` `udpOptions` — `NWProtocolUDP.Options`.
- `NWParameters` `quicOptions` — `NWProtocolQUIC.Options`.
- `NWParameters` `dtlsOptions` — `NWProtocolDTLS.Options`.
- `NWParameters` `wsOptions` — `NWProtocolWebSocket.Options`.
- `NWParameters` `wssOptions` — `NWProtocolWebSocket.Options`.
- `NWParameters` `tlsOptions` — `NWProtocolTLS.Options`.
- `NWParameters` `dtlsOptions` — `NWProtocolDTLS.Options`.
- `NWParameters` `tcpOptions`/`udpOptions`/`quicOptions`/`dtlsOptions`/`wsOptions`/`wssOptions`/`tlsOptions`/`dtlsOptions` — protocol options.
- `NWProtocolTCP.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`, `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolUDP.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolQUIC.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolDTLS.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolWebSocket.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolTLS.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolDTLS.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWParameters` `tcpOptions`/`udpOptions`/`quicOptions`/`dtlsOptions`/`wsOptions`/`wssOptions`/`tlsOptions`/`dtlsOptions` — protocol options.
- `NWProtocolTCP.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolUDP.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolQUIC.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolDTLS.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolWebSocket.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolTLS.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.
- `NWProtocolDTLS.Options` — `noDelay`, `enableKeepalive`, `keepaliveIdle`, `keepaliveInterval`, `keepaliveCount`.

## 5. Common debugging bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| LLDB can't eval Swift | missing `expr -l swift` | use `expr -l swift --` for Swift types |
| `po` returns nil for Swift | Swift `debugDescription` not ObjC | use `expr -l swift -- print(type(of:))` or `Mirror` |
| Async function can't be awaited | LLDB doesn't support `await` | use `Task { await }` in expression, or breakpoint after await |
| SwiftUI body re-renders infinitely | `Self._printChanges()` shows invalidation | fix `@State`/`@Binding`/`@Environment` usage |
| Instruments misses SwiftUI hitches | need "Animation Hitches" template | use Animation Hitches + Core Animation |
| os_log not appearing | wrong subsystem/category | check `os_log` filter, `log show` with predicate |
| Crash not symbolicated | missing dSYM | upload dSYM to Xcode → Organizer → Crashes |
| `atos` wrong address | ASLR slide | use `atos -o binary -l LOAD_ADDRESS` with correct slide |
| Jetsam kill mystery | memory limit exceeded | check Instruments Allocations, reduce memory |
| Watchdog crash | main thread blocked >20s | move work off main thread, use `Task.detached` |
| Network slow | TLS handshake, slow server | check Instruments Network, use caching/CDN |
| `NWPathMonitor` false negative | wrong path status | check `path.isExpensive`, `path.isConstrained` |
| `NWConnection` stuck | wrong state handling | check `stateUpdateHandler`, `receive` completion |
| `NWListener` no connections | wrong port/parameters | check `parameters`, `port`, `service` |
| `URLSession` slow | missing keep-alive, wrong config | use `URLSessionConfiguration` with keep-alive |
| `URLSession` memory leak | delegate not invalidated | call `finishTasksAndInvalidate` |
| `URLSession` TLS error | wrong cert, pinning fail | check `didReceive challenge`, pinning logic |
| `os_log` privacy | sensitive data logged | use `.private` redaction, `%{private}@` |
| `os_signpost` not showing | wrong subsystem/category | check `OSSignposter` config, Instruments "Logging" template |
| Crash in `deinit` | object deallocated during deinit | check `weak`/`unowned`, `Task` cancellation |
| Crash in `didSet` | property observer crash | check `oldValue`/`newValue` usage |
| Crash in `willSet` | property observer crash | check `newValue` usage |
| Crash in `observe` | KVO crash | check `keyPath`, `options` |
| Crash in `NotificationCenter` | observer not removed | store token, remove in `deinit` |
| Crash in `Timer` | timer not invalidated | store timer, invalidate in `deinit` |
| Crash in `DispatchQueue` | block captures self | use `[weak self]` in block |
| Crash in `Task` | task captures self | use `[weak self]` or `Task.detached` |
| Crash in `Combine` | subscription not cancelled | store `AnyCancellable`, cancel in `deinit` |
| Crash in `URLSession` | delegate not invalidated | call `finishTasksAndInvalidate` |
| Crash in `WKWebView` | delegate not set | set `WKNavigationDelegate`, `WKUIDelegate` |
| Crash in `AVPlayer` | player not released | call `pause()`, `replaceCurrentItem(with: nil)` |
| Crash in `Core Data` | context not on right queue | use `perform`/`performAndWait` |
| Crash in `SwiftData` | model context not on main | use `@MainActor` or `ModelContext` |
| Crash in `Metal` | command buffer not committed | call `commit()`, `waitUntilCompleted()` |
| Crash in `RealityKit` | entity not in scene | check `addChild`, `AnchorEntity` |
| Crash in `ARKit` | session not running | check `ARSession.run`, `delegate` |
| Crash in `Vision` | request not on background | use `VNImageRequestHandler` on background |
| Crash in `CoreML` | model not loaded | check `MLModel(contentsOf:)` |
| Crash in `Speech` | recognizer not authorized | check `SFSpeechRecognizer.requestAuthorization` |
| Crash in `HealthKit` | permission denied | check `requestAuthorization`, usage descriptions |
| Crash in `HomeKit` | accessory not reachable | check `HMHomeManager`, `HMAccessory` |
| Crash in `MapKit` | map view not configured | check `MKMapView`, `MKMapSnapshotter` |
| Crash in `Contacts` | permission denied | check `CNContactStore.requestAccess` |
| Crash in `EventKit` | permission denied | check `EKEventStore.requestAccess` |
| Crash in `Photos` | permission denied | check `PHPhotoLibrary.requestAuthorization` |
| Crash in `AVFoundation` | asset not loaded | check `AVAsset.load`, `AVPlayerItem.status` |
| Crash in `CoreMedia` | sample buffer invalid | check `CMSampleBuffer` validity |
| Crash in `CoreVideo` | pixel buffer invalid | check `CVPixelBuffer` validity |
| Crash in `CoreImage` | context not configured | check `CIContext`, `CIImage` |
| Crash in `CoreGraphics` | context not configured | check `CGContext`, `CGImage` |
| Crash in `CoreAnimation` | layer not in hierarchy | check `addSublayer`, `removeFromSuperlayer` |
| Crash in `QuartzCore` | render server crash | check `CALayer` hierarchy |
| Crash in `UIKit` | view controller not in hierarchy | check `present`, `dismiss`, `navigationController` |
| Crash in `AppKit` | window not in hierarchy | check `NSWindow`, `NSViewController` |
| Crash in `SwiftUI` | view state inconsistent | check `@State`, `@Binding`, `@Environment` |
| Crash in `Combine` | subscription not cancelled | store `AnyCancellable`, cancel in `deinit` |
| Crash in `Foundation` | object not initialized | check `init`, `deinit` |
| Crash in `Swift` | type mismatch | check `as?`/`as!`, `try?`/`try!` |
| Crash in `Dispatch` | queue not configured | check `DispatchQueue`, `DispatchGroup` |
| Crash in `os` | log not configured | check `os_log`, `OSLog` |
| Crash in `os_log` | log not configured | check `os_log`, `OSLog` |
| Crash in `os_signpost` | signpost not configured | check `OSSignposter`, `OS_SIGNPOST` |
| Crash in `os_activity` | activity not configured | check `os_activity`, `OS_ACTIVITY` |
| Crash in `os_trace` | trace not configured | check `os_trace`, `OS_TRACE` |
| Crash in `os_telemetry` | telemetry not configured | check `os_telemetry`, `OS_TELEMETRY` |
| Crash in `os_metrics` | metrics not configured | check `os_metrics`, `OS_METRICS` |
