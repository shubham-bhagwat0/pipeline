<!--
---
linkTitle: "Matrix"
weight: 406
---
-->

# Matrix

- [Overview](#overview)
- [Configuring a Matrix](#configuring-a-matrix)
  - [Generating Combinations](#generating-combinations)
  - [Explicit Combinations](#explicit-combinations)
- [Concurrency Control](#concurrency-control)
- [Parameters](#parameters)
  - [Parameters in Matrix.Params](#parameters-in-matrixparams-1)
  - [Parameters in Matrix.Include.Params](#parameters-in-matrixincludeparams)
  - [Specifying both `params` and `matrix` in a `PipelineTask`](#specifying-both-params-and-matrix-in-a-pipelinetask)
- [Context Variables](#context-variables)
- [Results](#results)
  - [Specifying Results in a Matrix](#specifying-results-in-a-matrix)
    - [Results in Matrix.Params](#results-in-matrixparams)
    - [Results in Matrix.Include.Params](#results-in-matrixincludeparams)
  - [Results from fanned out PipelineTasks](#results-from-fanned-out-pipelinetasks)
- [Retries](#retries)
- [Examples](#examples)
  - [`PipelineTasks` with `Tasks`](#pipelinetasks-with-tasks)
  - [`PipelineTasks` with `Custom Tasks`](#pipelinetasks-with-custom-tasks)

## Overview

`Matrix` is used to fan out `Tasks` in a `Pipeline`. This doc will explain the details of `matrix` support in
Tekton.

Documentation for specifying `Matrix` in a `Pipeline`:
- [Specifying `Matrix` in `Tasks`](pipelines.md#specifying-matrix-in-pipelinetasks)
- [Specifying `Matrix` in `Finally Tasks`](pipelines.md#specifying-matrix-in-finally-tasks)
- [Specifying `Matrix` in `Custom Tasks`](pipelines.md#specifying-matrix)

> :seedling: **`Matrix` is an [alpha](install.md#alpha-features) feature.**
> The `enable-api-fields` feature flag must be set to `"alpha"` to specify `Matrix` in a `PipelineTask`.

## Configuring a Matrix

A `Matrix` allows you to generate combinations and specify explicit combinations to fan out a `PipelineTask`.

### Generating Combinations

The `Matrix.Params` is used to generate combinations to fan out a `PipelineTask`.

```yaml
    matrix:
      params:
        - name: platform
          value:
          - linux
          - mac
        - name: browser
          value:
          - safari
          - chrome
  ...
```

### Explicit Combinations

> :warning: This feature is in a preview mode.
> It is still in a very early stage of development and is not yet fully functional.

The `Matrix.Include` is used to add explicit combinations to fan out a `PipelineTask`, but is not yet functional.

```yaml
    matrix:
      include:
        - name: s390x-no-race
          params:
              - name: GOARCH
                value: "linux/s390x"
              - name: flags
                value: "-cover -v"
  ...
```

## Concurrency Control

The default maximum count of `TaskRuns` or `Runs` from a given `Matrix` is **256**. To customize the maximum count of
`TaskRuns` or `Runs` generated from a given `Matrix`, configure the `default-max-matrix-combinations-count` in
[config defaults](/config/config-defaults.yaml). When a `Matrix` in `PipelineTask` would generate more than the maximum
`TaskRuns` or `Runs`, the `Pipeline` validation would fail.

Note: The matrix combination count includes combinations generated from both `Matrix.Params` and `Matrix.Include.Params`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-defaults
data:
  default-service-account: "tekton"
  default-timeout-minutes: "20"
  default-max-matrix-combinations-count: "1024"
  ...
```

For more information, see [installation customizations](install.md#customizing-basic-execution-parameters).

## Parameters

`Matrix` takes in `Parameters` in two sections:
- `Matrix.Params`: used to generate combinations to fan out the `PipelineTask`.
- `Matrix.Include.Params`: used to specify specific combinations to fan out the `PipelineTask`.

Note that:
- The names of the `Parameters` in the `Matrix` must match the names of the `Parameters` in the underlying
`Task` that they will be substituting.
- The names of the `Parameters` in the `Matrix` must be unique. Specifying the same parameter multiple times
will result in a validation error.
- A `Parameter` can be passed to either the `matrix` or `params` field, not both.

For further details on specifying `Parameters` in the `Pipeline` and passing them to
`PipelineTasks`, see [documentation](pipelines.md#specifying-parameters).

#### Parameters in Matrix.Params

`Matrix.Params` supports string replacements from `Parameters` of type String, Array or Object.

```yaml
tasks:
...
- name: task-4
  taskRef:
    name: task-4
  matrix:
    params:
    - name: param-one
      value:
      - $(params.foo) # string replacement from string param
      - $(params.bar[0]) # string replacement from array param
      - $(params.rad.key) # string replacement from object param
    - name: param-two
      value: $(params.bar) # array replacement from array param
```
#### Parameters in Matrix.Include.Params

`Matrix.Include.Params` takes string replacements from `Parameters` of type String, Array or Object.

```yaml
tasks:
...
- name: task-4
  taskRef:
    name: task-4
  matrix:
    include:
      - name: foo-bar-rad
        params:
        - name: foo
          value: $(params.foo) # string replacement from string param
        - name: bar
          value: $(params.bar[0]) # string replacement from array param
        - name: rad
          value: $(params.rad.key) # string replacement from object param
```

### Specifying both `params` and `matrix` in a `PipelineTask`

In the example below, the *test* `Task` takes *browser* and *platform* `Parameters` of type
`"string"`. A `Pipeline` used to run the `Task` on three browsers (using `matrix`) and one
platform (using `params`) would be specified as such and execute three `TaskRuns`:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: platform-browser-tests
spec:
  tasks:
  - name: fetch-repository
    taskRef:
      name: git-clone
    ...
  - name: test
    matrix:
      params:
        - name: browser
          value:
            - chrome
            - safari
            - firefox
    params:
      - name: platform
        value: linux
    taskRef:
      name: browser-test
  ...
```

## Context Variables

Similarly to the `Parameters` in the `Params` field, the `Parameters` in the `Matrix` field will accept
[context variables](variables.md) that will be substituted, including:

* `PipelineRun` name, namespace and uid
* `Pipeline` name
* `PipelineTask` retries

## Results

### Specifying Results in a Matrix

Consuming `Results` from previous `TaskRuns` or `Runs` in a `Matrix`, which would dynamically generate
`TaskRuns` or `Runs` from the fanned out `PipelineTask`, is supported. Producing `Results` in from a
`PipelineTask` with a `Matrix` is not yet supported - see [further details](#results-from-fanned-out-pipelinetasks).

See the end-to-end example in [`PipelineRun` with `Matrix` and `Results`][pr-with-matrix-and-results].

#### Results in Matrix.Params

`Matrix.Params` supports string replacements from `Results` of type String, Array or Object.

```yaml
tasks:
...
- name: task-4
  taskRef:
    name: task-4
  matrix:
    params:
    - name: values
      value:
      - $(tasks.task-1.results.foo) # string replacement from string result
      - $(tasks.task-2.results.bar[0]) # string replacement from array result
      - $(tasks.task-3.results.rad.key) # string replacement from object result
```

For further information, see the example in [`PipelineRun` with `Matrix` and `Results`][pr-with-matrix-and-results].

We plan to add support passing whole Array `Results` into the `Matrix` (#5925):

```yaml
tasks:
...
- name: task-5
  taskRef:
    name: task-5
  matrix:
    params:
    - name: values
      value: $(tasks.task-4.results.foo) # array
```

#### Results in Matrix.Include.Params

`Matrix.Include.Params` supports string replacements from `Results` of type String, Array or Object.

```yaml
tasks:
...
- name: task-4
  taskRef:
    name: task-4
  matrix:
    include:
      - name: foo-bar-duh
      params:
        - name: foo
          value: $(tasks.task-1.results.foo) # string replacement from string result
        - name: bar
          value: $(tasks.task-2.results.bar[0]) # string replacement from array result
        - name: duh
          value: $(tasks.task-2.results.duh.key) # string replacement from object result
```

### Results from fanned out PipelineTasks

Consuming `Results` from fanned out `PipelineTasks` will not be in the supported in the initial iteration
of `Matrix`. Supporting consuming `Results` from fanned out `PipelineTasks` will be revisited after array
and object `Results` are supported.


## Retries

The `retries` field is used to specify the number of times a `PipelineTask` should be retried when its `TaskRun` or
`Run` fails, see the [documentation][retries] for further details. When a `PipelineTask` is fanned out using `Matrix`,
a given `TaskRun` or `Run` executed will be retried as much as the field in the `retries` field of the `PipelineTask`.

For example, the `PipelineTask` in this `PipelineRun` will be fanned out into three `TaskRuns` each of which will be
retried once:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: matrixed-pr-with-retries-
spec:
  pipelineSpec:
    tasks:
      - name: matrix-and-params
        matrix:
          params:
            - name: platform
              value:
                - linux
                - mac
                - windows
        params:
          - name: browser
            value: chrome
        retries: 1
        taskSpec:
          params:
            - name: platform
            - name: browser
          steps:
            - name: echo
              image: alpine
              script: |
                echo "$(params.platform) and $(params.browser)"
                exit 1
```

## Examples

### `PipelineTasks` with `Tasks`

When a `PipelineTask` has a `Task` and a `Matrix`, the `Task` will be executed in parallel `TaskRuns` with
substitutions from combinations of `Parameters`.

In the example below, nine `TaskRuns` are created with combinations of platforms ("linux", "mac", "windows")
and browsers ("chrome", "safari", "firefox").

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: platform-browsers
  annotations:
    description: |
      A task that does something cool with platforms and browsers
spec:
  params:
    - name: platform
    - name: browser
  steps:
    - name: echo
      image: alpine
      script: |
        echo "$(params.platform) and $(params.browser)"
---
# run platform-browsers task with:
#   platforms: linux, mac, windows
#   browsers: chrome, safari, firefox
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: matrixed-pr-
spec:
  serviceAccountName: 'default'
  pipelineSpec:
    tasks:
      - name: platforms-and-browsers
        matrix:
          params:
            - name: platform
              value:
                - linux
                - mac
                - windows
            - name: browser
              value:
                - chrome
                - safari
                - firefox
        taskRef:
          name: platform-browsers
```

When the above `PipelineRun` is executed, these are the `TaskRuns` that are created:

```shell
$ tkn taskruns list

NAME                                         STARTED        DURATION     STATUS
matrixed-pr-6lvzk-platforms-and-browsers-8   11 seconds ago   7 seconds    Succeeded
matrixed-pr-6lvzk-platforms-and-browsers-6   12 seconds ago   7 seconds    Succeeded
matrixed-pr-6lvzk-platforms-and-browsers-7   12 seconds ago   9 seconds    Succeeded
matrixed-pr-6lvzk-platforms-and-browsers-4   12 seconds ago   7 seconds    Succeeded
matrixed-pr-6lvzk-platforms-and-browsers-5   12 seconds ago   6 seconds    Succeeded
matrixed-pr-6lvzk-platforms-and-browsers-3   13 seconds ago   7 seconds    Succeeded
matrixed-pr-6lvzk-platforms-and-browsers-1   13 seconds ago   8 seconds    Succeeded
matrixed-pr-6lvzk-platforms-and-browsers-2   13 seconds ago   8 seconds    Succeeded
matrixed-pr-6lvzk-platforms-and-browsers-0   13 seconds ago   8 seconds    Succeeded
```

When the above `Pipeline` is executed, its status is populated with `ChildReferences` of the above `TaskRuns`. The
`PipelineRun` status tracks the status of all the fanned out `TaskRuns`. This is the `PipelineRun` after completing
successfully:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: matrixed-pr-
  labels:
    tekton.dev/pipeline: matrixed-pr-6lvzk
  name: matrixed-pr-6lvzk
  namespace: default
spec:
  pipelineSpec:
    tasks:
    - matrix:
        params:
          - name: platform
            value:
              - linux
              - mac
              - windows
          - name: browser
              value:
                - chrome
                - safari
                - firefox
      name: platforms-and-browsers
      taskRef:
        kind: Task
        name: platform-browsers
  serviceAccountName: default
  timeout: 1h0m0s
status:
  pipelineSpec:
    tasks:
      - matrix:
          params:
            - name: platform
              value:
                - linux
                - mac
                - windows
            - name: browser
              value:
                - chrome
                - safari
                - firefox
        name: platforms-and-browsers
        taskRef:
          kind: Task
          name: platform-browsers
  startTime: "2022-06-23T23:01:11Z"
  completionTime: "2022-06-23T23:01:20Z"
  conditions:
    - lastTransitionTime: "2022-06-23T23:01:20Z"
      message: 'Tasks Completed: 1 (Failed: 0, Cancelled 0), Skipped: 0'
      reason: Succeeded
      status: "True"
      type: Succeeded
  childReferences:
  - apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    name: matrixed-pr-6lvzk-platforms-and-browsers-4
    pipelineTaskName: platforms-and-browsers
  - apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    name: matrixed-pr-6lvzk-platforms-and-browsers-6
    pipelineTaskName: platforms-and-browsers
  - apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    name: matrixed-pr-6lvzk-platforms-and-browsers-2
    pipelineTaskName: platforms-and-browsers
  - apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    name: matrixed-pr-6lvzk-platforms-and-browsers-1
    pipelineTaskName: platforms-and-browsers
  - apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    name: matrixed-pr-6lvzk-platforms-and-browsers-7
    pipelineTaskName: platforms-and-browsers
  - apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    name: matrixed-pr-6lvzk-platforms-and-browsers-0
    pipelineTaskName: platforms-and-browsers
  - apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    name: matrixed-pr-6lvzk-platforms-and-browsers-8
    pipelineTaskName: platforms-and-browsers
  - apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    name: matrixed-pr-6lvzk-platforms-and-browsers-3
    pipelineTaskName: platforms-and-browsers
  - apiVersion: tekton.dev/v1beta1
    kind: TaskRun
    name: matrixed-pr-6lvzk-platforms-and-browsers-5
    pipelineTaskName: platforms-and-browsers
```

To execute this example yourself, run [`PipelineRun` with `Matrix`][pr-with-matrix].

### `PipelineTasks` with `Custom Tasks`

When a `PipelineTask` has a `Custom Task` and a `Matrix`, the `Custom Task` will be executed in parallel `Runs` with
substitutions from combinations of `Parameters`.

In the example below, eight `Runs` are created with combinations of CEL expressions, using the [CEL `Custom Task`][cel].

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: matrixed-pr-
spec:
  serviceAccountName: 'default'
  pipelineSpec:
    tasks:
      - name: platforms-and-browsers
        matrix:
          params:
            - name: type
              value:
                - "type(1)"
                - "type(1.0)"
            - name: colors
              value:
                - "{'blue': '0x000080', 'red': '0xFF0000'}['blue']"
                - "{'blue': '0x000080', 'red': '0xFF0000'}['red']"
            - name: bool
              value:
                - "type(1) == int"
                - "{'blue': '0x000080', 'red': '0xFF0000'}['red'] == '0xFF0000'"
        taskRef:
          apiVersion: cel.tekton.dev/v1alpha1
          kind: CEL
```

When the above `PipelineRun` is executed, these `Runs` are created:

```shell
$ k get run.tekton.dev

NAME                                         SUCCEEDED   REASON              STARTTIME   COMPLETIONTIME
matrixed-pr-4djw9-platforms-and-browsers-0   True        EvaluationSuccess   10s         10s
matrixed-pr-4djw9-platforms-and-browsers-1   True        EvaluationSuccess   10s         10s
matrixed-pr-4djw9-platforms-and-browsers-2   True        EvaluationSuccess   10s         10s
matrixed-pr-4djw9-platforms-and-browsers-3   True        EvaluationSuccess   9s          9s
matrixed-pr-4djw9-platforms-and-browsers-4   True        EvaluationSuccess   9s          9s
matrixed-pr-4djw9-platforms-and-browsers-5   True        EvaluationSuccess   9s          9s
matrixed-pr-4djw9-platforms-and-browsers-6   True        EvaluationSuccess   9s          9s
matrixed-pr-4djw9-platforms-and-browsers-7   True        EvaluationSuccess   9s          9s
```

When the above `PipelineRun` is executed, its status is populated with `ChildReferences` of the above `Runs`. The
`PipelineRun` status tracks the status of all the fanned out `Runs`. This is the `PipelineRun` after completing:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: matrixed-pr-
  labels:
    tekton.dev/pipeline: matrixed-pr-4djw9
  name: matrixed-pr-4djw9
  namespace: default
spec:
  pipelineSpec:
    tasks:
      - matrix:
          params:
            - name: type
              value:
                - type(1)
                - type(1.0)
            - name: colors
              value:
                - '{''blue'': ''0x000080'', ''red'': ''0xFF0000''}[''blue'']'
                - '{''blue'': ''0x000080'', ''red'': ''0xFF0000''}[''red'']'
            - name: bool
              value:
                - type(1) == int
                - '{''blue'': ''0x000080'', ''red'': ''0xFF0000''}[''red''] == ''0xFF0000'''
        name: platforms-and-browsers
        taskRef:
          apiVersion: cel.tekton.dev/v1alpha1
          kind: CEL
  serviceAccountName: default
  timeout: 1h0m0s
status:
  pipelineSpec:
    tasks:
      - matrix:
          params:
            - name: type
              value:
                - type(1)
                - type(1.0)
            - name: colors
              value:
                - '{''blue'': ''0x000080'', ''red'': ''0xFF0000''}[''blue'']'
                - '{''blue'': ''0x000080'', ''red'': ''0xFF0000''}[''red'']'
            - name: bool
              value:
                - type(1) == int
                - '{''blue'': ''0x000080'', ''red'': ''0xFF0000''}[''red''] == ''0xFF0000'''
        name: platforms-and-browsers
        taskRef:
          apiVersion: cel.tekton.dev/v1alpha1
          kind: CEL
  startTime: "2022-06-28T20:49:40Z"
  completionTime: "2022-06-28T20:49:41Z"
  conditions:
    - lastTransitionTime: "2022-06-28T20:49:41Z"
      message: 'Tasks Completed: 1 (Failed: 0, Cancelled 0), Skipped: 0'
      reason: Succeeded
      status: "True"
      type: Succeeded
  childReferences:
    - apiVersion: tekton.dev/v1alpha1
      kind: Run
      name: matrixed-pr-4djw9-platforms-and-browsers-1
      pipelineTaskName: platforms-and-browsers
    - apiVersion: tekton.dev/v1alpha1
      kind: Run
      name: matrixed-pr-4djw9-platforms-and-browsers-2
      pipelineTaskName: platforms-and-browsers
    - apiVersion: tekton.dev/v1alpha1
      kind: Run
      name: matrixed-pr-4djw9-platforms-and-browsers-3
      pipelineTaskName: platforms-and-browsers
    - apiVersion: tekton.dev/v1alpha1
      kind: Run
      name: matrixed-pr-4djw9-platforms-and-browsers-4
      pipelineTaskName: platforms-and-browsers
    - apiVersion: tekton.dev/v1alpha1
      kind: Run
      name: matrixed-pr-4djw9-platforms-and-browsers-5
      pipelineTaskName: platforms-and-browsers
    - apiVersion: tekton.dev/v1alpha1
      kind: Run
      name: matrixed-pr-4djw9-platforms-and-browsers-6
      pipelineTaskName: platforms-and-browsers
    - apiVersion: tekton.dev/v1alpha1
      kind: Run
      name: matrixed-pr-4djw9-platforms-and-browsers-7
      pipelineTaskName: platforms-and-browsers
    - apiVersion: tekton.dev/v1alpha1
      kind: Run
      name: matrixed-pr-4djw9-platforms-and-browsers-0
      pipelineTaskName: platforms-and-browsers
```

[cel]: https://github.com/tektoncd/experimental/tree/1609827ea81d05c8d00f8933c5c9d6150cd36989/cel
[pr-with-matrix]: ../examples/v1beta1/pipelineruns/alpha/pipelinerun-with-matrix.yaml
[pr-with-matrix-and-results]: ../examples/v1beta1/pipelineruns/alpha/pipelinerun-with-matrix-and-results.yaml
[retries]: pipelines.md#using-the-retries-field
