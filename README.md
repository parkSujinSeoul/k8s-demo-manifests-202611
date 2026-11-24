# k8s-demo-manifests (manifest-repo)

[k8s-demo-app-202611](https://github.com/parkSujinSeoul/k8s-demo-app-202611)의 배포 매니페스트입니다. ArgoCD가 이 저장소의 `main` 브랜치를 감시해 클러스터에 자동 반영합니다.

## 구성

```
k8s-demo-manifests-202611/
├── namespace.yaml           demo 네임스페이스 (istio-injection=enabled)
├── deployment.yaml          demo-app-v1(2개), demo-app-v2(1개) · 같은 이미지, APP_VERSION만 다름
├── service.yaml             ClusterIP(demo-app:80) + NodePort(demo-app-nodeport:30080)
├── istio/
│   ├── gateway.yaml         host app.lab02.local, 80/HTTP
│   ├── virtualservice.yaml  헤더 x-version:v2 또는 /v2 → v2, 그 외 → v1
│   └── destinationrule.yaml version 라벨로 subset v1/v2
└── argocd/
    └── application.yaml     ArgoCD Application (자동 Sync, prune, selfHeal)
```

## 환경

| 서버 | IP | 역할 |
|---|---|---|
| bastion-02 | 10.255.27.167 | kubectl, Jenkins, dnsmasq, 시연 |
| master-02 | 10.255.27.192 | Control Plane |
| worker-02 | 10.255.27.230 | 앱, Istio, ArgoCD |

## 적용 순서

Istio(Gateway 등 CRD)와 ArgoCD가 먼저 설치되어 있어야 합니다.

```bash
# 0) deployment.yaml의 YOUR_DOCKERHUB_ID 변경 후 push (최초 1회)

# 1) 수동 확인용 (ArgoCD 전)
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml -f service.yaml
kubectl apply -f istio/

# 2) ArgoCD에 맡기기 (이후로는 Git push만으로 반영)
kubectl apply -f argocd/application.yaml
kubectl -n argocd get applications
```

## 연결 확인 (bastion)

```bash
# 3.1 NodePort
curl http://10.255.27.230:30080/api/info

# 3.2 Istio (ingressgateway NodePort 확인)
GW_PORT=$(kubectl -n istio-system get svc istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
GW=http://10.255.27.230:$GW_PORT
H="Host: app.lab02.local"
curl -s -H "$H" $GW/v1/api/info                      # 경로별 → 항상 v1
curl -s -H "$H" $GW/v2/api/info                      # 경로별 → 항상 v2
curl -s -H "$H" -H "x-version: v2" $GW/api/info      # 헤더 → 항상 v2

# 가중치(카나리): / 는 v1 80% : v2 20% → 100번 호출해서 개수 세기
for i in $(seq 100); do curl -s -H "$H" $GW/api/info | grep -o '"version":"v[12]"'; done | sort | uniq -c

# 비교: NodePort는 Istio를 거치지 않아 Pod 개수 비율(v1 2개 : v2 1개)대로 분산
for i in $(seq 100); do curl -s http://10.255.27.230:30080/api/info | grep -o '"version":"v[12]"'; done | sort | uniq -c

# 3.3 DNS (bastion dnsmasq: address=/lab02.local/10.255.27.230)
nslookup app.lab02.local
curl http://app.lab02.local:$GW_PORT/api/info
# 브라우저(bastion에서 터널 또는 hosts 등록): http://app.lab02.local:$GW_PORT/  (v2 화면: /v2/)
kubectl run dnstest --rm -it --image=busybox:1.36 -n demo -- nslookup demo-app.demo.svc.cluster.local
```

| 요청 | 응답 버전 |
|---|---|
| `/v1/...` | 항상 v1 |
| `/v2/...` 또는 헤더 `x-version: v2` | 항상 v2 |
| `/` | v1 약 80% : v2 약 20% (VirtualService 가중치) |
| NodePort 30080 | v1 약 2/3 : v2 약 1/3 (Pod 개수 비율, Istio 미경유) |

Istio를 거치면 설정한 비율대로, NodePort로 직접 가면 Pod 개수대로 나뉩니다. 이 차이로 Istio가 트래픽을 제어한다는 것을 보여 줄 수 있습니다. 비율을 바꿀 때는 `kubectl edit`이 아니라 Git에 커밋합니다(ArgoCD selfHeal이 수동 변경을 되돌림).

## 자동 배포

Jenkins가 `deployment.yaml`의 `image:` 두 줄 태그를 함께 바꿔 커밋합니다. 이 저장소를 직접 수정할 때는 Jenkins 커밋과 충돌하지 않도록 먼저 `git pull` 하세요.
