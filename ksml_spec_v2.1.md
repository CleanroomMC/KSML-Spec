# Kirino Shader Meta Language (KSML)

> Nov 2026

Specification Version 2.1

KSML is a runtime-agnostic engineering layer for GLSL. Unlike engine-specific shader systems, 
it is non-invasive and provides modules, namespaces, and first-class IDE support.

IntelliJ IDEA Plugin: https://github.com/tttsaurus/ksml-intellij-plugin
KSML Compiler: https://github.com/CleanroomMC/KSML-Compiler

## 1. Introduction

### 1.1. Scope

This specification defines:
- KSML source format (including `.ksml` itself & GLSL snippets in KSML & KSML snippets in GLSL shaders)
- KSML module
- KSML function visibility (`internal` & `export`)
- GLSL version requirements
- Namespace resolution
- Compiler diagnostics

KSML extends GLSL with module-level declarations and function annotations. 
The compiler resolves these constructs and produces GLSL source code.

Unless otherwise specified, GLSL syntax and semantics follow the corresponding OpenGL Shading Language specification.

### 1.2. Terminology

The terms must, must not, should, and may are used as follows:

- must / must not: Requirements for a conforming source or compiler.
- should: Recommended behavior, with exceptions permitted.
- may: Permitted behavior.

A shader refers to a GLSL source file (`.glsl`, `.vert`, `.frag`, etc.).

A module refers to a KSML source file (`.ksml`) identified by an `@module` declaration.

A code block refers to a GLSL function definition enclosed by an `@code` declaration.

A version requirement consists of a GLSL version number and, where applicable, a profile.

### 1.3. Source Structure

A KSML source file must contain exactly one `@module` declaration and one `@gl_version` declaration.

The `@module` declaration must appear first, excluding comments and whitespace. 
The `@requires` and `@gl_version` declarations must appear before the first code block. 
Their relative order is not significant.

A module may contain any number of `@requires` declarations and code blocks.

Must not contain duplicate `@requires` imports.

Example:
```ksml
@module lighting
@requires math
@requires color
@gl_version 330 core

@export
@code """
float intensity(float x) {
    return math.square(x);
}
"""
```

Function-level annotations apply only to the immediately following `@code` declaration.

## 2. Modules and Dependencies

### 2.1. `@import`

- Owner File Type: `.glsl` / (`.vert` / `.frag` / ...)
- Usage Pattern: `// @import <module_name>`

The `@import` directive imports a KSML module into a GLSL shader file.

Import directives must appear before the GLSL `#version` directive and before any shader declarations.

Example:
```glsl
// @import math
// @import lighting

#version 330 core

void main() {
    float value = math.square(2.0);
}
```

Rules:
- Each `// @import` directive must specify exactly one module.
- Multiple modules must be imported using separate directives.
- The compiler must resolve the dependencies declared by imported modules.
- A shader does not need to explicitly import the transitive dependencies of an imported module unless it directly references their functions.
- A shader must explicitly import a module before calling its functions, even if the module is already included as a transitive dependency.
- Importing an unresolved module is an error.
- The compiler must distinguish import directives from ordinary GLSL comments.
- Must not import duplicate modules explicitly. Transitive ones do not count.

Since `@import` is written as a GLSL comment, it does not alter GLSL's requirements concerning the placement of `#version`.

### 2.2. `@module`

- Owner File Type: `.ksml`
- Usage Pattern: `@module <module_name>`

The `@module` declaration defines the name and namespace of a KSML module.

Example:
```ksml
@module math

```

Rules:
- A module name must match `[A-Za-z_][A-Za-z_0-9]*`.
- Each KSML source file must contain exactly one `@module` declaration.
- Multiple source files may independently declare the same module name, but they must not be included together in one compilation unit.
- Duplicate module names within the same compilation unit are an error.
- The module name identifies the namespace used to reference its functions.

### 2.3. `@requires`

- Owner File Type: `.ksml`
- Usage Pattern: `@requires <module_name>`

The `@requires` declaration specifies a dependency on another KSML module.

Example:
```ksml
@module lighting
@requires math
@requires color
@requires geometry
@gl_version 330 core

```

