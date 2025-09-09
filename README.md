# 추가 데이터 전처리

1. 모델링 시 사용할 3개의 주요 칼럼은 CurLatitude(위도), CurLongitude(경도), BMSSOC(배터리 잔량)
    1. 위 칼럼에 대해 누락값 있는 행은 드롭 (`175178` → `3186`)
2. 위도와 경도에 대해 튀는 값 제거
    1. 위도: 37.0~38.3
    2. 경도: 126.5~127.53
    3. `3186` → `3116` 

# 모델링 아이디어

### : KMeans 가중치 클러스터링 알고리즘을 이용한 중심점 기반 접근

- 전기버스들이 주로 다니는 경로의 중간 지점을 찾아 충전소를 배치하는 방법입니다.
    - 전기버스 좌표값들을 클러스터링하여, 즉 군집으로 나눈 뒤
    - 각 군집에서 평균값들을 구하여 중심점, 충전소 좌표를 찾는 방법입니다.
    - 특이점은 배터리 잔량에 가중치를 두었다는 것인데,
        - 같은 거리의 좌표더라도 버스 배터리 잔량이 적을 경우 더 높은 가중치가 부여되게끔 설정하였습니다.
        - 코드상 구현
            
            ```python
            df['weight'] = (1000 - df['BMSSOC']) / 1000
            # 중략
            kmeans = KMeans(n_clusters=n_clusters, init='k-means++', random_state=42)
            df['cluster'] = kmeans.fit_predict(df[['CurLatitude', 'CurLongitude']], sample_weight=df['weight'])
            ```
            

- 군집(충전소) 개수는 사람이 직접 설정해주어야 하는 하이퍼파라미터로, 일단은 3~11의 값을 설정한 후 (충전 비용) / (충전소 수) 값으로 충전소 수에 따라 충전 효율이 어떻게 변화하는지 알아보고자 했습니다.
- 충전 비용은 다음과 같이 계산했습니다.
    
    ```
    충전 비용 = Σ{(충전해야 하는 버스 배터리 양) * (버스 위치와 제일 가까운 충전소와의 거리)} 
    ```
    

버스와 충전소 간 거리가 멀수록 중요한 데이터이고,

충전해야 하는 배터리 양이 많을수록 중요한 데이터이므로

이 둘을 곱한 값을 최소화하는 것이 이 모델링의 목표입니다.

Minimize Charge Cost = Σ{(1 - Battery level) * (Distance to Nearest Charging Station)}

- 충전소 1대 설치 및 관리 비용이 있다면
    
    ```
    Minimize Charge Cost + (Number of Charging Stations * Fixed Cost) 
    (Fixed Cost = Installation and Maintenance Cost per Station)
    ```
    
    로 목적함수를 잡아 최소화하면 되겠지만, 해당 값이 없으니
    
    ```
    Minimize Charge Cost * (Number of Charging Stations)
    ```
    
    를 목적함수로 잡아 최소화하는 걸로 하였습니다.
    
    (충전소 수 적을수록 이득이고, 충전비용이 적을수록 이득이므로)