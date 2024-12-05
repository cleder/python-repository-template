# python-repository-template
A Template repository for python projects, with testing and publishing.

## How to Publish Your Python Package with Trusted Publishing

This Repository provides a workflow that will publish your package to TestPyPi when a tag is created (on the `development` branch), or to PyPi when you merge to the `main` branch.

### Prepare Your Package for Publishing

Ensure your Python package follows PyPI’s packaging guidelines. At a minimum, you’ll need:

- A [~~setup.py or~~ pyproject.toml](https://dev.to/ldrscke/hypermodernize-your-python-package-3d9m "How to convert from setup.py to pyproject.toml") file defining your package metadata.
- Properly structured code with a clear directory layout.
- A README file to showcase your project on PyPI.

For a detailed checklist, refer to the [Python Packaging User Guide](https://packaging.python.org/).

### Configure GitHub Actions in Your Repository

Let's start by creating a new GitHub action `.github/workflows/test-build-publish.yml`.

```yaml
name: test-build-publish

on: [push, pull_request]

permissions:
  contents: read

jobs:

  build-and-check-package:
    name: Build & inspect our package.
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: hynek/build-and-inspect-python-package@v2
```

This [action](https://github.com/hynek/build-and-inspect-python-package) will build your package and uploads the built wheel and the source distribution (`SDist`) as GitHub Actions artefacts.

Next, we add a step to publish to [TestPyPI](https://test.pypi.org/). This step will run whenever a tag is created, ensuring that the build from the previous step has completed successfully. Replace PROJECT_OWNER and PROJECT_NAME with the appropriate values for your repository.

```yaml
  test-publish:
    if: >-
        github.event_name == 'push' &&
        github.repository == 'PROJECT_OWNER/PROJECT_NAME' &&
        startsWith(github.ref, 'refs/tags')
    needs: build-and-check-package
    name: Test publish on TestPyPI
    runs-on: ubuntu-latest
    environment: test-release
    permissions:
      id-token: write
    steps:
      - name: Download packages built by build-and-check-package
        uses: actions/download-artifact@v4
        with:
          name: Packages
          path: dist

      - name: Upload package to Test PyPI
        uses: pypa/gh-action-pypi-publish@release/v1
        with:
          repository-url: https://test.pypi.org/legacy/
```

This step downloads the artefacts created during the build process and uploads them to TestPyPI for testing.

In the last step, we will upload the package to PyPI when a pull request is merged into the `main` branch.

```yaml
  publish:
    if: >-
      github.event_name == 'push' &&
      github.repository == 'PROJECT_OWNER/PROJECT_NAME' &&
      github.ref == 'refs/heads/main'
    needs: build-and-check-package
    name: Publish to PyPI
    runs-on: ubuntu-latest
    environment: release
    permissions:
      id-token: write
    steps:
      - name: Download packages built by build-and-check-package
        uses: actions/download-artifact@v4
        with:
          name: Packages
          path: dist

      - name: Publish distribution 📦 to PyPI for push to main
        uses: pypa/gh-action-pypi-publish@release/v1
```

### Configure GitHub Environments

To ensure that only specific tags trigger the publishing workflow and maintain control over your release process.
Create a new environment `test-release` by navigating to Settings -> Environments in your GitHub repository.

Set up the environment and add a deployment tag rule.

![Add a new environment](https://github.com/cleder/talks/blob/main/images/pypi-publish/github-environments.png)

![Create test environment](https://github.com/cleder/talks/blob/main/images/pypi-publish/github-environment-test-release.png)

Limit which branches and tags can deploy to this environment based on rules or naming patterns.

![Add deployment tag rule](https://github.com/cleder/talks/blob/main/images/pypi-publish/github-environment--add-tag-ruleset-target-pattern.png)

Limit which branches and tags can deploy to this environment based on naming patterns.

![Add target tags](https://github.com/cleder/talks/blob/main/images/pypi-publish/github-environment--tag-ruleset-target.png)

Configure the target tags.

![Configure target tags](https://github.com/cleder/talks/blob/main/images/pypi-publish/github-environment--add-deployment-tag-pattern.png)

The pattern `[0-9]*.[0-9]*.[0-9]*` matches semantic versioning tags such as `1.2.3`, `0.1.0`, or `2.5.1b3`, but it excludes arbitrary tags like `bugfix-567` or `feature-update`.

Repeat this for the `release` environment to protect the `main` branch in the same way for the `release` environment, but this time targeting the `main` branch.

![Configure target branch](https://github.com/cleder/talks/blob/main/images/pypi-publish/github-environment--add-deployment-branch-pattern.png)



### Set Up a PyPI Project and Link Your GitHub Repository

Create an account on [TestPyPI](https://test.pypi.org/) if you don’t have one.
Navigate to your account, [Publishing](https://test.pypi.org/manage/account/publishing/) and add a new pending publisher.
Link your GitHub repository to the PyPI project by providing its name, your GitHub username, the repository name, the workflow name (`test-build-publish.yml`) and the environment name (`test-release`).

![Configure PyPI trusted publisher](https://github.com/cleder/talks/blob/main/images/pypi-publish/test-pypi-publishing.png)

Repeat the above on PyPI with the environment name set to `release`.

## Start Your Own Project

Just delete this README and use `uv init` to create a new python package.