Rules:
- A module may contain zero or more `@requires` declarations.
- Each declaration must specify exactly one module name.
- Dependencies must be declared on separate lines. A single `@requires` declaration containing multiple names is invalid.
- A module must declare a dependency before referencing functions belonging to that dependency.
- A module does not need to declare a dependency on itself.
- The compiler must resolve dependencies transitively.
- Cyclic dependencies are allowed. The compiler must resolve them without infinite recursion or duplicate module inclusion.
- An unresolved dependency is an error.
- Must not import duplicate modules with `@requires` explicitly. Transitive ones do not count.

## 3. GLSL Versions and Profiles

### 3.1. Supported GLSL Versions

KSML 2.1 recognizes the following desktop GLSL versions:

110, 120, 130, 140, 150,
330, 400, 410, 420, 430, 440, 450, 460

A version number outside this set is invalid.

For example:
```ksml
@gl_version 330 core  // valid
@gl_version 450 core  // valid
@gl_version 160 core  // error
@gl_version 320 core  // error
@gl_version 470 core  // error

```

The compiler must produce an error for unsupported or nonexistent GLSL versions.

This validation applies to:

- GLSL `#version` directives.
- KSML `@gl_version` declarations.
- KSML `@gl_requires` annotations.

OpenGL ES shading language versions are outside the scope of this specification.

### 3.2. GLSL Profiles

KSML follows the profile rules defined by the OpenGL Shading Language specification.

Profiles are available starting with GLSL version 150.

For GLSL versions 140 and earlier, no profile exists. A profile must not be explicitly declared.

For GLSL versions 150 and later, the following profiles are supported:

- `core`
- `compatibility`

When the profile is omitted for version 150 or later, it defaults to `core`.

These rules apply equally to GLSL shaders and KSML declarations.

Examples:
```glsl
#version 140                // valid, no profile
#version 150                // valid, defaults to core
#version 330                // valid, defaults to core
#version 330 core           // valid
#version 330 compatibility  // valid
```

The following are invalid:
```glsl
#version 140 core           // error
#version 140 compatibility  // error
#version 330 compat         // error
```

The effective profile of a shader is determined by its `#version` directive, regardless of the profile of the OpenGL context in which it is running.

A compatibility-profile OpenGL context does not change the default profile of a GLSL shader.

### 3.3. KSML Profile Abbreviation

KSML additionally accepts `compat` as an abbreviation for `compatibility`.

This abbreviation is valid only in KSML declarations and must not be emitted as a GLSL profile identifier.

Example:
```ksml
@gl_version 330 compat

```

This declaration is equivalent to:
```ksml
@gl_version 330 compatibility

```

The compiler must normalize both identifiers to the same internal profile.

A GLSL shader must continue to use the full `compatibility` identifier.

### 3.4. Default Profile Resolution

KSML uses the same default profile rules as GLSL.

For versions below 150, the profile is absent.

For versions 150 and later, an omitted profile is interpreted as `core`.

| Declaration                     | Effective Profile |
|---------------------------------|-------------------|
| `@gl_version 110`               | None              |
| `@gl_version 140`               | None              |
| `@gl_version 150`               | core              |
| `@gl_version 330`               | core              |
| `@gl_version 330 core`          | core              |
| `@gl_version 330 compat`        | compatibility     |
| `@gl_version 330 compatibility` | compatibility     |

The same rules apply to `@gl_requires`.

The compiler must not use the OpenGL context profile or a configurable profile hint to change the meaning of an omitted profile.

For example:
```ksml
@gl_version 450

```

Always represents GLSL 450 core, including when the program is running under a compatibility-profile OpenGL context.

For versions below 150, the absence of a profile must remain distinct from `core` and `compatibility`.
Such versions must not be treated as having a formally declared compatibility profile.

### 3.5. Version Requirement Compatibility

For GLSL versions 150 and later, the compatibility profile is a superset of the core profile at the same version.

A compatibility-profile requirement therefore satisfies a core-profile requirement of the same or an earlier supported version.

The reverse is not generally guaranteed, because functionality retained by the compatibility profile may be unavailable in the core profile.

