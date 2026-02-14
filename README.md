# Understanding: Python Packages, PyPI, GitHub Actions, Versioning and Publishing

So you want to understand:
- how to create and structure a Python package
- how to write tests for it
- how to write dope GitHub Actions workflows that
    - automate testing (with pytest)
    - automate versioning (with release-please and setuptools-scm)
    - automate building and publishing to PyPI (with Trusted Publishing)

well you're in the right place! This repo is an educational example package that demonstrates all of the above concepts.

## What this package 'does'
- A simple function `answer_to_life` that returns 42.
- A test for that function using pytest.

## Setup 

This checklist works whether you are
- forking this repo and following along
- adding stuff to an existing or blank repo

1. Create a new repository
    - option 1: fork this repo
        - 1. Fork this repo
        - 2. Rename package `pypi-example-package-<your-name>`, to make it unique on PyPI/TestPyPI
            - in the pyproject.toml file
            - the directory name
        - 3. reset the [.release-please-manifest.json] to 0.1.0 (or whatever you want your first version to be)
    - option 2: add this stuff to an existing repo (or blank repo).
        - 1. `pyproject.toml` with setuptools-scm configuration for dynamic version. Add the following lines to your `pyproject.toml` file:
            - `requires = ["setuptools >= 61", "setuptools-scm >= 8"]`
            - `dynamic = ["version"]` # if you have a `version = "X.Y.Z"` line replace it with this.
            - [tool.setuptools_scm]`
                - `version_file = "src/<package_dir>/_version.py" # create version file on build and install`
                - `local_scheme = "no-local-version"`
            - if you don't have a `pyproject.toml` and still using `setup.py`, get off that trash already!
        - 2. 'release-please.yml` workflow file for GitHub Actions
        - 3. `release-please-manifest.json` set version to current (or desired first) version
        - 4. `release-please-config.json` config file for release-please
        - 5. `__init__.py` that imports version from `_version.py`
            - if your IDE is complaining about an unresolved import for `_version.py`, don't worry it will get generated whe you 'install' the package.
        - 6. `.gitignore`: add `src/*/_version.py` to gitignore since it is generated on build


2. allow GitHub Actions to create releases in repo settings 
    - (Repo -> Settings -> Actions -> General -> Workflow permissions -> Allow GitHub Actions to create and approve pull requests and push to the repository)

3. Setup PyPI Trusted Publishing (can skip if not publishing to PyPI)
    - TestPyPI (recommended for making sure everything works before publishing to real PyPI)
        - 1. create a TestPyPI account if you don't have one (recommended for making sure everything works before publishing to real PyPI) 
        - 2. create environment named `testpypi` in GitHub repo settings (for trusted publishing)
            - (Repo -> Settings -> Environments -> New Environment -> name: testpypi)
        - 3. on TestPyPI, add trusted publisher with GitHub Actions and with 'testpypi' environment
    - PyPI
        - 1. create a PyPI account if you don't have one
        - 2. create environment named `pypi` in GitHub repo settings (for trusted publishing)
            - (Repo -> Settings -> Environments -> New Environment -> name: pypi)
        - 3. on PyPI, add trusted publisher with GitHub Actions and with 'pypi' environment
        - 4. remove the TestPyPI url from the `release-please.yml` workflow file so that it publishes to real PyPI instead of TestPyPI

## Concepts that made my head hurt but now make perfect sense

### pytest
- when you run `pytest`, it automatically discovers and runs tests in your codebase. By default, it looks for files that start with `test_` or end with `_test.py` and functions that start with `test_`.

### GitHub Actions
formal docs: https://docs.github.com/en/actions/get-started/understand-github-actions
- GitHub Actions are automations you can run, triggered by events like pushes, pull requests in your GitHub repo.
- Most commonly, they are used for CI/CD (continuous integration and continuous deployment), which means automatically running tests, bumping versions, and building when you push new code.
- GitHub Actions live in `workflows`. A workflow is defined in a YAML file in the `.github/workflows` directory of your repo. This makes them version controlled and consistent across the team.
- They run on a remote machine (hosted by GitHub) with a clean environment, so you have to set up everything you need to run your tests and publish your package in the workflow file.
- Its very common in workflows to use reusable actions defined in other repo. e.g.:
    - `actions/checkout` to check out your code, 
    - `actions/setup-python` to set up a Python environment, 
    -`googleapis/release-please-action` to automate releases.


#### Building a GitHub Actions Workflow
- `workflow` -> `job` -> `step`
- A `workflow` is defined in a YAML file and contains one or more `jobs`. 
- A `job` contains one or more `steps` that run **sequentially** on the same machine.
- By default, all `jobs` run in parallel.
- You can set dependencies between `jobs` with `needs: <prev-job>` so that a `job` only runs after another `job` finishes successfully (e.g. only publish after build is successful).
- `jobs` can also be conditionally run/skipped with an `if` statement 
    - e.g. `if: ${{ needs.release-please.outputs.release_created == 'true' }}` to only run the build job if a release was actually created by the release-please job.
- Each `job` runs on a fresh machine, so if you need to share data between `jobs` (e.g. build artifacts), you need to explicitly upload and download them using `actions/upload-artifact` and `actions/download-artifact`.


