# Independent external verification

A fresh-VM run was performed outside the main Kernsight pipeline, using the
rc3 kernel, the standard trixie image, and the archived
`repro.kernsight.prog`.

Command:

```sh
syz-execprog -procs=8 -threaded -repeat=0 \
    -executor ./syz-executor reproducer/repro.kernsight.prog
```

Result: target hit after 76 s.

- Run 1: `Oops: general protection fault` at `diRead+0x526/0xc40`
- Serial log: `serial-1.log`

The main pipeline also verified this state-dependent bug from the exported
state snapshot (215 replay programs, 2/2 snapshot verification). The fresh-VM
recipe hit once externally; this report does not claim that every fresh-VM
attempt is deterministic.