For example:
```text
GL330 compatibility >= GL330 core
GL450 compatibility >= GL330 core
GL450 compatibility >= GL330 compatibility
GL450 core          >= GL330 core
GL450 core          !>= GL330 compatibility
GL330 core          !>= GL330 compatibility
```

For version requirement checks where both versions are 150 or later, 
a provided requirement satisfies a required requirement if:
1. The provided version number is greater than or equal to the required version number.
2. The required profile is `core`, or the provided profile is `compatibility`.

For versions below 150, there is no profile distinction. 
Version requirements involving these versions are compared by version number only.

This comparison is used for KSML requirement diagnostics. 
It does not guarantee that arbitrary GLSL source written for an older version is valid in every newer profile. 
Some previously available GLSL features may be removed or restricted in later core profiles.

The compiler must not rely on ordinary ordering of `(version, profile)` pairs as a substitute for the compatibility rules above.

### 3.6. `@gl_version`

- Owner File Type: `.ksml`
- Usage Pattern: `@gl_version <version_number> <profile>?`

The `@gl_version` declaration specifies the baseline GLSL version requirement of a KSML module.

Example:
```ksml
@module math
@gl_version 330 core

```

The profile is optional and follows the rules in Sections 3.2 through 3.4.

A module must declare exactly one `@gl_version`.

Rules:
- The version number must be supported.
- For versions below 150, a profile must not be specified.
- For versions 150 and later, an omitted profile defaults to `core`.
- `compat` and `compatibility` are equivalent in KSML.
- The compiler must analyze version compatibility between the module and its dependencies.
- A module-level version mismatch must produce a `NOTICE`, as defined in Section 5.

### 3.7. `@gl_requires`

- Owner File Type: `.ksml`
- Usage Pattern: `@gl_requires <version_number> <profile>?`

The `@gl_requires` annotation overrides the version requirement of the immediately following code block.

Example:
```ksml
@module math
@gl_version 330 core

@gl_requires 450 core
@export
@code """
float advanced(float x) {
    return x;
}
"""
```

Rules:
- A code block without `@gl_requires` inherits the module's `@gl_version` requirement.
- A code block may have at most one `@gl_requires` annotation.
- The annotation may specify a higher or lower version requirement than the module.
- Profile resolution follows the same rules as `@gl_version`.
- A function-level version mismatch at a call site must produce a `WARNING`, as defined in Section 5.
- An `@gl_requires` annotation does not modify the module's baseline `@gl_version`.

## 4. Code Blocks and Function Visibility

### 4.1. `@code`

- Owner File Type: `.ksml`
- Usage Pattern: `@code """ <code_block> """`

The `@code` declaration defines a GLSL function inside a KSML module.

Each code block must contain exactly one function definition.

Example:
```ksml
@code """
float square(float x) {
    return x * x;
}
"""
```

Rules:
- A module may contain multiple code blocks.
- Each block must contain exactly one GLSL function definition.
- The function body must follow the GLSL syntax and semantics applicable to its declared version requirement.
- KSML/GLSL function overloading is supported across multiple code blocks.
- Multiple overloads may share a function name, provided their signatures satisfy GLSL overloading rules.
- Duplicate function definitions with identical signatures are invalid.

### 4.2. `@export`

- Owner File Type: `.ksml`
- Usage Pattern: `@export`

The `@export` annotation makes the immediately following function visible to GLSL shader files.

Example:
```ksml
@export
@code """
float square(float x) {
    return x * x;
}
"""

```

Rules:
- An exported function is accessible from GLSL shader files that import its module.
- An exported function is also accessible from KSML modules.
- An unexported function is internal and cannot be called directly from GLSL shader files.
- An unexported function remains accessible from KSML modules, subject to namespace and dependency rules.
- A call from GLSL to an unexported function is an error.
- A code block may have at most one `@export` annotation.

### 4.3. `@feature`

- Owner File Type: `.ksml`
- Usage Pattern: `@feature <feature_ident>`

The `@feature` annotation associates a function with a compiler-controlled feature.

Example:
```ksml
@feature PBR
@export
@code """
float pbrWeight(float roughness) {
    return roughness;
}
"""
```

