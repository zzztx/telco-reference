GOPATH  ?= $(HOME)/go
GOBIN ?= $(GOPATH)/bin
export HELMCONVERTOR := $(GOBIN)/helm-convert
DOWNLOAD_URL := $(shell curl -s https://api.github.com/repos/openshift/kube-compare/releases/latest | jq -r '.assets[] | select(.name? | match("linux_amd64")) | .browser_download_url')
ARCHIVE_NAME := kubectl-cluster_compare-linux_amd64.tar.gz
HELM_URL := https://get.helm.sh/helm-v3.16.1-linux-amd64.tar.gz
HELM_PKG := helm-linux-amd64.tar.gz
HELM_DIR := /tmp
HELM_BIN := $(HELM_DIR)/helm
HELM := $(if $(shell command -v helm 2>/dev/null), $(shell command -v helm), $(HELM_BIN))

# main target to check the differences between reference and RDS CRs. \
  Calling compare.sh script to find out differences at the present stage. \
  Will enable compare_crs target as part of the main CI once all differences are fixed.
.PHONY: check
check: compare


.PHONY: compare_crs
compare_crs: kubectl-cluster_compare
	@echo "Running kube-compare to compare reference against source-crs"
	@./kubectl-cluster_compare -r ./reference-crs-kube-compare/metadata.yaml -f ./reference-crs -R; \
	if echo $?; then \
		echo "Differences found between reference and source-crs"; \
		exit 1; \
	else \
		echo "Okay"; \
		exit 0; \
	fi

kubectl-cluster_compare:
	@echo "Downloading kube-compare tool"
	curl -sL $(DOWNLOAD_URL) -o $(ARCHIVE_NAME)
	tar zxvf $(ARCHIVE_NAME) $@
	rm -f $(ARCHIVE_NAME)


.PHONY: clean
clean:
	rm -rf kubectl-cluster_compare Chartv1 renderedv1 $(HELM_BIN)


.PHONY: convert
convert: $(HELMCONVERTOR) $(HELM)
	@echo "Converting reference files to Helm Charts."
	@rm -rf Chartv1 renderedv1
	@$(HELMCONVERTOR) -r ./reference-crs-kube-compare/metadata.yaml -n Chartv1 -v default_value.yaml
	@echo "Rendering Helm Charts to CR files."
	@$(HELM) template renderedv1 ./Chartv1 --output-dir renderedv1

.PHONY: $(HELMCONVERTOR)
$(HELMCONVERTOR):
	@echo "Installing helm-convert tool..."
	go install github.com/openshift/kube-compare/addon-tools/helm-convert@latest

$(HELM):
	@if [[ $(HELM) == $(HELM_BIN) ]] && [ ! -f $(HELM) ]; then \
		echo "Installing helm..." \
		&& cd $(HELM_DIR) && curl -fsSL $(HELM_URL) -o $(HELM_PKG) \
		&& tar -zxvf $(HELM_PKG) linux-amd64/helm --strip-components 1 && rm -f $(HELM_PKG); \
	fi

# compare target is used to find out missing files between reference and source-crs.
.PHONY: compare
compare: convert
	@./compare.sh reference-crs renderedv1
