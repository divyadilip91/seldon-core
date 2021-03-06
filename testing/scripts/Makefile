VERSION := $(shell cat ../../version.txt)

# This can be "Auto" or a Number
PYTEST_WORKERS ?= "4"

kind_create_cluster:
	kind create cluster --config ../resources/kind_config.yaml

kind_delete_cluster:
	kind delete cluster

kind_build_engine:
	cd ../../engine/ && make build_image
	kind load docker-image seldonio/engine:$(VERSION) --loglevel trace

kind_build_operator:
	cd ../../operator && make kind-image-install

kind_build_executor:
	cd ../../executor && make kind-image-install

kind_build_test_models:
	make -C ../docker/fixed-model kind_load_images
	make -C ../docker/echo-model kind_load_image

kind_build_prepackaged:
	cd ../../servers/sklearnserver && make kind_load
	cd ../../servers/xgboostserver && make kind_load
	cd ../../servers/mlflowserver && make kind_load
	cd ../../integrations/tfserving && make kind_load

kind_build_alibi:
	cd ../../components/alibi-detect-server && make kind_load
	cd ../../components/alibi-explain-server && make kind_load

kind_build_misc:
	cd ../../components/seldon-request-logger && make kind_load
	cd ../../components/storage-initializer && make kind_load

kind_build_images: build_protos kind_build_engine kind_build_operator kind_build_executor kind_build_test_models kind_build_prepackaged kind_build_alibi kind_build_misc

helm_setup:
	helm repo add stable https://kubernetes-charts.storage.googleapis.com/
	helm repo add seldonio https://storage.googleapis.com/seldon-charts
	helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
	helm repo update

install_ambassador:
	helm install ambassador \
		stable/ambassador \
		-f ../resources/ambassador_values.yaml \
		--set crds.keep=false \
		--namespace seldon \
		--set replicaCount=1 \
		--wait

install_jaeger:
	helm install jaeger-operator \
		jaegertracing/jaeger-operator \
		--set rbac.clusterRole=true \
		--namespace seldon \
		--wait
	kubectl apply -f ../resources/jaeger.yaml --namespace seldon


install_cert_manager:
	cd ../../operator && make install-cert-manager
	sleep 5 #https://github.com/jetstack/cert-manager/issues/2273

install_seldon: install_cert_manager
	kubectl create namespace seldon-system || echo "namespace seldon-system exists"
	helm install seldon \
		../../helm-charts/seldon-core-operator \
		--namespace seldon-system \
		--set istio.enabled=true \
		--set istio.gateway=istio-system/seldon-gateway \
		--set certManager.enabled=false \
		--set executor.enabled=true \
		--wait

install_istio:
	kubectl apply -f istio-1.4.2.yaml
	kubectl rollout status deployment.apps/istio-ingressgateway -n istio-system
	kubectl rollout status deployment.apps/istio-pilot -n istio-system
	kubectl rollout status deployment.apps/istio-citadel -n istio-system
	kubectl create -f ../resources/seldon-gateway.yaml -n istio-system


create_namespaces:
	kubectl create namespace seldon || echo "Namespace seldon already exists"
	kubectl create namespace test1 || echo "Namespace test1 already exists"

set_namespace:
	kubectl config set-context $$(kubectl config current-context) --namespace=seldon

# Derived from https://github.com/kubernetes-sigs/kind/issues/398#issuecomment-478325705
install_metrics:
	kubectl apply -f metrics.yaml

kind_setup: create_namespaces helm_setup install_ambassador install_istio install_jaeger install_seldon set_namespace install_metrics
port-forward-ambassador:
	kubectl port-forward $$(kubectl get pods -n seldon -l app.kubernetes.io/name=ambassador -o jsonpath='{.items[0].metadata.name}') -n seldon 8003:8080


s2i_build_base_images:
	cd ../../wrappers/s2i/python/build_scripts && ./build_all_local.sh

build_protos:
	cp ../../proto/prediction.proto ./proto
	cd ../../proto/tensorflow && make create_protos
	cp -vr ../../proto/tensorflow/tensorflow .
	python -m grpc.tools.protoc -I. --python_out=. --grpc_python_out=. ./proto/prediction.proto
	cd ../../notebooks && make build_protos
	cd ../../proto/k8s && make create_protos

#requires mvn, helm, ks, python, s2i and an accessible k8s cluster (if minikube increase mem and cpu)
#tests will create namespaces, deploy seldon core and create examples

install:
	pip install -r dev_requirements.txt
	pip install -e ../../python

test_parallel: build_protos install
	# Run the core tests in parallel
	pytest \
		--verbose \
		-s \
		-W ignore \
		-n $(PYTEST_WORKERS) \
		-m "not sequential and not notebooks" 2>&1

test_sequential: build_protos install
	# Then run the s2i tests in sequence (as they currently fail in parallel
	pytest \
		--verbose \
		-s \
		-W ignore \
		-n 0 \
		-m "sequential" 2>&1

test_notebooks: build_protos install
	pytest \
		--verbose \
		-s \
		-m "notebooks" \
		-W ignore test_notebooks.py::TestNotebooks 2>&1

.PHONY: clean
clean:
	rm -f proto/prediction*
	rm -rf run.log
	rm -rf tensorflow
	rm -rf my-model
	rm -rf ../../wrappers/s2i/python/_python/
