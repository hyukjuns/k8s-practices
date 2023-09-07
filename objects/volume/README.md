### EmptyDir
- html을 3초마다 생성하는 컨테이너를 통해 web컨테이너는 계속해서 다른 index.html파일을 보게 된다.
- 결과 확인용 nettool을 띄워 사용
``` shell
$ kubectl run nettool -it --image=c1t1d0s7/network-multitool
```
- nettool에서 curl을 사용해 3초마다 index.html이 바뀌는지 확인

### gitRepo
- 현재 gitRepo는 사용 불가 처리 되었으므로 initContainers 방식으로 gitrepo를 구현해볼 수 있다.
- 실제 git 저장소가 클론 됐는지 확인
```shell
kubectl exec hj-pod-gitrepo -- ls -l /repo
```

### hostPath
파드가 동작하는 쿠버네티스 클러스터의 노드(호스트)의 로컬 파일시스템의 파일 및 디렉토리를 파드가 사용할 수 있는 볼륨으로 제공, 같은 노드에 배치된 파드 및 컨테이너에게 공유 디렉토리를 제공할 목적으로 사용

### PV/PVC
- 정적 볼륨 프로비져닝
    마스터노드에 NFS스토리지를 구성하여 PV 및 PVC를 구성하고 RS를 통해 확인한다.
- 동적 볼륨 프로비져닝
    Cephfs의 Storage Class 를 통해 pv가 자동으로 생성된다.

