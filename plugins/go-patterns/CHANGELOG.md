# Changelog

## [v0.2.0]

### Added

- `references/` directory with twelve topic files for progressive disclosure
- `references/project-layout.md` — directory structures for library, CLI, and web service archetypes; sub-package vs sub-module depth nuance
- `references/package-design.md` — naming, dependency direction, interface ownership, embedding, semantic getters
- `references/configuration.md` — three-phase loading, ephemeral lifecycle, `Default*Config`/`Merge`/`Finalize` trio
- `references/error-handling.md` — sentinels, typed errors with `Unwrap`, functional options for enrichment, retry classification, `errors.Join` boundaries
- `references/constructors.md` — `New()` shapes, struct configs over functional options, runtime options as `map[string]any`, composition root
- `references/testing.md` — black-box layout, table-driven shape, `httptest.NewServer`, mock packages, `atomic.Int32` assertions
- `references/concurrency.md` — `sync.WaitGroup.Go`, context propagation, lifecycle Coordinator, streaming channels
- `references/modern-idioms.md` — Go 1.21–1.26 features (`maps.Copy`, `slices`, `min`/`max`, `for range n`, range-over-func iterators)
- `references/registry-factory.md` — pluggable extension pattern with thread-safe registry
- `references/ci-cd-release.md` — single-module, multi-module, and marketplace-plugin release workflows; CHANGELOG format
- `references/tooling-claude.md` — canonical Makefile, `.claude/settings.json`, CLAUDE.md modes, `doc.go` conventions
- `references/workspace-multimodule.md` — `go.work` semantics, sub-module decomposition, version coordination, `replace` vs workspace

### Changed

- Rewrote SKILL.md as a scannable principles index (~230 lines) with quick decision table and reference index
- Expanded plugin description triggers to cover CI/CD release workflows, multi-module workspaces, project layout, and registry pattern

## [v0.1.0]
- Initial release as standalone plugin (decomposed from monolithic `tau` plugin v0.1.3)
