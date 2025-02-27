---
title: "Testing plugins"
slug: "test-custom-plugin-goal"
excerpt: "How to write tests for your custom plugin code"
hidden: true
createdAt: "2022-02-07T05:44:28.620Z"
updatedAt: "2022-02-07T05:44:28.620Z"
---
# Introduction

In this tutorial, we'll learn how to test the custom plugin we wrote earlier. Pants documentation provides comprehensive coverage of the [plugin testing](doc:rules-api-testing) and this tutorial should help you get started writing own tests.

Most of the plugin code that needs to be tested is in the following files:
* `rules.py` where we implemented how a `VERSION` file needs to be read and how to use a `version_file` BUILD target
* `tailor.py` where we taught the `tailor` goal about the `VERSION` files and generation of `version_file` targets

To author a test suite, it may make sense to write a very high level test first to confirm our code does what we expect. Let's write some [integration tests for Pants](doc:rules-api-testing#approach-4-run_pants-integration-tests-for-pants) so that we could run our goal from a test!

## Testing with a complete Pants process

Pants provides a convenient way to run a full Pants process as it would run on the command line. Writing such a test would be equal to having, say, a Shell script to confirm that the output of the `./pants project-version myapp:` command is `{"path": "myapp/VERSION", "version": "0.0.1"}`. Keep in mind that running custom scripts with this type of tests would require having a Pants repository set up (including the `pants.toml` configuration), creating `BUILD` metadata files and so on. When writing custom acceptance tests using `pants.testutil` package, you, in contrast, don't have to worry about that and can focus on testing your plugin logic in the very minimalistic environment containing only what's absolutely necessary to run your plugin code. 

In the following code snippet, we define a set of files to be created (in a temporary directory that Pants manages for us), the backends to be used (Python and our custom plugin), and a Pants command to be run. By reading the `stdout` of a process, we can confirm the plugin works as expected (conveniently ignoring any unrelated warnings that Pants may have produced).

```python
import json
from pathlib import Path

from pants.testutil.pants_integration_test import run_pants, setup_tmpdir

build_root_marker = Path.cwd().joinpath("BUILDROOT")


def test_reading_project_version_target() -> None:
    """Run a full Pants process as it would run on the command line."""
    project_files = {
        "project/BUILD": "version_file(source='VERSION')",
        "project/VERSION": "10.6.1",
    }
    # This is a limitation of the current implementation.
    # See https://github.com/pantsbuild/pants/issues/12760.
    build_root_marker.touch()
    with setup_tmpdir(project_files) as tmpdir:
        result = run_pants(
            [
                (
                    "--backend-packages="
                    "['pants.backend.python', 'internal_plugins.project_version']"
                ),
                "project-version",
                "--as-json",
                f"{tmpdir}/project:",
            ],
        )
        result.assert_success()
        assert result.stdout.strip() == json.dumps(
            {"path": f"{tmpdir}/project/VERSION", "version": "10.6.1"}
        )
    build_root_marker.unlink()
```

These tests do not need any special bootstrapping and can be run just like any other tests you may have in the repository with the `test` goal. They, however, are slow, and if there are lots of test cases to check (e.g. you want to test usage of flags and targets with various fields set), it may soon become impractical to run them often enough. You would most likely want to test your plugin logic in a more isolated fashion.

## Testing goal rules

