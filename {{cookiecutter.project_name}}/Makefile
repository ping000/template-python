# Project settings
PROJECT := {{cookiecutter.project_name}}
PACKAGE := {{cookiecutter.package_name}}
REPOSITORY := {{cookiecutter.github_username}}/{{cookiecutter.github_repo}}

# Project paths
PACKAGES := $(PACKAGE) tests
CONFIG := $(wildcard *.py)
MODULES := $(wildcard $(PACKAGE)/*.py)

# Virtual environment paths
VIRTUAL_ENV ?= .venv

# MAIN TASKS ##################################################################

SNIFFER := poetry run sniffer

.PHONY: all
all: install

.PHONY: ci
ci: check test ## Run all tasks that determine CI status

.PHONY: watch
watch: install .clean-test ## Continuously run all CI tasks when files chanage
	$(SNIFFER)

.PHONY: run ## Start the program
run: install
	poetry run python $(PACKAGE)/__main__.py

# SYSTEM DEPENDENCIES #########################################################

.PHONY: doctor
doctor:  ## Confirm system dependencies are available
	bin/verchew

# PROJECT DEPENDENCIES ########################################################

DEPENDENCIES := $(VIRTUAL_ENV)/.poetry-$(shell bin/checksum pyproject.toml poetry.lock) *.egg-info

.PHONY: install
install: $(DEPENDENCIES)

$(DEPENDENCIES):
	@ poetry config settings.virtualenvs.in-project true
	poetry install
	@ touch $@

# CHECKS ######################################################################

ISORT := poetry run isort
PYLINT := poetry run pylint
PYCODESTYLE := poetry run pycodestyle
PYDOCSTYLE := poetry run pydocstyle

.PHONY: check
check: isort pylint pycodestyle pydocstyle ## Run linters and static analysis

.PHONY: isort
isort: install
	$(ISORT) $(PACKAGES) $(CONFIG) --recursive --apply

.PHONY: pylint
pylint: install
	$(PYLINT) $(PACKAGES) $(CONFIG) --rcfile=.pylint.ini

.PHONY: pycodestyle
pycodestyle: install
	$(PYCODESTYLE) $(PACKAGES) $(CONFIG) --config=.pycodestyle.ini

.PHONY: pydocstyle
pydocstyle: install
	$(PYDOCSTYLE) $(PACKAGES) $(CONFIG)

# TESTS #######################################################################

{% if cookiecutter.test_runner == "nose" -%}
NOSE := poetry run nosetests
{%- elif cookiecutter.test_runner == "pytest" -%}
PYTEST := poetry run pytest
{%- endif %}
COVERAGE := poetry run coverage
COVERAGE_SPACE := poetry run coveragespace

RANDOM_SEED ?= $(shell date +%s)
{% if cookiecutter.test_runner == "nose" %}
NOSE_OPTIONS := --with-doctest
ifndef DISABLE_COVERAGE
NOSE_OPTIONS += --with-coverage --cover-package=$(PACKAGE) --cover-html --cover-html-dir=htmlcov --cover-branches
endif

.PHONY: test
test: test-all ## Run unit and integration tests

.PHONY: test-unit
test-unit: install .clean-test
	$(NOSE) $(PACKAGE) $(NOSE_OPTIONS)
	$(COVERAGE_SPACE) $(REPOSITORY) unit

.PHONY: test-int
test-int: install .clean-test
	$(NOSE) tests $(NOSE_OPTIONS)
	$(COVERAGE_SPACE) $(REPOSITORY) integration

.PHONY: test-all
test-all: install .clean-test
	$(NOSE) $(PACKAGES) $(NOSE_OPTIONS)
	$(COVERAGE_SPACE) $(REPOSITORY) overall
{% elif cookiecutter.test_runner == "pytest" -%}
FAILURES := .cache/v/cache/lastfailed

PYTEST_OPTIONS := --random --random-seed=$(RANDOM_SEED)
ifdef DISABLE_COVERAGE
PYTEST_OPTIONS += --no-cov --disable-warnings
endif
PYTEST_RERUN_OPTIONS := --last-failed --exitfirst

.PHONY: test
test: test-all ## Run unit and integration tests

.PHONY: test-unit
test-unit: install
	@ ( mv $(FAILURES) $(FAILURES).bak || true ) > /dev/null 2>&1
	$(PYTEST) $(PACKAGE) $(PYTEST_OPTIONS)
	@ ( mv $(FAILURES).bak $(FAILURES) || true ) > /dev/null 2>&1
	$(COVERAGE_SPACE) $(REPOSITORY) unit

.PHONY: test-int
test-int: install
	@ if test -e $(FAILURES); then $(PYTEST) tests $(PYTEST_RERUN_OPTIONS); fi
	@ rm -rf $(FAILURES)
	$(PYTEST) tests $(PYTEST_OPTIONS)
	$(COVERAGE_SPACE) $(REPOSITORY) integration

.PHONY: test-all
test-all: install
	@ if test -e $(FAILURES); then $(PYTEST) $(PACKAGES) $(PYTEST_RERUN_OPTIONS); fi
	@ rm -rf $(FAILURES)
	$(PYTEST) $(PACKAGES) $(PYTEST_OPTIONS)
	$(COVERAGE_SPACE) $(REPOSITORY) overall
{% endif %}
.PHONY: read-coverage
read-coverage:
	bin/open htmlcov/index.html

# DOCUMENTATION ###############################################################

PYREVERSE := poetry run pyreverse
MKDOCS := poetry run mkdocs

MKDOCS_INDEX := site/index.html

.PHONY: docs
docs: uml mkdocs ## Generate documentation

.PHONY: uml
uml: install docs/*.png
docs/*.png: $(MODULES)
	$(PYREVERSE) $(PACKAGE) -p $(PACKAGE) -a 1 -f ALL -o png --ignore tests
	- mv -f classes_$(PACKAGE).png docs/classes.png
	- mv -f packages_$(PACKAGE).png docs/packages.png

.PHONY: mkdocs
mkdocs: install $(MKDOCS_INDEX)
$(MKDOCS_INDEX): mkdocs.yml docs/*.md
	ln -sf `realpath README.md --relative-to=docs` docs/index.md
	ln -sf `realpath CHANGELOG.md --relative-to=docs/about` docs/about/changelog.md
	ln -sf `realpath CONTRIBUTING.md --relative-to=docs/about` docs/about/contributing.md
	ln -sf `realpath LICENSE.md --relative-to=docs/about` docs/about/license.md
	$(MKDOCS) build --clean --strict

.PHONY: mkdocs-live
mkdocs-live: mkdocs
	eval "sleep 3; bin/open http://127.0.0.1:8000" &
	$(MKDOCS) serve

# BUILD #######################################################################

PYINSTALLER := poetry run pyinstaller
PYINSTALLER_MAKESPEC := poetry run pyi-makespec

DIST_FILES := dist/*.tar.gz dist/*.whl
EXE_FILES := dist/$(PROJECT).*
.PHONY: dist
dist: install $(DIST_FILES)
$(DIST_FILES): $(MODULES)
	rm -f $(DIST_FILES)
	poetry build

.PHONY: exe
exe: install $(EXE_FILES)
$(EXE_FILES): $(MODULES) $(PROJECT).spec
	# For framework/shared support: https://github.com/yyuu/pyenv/wiki
	$(PYINSTALLER) $(PROJECT).spec --noconfirm --clean

$(PROJECT).spec:
	$(PYINSTALLER_MAKESPEC) $(PACKAGE)/__main__.py --onefile --windowed --name=$(PROJECT)

# RELEASE #####################################################################

.PHONY: upload
upload: dist ## Upload the current version to PyPI
	git diff --name-only --exit-code
	poetry publish
	bin/open https://pypi.org/project/$(PROJECT)

# CLEANUP #####################################################################

.PHONY: clean
clean: .clean-build .clean-docs .clean-test .clean-install ## Delete all generated and temporary files

.PHONY: clean-all
clean-all: clean
	rm -rf $(VIRTUAL_ENV)

.PHONY: .clean-install
.clean-install:
	find $(PACKAGES) -name '*.pyc' -delete
	find $(PACKAGES) -name '__pycache__' -delete
	rm -rf *.egg-info

.PHONY: .clean-test
.clean-test:
	rm -rf .cache .pytest .coverage htmlcov xmlreport

.PHONY: .clean-docs
.clean-docs:
	rm -rf *.rst docs/apidocs *.html docs/*.png site

.PHONY: .clean-build
.clean-build:
	rm -rf *.spec dist build

# HELP ########################################################################

.PHONY: help
help: all
	@ grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

.DEFAULT_GOAL := help
