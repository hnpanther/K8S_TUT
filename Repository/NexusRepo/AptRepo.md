set repo for apt
in /etc/apt/ rename sources.list and rename files in sources.list.d the create new file => sources.list:

# Kubernetes packages(from opensuse) (v1.32)
deb [trusted=yes] http://192.168.220.1:8081/repository/opensuse-proxy/ /

# Kubernetes packages (v1.32)
deb [trusted=yes] http://192.168.220.1:8081/repository/kubernetes-proxy/ /

# Docker CE packages
deb [trusted=yes] http://192.168.220.1:8081/repository/docker-proxy-apt/ noble stable

# Ubuntu Security & Base
deb [trusted=yes] http://192.168.220.1:8081/repository/ubuntu-security-proxy/ noble-security main restricted universe multiverse
deb [trusted=yes] http://192.168.220.1:8081/repository/ubuntu-proxy/ noble main restricted universe multiverse

then apt clean and apt update