You can exercise the goal rule by using [`rule_runner.run_goal_rule()`](doc:rules-api-testing#testing-goal_rules) which runs very fast and does not start a full Pants process. In the test below, we register all rules from the `project_version` plugin with the `RuleRunner` so that the engine can find them when a test is run. These tests scale nicely and if your plugins are fairly simple, they may suffice.

```python
import pytest
from pants.engine.internals.scheduler import ExecutionError
from pants.testutil.rule_runner import RuleRunner

from internal_plugins.project_version.rules import ProjectVersionGoal
from internal_plugins.project_version.rules import rules as project_version_rules
from internal_plugins.project_version.target_types import ProjectVersionTarget


@pytest.fixture
def rule_runner() -> RuleRunner:
    return RuleRunner(
        rules=project_version_rules(), target_types=[ProjectVersionTarget]
    )


def test_project_version_goal(rule_runner: RuleRunner) -> None:
    """Test a `project-version` goal using VERSION files."""
    rule_runner.write_files(
        {
            "project/VERSION": "10.6.1",
            "project/BUILD": "version_file(source='VERSION')",
        }
    )
    result = rule_runner.run_goal_rule(
        ProjectVersionGoal, args=["--as-json", "project:"]
    )
    assert result.stdout.splitlines() == [
        '{"path": "project/VERSION", "version": "10.6.1"}'
    ]

    # Invalid version string is provided.
    rule_runner.write_files(
        {
            "project/VERSION": "foo.bar",
            "project/BUILD": "version_file(source='VERSION')",
        }
    )
    with pytest.raises(ExecutionError):
        rule_runner.run_goal_rule(ProjectVersionGoal, args=["project:"])
```

## Testing individual rules

If your plugin is more sophisticated, and there are many rules, you may want to test them in isolation. In our plugin, there are a couple of rules we could write tests for. For example, the `get_project_version_file_view` rule reads a target and returns an instance of `dataclass`, namely `ProjectVersionFileView`. This looks like a good candidate for a very isolated test.

```python
import pytest
from pants.build_graph.address import Address
from pants.testutil.rule_runner import QueryRule, RuleRunner

from internal_plugins.project_version.rules import (
    ProjectVersionFileView,
    get_project_version_file_view,
)
from internal_plugins.project_version.target_types import ProjectVersionTarget


@pytest.fixture
def rule_runner() -> RuleRunner:
    return RuleRunner(
        rules=[
            get_project_version_file_view,
            QueryRule(ProjectVersionFileView, [ProjectVersionTarget]),
        ],
        target_types=[ProjectVersionTarget],
    )


def test_get_project_version_file_view(rule_runner: RuleRunner) -> None:
    """Test plugin rules in isolation (not specifying what rules need to be run)."""
    rule_runner.write_files(
        {"project/VERSION": "10.6.1", "project/BUILD": "version_file(source='VERSION')"}
    )
    target = rule_runner.get_target(Address("project", target_name="project"))
    result = rule_runner.request(ProjectVersionFileView, [target])
    assert result == ProjectVersionFileView(path="project/VERSION", version="10.6.1")
```

Since we have extended the `tailor` goal to generate `version_file` targets in the directories containing `VERSION` files, let's write a test to confirm the goal does what we want. For this, we can continue using the [`RuleRunner`](doc:rules-api-testing#running-your-rules). Let's create a temporary build root, write necessary files, and then ask Pants to get a list of targets that it would have created for us.

It's often very difficult to know how testing of a particular functionality is done, so it's worth taking a look at the Pants codebase. For instance, this `tailor` test has been adopted from this [test suite](https://github.com/pantsbuild/pants/blob/8cb558592d00b228182e6bbcb667705dad73bb95/src/python/pants/backend/cc/goals/tailor_test.py#L1-L0).

```python
from pathlib import Path

import pytest
from pants.core.goals.tailor import AllOwnedSources, PutativeTarget, PutativeTargets
from pants.engine.rules import QueryRule
from pants.testutil.rule_runner import RuleRunner
from pants.util.frozendict import FrozenDict

from internal_plugins.project_version.tailor import (
    PutativeProjectVersionTargetsRequest,
    rules,
)
from internal_plugins.project_version.target_types import ProjectVersionTarget


@pytest.fixture
def rule_runner() -> RuleRunner:
    return RuleRunner(
        rules=[
            *rules(),
            QueryRule(
                PutativeTargets, (PutativeProjectVersionTargetsRequest, AllOwnedSources)
            ),
        ],
        target_types=[ProjectVersionTarget],
    )


def test_find_putative_avnpkg_files_targets(rule_runner: RuleRunner) -> None:
    """Test generating `version_file` targets in a project directory."""
    files = {
        "project/dir1/VERSION": "10.6.1",
        "project/dir2/file.txt": "",
        "project/dir3/VERSION": "10.7.1",
        # Note that dir3/VERSION already has the target and should be ignored.
        "project/dir3/BUILD": "version_file(source='VERSION')",
    }
    rule_runner.write_files(files)
    for filepath, _ in files.items():
        assert Path(rule_runner.build_root, filepath).exists()

    putative_targets = rule_runner.request(
        PutativeTargets,
        [
            PutativeProjectVersionTargetsRequest(
                ("project/dir1", "project/dir2", "project/dir3"),
            ),
            # Declare that all these files in the project are already owned by targets.  
            AllOwnedSources(["project/dir2/file.txt", "project/dir3/VERSION"]),
        ],
    )

    assert (
        PutativeTargets(
            [
                PutativeTarget(
                    path="project/dir1",
                    name="project-version-file",
                    type_alias="version_file",
                    triggering_sources=("VERSION",),
                    owned_sources=("VERSION",),
                    kwargs=FrozenDict({}),
                    comments=(),
                )
            ]
        )
        == putative_targets
    )
```

## Unit testing for rules

Finally, if your plugin is very complex and would benefit from a more rigorous testing, you may consider writing [unit tests for the rules](doc:rules-api-testing#approach-2-run_rule_with_mocks-unit-tests-for-rules) where some parts of the rules are going to be patched with mocks. For instance, there's `get_git_repo_version` rule which calls Git (in a subprocess) to describe the repository status. We could mock the `Process` call to make sure the inline logic of the rule is correct instead.

```python
from unittest.mock import Mock

from pants.base.build_root import BuildRoot
from pants.core.util_rules.system_binaries import (
    BinaryPath,
    BinaryPathRequest,
    BinaryPaths,
)
from pants.engine.process import Process, ProcessResult
from pants.testutil.rule_runner import MockGet, run_rule_with_mocks

from internal_plugins.project_version.rules import GitTagVersion, get_git_repo_version


def test_get_git_version() -> None:
    """Test running a specific rule returning a GitVersion."""

    def mock_binary_paths(request: BinaryPathRequest) -> BinaryPaths:
        return BinaryPaths(binary_name="git", paths=[BinaryPath("/usr/bin/git")])

    def mock_process_git_describe(process: Process) -> ProcessResult:
        return Mock(stdout=b"10.6.1\n")

    result: GitTagVersion = run_rule_with_mocks(
        get_git_repo_version,
        rule_args=[BuildRoot, ""],
        mock_gets=[
            MockGet(
                output_type=BinaryPaths,
                input_type=BinaryPathRequest,
                mock=mock_binary_paths,
            ),
            MockGet(
                output_type=ProcessResult,
                input_type=Process,
                mock=mock_process_git_describe,
            ),
        ],
    )
    assert result == GitTagVersion("10.6.1")
```

You could write the helper functions returning the mock objects as lambdas, if you like, for instance:

```python
MockGet(
    output_type=BinaryPaths,
    input_type=BinaryPathRequest,
    mock=lambda request: BinaryPaths(binary_name="git", paths=[BinaryPath("/usr/bin/git")]),
),
```

however if you have many `Get` requests that are being mocked, because `lambda`'s syntax does not support type annotations, it can make your tests slightly harder to read. For instance, in the example above, the type of the `request` argument is unknown.

---

This concludes the series of tutorials that should help you get started writing own plugins with Pants. We have by now done quite a lot of work! You have learned:
* how to create an own goal, custom options and custom targets
* how to extend existing Pants goals such as `tailor`
* how to run system tools in your plugins and how Pants interacts with the file system
* how to write unit and integration tests for your plugin code

You are now ready to design and implement your next Pants plugin!
