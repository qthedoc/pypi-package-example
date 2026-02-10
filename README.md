# PyPI Package Example

Testing a simple Python package for PyPI distribution with GitHub Actions CI.

## What it does
- A simple function `answer_to_life` that returns 42.
- A test for that function using pytest.

## im getting it now
- setuptools-scm is used to manage versioning based on git state (tags, commits) (more reliable than you or release-please which both just look at the last version number).
    - runs locally
    - is a setuptools plugin, normally is is installed in a temporary environment during build and then removed, so it is not available after installation
    - can be pip installed manually but this is almost never done unless debugging
    - note on editable installs: the version you get when running importlib.metadata.version("pypi-package-example") is frozen at the time of installation, so if you want to see changes to the version you must reinstall the package (pip install -e .) after making changes to the git state (commits, tags).
- GitHub Actions are set up to run tests on push and to publish to TestPyPI on release.


update v0.2.0