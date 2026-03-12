SHELL:=/usr/bin/env bash

.PHONY: help
# Run "make" or "make help" to get a list of user targets
# Adapted from https://marmelab.com/blog/2016/02/29/auto-documented-makefile.html
help: ## Show this help message
	@grep -E '^[a-zA-Z_-]+:.*?##.*$$' $(MAKEFILE_LIST) | awk 'BEGIN { \
	 FS = ":.*?## "; \
	 printf "\033[1m%-30s\033[0m %s\n", "TARGET", "DESCRIPTION" \
	} \
	{ printf "\033[32m%-30s\033[0m %s\n", $$1, $$2 }'

.PHONY: all
all: artifacts checks archives ## Run all pipelines

.PHONY: artifacts
artifacts: schemas ## Generate all artifacts

.PHONY: checks
checks: validate linkcheck ## Run all tests and checks

.PHONY: dependencies
dependencies: node_modules ## Ensure dependencies have been installed

node_modules: package.json package-lock.json
	npm ci

.PHONY: clean
clean: clean-schemas clean-linkcheck clean-archives ## Remove all generated content

# oscal-cli configuration
OSCAL_CLI_VERSION:=3.1.0
OSCAL_CLI_DOWNLOAD_URL:=https://repo1.maven.org/maven2/dev/metaschema/oscal/oscal-cli-enhanced/$(OSCAL_CLI_VERSION)/oscal-cli-enhanced-$(OSCAL_CLI_VERSION)-oscal-cli.tar.bz2
OSCAL_CLI_DIR:=bin/oscal-cli
OSCAL_CLI_ARCHIVE:=$(OSCAL_CLI_DIR)/oscal-cli-enhanced-$(OSCAL_CLI_VERSION)-oscal-cli.tar.bz2
OSCAL_CLI_BIN:=$(OSCAL_CLI_DIR)/bin/oscal-cli

# Use oscal-cli from PATH if available, otherwise use local installation
OSCAL_CLI_COMMAND:=$(or $(shell command -v oscal-cli 2>/dev/null),$(OSCAL_CLI_BIN))

.PHONY: install-oscal-cli
install-oscal-cli: ## Install oscal-cli if not available in PATH
ifeq ($(shell command -v oscal-cli 2>/dev/null),)
install-oscal-cli: $(OSCAL_CLI_BIN)
endif

$(OSCAL_CLI_BIN):
	@echo "oscal-cli not found in PATH, downloading version $(OSCAL_CLI_VERSION)..."
	@command -v curl >/dev/null 2>&1 || { echo "Error: curl is required but not installed."; exit 1; }
	@command -v bzip2 >/dev/null 2>&1 || { echo "Error: bzip2 is required but not installed."; exit 1; }
	@mkdir -p $(OSCAL_CLI_DIR)
	curl -fL -o "$(OSCAL_CLI_ARCHIVE)" "$(OSCAL_CLI_DOWNLOAD_URL)"
	@echo "Extracting oscal-cli..."
	@tar -xjf "$(OSCAL_CLI_ARCHIVE)" -C $(OSCAL_CLI_DIR)
	@rm "$(OSCAL_CLI_ARCHIVE)"
	@echo "oscal-cli installed to $(OSCAL_CLI_DIR)"
	@chmod +x $(OSCAL_CLI_BIN)

SRC_DIR:=../src/metaschema
# Contains all OSCAL metaschema modules, including those without exported roots
ALL_METASCHEMAS:=$(shell find $(SRC_DIR) -name '*_metaschema.xml')
# Contains the OSCAL metaschema modules that contain root assemblies
METASCHEMAS:=$(shell find $(SRC_DIR) -name '*_metaschema.xml' -a ! -name '*common*' -a ! -name '*metadata*')
GENERATED_DIR:=generated

#####################
# Schema Generation #
#####################

XSD_OUTPUTS:=$(patsubst $(SRC_DIR)/%_metaschema.xml,$(GENERATED_DIR)/%_schema.xsd,$(METASCHEMAS))
JSONSCHEMA_OUTPUTS:=$(patsubst $(SRC_DIR)/%_metaschema.xml,$(GENERATED_DIR)/%_schema.json,$(METASCHEMAS))

$(GENERATED_DIR)/%_schema.json $(GENERATED_DIR)/%_schema.xsd: $(SRC_DIR)/%_metaschema.xml
	@mkdir -p $(GENERATED_DIR)
	$(OSCAL_CLI_COMMAND) metaschema generate-schema --as=json --overwrite $(SRC_DIR)/$*_metaschema.xml $(GENERATED_DIR)/$*_schema.json
	$(OSCAL_CLI_COMMAND) metaschema generate-schema --as=xml --overwrite $(SRC_DIR)/$*_metaschema.xml $(GENERATED_DIR)/$*_schema.xsd
.PHONY: schemas

schemas: install-oscal-cli $(XSD_OUTPUTS) $(JSONSCHEMA_OUTPUTS) ## Generate the schemas

