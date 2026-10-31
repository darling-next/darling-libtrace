# Libtrace
The library implementing os_log and friends.

`_os_log_debug_impl`, `_os_log_error_impl`, and `_os_log_fault_impl` use the
same six arguments as `_os_log_impl`: DSO, logger, explicit `os_log_type_t`,
format, encoded buffer, and buffer size. They forward the supplied type.
This is the ABI declared by Apple's SDK `os/log.h`, including Xcode 26.3;
the specialized symbol name does not remove the type argument.

`os_log_compare_enablement(lhs, rhs)` compares log **types**, independently of
logger configuration. It returns the difference of their ranks in the order
DEBUG, INFO, DEFAULT, ERROR, FAULT; invalid types trap. The implementation is
based on native macOS 26.6.2 ARM64 and x86_64 code and a native valid-type
comparison matrix, rather than the numeric ordering of `os_log_type_t`.

`os_log_get_type(message)` reads the type byte at offset 1 in the opaque logging
hook record. It does not take a logger handle, normalize unknown type bytes, or
validate the record pointer. This layout is observed in both native architectures
above; the remaining private record layout is intentionally not exposed.

`os_log_set_hook(type, block)` copies a nonnull Block and returns the previous
owned Block (release it with `Block_release` when no longer needed). The first
argument is a type threshold, not a bitmask. Native registration retains the
lower enablement threshold across replacements, except that DEFAULT (zero)
acts as an unset sentinel: ERROR then FAULT still receives ERROR, whereas ERROR
then DEFAULT then FAULT receives only FAULT.

Hooks run synchronously on the emitting thread, without holding the registration
lock. Same-thread recursive logging does not invoke the hook again. The borrowed
record has its type at byte 1 and, for the consumed x86_64 ABI, subsystem and
category pointers at bytes 128 and 136. Default/disabled logger records have null
subsystem/category pointers. Direct `_os_log_impl` calls reach the hook even for
the disabled logger; ordinary enabled checks still suppress disabled logging.

`os_log_copy_message_string(record)` returns a separately malloc-owned string.
It uses the existing decoder with privacy enforcement: explicit private strings
are redacted and explicit public strings are rendered. Default `%s` arguments
are public only when the emitted range satisfies `_dyld_is_memory_immutable`:
dyld-owned, read-only, never-unloaded memory. Writable globals, stack/heap data,
read-only anonymous mappings, and strings in unloadable images remain private.
The existing dyld helper is reused; this is not a memory-permission heuristic.
The decoder skips `{...}` annotations by advancing within the original format,
preserving the conversion character, surrounding literal text, and later
conversions; the encoded argument descriptor remains the privacy authority.
Literal `%%` emits one percent without consuming an encoded argument, including
when adjacent to a string conversion or when the message has no arguments.
The message is snapshotted before invoking client code: mutating an original
argument inside a hook does not change later copies of that same record.
ASL output retains its build-default privacy enforcement switch. When ASL
already enforces privacy, its decoded snapshot is shared with the hook rather
than decoded or allocated twice.
Invocation retains the Block across replacement as an implementation safety
choice, not a native guarantee that callers may release their last reference
and continue accessing captures. Callbacks can register a replacement without
deadlocking. With no hook registered, the logging path adds no allocations,
record construction, decoding, or mutex acquisition.
