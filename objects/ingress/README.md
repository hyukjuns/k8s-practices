# Ingress with web services
## Architecture
<img src="./image/ingress.png"></img>
## Resources
- Ingress
- Service: LoadBalancer
- WebService: apache, nginx
- ReplicaSet
- ConfigMap by Volume
## Description
<p>
인그레스컨트롤러를 사용해 서비스를 외부로 노출시켜 L7 계층에서의 FQDN통신 가능하게 구성,
컨피그맵을 사용해서 apache pod에는 설정파일과 index.html 파일을 사용자화 하였고,
nginx는 컨피그맵을 사용해서 포트번호를 사용자화 하였습니다.
</p>
