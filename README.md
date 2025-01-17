# Playwright finish tests in x minutes

A GitHub action to calculate required runners to finish your tests in x mins. 

## Known limitations

- This action would work best when tests are more or less of same execution time length and they are all run in parallel using `fully parallel = true`. 

However, if there are some very extremely slow tests, the time taken maybe more than what user desires. This limitation comes from the playwright sharder itself which does not know about how much time each test takes and thus may end up clubbing many slow tests together in the same runner rather than properly land balancing them based on their execution time. 

A workaround for this would be to increase the expected time and it would hopefully add more runners and bring down total execution time. 

- This action assumes that user is running tests with flag `fully parallel = true`. Since it calculates and uses the number of cores of runner machine to decide how many runners are required (both distributed runs on multiple machines and parallel test run on the same machine). The logic to add for `fully parallel = false` is not yet in the action and is yet to be added. 

- Within a framework a user may run certain tests in `serial` or `default` mode, in which case the calculations would be a bit off. 

## Why this action?

Playwright provides an option to run tests on multiple runners using its sharding option as shown [here](https://playwright.dev/docs/test-sharding#github-actions-example). Sharding is a great way to create multiple runners to bring down total execution time. For instance, when a branch is merged to the main branch, post deployment of dev environment, you may want to run all your tests on newly deployed dev environment to see everyhing works as expected.

However, there are also instances when you just want to run a subset of tests, say at a CRON job every hour or so to check if the test environments are up and running. OR we may want to run frontend tests for a particular app if the backend service for that app is newly deployed.

In all these cases, we want to run a very small amount of tests compared to what the max number of runners are optimised for.

To take this with an example, imagine you have 600 tests and you use 6 github runners to run all these tests on a sharded manner. You have hardcoded this matrix as `shardIndex: [1, 2, 3, 4, 5, 6]`, as shown in the example from Playwright above.

Now for one of the above use cases, imagine the CRON job now needs to run only 5 or 6 tests that are tagged as @smoke-test or @health-check. If you use a reusable workflow, you may still be creating 6 runners to run just these 5 or 6 tests even though just one runner would be enough. If this happens frequently, it would result in wasted billable GitHub runner minutes over a period of time for no value in return.

Wouldn't it be nice if based on the ratio of tests to run to the total tests, we could generate a dynamic GitHub matrix on run time? This is exactly the problem this action was created to solve.

> [!NOTE]
>
> If you want to optimise runners for the use case of changed test files in a pull request, you may want to refer this other action that I created.

## Inputs

```yaml {"id":"01J2XFHJFST5N0A1651KZ5JCAT"}
inputs:
  max-runners:  
    description: 'maximum number of runners to be used'
    required: true
  
  # Example (run all projects): npx playwright test --list    
  pw-cmd-to-list-all-tests:  
    description: 'playwright command to find total tests'
    required: false
    default: 'npx playwright test --list'
  
  # Example (run a single project): npx playwright test --project=chromium --grep=@smoke-test --list 
  pw-cmd-to-list-selected-tests:  
    description: 'playwright command to run tests in this run'
    required: true

```

## Outputs

```yaml {"id":"01J2XFHJFST5N0A1651MMCD9FR"}
outputs:
  dynamic-matrix:
    description: "dynamic matrix to use"
    value: ${{ steps.set-matrix.outputs.dynamic_matrix }}

```

## Example usage

Below is working and tested example of a workflow that uses this action.

```yaml {"id":"01J2NSXS32KV8TSMM4W64D9WMT"}
# Simple workflow for deploying static content to GitHub Pages
# Reference: https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site#publishing-with-a-custom-github-actions-workflow
name: Run Tests on dynamic shards

on:
  # Runs on pushes targeting the default branch

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:
    inputs:
      pw-cmd-to-run-all-tests:
        description: "Command to run all tests"
        required: false
        default: "npx playwright test"
        type: string

      pw-cmd-to-run-selected-tests:
        description: "Command to run selected tests"
        required: true
        type: string

# Grant GITHUB_TOKEN the permissions required to make a Pages deployment
permissions:
  contents: read
  pages: write
  id-token: write

# Allow only one concurrent deployment, skipping runs queued between the run in-progress and latest queued.
# However, do NOT cancel in-progress runs as we want to allow these production deployments to complete.
concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  generate-matrix:
    runs-on: ubuntu-latest
    container:
      image: mcr.microsoft.com/playwright:latest
    outputs:
      dynamic_matrix: ${{ steps.get-dynamic-matrix.outputs.dynamic-matrix }}
    steps:
      - name: Get dynamic matrix
        uses: PramodKumarYadav/playwright-loadbalancer-based-on-tests-to-run@main
        id: get-dynamic-matrix
        with:
          max-runners: 6
          pw-cmd-to-list-all-tests: "${{ inputs.pw-cmd-to-run-all-tests }} --list"
          pw-cmd-to-list-selected-tests: "${{ inputs.pw-cmd-to-run-selected-tests }} --list"

  test:
    timeout-minutes: 60
    needs: generate-matrix
    runs-on: ubuntu-latest
    environment: dev
    container:
      image: mcr.microsoft.com/playwright:v1.47.2-jammy
    strategy:
      fail-fast: false
      matrix:
        runner: ${{ fromJSON(needs.generate-matrix.outputs.dynamic_matrix) }}
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Mark Repository as Safe
        run: git config --global --add safe.directory $GITHUB_WORKSPACE

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18

      - name: Install project dependencies
        run: npm ci

      - name: Run Playwright tests
        # Index is zero based so add one to get shards index as 1,2,3 and thus ratio as 1/3, 2/3, 3/3.
        run: NODE_ENV=dev ${{ inputs.pw-cmd-to-run-selected-tests }} --shard=$((${{ strategy.job-index }} + 1))/${{ strategy.job-total }} --reporter=blob
        env:
          # HOME: /root
          HOST: ${{ secrets.HOST}}

      - name: Upload blob report to GitHub Actions Artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: blob-report-${{ matrix.runner }}-${{ strategy.job-index }}
          path: blob-report
          retention-days: 1

  merge-reports:
    # Merge reports after playwright-tests, even if some shards have failed
    needs: [test]
    if: always()
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 18

      - name: Install project dependencies
        run: npm ci

      - name: Download blob reports from GitHub Actions Artifacts
        uses: actions/download-artifact@v4
        with:
          path: all-blob-reports
          pattern: blob-report-*
          merge-multiple: true

      - name: Merge into HTML Report
        run: npx playwright merge-reports --reporter html ./all-blob-reports

      - name: Upload HTML report
        uses: actions/upload-artifact@v4
        with:
          name: html-report--attempt-${{ github.run_attempt }}
          path: playwright-report
          retention-days: 14

      - name: Setup and Enable Pages
        uses: actions/configure-pages@v4

      - name: Upload artifact playwright-report
        uses: actions/upload-pages-artifact@v3
        with:
          path: "playwright-report"
          retention-days: 30

  # Publish test results job
  publish-results:
    # Add a dependency to the test job
    needs: merge-reports
    if: always()

    # Deploy to the github-pages environment
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    # Specify runner + deployment step
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4 # or specific "vX.X.X" version tag for this action

```

## Boundray value Tests

### Test result when there are no tests to run

![zero-tests-to-run](docs/0-zero-tests-to-run.png)

### Test result when there are very few tests to run

![seven-percent-tests-to-run](docs/1-seven-percent-tests-to-run.png)

### Test result when fifty percent tests to run

![fifty-percent-tests-to-run](docs/2-fifty-percent-tests-to-run.png)

### Test result when all tests to run

![all-tests-are-run](docs/3-all-tests-are-run.png)

## Reference

To create and push new tags:

```sh {"id":"01J2XFHJFT1K765K3D5J6BDSSC"}
pramodyadav@Pramods-Laptop playwright-loadbalancer % git tag -a -m "add your message here" v1                   
pramodyadav@Pramods-Laptop playwright-loadbalancer % git push --follow-tags   
```
