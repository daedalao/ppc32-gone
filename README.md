# go for Arch POWER 32-bit (linux/ppc)

`go/` is a PKGBUILD in the style of the Arch `go` package, plus a 12-patch
series that adds a `GOARCH=ppc` (32-bit big-endian PowerPC) port to upstream
Go. It builds the same way any other Arch POWER package does: upstream
source, patches applied in `prepare()`.

Upstream Go has no 32-bit PowerPC target and the port was written against
Go master (commit a14b5433261d, 2026-09-08), so the PKGBUILD fetches that
commit from go.googlesource.com rather than a release tarball. The patches
apply cleanly only there.

## Building

Cross, from any Arch machine with a working `go` (this is how the package
was produced and tested; op4k, ppc64le, about 10 minutes):

    cd go && CARCH=powerpc makepkg

`make.bash` uses the host go as GOROOT_BOOTSTRAP and builds the ppc32
compiler, linker, tools and standard library; `build()` then lays the tree
out like Go's own `bootstrap.bash` (only the linux/ppc tools and packages
are kept). No separate bootstrap package is needed.

Natively, on a powerpc machine that already has this package installed:

    makepkg

(`GOROOT_BOOTSTRAP=/usr/lib/go`; about 70 minutes on a 1.67 GHz G4.)

`check()` is intentionally absent: `run.bash` cannot run in a cross build,
and natively it takes many hours on a G4. The port was validated by running
the standard library test suites and real programs on a PowerBook G4; see
the notes in the top-level README of the port for details.

## Notes for packagers of Go software on powerpc

- `-buildmode=pie` works (external linking, DT_TEXTREL). cgo works.
  c-shared, c-archive, plugin, `-buildmode=shared` and `-linkshared` work
  (external linking, no text relocations). `go install -buildmode=shared
  std` needs a writable GOROOT (run it as root against /usr/lib/go, or use
  a GOROOT copy).
- Large builds need `GOTMPDIR` and `TMPDIR` on disk if `/tmp` is a small
  tmpfs; the external linker stages the whole program under `TMPDIR`.
- golang.org/x/sys has no linux/ppc support for the gc compiler upstream;
  this toolchain supplies the two missing files automatically (see
  `/usr/lib/go/misc/ppc/xsys`), so modules using x/sys build unchanged.
- Generated code that enumerates GOARCH values may not know `ppc`.
  Example: ncruces/go-sqlite3-wasm (used by charmbracelet/crush) has a
  compile-time endianness table that lacks `ppc`, so both `big` and
  `little` are false and it fails to compile. In the package's
  `prepare()`, vendor and patch it, then build with `-mod=vendor`:

      go mod vendor
      sed -i 's/runtime.GOARCH == "ppc64" || runtime.GOARCH == "s390x" ||/runtime.GOARCH == "ppc" || &/' \
        vendor/github.com/ncruces/go-sqlite3-wasm/v3/sqlite3.go

## Licensing

Packaging files (PKGBUILD, .SRCINFO, this README) are 0BSD, like Arch
Linux's own go package from which the PKGBUILD is derived. The patches
modify and add to the Go distribution and are under Go's BSD-3-Clause
license; the built package installs Go's LICENSE under
/usr/share/licenses/go. See REUSE.toml and LICENSES/.

## Port summary

New GOARCH `ppc`: sys.ArchPPC (family PPC), ABI0 only, R2 = C TLS, R30 = g,
R31 = REGTMP, R12 scratch for linker trampolines. Assembler shares
cmd/internal/obj/ppc64 in a 32-bit mode; new SSA backend
cmd/compile/internal/ppc; new linker cmd/link/internal/ppc (ELF32, RELA,
trampolines, PIE via external link); runtime with the 32-bit Linux ppc
syscall ABI, time64 syscalls, ppc32 signal frames, async preemption, cgo.
No isel/popcnt/fsqrt/lwsync/lbarx are used (the G4 lacks them).
Build modes: exe, pie (absolute code with dynamic text relocations),
c-shared, c-archive, plugin, shared and -linkshared (position independent
code with R13 as the module base register, initial-exec TLS, and
linker-generated PC-relative stubs for calls that bind at load time, since
GNU ld leaves those as 32 MB-limited dynamic REL24 relocations on ppc32).
Known limitations: no vdso, no race/msan/asan (their runtimes are 64-bit
only), no internal-linking PIE.