### Trusted publishing to PyPI (the new way) vs using API tokens (the old way)
- Traditionally, to publish a package to PyPI, you would create an API token on PyPI, then enter that token after running the publish command (e.g. `twine upload dist/*). (or automate it in CI by setting the token as a secret and using it in the workflow). Static API's are a security risk because they can be leaked or misused, but most of all they are annoying to manage( creating, rotating, revoking).
- Trusted publishing is a sleeker and more secure way to publish. Setting up a trusted publisher is essentially telling PyPI "only trust publishes coming from a specific GitHub Action with user, repository, workflow name and environment". 

### Versioning: setuptools-scm and release-please

#### What the heck is setuptools-scm and why do I need it?
- setuptools-scm is a tool that automatically generates version numbers *during build and install events* based on your git state (tags and commits). This can be more reliable than manually reading the version number from a file (e.g. pyproject.toml or __init__.py).
    - runs locally
    - is a setuptools plugin, normally is is installed in a temporary environment during 'build and install events' and then removed, so it is not available after installation
    - can be pip installed manually but this is almost never done unless debugging
    - note on editable installs: the version you get when running importlib.metadata.version("pypi-package-example") is frozen at the time of installation, so if you want to see changes to the version you must reinstall the package (pip install -e .) after making changes to the git state (commits, tags).

#### How does release-please know what the previous version was?
- release-please looks at pyproject.toml but if dynamic versioning is used (e.g. setuptools-scm), it will use [.release-please-manifest.json].

Process with release-please and setuptools_scm
1. release-please looks at [.release-please-manifest.json] and commit messages to determine next version.
2. when user merges in the next version, release-please creates a git tag for that version (e.g. v0.1.0)
3. setuptools-scm looks at the git tags to determine the version number when building and installing the package. 
    - If the git state is exactly at a tag (e.g. v0.1.0), it will use that version number. 
    - If there have been commits since the last tag, it will use the last tag plus a dev suffix (e.g. 0.1.0.dev1).

## Optional Alterations
I did explore a few other approaches before settling on the ones in this repo but here are some honorable mentions:

### Alternatives to CI
- Just keep it stupid simple: manually run tests locally, manually build and manually publish to PyPI using `twine upload dist/*`. This is how the cavemen did it and it works just fine for projects you don't update often!

### how to get user facing version info (`package.__version__`) from setuptools-scm
`package.__version__` is usually accesible in your `__init__.py` and can be derived in a few different ways:
- hard coded in `__init__.py` (e.g. `__version__ = "0.1.0"`) - this is the simplest way but requires manual updating
- importlib.metadata.version("package-name")
    - PROs
        - no extra files or gitignore entries needed since it derives version info directly from the installed package metadata
    - CONs
        - If you want dev build numbers (e.g. 0.1.0.dev1), dev builds will just show the version as the last released version since it looks at the installed package metadata
        - importlib becomes a runtime dependency
- `_version.py` generated by setuptools-scm
    - PROs
        - can include dev build numbers (e.g. 0.1.0.dev1) since it is generated on build and install based on the git state at that time
    - CONs
        - requires an extra file and gitignore entry since it is generated on build and install
note: editable installs will not get the dev version since you aren't really building or installing the package (which is when setuptools_scm runs) you are really just directly reading code

### Set a branch protection rule
- you can set a branch protection rule for main so that tests must pass before being allowed to merge. do this in repo settings.
- Problem: however theres a problem when release-please creates a release pr, it wont trigger the testing workflow (because github action can't trigger other github actions), then you will be stuck or have to do a manual commit to trigger it.
- Solution: you then must then run release-please with PAT so that it can trigger other action runs
- I will opt not to do this a really simple lib thats just me, i can just manually wait to make sure tests pass.

## Other notes

### Manual Release Checklist (What I used to use):
1. Make code changes
2. Run tests locally
3. Update version number 
    - in `__init__.py`
    - in `pyproject.toml`
4. Update changelog
5. Commit and push changes (or merge PR if using a branch for development)
6. Build package using `python -m build`
7. Publish to PyPI using `twine upload dist/*`
8. Create GitHub release and tag
    - run: `git tag vX.Y.Z`
    - run: `git push origin vX.Y.Z`
    - create release in GitHub UI and link to tag.


TODO:
- clean this up so it can be a template for future packages
    - better readme
    - remove unnecessary files
- add links to docs
- rename to `understanding-python-packaging-qthedoc
- other topics to cover
    - Running tests locally (taking care of the `_version.py`)
        - `pip install .` local static install (this will generate the `_version.py`.)
        - `pip install -e .` editable install. this will not generate the `_version.py` since an editable install si really just a symlink to your code
        - even if you want an editable install, you should still do a static install at least once to generate the `_version.py` so that it quits complaining about the missing version file.
    - Using your package locally
    - understanding editable vs static installs

#### Top Level Concepts starting from nothing:
what is a python package? how is it different from simply running a pytohn file (module)?
- a package is a collection of modules (python files) that are organized in a directory structure and can be distributed and installed as a unit. once installed, you can import the package and use its functionality in other code.
- a module is a single python file that can be run in a command line usually using `path/to/python_interpreter path/to/module.py` or just `python module.py` python is set as a PATH variable that points to the python interpreter. 

## Contributing
Contributions are welcome! Please open an issue or submit a pull request with any improvements or suggestions.