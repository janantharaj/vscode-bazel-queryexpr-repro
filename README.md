# repro: queryExpression partially applied to Bazel Targets tree

Small workspace to show that `bazel.commandLine.queryExpression` only
half-filters the Bazel Targets tree: the package layer is filtered, but
target listings inside each package (and at the workspace root) are
hardcoded to `:all` and leak non-matching targets through.

Uses only built-in Bazel rules (`genrule`, `filegroup`, `alias`) so no
external module dependencies are required — `bazel query //...` works
out of the box.

## Layout

- `//` (root) — `genrule :root_gen`, `filegroup :root_files`, `alias :root_alias`
- `pkg_app` — `genrule :app`, `alias :app_alias`
- `pkg_lib` — `filegroup :sources`, `genrule :concat`
- `pkg_data`, `pkg_data/nested` — filegroups only

Full target list:

```
//:root_alias              (alias)
//:root_files              (filegroup)
//:root_gen                (genrule)
//pkg_app:app              (genrule)
//pkg_app:app_alias        (alias)
//pkg_data:all_data        (filegroup)
//pkg_data:readme          (filegroup)
//pkg_data/nested:fixture  (filegroup)
//pkg_lib:concat           (genrule)
//pkg_lib:sources          (filegroup)
```

Root-level targets are rendered directly under the workspace node in the
tree (no enclosing package folder).

## Repro

1. Clone, open in VS Code with the Bazel extension installed.
2. Open the Bazel Targets view in the Explorer.
3. Set a filter in `.vscode/settings.json`, e.g.
   ```json
   { "bazel.commandLine.queryExpression": "kind('genrule', //...)" }
   ```
4. Reload the window (or refresh the tree).

### Example: `"bazel.commandLine.queryExpression": "kind('genrule', //...)"`

**Current Behavior:** `pkg_data` is hidden (no genrules there), but every
other level leaks — root-level targets are all shown and surviving
packages still list non-genrule targets.

```
  :root_alias               (alias)       <- not a genrule, still shown
  :root_files               (filegroup)   <- not a genrule, still shown
  :root_gen                 (genrule)
▾ pkg_app
    :app                    (genrule)
    :app_alias              (alias)       <- not a genrule, still shown
▾ pkg_lib
    :concat                 (genrule)
    :sources                (filegroup)   <- not a genrule, still shown
```

**Expected Behavior:** only matching targets remain at every level.

```
  :root_gen                 (genrule)
▾ pkg_app
    :app                    (genrule)
▾ pkg_lib
    :concat                 (genrule)
```

### Example: `"bazel.commandLine.queryExpression": "kind('filegroup', //...)"`

**Current Behavior:**

```
  :root_alias               (alias)       <- not a filegroup, still shown
  :root_files               (filegroup)
  :root_gen                 (genrule)     <- not a filegroup, still shown
▾ pkg_data
    :all_data               (filegroup)
    :readme                 (filegroup)
  ▾ nested
      :fixture              (filegroup)
▾ pkg_lib
    :concat                 (genrule)     <- not a filegroup, still shown
    :sources                (filegroup)
```

**Expected Behavior:**

```
  :root_files               (filegroup)
▾ pkg_data
    :all_data               (filegroup)
    :readme                 (filegroup)
  ▾ nested
      :fixture              (filegroup)
▾ pkg_lib
    :sources                (filegroup)
```

### Example: `"bazel.commandLine.queryExpression": "//pkg_app:app"`

**Current Behavior:** even though the filter names a single target, the
root targets and `pkg_app`'s sibling target all appear.

```
  :root_alias               (alias)       <- still shown
  :root_files               (filegroup)   <- still shown
  :root_gen                 (genrule)     <- still shown
▾ pkg_app
    :app                    (genrule)
    :app_alias              (alias)       <- still shown
```

**Expected behavior:**

```
▾ pkg_app
    :app                    (genrule)
```
