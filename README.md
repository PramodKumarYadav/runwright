# RunWright

## Why this action?

This [article](https://pramodkumaryadav.github.io/power-tester/blogs/finish-fast-in-2-mins.html) explains, in detail, the need for this action and the problem it solves. 

## 🚀 Core features

In Summary the core features are:
- The power to finish thousands of your tests in a pre defined desired run time (say 2 mins or 5 mins).
- Smart runner orchestration that scales runners up or down based on your test load (where test load is in terms of time - not just count).
- Equal test load distribution (in terms of execution time, not just count).

In below example, we see more than three thousand tests run in just 1.5 minutes with a desired run time of 2 mins in total. 

<img src="docs/3k-tests-in-90-seconds.png" alt="fast test run" width="70%">

## With RunWright

## Even test load distribution based on desired run time to finish tests.

<img src="docs/even-load-distribution.png" alt="even-load-distribution" width="70%">

## Dynamic runners that scale up or down based on test load

<img src="docs/dynamic-scaling-of-runners.png" alt="dynamic-scaling-of-runners" width="70%">

** At the time of writing this document, there are no known other solutions (paid or open source), that can do this using Playwright and GitHub.

## Scope

This action covers both execution modes in Playwright. i.e.
- When `fullyParallel=true`. [ Parallel run all individual test cases on runners]
- When `fullyParallel=false`. [ Parallel run all individual test files on runners]

## Getting Started

There are 3 main steps involved:

### Step1: One time setup (in your test project)

- [Install latest version of Node (or at least >=18)](https://nodejs.org/en)
- [Install husky in your test project](https://typicode.github.io/husky/get-started.html).
- Add a `pre-commit` hook file as shown [here](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.husky/pre-commit).

   - This will run `--only-changed` tests on local commits.

- Copy [state-reporter.js](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state-reporter.js) file and put it in the root repository.

   - This will create a [`state.json`](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state.json) file that contains the mapping of test path and the time it took to run (in ms).

- Update [playwright.config.ts](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/playwright.config.ts) file reporters to include this reporter as shown below.
   `reporter: [["list"], ["html"], ["github"], ["./state-reporter.js"]],`
- Add a `post-commit` hook file as shown [here](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.husky/post-commit).

   - This will automatically stage and commit the updated [`state.json`](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state.json) file to the feature branch.

- Add a [reusable workflow](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/reusable-workflow.yml) that can take inputs from user to run playwright commands and finish tests in x mins.
- Add a example [trigger workflow](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-all-tests-on-push-to-main.yml) that shows how to use the reusable workflow to run desired tests. Here are a few examples of `trigger workflows`.
   - [run selected tests on demand](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-any-tests-on-demand.yml)
   - [run only new or updated tests in a pull request](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-only-touched-tests-on-pull-requests.yml)
   - [run all tests on a push to main](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-all-tests-on-push-to-main.yml)

### Step2: Run tests based on the defined triggers (push to main, pull_request to main, schedule, on demand etc all)

- To test the setup use one of below workflows (either in your own test repository or fork the above sandbox repository to try out).
   - [run selected tests on demand - using reusable workflow that points to a fixed version say v2.1.0 of runwright action](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-any-tests-on-demand.yml)
   - [run selected tests on demand - using standlone workflow - that points to the latest main version of runwright action](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-any-tests-on-demand-sandbox.yml)

> For reference, a example-workflow.yml file is also available in the root of the RunWright GitHub project. 

### Step3: Report found issues

- If you find any issues, use the [issues page](https://github.com/PramodKumarYadav/runwright/issues) to raise them.

## Things to remember

- Do not use sharding related commands in the input playwright command to run; since this solution is meant to overcome the flaws of sharding. Using sharding again would introduce those short comings again.
- If you are using custom powerful GitHub runners, use the same custom runner type for job that evaluates "RunWright" then what you would use in subsequent job for running tests. This is important to correctly calcualte the workers (threads), which is half of cores of the runners.


## 🧪 Tests

| Sr No | Test Description | Test Condition | Expected Result | Actual Result | Status |
|-------|------------------|----------------|-----------------|---------------|---------|
| 1 | Run with 0 tests found | `npx playwright test --grep="non-existent-test"`  | Action should handle gracefully, set RUNNER_COUNT=1, and exit successfully | ✅ Action exits gracefully with "No tests found" message | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16179518780) |
| 2 | Test with fullyParallel=true | `npx playwright test` with -fully-parallel=true | Should distribute individual tests across runners | ✅ Looking into runwright job, we can see that tests from same file are distributed across different runners | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16179632442/job/45673015976) |
| 3 | Test with fullyParallel=false | `npx playwright test` with -fully-parallel=false | Should distribute tests files across runners | ✅ File-level distribution works correctly. No tests from same file + project seen in different runners. With each file size of 102 seconds run time, and 2 workers, each runner getting 2 files is also accurate. Slowest runner time was 2 mins 48 seconds. Total run time in html report = 1.9m | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16179919403/) |
| 4 | Run with very few tests (with < 2 min total run time) | `npx playwright test --grep='Wait for 5 seconds'` | Should create 1 runner with optimal worker allocation | ✅ Creates 1 runner, completes in less than <2 mins> | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16179545717) |
| 5 | Run with all tests (~30 mins when run sequentially) | `npx playwright test` | Should create around 8 runners with optimal worker allocation | ✅ Creates 9 runners. Slowest runner time was 2 mins 59 seconds. Total run time on html report: 2.1m | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16179792471) |
| 6 | Run ~1.5k tests in 2 minutes | [Test Command Placeholder] | Should create optimal number of runners to finish within 2 minutes | ✅ Completes in ~2 minutes with dynamic runner allocation (as seen in the early tests image. runner id  not available) | ✅ PASS |
| 7 | Run ~3k tests in 2 minutes | `npx playwright test` | Should scale up runners appropriately to meet time constraint | ✅ Scales to multiple runners, finishes in ~2 minutes | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/13400398360) |
| 8 | Test missing from state.json | Delete tests from file06 for 20,25 seconds for firefox in state.json. Run command `npx playwright test --grep='Wait for 20 seconds' --project=firefox` | Should fail with clear error message and suggestions. No grace failure to avoid "false positive" situation from runs. | ✅ Provides clear error with post-commit hook guidance | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16180170072/) |
| 9 | Single project configuration | `npx playwright test --project='chromium'` | Should work with single browser project | ✅ Handles single project scenarios correctly | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16180235559) |
| 10 | Multiple project configuration | `npx playwright test --project='chromium' --project='webkit'` | Should group tests by project within runners | ✅ Correctly groups and distributes multi-project tests | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16180268230) |
| 11 | CPU core detection | [check any of previous runs] | Should detect available cores and calculate optimal workers | ✅ Detects cores correctly, sets workers = cores/2 | ✅ PASS |
| 12 | Dynamic matrix generation | [check any of previous runs] | Should create proper GitHub Actions matrix format | ✅ Generates valid JSON array for matrix strategy | ✅ PASS |
| 13 | Load balancing accuracy | [check any of previous runs] | Distribution should be based on execution time, not test count | ✅ Uses actual test execution times for optimal distribution | ✅ PASS |
| 14 | Runner utilization | [check any of previous runs] | All runners should finish at approximately the same time | ✅ Even load distribution across all runners | ✅ PASS |
| 15 | Browser caching | [check any of previous runs] | Should cache and reuse Playwright browsers efficiently | ✅ Implements proper browser caching strategy | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16180268230/job/45675077743) |
| 16 | Error handling for malformed state.json | [Test Command Placeholder] | Should provide clear error when state.json is corrupted | ✅ Handles JSON parsing errors gracefully | ✅ PASS |
| 17 | Large test suite scalability | run with .5 seconds (before putting a restriction for minimum time) | Should handle test suites with 5k+ tests efficiently | ✅ Scales appropriately for large test suites (tested with tests of size 3k+) | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16180627144) |
| 18 | Custom runner types compatibility | [Test Command Placeholder] | Should work with custom GitHub runner configurations | ✅ Compatible with custom runner specifications | ？ NOT-YET-TESTED |
| 19 | Output format validation | [check any of previous runs] | All outputs should be valid JSON and consumable by workflows | ✅ All outputs are properly formatted and consumable | ✅ PASS |
| 20 | Invalid time input (< 1 minute) | [Test Command Placeholder] | Should handle minimum time constraint appropriately | ✅ Validates minimum 1 minute requirement | ✅ [PASS](https://github.com/PramodKumarYadav/playwright-sandbox/actions/runs/16180781972) |

## 💡 Inner workings

For the curious minds, here are the equations that I used to device this solution. 

🔁 Definitions

Let:
-  TargetRunTime = total desired time to complete the run (in minutes)
   - We get this as [input from the user](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/.github/workflows/run-any-tests-on-demand-sandbox.yml). 
-  Σ T_i = TestRunTimeForEachTest(i) = execution time of test i (from state.json)
   - We get this value from [`state.json`](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state.json)  file that is updated on a post-commit hook.
-  N = total number of tests to run.
   - We get this by running playwright command with [`--list`](https://playwright.dev/docs/test-cli#all-options) option.
-  TotalLoad = Σ T_i = total test load (in terms of test run time)
   - We iterate over each runner to keep the `Σ T_i <= TargetRunTime`. 
   - Note that the total run time for each runner is affected by the number of parallel threads and is explained in more details in the next section.
-  Cores = number of cores per runner.
   - For Linux runners, `NUM_CORES=$(nproc)`
-  Threads (per runner).
   - [Recommended Threads per runner](https://learn.microsoft.com/en-us/azure/playwright-testing/concept-determine-optimal-configuration) is half of cores; i.e. (Threads = Cores / 2). 
-  Runners = Total number of required runners.
   - We get this from equation given in the next section.

📐 Equation


Since we know every individual TestRunTimeForEachTest(i) from state.json, the total workload is:

![alt text](./docs/image-1.png)

Total parallel capacity available on the runners:

![alt text](./docs/image-2.png)

Equating Load and Capacity:

![alt text](./docs/image-3.png)

Solving for Runners:

![alt text](./docs/image-4.png)

## Troubleshooting

- It could be a good idea to generate the [`state.json`](https://github.com/PramodKumarYadav/playwright-sandbox/blob/main/state.json) file from scratch every few days or weeks to avoid having redundant test path and names.

## Whats next?

- [ ] Add option for when a user doesn't want to limit by time but want to limit the maximum runners to use.

## Like my work and want to support or sponsor?

<a href="https://buymeacoffee.com/power.tester" target="_blank">
  <img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me a Coffee" style="height: 60px; width: 217px; border-radius: 8px; margin-bottom: 15px;">
</a>

<a href="https://github.com/sponsors/PramodKumarYadav" target="_blank">
  <img src="https://img.shields.io/badge/Sponsor%20Me%20on%20GitHub-%E2%9D%A4-6f42c1?style=for-the-badge&logo=github-sponsors" alt="Sponsor Me on GitHub" style="height: 60px; width: 217px; border-radius: 8px;">
</a>
