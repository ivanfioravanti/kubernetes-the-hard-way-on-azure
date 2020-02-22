# 데이터 암호화 구성 및 키 생성

쿠버네티스는 클러스터 상태, 응용 프로그램 구성 및 비밀 데이터를 포함한 다양한 데이터를 저장합니다. 쿠버네티스는 유휴 클러스터 데이터를 [암호화](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) 하는 기능을 지원합니다.

이 실습에서는 쿠버네티스 비밀 데이터를 암호화에 사용할 암호화 키 및 생성 [암호화 설정](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration)을 적용할 것입니다.

## 암호화 키

암호화 키를 생성합니다.

```shell
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## 암호화 구성 파일

`encryption-config.yaml` 암호화 구성 파일을 만듭니다.

```shell
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

`encryption-config.yaml` 암호화 구성 파일을 각 컨트롤러 인스턴스에 복사합니다.

```shell
for instance in controller-0 controller-1 controller-2; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -otsv)

  scp -o StrictHostKeyChecking=no encryption-config.yaml kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```

다음: [etcd 클러스터 부트 스트랩](07-bootstrapping-etcd.md)
