# RunWright

The one-and-only**, GitHub action (and solution), that allows you to finish your PlayWright tests in your chosen x minutes, while using optimal minimum number of GitHub runners. 

** At the time of writing, there are no known other solutions (paid or open source), that can do this. 

## Why this action?

This [article](https://pramodkumaryadav.github.io/power-tester/blogs/blog2.html) explains the need for this action, in detail. 

## How does this work?
There are 3 main steps involved. 

### Step1: Do one time setup. 

- [Install husky](https://typicode.github.io/husky/get-started.html).

- Add a pre-commit hook as shown [here](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.husky/pre-commit). 
  - This will run `--only-changed` tests on local commits. 

- Copy [state-reporter.json](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state-reporter.js) file and put it in the root repository.
  - This will create a `state.json` file that contains the mapping of test path and the time it took to run (in ms).

- Update [playwright.config.ts](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/playwright.config.ts) file reporters to include this reporter as shown below. 
`reporter: [["list"], ["html"], ["github"], ["./state-reporter.js"]],`

- Add a post-commit as shown [here](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.husky/post-commit).
  - This will automatically add `state.json` file to the branch.

- Add a [reusable workflow](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/reusable-workflow.yml) that can take inputs from user to run playwright commands and finish tests in x mins. 

- Add a example [trigger workflow](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-all-tests-on-push-to-main.yml) that shows how to use the reusable workflow to run desired tests.

### Step2: Run tests based on the defined triggers (push to main, pull_request to main, schedule etc all)

- Create an event that triggers the workflow and verify if the tests finish in approx same projected time. 
 - For now, you can use the [trigger workflow](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-all-tests-on-push-to-main.yml) you copied above to trigger and run these tests. Fork the repo and push something on your main branch to trigger tests. 

### Step3: Report found issues.

- If you find any issues, use the [issues page](https://github.com/PramodKumarYadav/runwright/issues) to raise them. 

## Things to rememember

- This action is made to work with playwright option `fully parallel = true`. 

- Do not use sharding related commands in the input playwright command to run since this solution is meant to overcome the flaws of sharding. Using sharding again would introduce them again. 

- If you are using custom powerful GitHub runners, use the same custom runner type for job that evaluates "RunWright" then what you would use for running tests. 

- The action is not meant to deal with tests run in `serial` or `default` mode and thus can have side effects if your tests are not running fully parallel. This may be addressed in one of future releases. 

TODO: More documentation to be updated in a day or two. 

<!-- ## Inputs

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
``` -->
