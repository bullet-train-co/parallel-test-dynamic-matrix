# bullet-train-co/parallel-test-dynamic-matrix

A GitHub action to calculate a dynamic list of test groups for use with `parallel_test`.

## Usage

```yaml
- name: Dynamic Matrix
  id: dynamic-matrix
  uses: bullet-train-co/parallel-test-dynamic-matrix@v1
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
| `totalRunners` | The total number of runners across all nodes | `16` |
| `numberOfTestNodes` | The number of "physical" test nodes to spin up. | `4` |
| `testRunnersPerNode` | The number of parallel test groups to run on each node. | `4` |


## Reference Example

Here is a complete example worklow.

```yaml
name: " 💻 PR CI (staging)"
on:
  workflow_dispatch:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  # First a separate job to host this action. The output of this job
  # is used to configure the matrix in the test job.
  calculate_matrix:
    name: Calculate Dynamic Matrix
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.dynamic-matrix.outputs.matrix }}
      totalRunners: ${{ steps.dynamic-matrix.outputs.totalRunners }}
      testRunnersPerNode: ${{ steps.dynamic-matrix.outputs.testRunnersPerNode }}
    steps:
      - name: Dynamic Matrix
        id: dynamic-matrix
        uses: bullet-train-co/parallel-test-dynamic-matrix@v1
        with:
          # You can explicitly set the number of test nodes, or it will default to 4
          numberOfTestNodes: 4
          # You can explicitly set the number of runners per node, or it will default to 4 for public repos and 2 for private
          # testRunnersPerNode: 2
  parallel-tests:
    name: Run Test
    needs: [calculate_matrix]
    strategy:
      fail-fast: false
      matrix:
        test_groups: ${{ fromJson(needs.dynamic-matrix.outputs.matrix) }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Ruby and gems
        uses: ruby/setup-ruby@v1

      - name: Set up database schema
        run: bin/rake parallel:setup[${{ fromJson(needs.calculate_matrix.outputs.testRunnersPerNode) }}]

      - name: Run tests
        id: run-tests
        run: |
          bundle exec parallel_test test \
          -n ${{needs.calculate_matrix.outputs.totalRunners}} \
          --only-group ${{ matrix.test_groups }} \
          --group-by runtime \
          --allowed-missing 100 \
          --verbose \
          --verbose-command
        working-directory: tmp/starter
        continue-on-error: false
        shell: bash
```

To see this in action in a real repo you can check out the [`_run_tests.yml` workflow in the Bullet Train starter repo](https://github.com/bullet-train-co/bullet_train/blob/main/.github/workflows/_run_tests.yml)
and you can see [example test runs](https://github.com/bullet-train-co/bullet_train/actions/workflows/ci-cd-pipeline.yml).

