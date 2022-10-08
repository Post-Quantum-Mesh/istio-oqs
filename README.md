# Post-Quantum Istio Deployment

<a href="https://istio.io/">
    <img src="https://github.com/istio/istio/raw/master/logo/istio-bluelogo-whitebackground-unframed.svg"
         alt="Istio logo" title="Istio" height="150" width="150" />
</a>

Open source implementation of quantum-resistant encryption algorithms for mesh microservice deployement

- [Components]()
- [Overview]()
- [Quick Start]()
  - [Quick Install Demo]()
  
## Components

### Quantum-Resistant Library/TLS Protocol
- [liboqs](https://github.com/open-quantum-safe/liboqs)
- [modified boringssl-liboqs fork](https://github.com/drouhana/boringssl.git)
- [openssl-liboqs fork](https://github.com/open-quantum-safe/openssl)
- [openssl-1.1.1](https://github.com/openssl/openssl/tree/OpenSSL_1_1_1-stable)

### Istio Build
- [istio/proxy-1.15.0](https://github.com/drouhana/proxy)
- [istio/istio-1.15.0](https://github.com/drouhana/istio)


## Overview

< space holder >  

## Quick Start

### Quick Install Demo

Installation Notes:
- Certain commands ran as "sudo" may require re-installation of dependencies as root user
- Clang must be installed and added to path for compilation
- Go must be installed, instructions found [here](https://go.dev/doc/install)
- Docker engine must be installed and configured
- Install and set-up minikube with following config parameters:
    - cpus > 4
    - memory > 16384
    - driver=docker
- Install and set-up [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- Enable bash completion for make by running the following command

       echo "complete -W \"\`find . -iname \"?akefil*\" | xargs -I {} grep -hoE '^[a-zA-Z0-9_.-]+:([^=]|$)' {} | sed 's/[^a-zA-Z0-9_.-]*$//' | sort -u\`\" make" >> ~/.profile

1. Update package manager

        sudo apt-get update

2. Install Envoy dependencies

        sudo apt-get install -y autoconf automake cmake curl libtool make ninja-build patch python3-pip unzip virtualenv

3. Install bazelisk as bazel

        sudo wget -O /usr/local/bin/bazel https://github.com/bazelbuild/bazelisk/releases/latest/download/bazelisk-linux-$([ $(uname -m) = "aarch64" ] && echo "arm64" || echo "amd64")
        sudo chmod +x /usr/local/bin/bazel

4. Clone istio repos into current directory

        cd /usr/local/go/src/istio.io
        sudo git clone https://github.com/drouhana/istio.git
        sudo git clone https://github.com/drouhana/proxy.git

5. Build istio/proxy using [envoy-oqs image](https://hub.docker.com/layers/drouhana/envoy-oqs/envoy/images/sha256-e779ccfd8707e31fbf3f47f1f2ac99cb52ea56f6e923a87fbb12b7fa1dbca114?context=repo) (Note: "sudo" is not needed for build_envoy command, as the previous commands opens a bash terminal as root)

        cd proxy
        sudo docker pull drouhana/envoy-oqs:envoy
        sudo docker run -it -w /work -v $PWD:/work drouhana/envoy-oqs:envoy bash
        make build_envoy

6. Start minikube cluster

        minikube start

7. Build istio

        cd ../istio
        sudo make build

8. Build istio docker images

        sudo make docker

9. Install istio from previous step into cluster

        go run ./istioctl/cmd/istioctl install
  
10. Label default namespace with sidecar auto-injection

        kubectl label namespace default istio-injection=enabled

11. Deploy bookinfo application

        kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

12. Confirm all pods are correctly defined and running

        kubectl get services
        kubectl get pods

Console Output: \
<img width="539" alt="Screen Shot 2022-10-08 at 09 44 43" src="https://user-images.githubusercontent.com/56026339/194718243-b5094a7a-cc31-460b-8022-c5811a2b5a98.png">

13. Define the ingress gateway for the application

        kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml

14. Confirm gateway has been created

        kubectl get gateway

Console Output: \
<img width="497" alt="Screen Shot 2022-10-08 at 09 46 22" src="https://user-images.githubusercontent.com/56026339/194718309-b2c52ada-89d7-4506-8ac7-f6793735022e.png">

15. Start minikube external load balancer (Note: do this in a different terminal)

        minikube tunnel

16. Get INGRESS_PORT (Note: will be under columnn "exernal-IP." If having issues, troubleshoot using [istio ingress documentation](https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports))

        kubectl get svc istio-ingressgateway -n istio-system

17. Set ingress and IP ports

        export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
        export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
        export TCP_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="tcp")].port}')

18. Set gateway URL

        export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT

19. Confirm pre-quantum curl implementation can access productpage

        curl -v "http://${GATEWAY_URL}/productpage" | grep -o "<title>.*</title>"

Console Output: \
<img width="870" alt="Screen Shot 2022-10-08 at 09 52 06" src="https://user-images.githubusercontent.com/56026339/194718568-2e31d5c9-379a-462f-a617-cb0a58de03be.png">

20. Confirm post-quantum curl implementation can access productpage (Note: post-quantum curl is run from a docker container. Adding the grep addendum to the end of the command will remove the verbose curl printout, so it has been left out of so the TLS output can be displayed. Console output continues beyond screenshot.)

        sudo docker run --network host -it openquantumsafe/curl curl -k -v "http://${GATEWAY_URL}/productpage" -e SIG_ALG=dilithium3

Console Output: \
<img width="1112" alt="Screen Shot 2022-10-08 at 09 57 27" src="https://user-images.githubusercontent.com/56026339/194718796-c3f9f8d9-dd63-4616-8efc-8136991c64b6.png">

21. Uninstall istio installation

        samples/bookinfo/platform/kube/cleanup.sh

22. Confirm clean-up

        kubectl get virtualservices
        kubectl get destinationrules
        kubectl get gateway
        kubectl get pods

Console Output: \
<img width="581" alt="Screen Shot 2022-10-08 at 10 00 48" src="https://user-images.githubusercontent.com/56026339/194718932-98aa502d-90fa-4b62-90ba-4d7d95adc597.png">
