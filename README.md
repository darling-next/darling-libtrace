# Libtrace
The library implementing os_log and friends.

`os_log_compare_enablement(lhs, rhs)` compares log **types**, independently of
logger configuration. It returns the difference of their ranks in the order
DEBUG, INFO, DEFAULT, ERROR, FAULT; invalid types trap. The implementation is
based on native macOS 26.6.2 ARM64 and x86_64 code and a native valid-type
comparison matrix, rather than the numeric ordering of `os_log_type_t`.
