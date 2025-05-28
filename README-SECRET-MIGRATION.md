# EUM-AI Secret Migration Guide

## 🚨 중요: Sealed Secret 네임스페이스 바인딩 문제

Sealed Secret은 **특정 네임스페이스에 바인딩**되어 있어 다른 네임스페이스에서 사용할 수 없습니다.

### 문제 상황
- 기존: `eum-ai` 네임스페이스용 Sealed Secret
- 새로운: `eum-ai-helm` 네임스페이스에서 사용 시 **복호화 실패**

## 🔧 해결 방안

### Option 1: 기존 Secret 마이그레이션 (추천)

#### 1. Secret 마이그레이션 스크립트 실행
```bash
# 스크립트 실행 권한 부여
chmod +x scripts/migrate-secrets.sh

# Secret 마이그레이션 실행
./scripts/migrate-secrets.sh
```

#### 2. Helm Chart 배포 (Sealed Secret 비활성화)
```bash
# values.yaml에서 sealedSecrets.enabled: false 확인 후
helm install eum-ai-test ./eum-ai --namespace eum-ai-helm
```

### Option 2: 새로운 Sealed Secret 생성

#### 1. 기존 Secret 값들을 추출
```bash
# 각 Secret의 값들을 확인
kubectl get secret discussion-room-secret -n eum-ai -o yaml
kubectl get secret agentic-secret -n eum-ai -o yaml
kubectl get secret chatbot-secret -n eum-ai -o yaml
```

#### 2. 새 네임스페이스용 Sealed Secret 생성
```bash
# kubeseal을 사용하여 새 네임스페이스용으로 암호화
echo -n "your-secret-value" | kubectl create secret generic discussion-room-secret \
  --dry-run=client --from-file=GROQ_API_KEY=/dev/stdin -o yaml | \
  kubeseal --namespace eum-ai-helm -o yaml > new-discussion-room-sealed-secret.yaml
```

## 📋 현재 Secret 상태

### 기존 네임스페이스 (eum-ai)
```
NAME                     TYPE     DATA   AGE
agentic-secret           Opaque   8      5d1h
chatbot-secret           Opaque   3      5d1h
discussion-room-secret   Opaque   6      5d1h
```

### 누락된 Secret
- `classifier-secret`: Sealed Secret은 있으나 실제 Secret 미생성

## 🔍 Verification Steps

### 1. Secret 마이그레이션 확인
```bash
kubectl get secret -n eum-ai-helm
```

### 2. Helm 배포 테스트
```bash
helm install eum-ai-test ./eum-ai --namespace eum-ai-helm --dry-run
```

### 3. Pod 시작 확인
```bash
kubectl get pods -n eum-ai-helm
kubectl logs -n eum-ai-helm <pod-name>
```

## ⚙️ Configuration Changes

### values.yaml 설정
```yaml
# Sealed Secrets 비활성화
sealedSecrets:
  enabled: false

# Secret 마이그레이션 모드 활성화
secretMigration:
  enabled: true
  sourceNamespace: "eum-ai"
```

## 🚀 Migration Workflow

1. **Secret 마이그레이션**: `./scripts/migrate-secrets.sh`
2. **Helm 테스트**: `helm install eum-ai-test ./eum-ai --namespace eum-ai-helm`
3. **Pod 동작 확인**: Secret 읽기 및 앱 시작 확인
4. **ArgoCD 설정**: Helm 기반 배포로 전환
5. **기존 리소스 정리**: 성공 후 기존 매니페스트 기반 리소스 삭제

## ⚠️ 주의사항

- **Blue-Green 배포**: 기존 서비스가 계속 실행되므로 안전
- **Secret 보안**: 마이그레이션 중 Secret 값 노출 주의
- **네임스페이스 분리**: 테스트 완료 후 기존 네임스페이스 정리
- **Rollback 계획**: 문제 발생 시 기존 설정으로 복원 가능 