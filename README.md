# Chaoslab_GitopsRepo
![Kubernets](https://img.shields.io/badge/-Kubernetes-326CE5?style=for-the-badge&logo=Kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/-Helm-0F1689?style=for-the-badge&logo=Helm&logoColor=white)
![Argo](https://img.shields.io/badge/-ArgoCD-fe733d?style=for-the-badge&logo=&logoColor=white)

Chaos Lab 실험에 필요한 Chaos Mesh, 모니터링 도구, podinfo를 설치하는 ArgoCD ApplicationSet 기반 GitOps 저장소입니다.

## 배포 대상

| 앱 | 출처 | 용도 |
|----|------|------|
| podinfo | Helm chart (외부) | 카오스 테스트 대상 앱 |
| kube-prometheus-stack | Helm chart (외부) | Prometheus + Grafana 모니터링 |
| chaos-mesh | Helm chart (외부) | 장애 주입 도구 |
| loki-stack | Helm chart (외부) | 로그 수집 |
| chaoslab-base | Helm chart (로컬) | Chaos Mesh RBAC, SA Token 등 기반 리소스 |

## 구조
```
Chaoslab_GitopsRepo/
├── bootstrap/
│   ├── appset-cloud.yaml          # 클라우드 환경용 ApplicationSet
│   └── appset-local.yaml          # 로컬 환경용 ApplicationSet
├── env/
│   └── config-cloud.json          # 환경별 도메인 설정
├── values/
│   ├── argocd/
│   │   ├── cloud.yaml             # ArgoCD Helm values (클라우드)
│   │   └── local.yaml             # ArgoCD Helm values (로컬)
│   └── apps/
│       ├── podinfo.yaml
│       ├── prometheus.yaml
│       ├── chaos-mesh.yaml
│       └── loki.yaml
├── charts/
│   └── chaoslab-base/
│       ├── Chart.yaml
│       └── templates/
│           ├── rbac.yaml
│           └── chaos-token.yaml
└── manifests/
    └── storageclass/
        └── storageclass.yaml      # 환경별 수동 적용
```

## 선수요건

1. NGINX Ingress Controller가 설치되어 있어야 합니다.
2. StorageClass 설정 (환경별)
```bash
kubectl apply -f manifests/storageclass/storageclass.yaml
```

## 1. Ingress IP 확인
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# 확인된 IP를 변수로 설정
INGRESS_IP=<확인된 IP>
```

## 2. ArgoCD 설치

Helm chart로 설치하며, Ingress를 동시에 활성화합니다.
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --namespace argocd --create-namespace \
  -f values/argocd/[env].yaml # 각 환경별 /values/argocd/ 에서 확인 후 설정

# 설치 확인
kubectl get pods -n argocd
kubectl get ingress -n argocd

# 초기 admin 비밀번호 확인
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

접속: `https://argocd.<INGRESS_IP>` (ID: admin)

## 3. config.json 설정
```bash
# env/config-cloud.json을 편집 후 push
vi env/config-cloud.json
# e.g. { "domain": "<새 IP>.nip.io" }
git add env/config-cloud.json
git commit -m "set ingress IP"
git push
```

## 4. ApplicationSet 적용
```bash
kubectl apply -f bootstrap/appset-cloud.yaml   # 클라우드
kubectl apply -f bootstrap/appset-local.yaml    # 로컬

# 5개 앱 생성 확인
kubectl get applications -n argocd
```

## 도메인 변경 시

env/config-cloud.json만 수정하면 5개 앱에 자동 반영됩니다.
```bash
vi env/config-cloud.json
# { "domain": "<새 IP>.nip.io" }
git add env/config-cloud.json && git commit -m "update domain" && git push
```