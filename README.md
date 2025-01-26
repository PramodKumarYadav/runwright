# RunWright

The one-and-only known GitHub action (and solution), that allows you to finish your PlayWright tests in your chosen x minutes, while using optimal minimum number of GitHub runners. **

![fast test run](docs/3k-tests-in-90-seconds.png)

** At the time of writing, there are no known other solutions (paid or open source), that can do this.

## Without RunWright

## Uneven run time on each runner

![uneven-run-time](docs/uneven-run-time.png)

## Fixed runners that does not scale up or down based on test load

![fixed-runners](docs/fixed-runners.png)

## With RunWright

## Even test load distribution based on desired run time to finish tests.

![even-load-distribution](docs/even-load-distribution.png)

## Dynamic runners that scale up or down based on test load

![dynamic-scaling-of-runners](docs/dynamic-scaling-of-runners.png)

## 🚀 Core features

- **🚀 Faster than Playwright Sharding**: ✅
  - **Smart load balancing based on execution time and not just test count**: We create "bundle of tests and runners" based on:
    - The time each test takes. 
    - The total exepected time for run provided by user. 
    - The cores availablity of runners. 

    This makes make our total run time predictable and fast.
  - **Distributed Run**: The number of bundles translates to number of runners.
  - **Parallel Run**: The number of cores translates to number of workers (where workers = cores/2).

- **↗️↘️ Dynamic sharding over Playwrights hard coded Sharding**: ✅
  - Although this is not directly a Playwright sharder shortcoming but since not everyone is an expert with GitHub actions and the [playwright sharding github example](https://playwright.dev/docs/test-sharding#github-actions-example)
  hard codes the number of shards in the workflow file, most teams end up using the example as-is in their projects and thus results into a inefficient hard coded runner strategy.  

  Teams often increase runners to finish tests faster but when they do maintenance or add a few new tests, they end up spinning all those runners, resulting into waste and under utilisation of runners. 

- **💸 Cheaper than Playwright Sharding**: ✅
  - Since we dynamically scale runners up and down to always create only the bare minimum runners required to do the job, we avoid waste in terms of CI minutes. 
  For comparision, with playwright sharding teams end up hard coding more runners to bring down execution time and as a side affect increase cost to the company.

- **🌿 Greener than Playwright Sharding**: ✅
  - Since we always only create the exact amount of runners we need to do the job (no less, no more), and since each runner is optimially utilised to finish all runners at approx same time, 
  this is also a very efficient and thus greener alternative to Playwright Sharding. 

## Why this action?

This [article](https://pramodkumaryadav.github.io/power-tester/blogs/blog2.html) explains, in detail, the need for this action and the problem it solves.

## Getting Started

There are 3 main steps involved:

### Step1: One time setup (in your test project)

- [Install latest version of Node (or at least >=18)](https://nodejs.org/en)

- [Install husky in your test project](https://typicode.github.io/husky/get-started.html).

- Add a `pre-commit` hook file as shown [here](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.husky/pre-commit).
  - This will run `--only-changed` tests on local commits.

- Copy [state-reporter.js](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state-reporter.js) file and put it in the root repository.
  - This will create a `[state.json](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state.json)` file that contains the mapping of test path and the time it took to run (in ms).

- Update [playwright.config.ts](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/playwright.config.ts) file reporters to include this reporter as shown below.
`reporter: [["list"], ["html"], ["github"], ["./state-reporter.js"]],`

- Add a `post-commit` hook file as shown [here](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.husky/post-commit).
  - This will automatically add `[state.json](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state.json)` file to the branch.

- Add a [reusable workflow](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/reusable-workflow.yml) that can take inputs from user to run playwright commands and finish tests in x mins.

- Add a example [trigger workflow](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-all-tests-on-push-to-main.yml) that shows how to use the reusable workflow to run desired tests.

### Step2: Run tests based on the defined triggers (push to main, pull_request to main, schedule etc all)

- Create an event that triggers the workflow and verify if the tests finish in approx same projected time.
 - For now, you can use the [trigger workflow](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-all-tests-on-push-to-main.yml) you copied above to trigger and run these tests. Fork the repo and push something on your main branch to trigger tests.

### Step3: Report found issues

- If you find any issues, use the [issues page](https://github.com/PramodKumarYadav/runwright/issues) to raise them.

## Things to remember

- This action is made to work with playwright option `fully parallel = true`. The action is not meant to deal with tests run in `serial` or `default` mode and thus can have side effects if your tests are not running fully parallel. This is intended to be addressed in one of future releases.

- Do not use sharding related commands in the input playwright command to run; since this solution is meant to overcome the flaws of sharding. Using sharding again would introduce those short comings again.

- If you are using custom powerful GitHub runners, use the same custom runner type for job that evaluates "RunWright" then what you would use in subsequent job for running tests.

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

Follow the instructions in `Getting Started section` that shows how to use this action. Other than that, below are some common examples:

- [run only new or updated tests in a pull request](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-only-touched-tests-on-pull-requests.yml)
- [run all tests on a push to main](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-all-tests-on-push-to-main.yml)
- [run selected tests on demand](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-any-tests-on-demand.yml)

## Reference

To create and push new tags (releases) of this action:

```sh {"id":"01J2XFHJFT1K765K3D5J6BDSSC"}
pramodyadav@Pramods-Laptop runwright % git tag -a -m "add your message here" v1                   
pramodyadav@Pramods-Laptop runwright % git push --follow-tags   

```

## Boundary value Tests

Below are some tests to verify for some edge case scenarios and validate that the action works as expected. 

### Test result when there are no tests to run

![zero-tests-to-run](docs/0-zero-tests-to-run.png)

### Test result when there are very few tests to run

![just a few tests](docs/less-tests-equals-less-runners-and-less-time.png)

### Test result for approx (~1.5k) tests in total expected time of 2 mins

![finish all 1.5k tests in 2 mins](docs/finish-1.5k-tests-in-2-mins-jobs.png)

![finish all 1.5K tests in predefined time](docs/finish-1.5k-tests-in-2mins.png)

[all 1.5k tests finished in less than 2 mins](https://www.loom.com/share/c13973941f60401797d840a31e3a6767?sid=c8741b3b-4863-4509-8d0a-43fb7aad8945)

### Test result for approx (~3k) tests in total expected time of 2 mins

![finish all 3k tests in predefined time](docs/finish-3k-tests-in-2mins.png)

[all 3k tests finished in approx 2 mins](https://www.loom.com/share/7e2a3f093d264619886c6b261696af86?sid=d75b3fb8-0e11-4573-bc00-f575c99db6b9)

## Troubleshooting

- It could be a good idea to generate the `[state.json](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state.json)` file from scratch every few days or weeks to avoid having redundant test path and names.

## Love this action? Want to support?

![coffee](docs/coffee.png) 

https://buymeacoffee.com/power.tester

[![Sponsor PramodKumarYadav](https://img.shields.io/badge/🤍%20Sponsor%20Me-%23EA4AAA.svg?style=for-the-badge&logo=github)](https://github.com/sponsors/PramodKumarYadav)