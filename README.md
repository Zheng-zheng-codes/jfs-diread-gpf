# general protection fault in diRead (fs/jfs/jfs_imap.c:342, v7.4-rc3)

Syzkaller-found JFS general protection fault on Linux v7.4-rc3, reproduced
and verified by the KernSight state-primed pipeline (fresh-VM 2/2).

## Summary

A general protection fault / KASAN null-ptr-deref occurs while reading a
JFS disk inode during lookup:

```
Oops: general protection fault, probably for non-canonical address 0xdffffc0000000006
KASAN: null-ptr-deref in range [0x0000000000000030-0x0000000000000037]
RIP: 0010:diRead+0x526/0xc40 fs/jfs/jfs_imap.c:342
```

The faulting source line is:

```c
agno = BLKTOAG(agstart, JFS_SBI(ip->i_sb));
```

`BLKTOAG()` is:

```c
#define BLKTOAG(b, sbi) ((b) >> ((sbi)->bmap->db_agl2size))
```

The faulting address/offset (0x34) is consistent with dereferencing a NULL
`sbi->bmap` in this path.

Call trace:

```
diRead+0x526/0xc40 fs/jfs/jfs_imap.c:342
 jfs_iget+0xa0/0x5b0 fs/jfs/inode.c:35
  jfs_lookup+0x27f/0x340 fs/jfs/namei.c:1472
   lookup_open+0xb0f/0x1770 fs/namei.c:4536
    path_openat+0xede/0x2240 fs/namei.c:5013
     do_sys_openat2+0x13e/0x1d0 fs/open.c:1443
      __x64_sys_openat+0x144/0x200
       do_syscall_64
```

This is a crafted JFS image leading `diRead()` to use an aggregate-map
pointer that is not initialized/NULL. The exact image condition needs
maintainer analysis.

## Verification

- Pipeline attempts 1-2: budget exhausted
- Pipeline attempts 3-4: not reproduced
- Pipeline attempt 5: `confirmed=true`, `status=verified`
- Time: `2026-11-18T00:13:39Z`
- Reproduced during crash-log replay
- Fresh-VM verification: **2/2** (main pipeline)
- Independent external fresh-VM run: **1 hit**, 76 s
  - `external-verify/serial-1.log`
- 215 replay programs, 0 priming programs
- The verified pipeline path is the exported state snapshot; the fresh-VM
  recipe was not deterministic in every external attempt
- Evidence: `reproducer/repro.outcomes.jsonl`, `reproducer/repro.stats`,
  `reproducer/repro.report`, `external-verify/`

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc3 with CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y,
CONFIG_JFS_FS=y, booted with `oops=panic` and `panic_on_warn=1`. See
`environment.txt` and `kernel.config`.

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 \
    -executor ./syz-executor reproducer/repro.kernsight.prog
```

`-repeat=0` repeats until the fault fires. The recipe has 216 entries and is
self-contained; an independent fresh-VM run hit in 76 s.

## Files

- `description`, `environment.txt`, `kernel.config`
- `reproducer/repro.kernsight.prog` — 216-entry state recipe
- `reproducer/repro.priming.prog`, `reproducer/repro.prog`
- `reproducer/repro.report`, `repro.stats`, `repro.outcomes.jsonl`,
  `repro.context.json`
- `logs/log-0.txt`, `logs/report-0.txt`, `logs/machineInfo-0.txt`,
  `logs/title-stat.json`
- `external-verify/` — independent fresh-VM run, serial log and timing

## Notes

- Crash title: `general protection fault in diRead`
- Subsystem: JFS inode map / `jfs_lookup` -> `jfs_iget` -> `diRead`
- The fault is at `BLKTOAG(agstart, JFS_SBI(ip->i_sb))`, consistent with a
  NULL `sbi->bmap` on this crafted-image path.
