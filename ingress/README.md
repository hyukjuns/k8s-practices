# 다중 호스트 인그레스 구성
1. 각기 다른 레이블 셀렉터를 가지며 다른 기능을 제공하는 ReplicSet 생성(myweb(nginx), httpd(apache))
2. 연결할 서비스는 ClusterIP 사용, 마찬가지로 각각 다른 레이블 셀렉터를 가지며 파드를 따로 관리
3. 인그레스를 생성할때 다중 호스트 인그레스 구성(Spec 필드의 host 필드를 두개 생성)

```
hjwebapp.com -> svc1 -> myweb(nginx)
namwebapp.com -> svc2 -> httpd(apache)
```