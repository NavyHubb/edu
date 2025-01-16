# lecture-99 : 실습 과제

### 서비스 구성
![실습과제](/lecture99/img/lecture10-homework.png)

```dtd
# MVP 발표 진행
  일시 : 1/23(목) 16:00

# 시나리오
1. namespace를 2개 생성 : edu-user, edu-goods
2. demo 서비스를 참고하여 edu-user, edu-goods 서비스 생성
   - API : edu-user.xxx.xxx.xxx.xxx.sslip.io/api/v1/user/{userNo}
           -> goods/api/v1/{goodsNo} 호출 결과 받아서
           -> return : userNo, userName(prod-userNo), goodsNo(userNo), goodsName(goods-prod-userNo)
   - API : edu-goods.xxx.xxx.xxx.xxx.sslip.io/api/v1/goods/{goodsNo}
           ->  goodsNo(goodsNo), goodsName(goods-prod-goodsNo) 
3. HPA 설정 : edu-user(min:2, max:4), edu-goods(min:2, max:3)
   - 조건 : cpu 평균 사용량 30% 이상일때 1개씩 pod 증가
4. porb 설정하여 pod 내 어플리케이션이 다운되면 자동으로 pod start하게 설정
   - 체크 조건 : "/" 호출 하여 상태 체크
5. argocd 통해 배포
```