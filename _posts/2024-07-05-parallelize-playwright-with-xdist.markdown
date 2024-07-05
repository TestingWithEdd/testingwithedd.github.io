---
layout: post
title:  "Parallelize Playwright with Xdist!"
date:   2024-07-05
categories: testing playwright
---

UI-driven E2E tests are usually the slowest tests you can have, often observed when your CI pipeline spends 20 minutes running them compared to ~30 seconds for your other tests. There are a few tricks to reduce this slowness: running a subset of tests, using APIs to set up/tear down data rather than going through the UI, caching data from previous tests like auth tokens, and parallelizing tests. 

This blog post will focus on the latter, how to do it, and a few tricks I learned along the way :)

A few assumptions here - you're using Python and Pytest. The tool we will use here is `xdist`, installed like so:

```
pip install pytest-xdist
```

Out of the box, we can now parallelize tests with `pytest --numprocess auto`, exactly as the Playwright docs tell us. This will speed up the execution time of your tests! You can customize this to run a specific number of processes with `--numprocess {N}`, use only logical processes with `--numprocess logical`, or set an environment variable to set this value with `PYTEST_XDIST_AUTO_NUM_WORKERS={N}`. 

For some of you, this may actually be enough - try it and see what happens! I'll go into a few more advanced details below.

### How does xdist work?
Xdist follows the controller/worker pattern where it spins up a variable number of workers that are pytest runners. Each worker gets the full list of tests to run and does some verification by sending back a list of all the test IDs to check each worker has all the tests required.

Once the controller knows the workers all have the full set of tests, it will do one of two things depending on if `dist-mode` is `each` or `load` (or a variation of load).

`each`: the controller sends the full list of test indexes to each node, so basically each node will run all the tests.

`load`: the controller distributes ~25% of the tests to all workers via round-robin scheduling and then distributes the rest as and when workers are available.

Why bother sending all the tests to each worker? This is useful if each worker is a different platform and you are using this to test across multiple systems. Because we're interested in parallelizing tests here, we will be focusing on `load`. The default setting is `--dist=load`.

### Avoiding disaster
If you try to run all your tests in parallel, randomly allocated to workers, you open the door to flaky tests. For example, if you are logging in as the same user, you don't want two tests doing conflicting actions like changing settings to different values. Introducing groups!

```
def test_setting_private_to_on(page):
    # do something...

def test_setting_private_to_off(page):
    # do something...
```

These two tests do totally different things that, if executed at the same time, will break the other's assertion. We need to group these so they run on the same worker sequentially.

```
@pytest.mark.xdist_group(name="settings")
def test_setting_private_to_on(page):
    # do something...

@pytest.mark.xdist_group(name="settings")
def test_setting_private_to_off(page):
    # do something...
```

We can run the tests taking groups into account with `pytest -n auto --dist=loadgroup`, which acts the same as `pytest -n auto --dist=load` except guaranteeing groups are sent to the same worker. If the tests are already in the same file, you can get away with `pytest -n auto --dist=loadfile`, but if you refactor this into different files, you'll find they start breaking each other.

### Issues with xdist
One heads-up with xdist is that it will swallow your `logging`. On top of this, logging will not always be ordered. If you need to see your prints, you need to write to `sys.stderr`. You can hack this with `sys.stdout = sys.stderr`.

The way I personally handle this is running the tests with xdist and, on failure, re-running the failing test regularly with logging. It still saves time in test execution, vastly saves time in debugging half-swallowed logs, and avoids redirecting out to err.

--------
Xdist is a great way of quickly speeding up your Playwright tests (and all of this applies to other types of testing, including regular pytest unit tests). Just remember to group tests that, if run at the same time, could break each other. Testing flaky UI tests is a pain already, making them even more non-deterministic is a great way to waste a workday!