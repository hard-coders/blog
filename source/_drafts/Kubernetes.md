---
title: Kubernetes
tags: kubernetes, k8s, minikube
---

## 노드?

### 마스터 노드

- kube-apiserver: API 앤드포인트
- etcd: 키-값 저장소
- kube-scheduler: 새로 생긴 팟 감지 후 노드에 배정
- kube-controller-manager: 프로세스 관리
- cloud-controller-manager: 클라우드

### 일반 노드

- kubelet: 각 노드의 에이전트
- kube-proxy: 네트워크
- Container Runtime: docker, containerd, rktlet etc.
