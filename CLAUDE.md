# CLAUDE.md

## Project

innix — a Go TUI that scaffolds and manages nix-darwin macOS configuration repos. Built with Charm (Huh v2, BubbleTea v2, Lipgloss v2).

## Repo layout

- `cmd/` — entry point
- `internal/` — all application logic (not a library)
- `templates/` — Nix files embedded via `go:embed`
- `docs/specs/` — design specs

## Commands

```bash
go build ./cmd/...          # build
go test ./...               # test
go run ./cmd/main.go        # run
```

## Conventions

- Go 1.22+
- Use `internal/` for all packages — this is a CLI, not a library
- Error handling: return errors, don't panic
- Templates embedded via `go:embed` in the package that uses them
- Nix files in `templates/` are the source of truth for generated repos
