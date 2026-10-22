# DE-5216 spec review
## Verdict
PASS
## Issues
(none)
## Residual nits (non-blocking)

- §6.6’s #824 test table does not list Lerna GET JSON `not_found` → list 502 (or Lerna GET 3xx). Behavior is already in §6.2.2 and in the additional AC that requires those buckets.
- Cap-hit log message and level are unspecified. §5.8 only needs CI to prove a cap-hit was logged and that the 201st `ORDER BY` row was dropped.
- Bruno §7.3 step 10 “the TMA list handle” is implicit after several mint/DELETE steps (the surviving step-7 row).
- §6.2.2 “Fail list / no token table” is list-page language. Invalidate has no matching table row for dual-read Lerna GET ≠ 200; the §6.3 numbered order plus §4.1 “absent from this merge” already continue to classify and DELETE.
