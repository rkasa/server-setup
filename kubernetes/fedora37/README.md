# Installing Harbor and Kubernetes Cluster on Fedora37

Proxy 経由でインターネット疎通できる環境に以下を構築する。OSは全て Fedora37 を用いる。 \
構築作業は以下順番で行うこと。

1. [管理クライアント](10-management.md)
1. [Harbor](20-harbor.md)
1. [Kubernetes クラスタ](30-kubernetes.md)

## Appendix: Proxyサーバ疎通許可

- Proxy サーバで疎通先を制限する場合、以下ドメインへの通信を許可すること
  - amazonaws.com
  - pkg.dev
  - docker.com
  - docker.io
  - github.com
  - githubusercontent.com
  - googleapis.com
  - gstatic.com
  - k8s.io
  - librivox.org
  - quay.io
  - riken.jp
  - projectcontour.io
  - ghcr.io
  - tigera.io
  - helm.sh
  - github.io

- Squid での設定例
  ```text
  acl whitelist dstdomain .amazonaws.com .pkg.dev .docker.com .docker.io .github.com .githubusercontent.com .googleapis.com .gstatic.com .k8s.io .librivox.org .quay.io .riken.jp .projectcontour.io ghcr.io .tigera.io .helm.sh .github.io
  http_access allow whitelist
  ```

