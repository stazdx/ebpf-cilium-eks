# eBPF - Cilium - EKS

Para la siguiente demo usaremos lo siguiente:

- [Multipass](https://multipass.run/)
- [Helm](https://helm.sh/)

### Instalamos Multipass

Podemos instalarlo tanto en Windows, Mac como Linux.

| OS | Referencia |
| ------ | ------ |
| Windows | https://multipass.run/docs/installing-on-windows |
| Mac | https://multipass.run/docs/installing-on-macos |
| Linux | https://multipass.run/docs/installing-on-linux |

Para esta Demo, utilizaremos la instalación en mac utilizando Homebrew

```sh
# Instalamos Multipass con brew cask
$ brew install --cask multipass

# Verificamos que lo tenemos instalado
$ multipass
```

### Creamos nuestro nodo Linux (Ubuntu)

```sh
# Crearemos una instancia Ubuntu pasando como parámetros nombre del nodo, ram y disco que le asignaremos
$ multipass launch --name ebpfdemo --mem 2G --disk 5G

# Ingresaremos dentro de nuestra instancia ya creada
$ multipass shell ebpfdemo

# Verificamos que estamos dentro de la instancia y escalaremos a root
$ sudo su

# Instalamos dependencias
$ apt update
$ apt install -y bpfcc-tools linux-headers-$(uname -r)

# Clonamos el repo
$ git clone https://github.com/stazdx/ebpf-cilium-eks.git

# ejecutamos el ejemplo ebpf, 
$ cd ebpf-cilium-eks/
$ python3 hello.py

# En otra sesion del terminal ingresan a la instancia Ubuntu 
$ multipass shell ebpfdemo
$ sudo su

# Ejecutan cualquier comando linux y regresaran a la sesion anterior para ver las trazas resultado
$ ps ax
$ ls

# Al finalizar la prueba, salen de la instancia
$ exit
$ exit

# Eliminan la instancia para liberar recursos
$ multipass stop ebpfdemo
$ multipass delete ebpfdemo
$ multipass purge

```

### Cilium en EKS

```
# Creamos un cluster de EKS con eksctl sin nodegroup asociado
$ brew install weaveworks/tap/eksctl
$ eksctl create cluster --name ciliumdemo --without-nodegroup

# Eliminamos aws-node para instalar Cilium 
$ kubectl -n kube-system delete daemonset aws-node

# Instalamos Cilium con helm
$ brew install helm@3
$ helm repo add cilium https://helm.cilium.io/
$ helm install cilium cilium/cilium --version 1.9.18 \
  --namespace kube-system \
  --set eni=true \
  --set ipam.mode=eni \
  --set egressMasqueradeInterfaces=eth0 \
  --set tunnel=disabled \
  --set nodeinit.enabled=true

# Ahora si, creamos el nodegroup con 2 nodos
$ eksctl create nodegroup --cluster ciliumdemo --nodes 2
$ kubectl -n kube-system get pods --watch

# Cuando los pods esten en Ready, creamos un namespace para nuestro ejemplo
$ kubectl create ns cilium-test

# Instalamos ejemplos de Cilium
$ kubectl apply -n cilium-test -f https://raw.githubusercontent.com/cilium/cilium/v1.9/examples/kubernetes/connectivity-check/connectivity-check.yaml

# Verificamos que esten creados
$ kubectl get pods -n cilium-test

# Instalamos hubble, para poder ver el tracing del cluster con Cilium
$ helm upgrade cilium cilium/cilium --version 1.9.18 \
   --namespace kube-system \
   --reuse-values \
   --set hubble.listenAddress=":4244" \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true

# Realizamos un port-forward para poder acceder a la interfaz localmente
$ kubectl port-forward -n kube-system svc/hubble-ui --address 0.0.0.0 --address :: 12000:80
```

Finalmente ingresamos a http://localhost:12000 en nuestro navegador para ver la UI.

# Para borrar el cluster EKS ejecutamos

```
$ eksctl delete cluster --name ciliumdemo
```

Referencia: https://docs.cilium.io/en/v1.9/gettingstarted/k8s-install-eks/



Happy coding! 
