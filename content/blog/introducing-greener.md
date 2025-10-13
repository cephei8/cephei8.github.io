+++
title = "Introducing Greener: store/view test results"
date = 2025-11-21
draft = false
+++

## Problem {#problem}

Test frameworks usually produce custom test output by default (along with exit code).<br />
For example, pytest (test framework for Python) gives something like this:

```nil
tests/test_apikey_controller.py::test_create[db_sqlite] PASSED        [ 25%]
tests/test_apikey_controller.py::test_get[db_sqlite] PASSED           [ 50%]
tests/test_apikey_controller.py::test_list[db_sqlite] PASSED          [ 75%]
tests/test_apikey_controller.py::test_delete[db_sqlite] PASSED        [100%]
```

This is fine if a number of tests is small, or we mostly care about overall test result - if all the tests in the test session passed.

However, once we need to investigate test failure, especially for larger/longer test sessions (e.g. end-to-end tests), often the following questions need answers:

-   What are the other tests that failed? Did they fail because of the same reason?
-   How did this test behave over time? Is it flaky?
-   How to match applications logs with specific tests?

It may be hard or not possible to answer these questions without additional infrastructure/tools.<br />
Normally, the following comes to help:

-   Export test results as JUnit XML and visualize the test results in the CI system being used (Jenkins, Azure DevOps etc.)
-   Export test results as JUnit XML, process and store them in database, and use Redash/Grafana for visualization
-   Instrument test framework to log additional details (e.g. start/end time), and then use the test log in addition to test results
-   Build ad-hoc solution that incorporates the previous points, and adds more convenience e.g. by adding each search for tests etc.

The problem is that these additional infrastructure/tools require implementation, setup and maintenance, and that takes resources/focus from doing work that brings the actual value (like building a product).


## Introducing Greener {#introducing-greener}

[Greener](https://cephei8.github.io/greener/) is a platform for storing and viewing test results.

It strives to:

-   Be a simple and focused tool
-   Be easy to integrate
    -   The platform is hosted as a Docker container that works with SQLite/PostgreSQL/MySQL
    -   Test framework plugins don't require code changes
-   Require "zero" setup and work out of the box
-   No "lock in" - switch to some other solution at any time (simply disable test framework plugins)
-   Enable further automation and/or advanced use cases via API
-   Bring simple but powerful query language
    -   Find specific tests by certain properties (name, session, custom labels)
    -   Group results
-   Enable adding custom metadata to test sessions and test cases (labels, arbitrary JSON)


## Motivational examples {#motivational-examples}


### Track a subset of tests {#track-a-subset-of-tests}

You can prepare a query to select a certain subset of tests and group the results by session.<br />
In this case you will be able to see how this specific subset of tests behaved over time.<br />
Additionally, group by label e.g. "target" label in case cross-platform builds.


### Match application logs to test results {#match-application-logs-to-test-results}

This example requires code changes, but it is can be powerful for end-to-end tests.<br />
The idea is to create OpenTelemetry trace context, and both store it in test case metadata and pass with requests to your API.<br />
It will allow finding traces for specific test cases and finding a test case for specific trace.


## Links {#links}

-   [Documentation](https://cephei8.github.io/greener/)
-   Packages
    -   [Platform Docker image](https://hub.docker.com/r/cephei8/greener)
    -   [pytest plugin](https://pypi.org/project/pytest-greener/)
    -   [Jest reporter](https://www.npmjs.com/package/jest-greener)
-   Repositories
    -   [Platform (main repository)](https://github.com/cephei8/greener)
    -   [Native library for implementing test reporters](https://github.com/cephei8/greener-reporter)
    -   [JavaScript package for implementing test reporters](https://github.com/cephei8/greener-reporter-js)
    -   [Python package for implementing test reporters](https://github.com/cephei8/greener-reporter-py)
    -   [Jest reporter](https://github.com/cephei8/jest-greener)
    -   [pytest plugin](https://github.com/cephei8/pytest-greener)


## Feedback and contact information {#feedback-and-contact-information}

Let me know your thoughts at [Greener GitHub Discussions](https://github.com/cephei8/greener/discussions).<br />
You can find me personally [here](https://cephei8.github.io/contact/).
