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
