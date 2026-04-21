# evepcr

Go library and CLI tools for predicting and validating TPM PCR values
across [EVE OS](https://github.com/lf-edge/eve) updates.

## What this solves

EVE measures its boot chain into TPM Platform Configuration Registers and
records the measurements in an event log. A remote controller can replay
that log to verify the device booted trusted software.

The problem comes after an OS update: the PCR values change (new rootfs,
different partition layout, updated GRUB commands), so the controller
needs to know what the new values will be *before* the device reboots.
This library predicts those values by parsing and splicing event logs,
patching the entries that differ between versions, and replaying the
result.

## API

### Verification and validation

Event log verification replays the log against TPM PCR values to confirm
integrity. Validation runs security rules (e.g. checking for disabled DMA
protection or debug mode) without needing PCR values.

```go
// Verify event log integrity against PCR values.
func VerifyEventLogFromBytes(eventLogBytes []byte, pcrValues map[int][]byte) ([]attest.Event, error)
func VerifyEventLogFromFile(eventLogFile, pcrYaml string) error

// Run security validation rules on the event log (no PCR values needed).
func ValidateEventLogFromBytes(eventLogBytes []byte) ([]attest.Event, error)
func ValidateEventLogFromFile(eventLogFile string) error
```

### PCR prediction

Predict what TPM PCR values will look like after an OS update by splicing
and patching event logs.

```go
// Predict PCRs by diffing a baseline and incoming event log.
func PredictPCRs(srcLog, dstLog []byte, rootfsHash []byte) (map[int][][]byte, []byte, error)
func PredictPCRsFromFiles(srcFile, dstFile string, rootfsHash []byte) (map[int][][]byte, []byte, error)

// Predict PCRs from a single baseline log, optionally patching the
// EVE version and overriding individual PCR values.
func PredictPCRsFromBaseline(srcLog []byte, rootfsHash []byte, eveVersion string, pcrOverrides map[int][]byte) (map[int][][]byte, []byte, error)
func PredictPCRsFromBaselineFile(srcFile string, rootfsHash []byte, eveVersion string, pcrOverrides map[int][]byte) (map[int][][]byte, []byte, error)
```

### Rootfs hashing

```go
// Hash a squashfs rootfs image the same way EVEs's GRUB measurefs does
// before extending PCR 13.
func HashRootfsImage(imageBytes []byte) ([]byte, error)
```

## Building

```sh
make build
```

This compiles the library and the CLI tools under `cmd/`. See
[cmd/README.md](cmd/README.md) for a description of each tool.

## Testing

```sh
make test
```

Downloads a rootfs test fixture on first run, then runs `go test ./...`.

End-to-end integration tests live in `test/script/` and boot real EVE images
in QEMU with a software TPM. See [docs/tests.md](docs/tests.md) for
details.
