# RunWright

The one-and-only known GitHub action (and solution), that allows you to finish your PlayWright tests in your chosen x minutes, while using optimal minimum number of GitHub runners. **

![fast test run](docs/3k-tests-in-90-seconds.png)

** At the time of writing, there are no known other solutions (paid or open source), that can do this. 

## Why this action?

This [article](https://pramodkumaryadav.github.io/power-tester/blogs/blog2.html) explains the need for this action, in detail. 

## Getting Started
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

## Inputs

```yaml {"id":"01J2XFHJFST5N0A1651KZ5JCAT"}
inputs:
  total-run-time-in-mins:  
    description: 'desired-total-test-run-time-in-mins'
    required: true

  pw-command-to-execute:  
    description: 'playwright command to run tests'
    required: true

```

## Outputs

```yaml {"id":"01J2XFHJFST5N0A1651MMCD9FR"}
outputs:
  dynamic-matrix:
    description: "dynamic matrix to use"
    value: ${{ steps.set-matrix.outputs.dynamic_matrix }}

  test-load-distribution-json:
    description: "test load distribution json"
    value: ${{ steps.calculate-required-runners.outputs.test_load_json }}

  recommended-workers:  
    description: 'optimal number of workers to run tests'
    value: ${{ steps.get-number-of-cpu-cores-to-decide-on-worker-count.outputs.RECOMMENDED_WORKERS }}

```

## Example usage

Follow the instructions in `Getting Started section` that shows how to use this action.

## Reference

To create and push new tags (releases) of this action:

```sh {"id":"01J2XFHJFT1K765K3D5J6BDSSC"}
pramodyadav@Pramods-Laptop runwright % git tag -a -m "add your message here" v1                   
pramodyadav@Pramods-Laptop runwright % git push --follow-tags   

```

## Boundray value Tests

TODO: Add examples that shows if the action can deliver on what it promises, with a low test load and high test load. 

<!-- 
### Test result when there are no tests to run

![zero-tests-to-run](docs/0-zero-tests-to-run.png)

### Test result when there are very few tests to run

![seven-percent-tests-to-run](docs/1-seven-percent-tests-to-run.png)

### Test result when fifty percent tests to run

![fifty-percent-tests-to-run](docs/2-fifty-percent-tests-to-run.png)

### Test result when all tests to run

![all-tests-are-run](docs/3-all-tests-are-run.png) 
-->