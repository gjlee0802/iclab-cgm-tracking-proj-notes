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
- onStartCommand : 서비스 시작과 끝이 onCreate()와 onDestroy 였다면, 서비스의 활성 수명은 onStartCommand()와 onBind()에서 시작함.
    - 1. Context 및 데이터베이스 초기화
        ~~~
        Context context = getApplicationContext();
        xdrip.checkAppContext(context);
        Sensor.InitDb(context); // ensure db is initialized
        ~~~
    - 2. WakeLock 설정
        ~~~
        final PowerManager.WakeLock wl = JoH.getWakeLock("g5-start-service", 120000);
        ~~~
    - 3. 서비스 실행 상태 확인 및 초기 설정
        ~~~
        if ((!service_running) && (keep_running)) {
            service_running = true;

            checkWakeupTimeLatency();
            logWakeTimeLatency();

            Log.d(TAG, "onG5StartCommand wakeup: " + JoH.dateTimeText(JoH.tsl()));
            Log.e(TAG, "settingsToString: " + settingsToString());

            lastState = "Started: " + JoH.hourMinuteString();
        }
        ~~~
    - 4. 서비스 실행 조건 확인
        ~~~
        if (!shouldServiceRun()) {
            Log.e(TAG,"Shutting down as no longer using G5 data source");
            service_running = false;
            keep_running = false;
            stopSelf();
            return START_NOT_STICKY;
        }
        ~~~
    - 5. 블루투스 초기화 및 스캔 설정
        ~~~
        scanCycleCount = 0;
        mBluetoothManager = (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);
        mBluetoothAdapter = mBluetoothManager.getAdapter();

        if (mGatt != null) {
            try {
                Log.d(TAG, "onStartCommand mGatt != null; mGatt.close() and set to null.");
                mGatt.close();
                mGatt = null;
            } catch (NullPointerException e) { //
            }
        }

        if (Sensor.isActive()) {
            setupBluetooth();
            Log.d(TAG, "Active Sensor");
        } else {
            stopScan();
            Log.d(TAG, "No Active Sensor");
        }
        ~~~
    - 6. 서비스 상태 갱신 및 종료 처리
        ~~~
        service_running = false;
        return START_STICKY;
        ~~~
    - 7. WakeLock 해제
        ~~~
        } finally {
            JoH.releaseWakeLock(wl);
        }
        ~~~