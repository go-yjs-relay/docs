# Contributing

Contributions are welcome under the org's uniform standard.

## Quality gate

- **Pure Go, `CGO_ENABLED=0`.** No cgo, no external dependencies beyond the
  standard library.
- **100% test coverage under `-race`,** including every error and edge branch —
  double-`Leave` idempotency, room GC on the last leave, drop-on-full-buffer, and
  both arms of `LeaveOnContextDone`. It is enforced as a CI gate.
- **`gofmt` + `go vet` clean.**
- **Green across all six 64-bit Go targets** — amd64, arm64 (native) and riscv64,
  loong64, ppc64le, s390x (under qemu-user).

## Running the suite

```sh
COVERPKG=$(go list ./... | paste -sd, -)
go test -race -coverpkg="$COVERPKG" -coverprofile=cover.out ./...
go tool cover -func=cover.out | tail -1   # 100.0%
```

## Cross-arch check

```sh
for a in amd64 arm64 riscv64 loong64 ppc64le s390x; do
  GOOS=linux GOARCH=$a go build ./... && echo "linux/$a ok"
done
```

## License

BSD-3-Clause. By contributing you agree your work is licensed under it.
Copyright the go-yjs-relay/yrelay authors.
