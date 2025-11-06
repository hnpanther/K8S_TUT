0 - create nexus repositry:
    create two blob => blobdocker,blob-apt
    enbale docker v2
    http port
    docker-all-group:
    proxy docker repo:
                docker-hub-proxy => https://registry.hub.docker.com (and another one with https://registry-1.docker.io)
                k8s-gcr-proxy1 => https://k8s.gcr.io
                k8s-gcr-proxy2 => https://registry.k8s.io
                ghcr-proxy => https://ghcr.io
                quay-proxy => https://quay.io
                gcr-proxy => https://gcr.io

    test -> docker pull 192.168.79.1:5000/pause:3.9

    apt-all-group
    proxy apt repo:
                ubuntu-proxy => https://archive.ubuntu.com/ubuntu/
                ubuntu-security-proxy => http://security.ubuntu.com/ubuntu/
                kubernetes-proxy(set flat) => https://pkgs.k8s.io/core:/stable:/v1.32/deb/
                docker-proxy-apt => https://download.docker.com/linux/ubuntu
                opensuse-proxy(set flat and dist name ubuntu) => http://download.opensuse.org/repositories/isv:/kubernetes:/core:/stable:/v1.32/deb/