.PHONY: clean-schemas
clean-schemas: ## Remove generated schemas
	rm -fr $(GENERATED_DIR)/*_schema.*


######################
# Archive Generation #
######################

RELEASE:=SNAPSHOT
ZIP_ARCHIVE:=$(GENERATED_DIR)/oscal-$(RELEASE).zip
TARBZ2_ARCHIVE:=$(GENERATED_DIR)/oscal-$(RELEASE).tar.bz2
ARCHIVE_TEMP_DIR:=$(GENERATED_DIR)/archive_temp

.PHONY: archives
archives: $(ZIP_ARCHIVE) ## Archive schemas and metaschemas

$(ZIP_ARCHIVE) $(TARBZ2_ARCHIVE): schemas
	@echo Generating archive
	@command -v zip >/dev/null 2>&1 || { echo "Error: zip is required but not installed."; exit 1; }
	mkdir -p $(ARCHIVE_TEMP_DIR)/{{json,xml}/schema,metaschema}
	cp ../src/release/release-readme.txt "$(ARCHIVE_TEMP_DIR)/README.txt"
	cp $(XSD_OUTPUTS) "$(ARCHIVE_TEMP_DIR)/xml/schema"
	cp $(JSONSCHEMA_OUTPUTS) "$(ARCHIVE_TEMP_DIR)/json/schema"

	cp $(ALL_METASCHEMAS) "$(ARCHIVE_TEMP_DIR)/metaschema"
	(cd "$(ARCHIVE_TEMP_DIR)" && zip -r $(abspath $(ZIP_ARCHIVE)) .)
	tar -jcvf "$(TARBZ2_ARCHIVE)" -C "$(ARCHIVE_TEMP_DIR)" .

	@echo Removing temporary archive staging directory $(ARCHIVE_TEMP_DIR)
	rm -fr $(ARCHIVE_TEMP_DIR)

.PHONY: clean-archives
clean-archives: ## Clean generated archives
	rm -fr $(ARCHIVE_TEMP_DIR)
	rm -fr $(ZIP_ARCHIVE) $(TARBZ2_ARCHIVE)

#######################
# Markdown Link Check #
#######################

LINKCHECK_OUTPUT:=$(GENERATED_DIR)/mlc_report.log
LINKCHECK_CONFIG:=markdown-link-check.json

.PHONY: linkcheck
linkcheck: $(LINKCHECK_OUTPUT) ## Perform Markdown link checking

# WARNING: will fail if locally deleted markdown files are not staged.
# 1. cd into the parent directory
# 2. list all markdown files known to git
# 3. Pass the files as an argument to the markdown link checker
# 4. Write the log to a file
# 5. Exit with a failure if any of the link checker invocations failed
$(LINKCHECK_OUTPUT): node_modules $(LINKCHECK_CONFIG)
	@mkdir -p $(GENERATED_DIR)
	@echo Checking Markdown links...
	cd ..; git ls-files -z "*.md" | \
		xargs -0n1 npx --prefix build/ markdown-link-check -c build/$(LINKCHECK_CONFIG) | \
		tee build/$(LINKCHECK_OUTPUT); \
		echo ""; echo "... Done checking Markdown links."; \
	exit $${PIPESTATUS[1]}

.PHONY: clean-linkcheck
clean-linkcheck: ## Remove linkcheck log
	rm -fr $(LINKCHECK_OUTPUT)

#######################
# Artifact Validation #
#######################

.PHONY: validate
validate: validate-jsonschemas validate-xsds validate-metaschemas ## Validate all generated content

.PHONY: validate-jsonschemas
validate-jsonschemas: node_modules $(JSONSCHEMA_OUTPUTS) ## Validate generated JSON Schemas
	@echo Validating generated JSON Schemas
	npx ajv compile -c "ajv-formats" -s "$(GENERATED_DIR)/*_schema.json"

.PHONY: validate-xsds
validate-xsds: $(XSD_OUTPUTS) ## Validate generated XSD files
	@echo Validating generated XSDs
	@command -v xmllint >/dev/null 2>&1 || { echo "Error: xmllint is required but not installed. Install libxml2-utils."; exit 1; }
	xmllint --noout $(XSD_OUTPUTS)

.PHONY: validate-metaschemas
validate-metaschemas: install-oscal-cli ## Validate source metaschemas
	@for metaschema in $(ALL_METASCHEMAS); do \
		echo "Validating $$metaschema"; \
		$(OSCAL_CLI_COMMAND) metaschema validate "$$metaschema" || exit 1; \
	done

###################
# Utility Targets #
###################

# These targets may be used by consumers of the OSCAL repository

# All artifacts typically included in a release
RELEASE_ARTIFACTS:=$(XSD_OUTPUTS) $(JSONSCHEMA_OUTPUTS) $(ZIP_ARCHIVE) $(TARBZ2_ARCHIVE)

# This target is used by OSCAL-Reference to generate meta-redirects for release assets
.PHONY: list-release-artifacts
list-release-artifacts: ## Print out a list of all artifacts typically included in an OSCAL release
	@echo $(RELEASE_ARTIFACTS)
