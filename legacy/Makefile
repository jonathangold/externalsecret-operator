DOCKER_IMAGE ?= containersol/externalsecret-operator"

NAMESPACE ?= "default"
BACKEND ?= "asm"
OPERATOR_NAME ?= "asm-example"

GIT_HASH	:= $(shell git rev-parse --short HEAD)
GIT_BRANCH 	:= $(shell git rev-parse --abbrev-ref HEAD | sed 's/\//-/')
GIT_TAG 	:= $(shell git describe --tags --abbrev=0 --always)
DOCKER_TAG 	:= $(shell ./build/scripts/determine_docker_tag.sh $(GIT_HASH) $(GIT_BRANCH) $(GIT_TAG))

.PHONY: build
build: operator-sdk
	go build -o ./build/_output/bin/externalsecret-operator ./pkg/cmd
	./operator-sdk build $(DOCKER_IMAGE)

.PHONY: push
.EXPORT_ALL_VARIABLES: push
push: build
	# tag and push docker image with commit hash as tag
	docker tag $(DOCKER_IMAGE) $(DOCKER_IMAGE):$(GIT_HASH)
	docker push $(DOCKER_IMAGE):$(GIT_HASH)

.PHONY: release
.EXPORT_ALL_VARIABLES: push
release:
	# tag an existing image with a proper release tag (i.e. latest or 0.0.4)
	docker pull $(DOCKER_IMAGE):$(GIT_HASH)
	docker tag $(DOCKER_IMAGE):$(GIT_HASH) $(DOCKER_IMAGE):$(DOCKER_TAG)
	docker push $(DOCKER_IMAGE):$(DOCKER_TAG)

.PHONY: deploy
.EXPORT_ALL_VARIABLES: deploy
deploy:
	# deploy using vanilla manifests with envsubsts
	kubectl apply -n $(NAMESPACE) -f ./deployments/service_account.yaml
	kubectl apply -n $(NAMESPACE) -f ./deployments/role.yaml
	envsubst < ./deployments/role_binding.yaml | kubectl apply -n $(NAMESPACE) -f  -
	kubectl apply -n $(NAMESPACE) -f ./deployments/crds/externalsecret-operator_v1alpha1_externalsecret_crd.yaml
	envsubst < deployments/secret-${BACKEND}.yaml | kubectl apply -n $(NAMESPACE) -f -
	envsubst < deployments/deployment.yaml | kubectl apply -n $(NAMESPACE) -f -

.PHONY: apply-onepassword
OPERATOR_NAME=onepassword
BACKEND=onepassword
.EXPORT_ALL_VARIABLES: apply-onepassword
apply-onepassword:
	@echo "Deploying service account..."
	@kubectl apply -n $(NAMESPACE) -f ./deployments/service_account.yaml
	@echo "Deploying role..."
	@kubectl apply -n $(NAMESPACE) -f ./deployments/role.yaml
	@echo "Deploying rolebinding..."
	@envsubst < ./deployments/role_binding.yaml | kubectl apply -n $(NAMESPACE) -f  -
	@echo "Deploying external operator CRD..."
	@kubectl apply -n $(NAMESPACE) -f ./deployments/crds/externalsecret-operator_v1alpha1_externalsecret_crd.yaml
	@echo "Deploying 1password operator config secret..."
	@envsubst < deployments/secret-${BACKEND}.yaml | kubectl apply -n $(NAMESPACE) -f -
	@echo "Deploying operator deployment..."
	@envsubst < deployments/deployment.yaml | kubectl apply -n $(NAMESPACE) -f -

.PHONY: apply-gsm
OPERATOR_NAME=gsm-local
BACKEND=gsm
.EXPORT_ALL_VARIABLES: apply-onepassword
apply-gsm:
	@echo "Deploying service account..."
	@kubectl apply -n $(NAMESPACE) -f ./deployments/service_account.yaml
	@echo "Deploying role..."
	@kubectl apply -n $(NAMESPACE) -f ./deployments/role.yaml
	@echo "Deploying rolebinding..."
	@envsubst < ./deployments/role_binding.yaml | kubectl apply -n $(NAMESPACE) -f  -
	@echo "Deploying external operator CRD..."
	@kubectl apply -n $(NAMESPACE) -f ./deployments/crds/externalsecret-operator_v1alpha1_externalsecret_crd.yaml
	@echo "Deploying gsm operator config secret..."
	@envsubst < deployments/secret-gsm.yaml | kubectl apply -n $(NAMESPACE) -f -
	@echo "Deploying operator deployment..."
	@envsubst < deployments/deployment.yaml | kubectl apply -n $(NAMESPACE) -f -

.PHONY: delete-onepassword
.EXPORT_ALL_VARIABLES: delete-onepassword
delete-onepassword:
	@echo "Deleting 1password operator config secret..."
	kubectl delete secret externalsecret-operator-config
	@echo "Deleting operator deployment..."
	kubectl delete deployment externalsecret-operator

.PHONY: deploy-onepassword
.EXPORT_ALL_VARIABLES: deploy-onepassword
deploy-onepassword: push apply-onepassword
	
.PHONY: test
test:
	# run unit tests
	go test -v -short ./... -count=1

.PHONY: coverage
# include only code we write in coverage report, not generated
COVERAGE := ./...
coverage:
	# generate coverage report
	go test -short -race -coverprofile=coverage.txt -covermode=atomic $(COVERAGE)
	curl -s https://codecov.io/bash | bash

.PHONY: test-helm
RELEASE := test$(shell echo $$$$)
OPERATOR_NAME=$(RELEASE)
BACKEND=dummy
HELM_IMAGE_TAG ?= $(GIT_HASH)
.EXPORT_ALL_VARIABLES: test-helm
test-helm:
	# end-to-end test using test included in the helm chart
	helm install --wait $(RELEASE) \
		--set test.create=true \
		--set image.tag=$(HELM_IMAGE_TAG) \
		./deployments/helm/externalsecret-operator/.
	helm test  $(RELEASE)
	helm uninstall $(RELEASE)

PLATFORM := $(shell bash -c '[ "$$(uname -s)" = "Linux" ] && echo linux-gnu || echo apple-darwin')
OPERATOR_SDK_VERSION := v0.9.0
OPERATOR_SDK_URL := https://github.com/operator-framework/operator-sdk/releases/download/${OPERATOR_SDK_VERSION}/operator-sdk-${OPERATOR_SDK_VERSION}-x86_64-$(PLATFORM)
operator-sdk:
	# ensure operator-sdk is available
	curl -LJ -o $@ $(OPERATOR_SDK_URL)
	chmod +x $@
