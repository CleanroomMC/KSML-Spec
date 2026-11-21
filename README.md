# Kirino Shader Meta Language (KSML)

KSML is a runtime-agnostic engineering layer for GLSL. Unlike engine-specific shader systems,
it is non-invasive and provides modules, namespaces, and first-class IDE support.

- IntelliJ IDEA Plugin: https://github.com/tttsaurus/ksml-intellij-plugin
- Official KSML Compiler: https://github.com/CleanroomMC/KSML-Compiler

# Changelog

Changes to the Kirino Shader Meta Language (KSML) specification.

## v1 Initial Draft (January 2026)

- Introduced `.ksml` source files, modules, and explicit namespaces.
- Defined `@module`, `@requires`, and `@import` for module declarations and dependency resolution.
- Proposed `@gl_version` and `@gl_requires` for compile-time GLSL version diagnostics.
- Introduced `@export` for controlling function and struct visibility.
- Proposed GLSL macro-based feature handling and source mapping through `#line`.
- Established source transpilation without runtime shader assembly or full GLSL AST analysis.

## v2 Syntax Revision (July 2026)

### Changed
- Changed GLSL imports to the comment-based `// @import` syntax.
- Introduced `@code """ ... """` blocks, each containing one GLSL function definition.
- Moved `@export` and `@gl_requires` to annotations on individual code blocks.
- Replaced the proposed macro-based feature handling with `@feature` annotations controlled by compiler configuration.
- Added GLSL profile requirements to `@gl_version` and `@gl_requires`.
- Clarified transitive dependency resolution and namespace rewriting.

### Removed
- Dropped the proposed export aliases and struct declarations from the defined syntax.

## v2.1 Spec Refinement (November 2026)

### Changed
- Aligned profile resolution with the GLSL specification.
- Removed context-dependent profile inference.
- Required explicit namespace qualification for all KSML function calls, including calls within the same module.
- Defined version requirement compatibility using GLSL version and profile semantics.
- Distinguished module-level version mismatches (`NOTICE`) from function-level mismatches (`WARNING`).

### Added
- Explicit support for multiple `@requires` declarations, with one dependency per line.
- Validation against the supported desktop GLSL version set.
- Profile validation for versions below GLSL 150, where profiles do not exist.
- Support for `compat` as a KSML-only abbreviation for `compatibility`.
- Explicit rules against duplicate `@import` and `@requires` declarations.
- Identifier requirements for feature names.
- Defined diagnostic severity and compilation behavior.
- More formal rules.
