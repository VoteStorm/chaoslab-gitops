# VoteStorm_GitopsRepo
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
| votestorm-base | Helm chart (로컬) | RBAC, StorageClass 등 기반 리소스 |

## 구조
```
VoteStorm_GitopsRepo/
├── config.json                  # [SSOT] 전역 설정 (IP 등)
├── bootstrap/
│   └── appset.yaml              # [Control] 5개 앱을 관리하는 ApplicationSet
└── charts/
    └── votestorm-base/          # [Local Chart] chart에 없는 순수 K8s 리소스
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── rbac.yaml            # Chaos Mesh 전용
            ├── chaos-token.yaml     # Chaos Mesh 전용
            └── storageclass.yaml    # 인프라 레벨
```

## 선수요건

NGINX Ingress Controller가 설치되어 있어야 합니다.

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

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 5.55.0 \
  --set server.ingress.enabled=true \
  --set server.ingress.ingressClassName=nginx \
  --set "server.ingress.hosts[0]=argocd.${INGRESS_IP}.nip.io" \
  --set "server.ingress.tls[0].hosts[0]=argocd.${INGRESS_IP}.nip.io" \
  --set 'server.ingress.annotations.nginx\.ingress\.kubernetes\.io/ssl-passthrough=true' \
  --set 'server.ingress.annotations.nginx\.ingress\.kubernetes\.io/backend-protocol=HTTPS' \
  --wait --timeout 120s

# 설치 확인
kubectl get pods -n argocd
kubectl get ingress -n argocd

# 초기 admin 비밀번호 확인
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d && echo
```

접속: `https://argocd.<INGRESS_IP>.nip.io` (ID: admin)

## 3. config.json 설정
```bash
# config.json에 확인된 IP 반영
echo "{ \"ingressIP\": \"${INGRESS_IP}\" }" > config.json
git add config.json
git commit -m "set ingress IP"
git push
```

## 4. ApplicationSet 적용
```bash
kubectl apply -f bootstrap/appset.yaml

# 5개 앱 생성 확인
kubectl get applications -n argocd
```

## IP 변경 시

config.json만 수정하면 5개 앱에 자동 반영됩니다.
```bash
echo '{ "ingressIP": "<새 IP>" }' > config.json
git add config.json && git commit -m "update IP" && git push
```