# KIND

O kind é um projeto para executar um Cluster Kubernetes localmente em seu computador de uma forma muito simples.

Ele utiliza o Docker para isso.

https://kind.sigs.k8s.io/docs/user/quick-start/


## Instalação

### Linux:

```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Windows Powershell:
```
curl.exe -Lo kind-windows-amd64.exe https://kind.sigs.k8s.io/dl/v0.20.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe
```

## Comandos

Criar um Cluster
```
kind create cluster
```

Visualizar o Cluster
```
kind get clusters
```

Para visualizar os Nodes do seu Cluster.

*Tenha o kubectl instalado*
```
kubectl get nodes
```

Apagar um Cluster
```
kind delete cluster
```

## Avançado

Criar um cluster com 1 Control Plane e 2 Workers.

Necessita criar um arquivo de configuração:
*Já inclui opções para poder utilizar um ingress controler (Traefik)*

kind-config.yaml
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
    - |
      kind: InitConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "ingress-ready=true"
  extraPortMappings:
    - containerPort: 80
      hostPort: 80
      protocol: TCP
    - containerPort: 443
      hostPort: 443
      protocol: TCP

- role: worker
- role: worker

```

Executando este cluster:
```
kind create cluster --config kind-config.yaml
```


# LoadBalancer para o Kind (Metallb)

O kind permite que se crie um LoadBalancer para fins de testes e estudos.

https://kind.sigs.k8s.io/docs/user/loadbalancer/


## Configuração

Aplique este manifesto abaixo:
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

Verifique o Pool de Endereço que poderá ser utilizado:
```
docker network inspect -f '{{.IPAM.Config}}' kind
```
A saida deve ser algo como:

172.18.0.0/16 ou 
<br>
172.19.0.0/16

Crie um arquivo chamado metallb.yaml com o conteúdo abaixo:
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: example
  namespace: metallb-system
spec:
  addresses:
  - 172.19.255.200-172.19.255.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
```
*altere a linha do ip address, informando a rede de acordo com o valor obtido no passo anterior.*

Aplique o manifesto:
```
kubectl apply -f metallb.yaml
```

Para testar o LoadBalancer, execute o manifesto abaixo:
*Este manifesto tem porta padrão 5678*
```
kubectl apply -f https://kind.sigs.k8s.io/examples/loadbalancer/usage.yaml
```

Veja o IP que o LoadBalancer obteve para o Serviço:
```
kubectl get service
```

```
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
foo-service   LoadBalancer   10.96.134.133   172.18.255.200   5678:32386/TCP   8s
kubernetes    ClusterIP      10.96.0.1       <none>           443/TCP          31m
```

Verifique o IP na coluna External-IP.

Abra o Navegador e informe o IP + porta do Manifesto:

http://172.18.255.200:5678

*Se aparecer alguma coisa é porque funcionou.*


# Metrics Server

https://github.com/kubernetes-sigs/metrics-server

O Metrics Server coleta Metricas de CPU e Memória dos Containers (Pods) do Kubernetes.

Para instalar, o ideal é fazer download do arquivo de manifesto e acrescentar uma linha de argumento no Deployment.
```
--kubelet-insecure-tls
```

Em seguida, aplique o Manifesto.

```
kubectl apply -f metrics-server.yaml
```

Para visualizar o consumo de CPU e Memória de um POD:
```
kubectl top pod nomedopod
```

# NFS Storage Class

Requisito: \
Tenha o helm instalado. \
Tenha um Servidor NFS.

Adicionar o repositório do nfs-subdir-external-provisioner
```
$ helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```

Instalar e setar as configurações. \
Atenção: Altere os valores conforme necessário.
```
$ helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --set nfs.server=127.0.0.1 \
    --set nfs.path=/nfs \
    --set storageClass.reclaimPolicy=Retain \
    --create-namespace -n nfs-provisioner
```

# Traefik

O Traefik é um Proxy (Edge Router) para ser utilizado como Ingress Controller.

https://doc.traefik.io/traefik/getting-started/install-traefik/


Para Instalar é bem simples:

```
helm repo add traefik https://traefik.github.io/charts
```

Se quiser visualizar os Values para modificar alguma coisa.
```
helm show values traefik/traefik > traefik.yaml
```
```
helm upgrade --install traefik traefik/traefik --create-namespace -n traefik
```

# Weave CNI 

Este CNI permite trabalhar com regras de rede (Network Policies)

https://www.weave.works/docs/net/latest/kubernetes/kube-addon/

Deve primeiro não adicionar o CNI padrão no KIND, modificando o arquivo de configuração do Cluster.

Depois deve-se executar o comando:

```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```