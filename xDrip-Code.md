# xDrip application code 해부


## com/eveningoutpost/dexdrip

### Home.java
- 기본 홈 화면 코드
- updateCurrentBgInfo

## com/eveningoutpost/dexdrip/models/
### LibreBluetooth.java
LibreBluetooth 클래스 정의, Libre 블루투스 연결 코드  

## com/eveningoutpost/dexdrip/services/
### DexCollectionService.java
LibreBluetooth 모델을 이용하여 블루투스로 Libre 데이터를 수집하는 서비스 코드  

### G5CollectionService.java
com.eveningoutpost.dexdrip.g5model 내의 모델을 이용하여 블루투스로 Dexcom 데이터를 수집하는 서비스 코드