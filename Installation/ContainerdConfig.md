after creating config file 
set  SystemdCgroup = true:
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = true
            
and set path config:
[plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"
and comment another things in this part

create certs.d folder and create folder for every repo and private repo:

docker.io:
server = "https://registry-1.docker.io"

[host."http://192.168.44.1:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true



quay.io:
server = "https://quay.io"

[host."http://192.168.44.1:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true


k8s.gcr.io:
server = "https://k8s.gcr.io"

[host."http://192.168.44.1:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true

registry.k8s.io:
server = "https://registry.k8s.io"

[host."http://192.168.44.1:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true


gcr.io:
server = "https://gcr.io"

[host."http://192.168.44.1:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true


ghcr.io:
server = "https://ghcr.io"

[host."http://192.168.44.1:5000"]
  capabilities = ["pull", "resolve"]
  skip_verify = true

and finally private reop:
server = "http://192.168.44.1:5000"

[host."http://192.168.44.1:5000"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