Rules:
- Feature identifiers may be freely defined without prior registration, but must follow `[A-Za-z_][A-Za-z_0-9]*`.
- The compiler must support enabling or disabling features through its configuration.
- A code block is included only when all of its declared feature requirements are enabled.
- A disabled code block must be treated as absent for function resolution and GLSL generation.
- A code block may declare multiple feature requirements.

## 5. Namespaces and Function Calls

### 5.1. Qualified Function Calls

- Owner File Type: `.ksml` / `.glsl` / (`.vert` / `.frag` / ...)
- Usage Pattern: `<module_name>.<function_name>(<arguments>)`

A KSML function must be referenced through its module namespace.

This applies to calls from GLSL shaders, calls between different KSML modules, and calls between functions within the same KSML module.

Example:
```ksml
@module math
@gl_version 110

@code """
float square(float x) {
    return x * x;
}
"""

@export
@code """
float fourthPower(float x) {
    return math.square(math.square(x));
}
"""
```

A function in module `math` must call another function in `math` using the `math.` namespace.

The following is not valid as a call to a KSML-defined function:
```ksml
@code """
float fourthPower(float x) {
    return square(square(x));
}
"""
```

Rules:
- All KSML-defined function calls must be explicitly namespace-qualified.
- A function must use its own module name when calling functions declared in the same module.
- A function must use the target module name when calling functions declared in another module.
- Calls to functions in another module require a corresponding `@requires` declaration.
- Calls to functions in the current module do not require a self-dependency declaration.
- Unqualified GLSL built-in function calls remain valid and retain their original GLSL semantics.

### 5.2. Cross-Module Function Calls

Example:
```ksml
@module effects
@requires math
@gl_version 330 core

@code """
float apply(float x) {
    return math.square(x);
}
"""
```

The `math.square()` call is valid because `effects` declares `math` as a dependency.

The compiler must report an error if the required module is missing or the referenced function cannot be resolved.

### 5.3. Function Name Rewriting

During compilation, module-qualified function names must be converted into valid GLSL identifiers.

The compiler must preserve the namespace association and avoid collisions between functions declared in different modules.

The generated naming convention is implementation-defined.

For example, the KSML expression:
```text
math.square(x)
```

may be rewritten into a compiler-generated GLSL function call.

The period in a KSML function call represents namespace resolution, not a GLSL structure member access.

## 6. Compiler Diagnostics

The compiler distinguishes three diagnostic levels:

- `ERROR`
- `WARNING`
- `NOTICE`

The diagnostic level depends on the nature of the problem, not merely on whether compilation can continue.

### 6.1. `ERROR`

An `ERROR` indicates invalid source code or a failed requirement necessary for correct KSML compilation.

Errors must terminate compilation.

Examples include:
- Invalid or unsupported GLSL version numbers.
- Profile declarations for GLSL versions below 150.
- Invalid profile identifiers.
- The `compat` abbreviation in a GLSL `#version` directive.
- Missing or duplicate mandatory module declarations.
- Duplicate module names.
- Unresolved imports or dependencies.
- Calls to unresolved module functions.
- Unqualified calls to KSML-defined functions.
- Calls to internal KSML functions from GLSL shader files.
- Invalid function declarations or duplicate signatures.

### 6.2. `NOTICE`

A `NOTICE` indicates a module-level GLSL version requirement mismatch.

A notice is produced when a shader imports a module, or a module requires another module, 
whose baseline `@gl_version` requirement is not satisfied by the importing source.

Example:
```text
Shader:
GL330 core

Imported Module:
GL450 core

Diagnostic:
NOTICE
```

Module-to-module example:
```text
Module A:
GL330 core

Module B:
GL450 core

A requires B:
NOTICE
```

Rules:
- Module-level version mismatches must produce notices.
- Notices must not terminate compilation.
- Notices must not be counted as warnings.
- A notice may be produced for either direct or transitive dependencies.
- The compiler should identify the affected module and the incompatible version requirements.

### 6.3. `WARNING`

A `WARNING` indicates a function-level GLSL version requirement mismatch.

A warning is produced when the caller's effective version requirement does not satisfy the called function's requirement.

Example:
```ksml
@module math
@gl_version 330 core

@gl_requires 450 core
@export
@code """
float advanced(float x) {
    return x;
}
"""
```

If `advanced()` is called from a GL330 core shader, the compiler must produce a warning.

