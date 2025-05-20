prepare:
	# Обновление системы
	sudo apt update && sudo apt upgrade -y

	# Отключение swap (обязательно для kubelet)
	sudo swapoff -a
	sudo sed -i '/ swap / s/^/#/' /etc/fstab

	# Настройки sysctl для Kubernetes
	echo "br_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
	echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee /etc/sysctl.d/k8s.conf
	echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.d/k8s.conf

	sudo modprobe br_netfilter
	sudo sysctl --system

	# Установка containerd
	sudo apt install -y containerd

	# Настройка containerd
	sudo mkdir -p /etc/containerd
	containerd config default | sudo tee /etc/containerd/config.toml
	sudo systemctl restart containerd
	sudo systemctl enable containerd


install:
	sudo apt-get update
	sudo mkdir /etc/apt/keyrings
	sudo apt-get install -y apt-transport-https ca-certificates curl gpg
	curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
	echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
	sudo apt-get update
	sudo apt-get install -y kubelet kubeadm kubectl
	sudo apt-mark hold kubelet kubeadm kubectl
	sudo systemctl enable --now kubelet

	sudo sysctl -w net.ipv4.ip_forward=1
	echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf


master-settings:
	sudo kubeadm init --pod-network-cidr=192.168.0.0/16 \
	--apiserver-cert-extra-sans=158.160.180.27
	# Добавить внешний ip

	mkdir -p $(HOME)/.kube
	sudo cp /etc/kubernetes/admin.conf $(HOME)/.kube/config
	sudo chown $$(id -u):$$(id -g) $(HOME)/.kube/config

	kubectl apply --validate=false -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Пересоздать токен для подключения воркеров
# kubeadm token generate
# kubeadm token create <generated-token> --print-join-command --ttl=0


helm:
	curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
	helm version


dashboard:
	helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
	helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

# Создание сервисного аккаунта
#kubectl apply -f - <<EOF
#apiVersion: v1
#kind: ServiceAccount
#metadata:
#  name: admin-user
#  namespace: kubernetes-dashboard
#---
#apiVersion: rbac.authorization.k8s.io/v1
#kind: ClusterRoleBinding
#metadata:
#  name: admin-user-binding
#roleRef:
#  apiGroup: rbac.authorization.k8s.io
#  kind: ClusterRole
#  name: cluster-admin
#subjects:
#- kind: ServiceAccount
#  name: admin-user
#  namespace: kubernetes-dashboard
#EOF

# получить токен kubectl -n kubernetes-dashboard create token admin-user
# локально kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
# https://localhost:8443


monitoring:
	helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
	helm repo update
	kubectl create namespace monitoring
	helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring
	# User: admin, пароль можно посмотреть:
	# kubectl get secret --namespace monitoring prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d


master: prepare install master-settings helm dashboard monitoring

worker: prepare install
# join Вручную добавить токен мастера