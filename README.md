# bullet-train-co/parallel-test-dynamic-matrix

A GitHub action to calculate a dynamic list of test groups for use with `parallel_test`.

## Usage

```yaml
- name: Dynamic Matrix
  id: dynamic-matrix
  uses: bullet-train-co/parallel-test-dynamic-matrix
```

## Action Inputs

| Name | Description | Default |
| --- | --- | --- |
| `numberOfTestNodes` | The number of "physical" test nodes to spin up. | 4 |
| `testRunnersPerNode` | The number of parallel test groups to run on each node. | For public repos the default is 4, otherwise it's 2 |

## Action Outputs

| Name | Description | Example |
| --- | --- | --- |
| `matrix` | An array containing the test groups to use with `parallel_test` | `[ '1,2,3,4', '5,6,7,8', '9,10,11,12', '13,14,15,16' ]` |



## Reference Example

Here is a complete example worklow.

```yaml
name: " 💻 PR CI (staging)"
on:
  workflow_dispatch:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  dynamic-matrix:
    name: Dynamic Matrix
    uses: bullet-train-co/parallel-test-dynamic-matrix/workflow.yml
  parallel-tests:
    name: Run Test
    strategy:
      fail-fast: false
      matrix:
        test_group: ${{ fromJson(needs.dynamic-matrix.outputs.matrix) }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run tests
        id: run-tests
        run: |
          bundle exec parallel_test test \
          -n ${{needs.dynamic-matrix.outputs.totalRunners}} \
          --only-group ${{ matrix.test_group }} \
          --group-by runtime \
          --allowed-missing 100 \
          --verbose \
          --verbose-command
        working-directory: tmp/starter
        continue-on-error: false
        shell: bash
```