```text
Caller:
GL330 core

Function:
GL450 core

Diagnostic:
WARNING
```

Rules:
- Function-level version mismatches must produce warnings.
- Warnings must not terminate compilation by themselves.
- The compiler must use the effective version requirement of the called function, including any `@gl_requires` override.
- A warning may occur for a function whose requirement is inherited from the module's `@gl_version`.
- When overloads are present, the compiler must check the applicable overload candidates. 
  A warning is required when none of those candidates satisfies the caller's requirement.

### 6.4. Diagnostic Separation

Module-level and function-level version checks are distinct.

| Condition                                                                        | Diagnostic |
|----------------------------------------------------------------------------------|------------|
| Unsupported GLSL version                                                         | ERROR      |
| Invalid profile declaration                                                      | ERROR      |
| Unresolved module or function                                                    | ERROR      |
| Missing namespace qualification                                                  | ERROR      |
| Shader imports a module with an incompatible baseline version requirement        | NOTICE     |
| Module requires another module with an incompatible baseline version requirement | NOTICE     |
| Caller uses a function with an incompatible effective version requirement        | WARNING    |

The same module may generate both a notice and a warning if its baseline requirement is incompatible
and one of its functions is also called from an incompatible context.

The compiler must preserve this distinction rather than promoting every module-level mismatch to a warning.

## 7. Complete Example

`math.ksml`:
```ksml
@module math
@gl_version 330

@export
@code """
float square(float x) {
    return x * x;
}
"""
```

`lighting.ksml`:
```ksml
@module lighting
@requires math
@requires color
@gl_version 330

@code """
float saturate(float x) {
    return clamp(x, 0.0, 1.0);
}
"""

@export
@code """
float intensity(float x) {
    return lighting.saturate(math.square(x));
}
"""
```

`main.frag`:
```glsl
// @import lighting

#version 330

out vec4 fragColor;

void main() {
    float value = lighting.intensity(0.5);
    fragColor = vec4(value);
}
```

In this example:
- All omitted profiles for GLSL 330 resolve to `core`.
- The compiler resolves `math` and `color` through `lighting`'s dependency declarations.
- `lighting.intensity()` is accessible to the shader because it is exported.
- `lighting.saturate()` is an internal function and cannot be called directly from the shader.
- The call from `intensity()` to `saturate()` is valid because it uses the `lighting` namespace.
- The shader does not need to import `math` separately because it does not call `math` functions directly.

The example assumes that a valid `color` module is supplied as part of the compilation unit.

## 8. Compiler Output and Source Mapping

### 8.1. GLSL Assembly

The compiler must produce a single GLSL shader source from the input shader and its selected KSML modules.

The generated source must follow this order:
1. GLSL `#version` and extension directives.
2. Source file ID mapping.
3. Function prototypes from all selected modules.
4. KSML module function definitions.
5. Generated exported function wrappers.
6. Original GLSL shader declarations.

Function prototypes must be emitted before module definitions to support cross-module calls and cyclic dependencies.

### 8.2. Source Mapping

The compiler must preserve original source locations using GLSL `#line` directives.

Each source file must be assigned a unique integer source ID within a compilation.

The compiler must insert:

```glsl
#line <original_line> <source_id>
```

before each KSML function definition and each original GLSL top-level declaration. 
Generated exported wrappers should also retain the source location of their corresponding function declarations.

A source ID mapping table must be included in the generated GLSL as a comment.

Example:
```glsl
/* File Mapping:
 * math.ksml <-> 0
 * main.frag <-> 1
 */

#line 12 0
float ksml_4_math_square(float x) {
    return x * x;
}

#line 6 1
void main() {
    // ...
}
```

The `#line` directives allow downstream GLSL compilers to report errors using the original source line numbers 
and source IDs rather than the assembled shader's physical line numbers.

### 8.3. Compilation Diagnostics

KSML parsing and semantic diagnostics must reference the original source file and location.

Generated GLSL source IDs must remain consistent throughout a compilation so downstream shader diagnostics 
can be mapped back to their corresponding files.

The KSML compiler is responsible for source generation and KSML diagnostics. 
Actual OpenGL shader compilation and driver error reporting remain outside its scope.
