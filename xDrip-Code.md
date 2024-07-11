# xDrip application code 해부


## com/eveningoutpost/dexdrip/

### Home.java
기본 홈 화면 코드  
BroadcastReceiver을 이용한 데이터 수신 코드 확인 필요  
BroadcastReceiver의 onReceive 메소드를 오버라이드함.  
onReceive 메소드 내에서 updateCurrentBgInfo()를 통해 Blood Glucose 정보 업데이트함.  
아래 onResume 메소드 내의 BroadcastReceiver 인스턴스 생성 참고  
~~~
        _broadcastReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context ctx, Intent intent) {
                if (intent.getAction() != null && intent.getAction().equals(Intent.ACTION_TIME_TICK)) {
                    if (msSince(lastDataTick) > SECOND_IN_MS * 30) {
                        Inevitable.task("process-time-tick", 300, () -> runOnUiThread(() -> {
                            updateCurrentBgInfo("time tick");
                            updateHealthInfo("time_tick");
                        }));
                    }
                }
            }
        };
        newDataReceiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context ctx, Intent intent) {
                lastDataTick = tsl();
                updateCurrentBgInfo("new data");
                updateHealthInfo("new_data");
            }
        };
~~~

updateCurrentBgInfo() 메소드도 중요함으로 확인 필요함.  

#### updateCurrentBgInfo() : 
특정 소스에서 가져온 데이터를 기반으로 현재 혈당 정보를 업데이트함. 이 과정은 UI 요소의 업데이트와 다양한 설정 및 상태를 확인하는 과정을 포함.

#### setupCharts() : 
차트 셋업 코드  
아래처럼 setLineChartData 메서드를 통해 시각화할 데이터를 입력
~~~
chart.setLineChartData(bgGraphBuilder.lineData());
~~~
utilityModels/BgGraphBuilder.java에 정의된 BgGraphBuilder는 수집되는 데이터를 차트에 반영할 수 있도록 함.  

### BestGlucose.java
BestGlucose는 models/BgReading.java의 postProcess 함수에서 사용됨.

#### minutesAgo() :
몇분 전(mssince 값)의 데이터가 갱신되었는지 알려주는 string을 반환함.

#### getDisplayGlucose() :
TODO

### NSEmulatorReceiver.java
CareSens 사용할 경우에 해당.  
BroadcastReceiver을 상속함.
#### onReceive() : 
Broadcast를 수신할 경우에 호출되는 메서드.  
Intents.XDRIP_PLUS_NS_EMULATOR 액션에 대한 처리.  
SGV를 수신할 경우 bgReadingInsertFromData() 메서드를 호출함.  

collection 값을 가져와서, 그 값에 따라 데이터 처리:  
- entries:
    - data 문자열을 가져와서 JSON 배열로 변환.
    - JSON 배열의 길이에 따라 다른 처리:
        - 길이가 1 이상인 경우:
            - 첫 번째 JSON 객체에서 ROW_ID를 가져와서 프로세스 ID를 확인하고, OOP 결과를 처리.
        - 길이가 1인 경우:
            - type 값을 가져와서:
                - sgv: 혈당값과 방향을 처리하여 bgReadingInsertFromData 메서드를 호출.
                - 다른 타입의 경우, 알 수 없는 타입으로 로그를 기록.

#### bgReadingInsertFromData() : 
**BgReading.java의 bgReadingInsertFromJson() 메서드 호출**.  
이 메서드는 주어진 타임스탬프, 혈당값(sgv), 경사(slope) 데이터를 기반으로 새로운 BgReading 객체를 생성하고, 이를 JSON 형태로 변환하여 삽입함.  
데이터 삽입 후 알림을 발생시키고, 필요한 경우 기본 센서를 초기화.  
주요 작업 중 예외가 발생하면 로그에 기록하고 null을 반환.  

### LibreReceiver.java
Freestyle Libre 사용할 경우에 해당.  
BroadcastReceiver을 상속함.

#### onReceive() : 
Broadcast를 수신할 경우에 호출되는 메서드.  
액션별 처리 : 
- Intents.LIBRE2_ACTIVATION: 센서 활성화 처리
    - 센서 시작 시간 저장
    - 기본 센서 생성
- Intents.LIBRE2_SCAN: 스캔된 데이터 처리
    - 실시간 혈당 데이터와 과거 혈당 데이터 처리
    - 그래프 갱신
- Intents.LIBRE2_CONNECTION: 연결 상태 처리
    - 블루투스 주소 및 연결 상태 저장
- Intents.LIBRE2_BG: 혈당 값 처리
    - 현재 값 처리 및 저장
    - NFC 센서 나이 초기화

#### processValues() : 
**BgReading.java의 bgReadingInsertLibre2() 메서드 호출**

## com/eveningoutpost/dexdrip/utilitymodels/

### BgSendQueue.java
#### handleNewBgReading() : 
BgReading의 postProcess함수에서 호출되는 함수로, 새로운 데이터를 전달(브로드캐스트)하는 역할. 동시에 WakeLock을 사용하여 중요한 작업이 수행되는 동안 CPU가 절전 모드로 전환되지 않도록 함.  
- JoH.getWakeLock을 사용하여 120초 동안 유지되는 WakeLock을 획득. WakeLock은 CPU가 잠들지 않도록 방지함.
- UploaderQueue.newEntry 메서드를 호출하여 bgReading을 업로드 큐에 추가. (코드에서 is_follower에 따른 조건은 주석 처리되어 있음.)  
- UI 업데이트: quick가 false인 경우 추가 UI 업데이트를 수행함. Home.activityVisible이 true인 경우 ACTION_NEW_BG_ESTIMATE_NO_DATA 인텐트를 브로드캐스트함. xDripWidget이 있는 경우 WidgetUpdateService를 시작함.  
- 로컬 브로드캐스트 전송: BroadcastGlucose.sendLocalBroadcast 메서드를 호출하여 bgReading을 로컬 브로드캐스트.
- 추가 WakeLock: quick가 false이고 excessive_wakelocks 설정이 true인 경우, 3초 동안 추가 WakeLock을 획득.  
- 새 데이터 관찰자 호출: quick가 false인 경우 NewDataObserver.newBgReading 메서드를 호출하여 새로운 혈당 데이터를 관찰자에게 전달함.
- 팔로워 동기화: is_follower가 false이고 plus_follow_master 설정이 true인 경우, 혈당 데이터를 플러그인을 통해 동기화함.
- 업로더 큐 처리: JoH.ratelimit 메서드를 사용하여 30초 내에 한 번만 SyncService를 시작하도록 함.
- WakeLock 해제: 모든 작업이 완료된 후 JoH.releaseWakeLock 메서드를 호출하여 WakeLock을 해제함.

### BgGraphBuilder.java
BgGraphBuilder 초기화 함수
~~~
    public BgGraphBuilder(Context context, long start, long end, int numValues, boolean show_prediction, final boolean useArchive) {
        // swap argument order if needed
        if (start > end) {
            long temp = end;
            end = start;
            start = temp;
            if (d) Log.d(TAG, "Swapping timestamps");
        }
        if (d)
            Log.d(TAG, "Called timestamps: " + JoH.dateTimeText(start) + " -> " + JoH.dateTimeText(end));
        this.prefs = PreferenceManager.getDefaultSharedPreferences(context);
        prediction_enabled = show_prediction;
        if (prediction_enabled)
            simulation_enabled = prefs.getBoolean("simulations_enabled", true);
        end_time = end / FUZZER;
        start_time = start / FUZZER;

        readings_lock.lock();
        try {
            // store the initialization values used for this instance
            loaded_numValues = numValues;
            loaded_start = start;
            loaded_end = end;
            bgReadings = BgReading.latestForGraph(numValues, start, end);
            if (DexCollectionType.getDexCollectionType() == DexCollectionType.LibreReceiver)
                Libre2RawValues = Libre2RawValue.latestForGraph(numValues, start, end);
            plugin_adjusted = false;
            smoother_adjusted = false;
        } finally {
            readings_lock.unlock();
        }

        if ((end - start) > 80000000) {
            try {
                capturePercentage = ((bgReadings.size() * 100) / ((end - start) / 300000));
                //Log.d(TAG, "CPTIMEPERIOD: " + Long.toString(end - start) + " percentage: " + JoH.qs(capturePercentage));
            } catch (Exception e) {
                capturePercentage = -1; // invalid reading
            }
        }
        bloodtests = BloodTest.latestForGraph(numValues, start, end);
        // get extra calibrations so we can use them for historical readings
        calibrations = Calibration.latestForGraph(numValues, start - (3 * Constants.DAY_IN_MS), end);
        treatments = Treatments.latestForGraph(numValues, start, end + (120 * 60 * 1000));
        this.context = context;
        this.highMark = tolerantParseDouble(prefs.getString("highValue", "170"), 170);
        this.lowMark = tolerantParseDouble(prefs.getString("lowValue", "70"), 70);
        this.doMgdl = (prefs.getString("units", "mgdl").equals("mgdl"));
        defaultMinY = unitized(40);
        defaultMaxY = unitized(250);
        if (custimze_y_range) { // If Customize y axis range is enabled
            defaultMinY = unitized(Pref.getStringToInt("default_ymin", 40)); // Use the user-defined ymin
            defaultMaxY = unitized(Pref.getStringToInt("default_ymax", 250)); // Use the user-defined ymax
        }
        pointSize = isXLargeTablet(context) ? 5 : 3;
        axisTextSize = isXLargeTablet(context) ? 20 : Axis.DEFAULT_TEXT_SIZE_SP;
        previewAxisTextSize = isXLargeTablet(context) ? 12 : 5;
        hoursPreviewStep = isXLargeTablet(context) ? 2 : 1;
    }
~~~

### SourceWizard.java
Home 화면의 START SOURCE SETUP WIZARD 기능에 대한 코드  
Data Source를 결정하는 역할

~~~
    private Tree<Item> root = new Tree<>(new Item(gs(R.string.choose_data_source), gs(R.string.which_system_do_you_use)));

    {
        Tree<Item> g5g6 = root.addChild(new Item("G4, G5, G6, G7, 1", gs(R.string.which_type_of_device), R.drawable.g5_icon));
        {
            Tree<Item> g4 = g5g6.addChild(new Item("G4", gs(R.string.what_type_of_g4_bridge_device_do_you_use), R.drawable.g4_icon));
            {
                Tree<Item> wixel = g4.addChild(new Item(gs(R.string.bluetooth_wixel), gs(R.string.which_software_is_the_wixel_running), R.drawable.wixel_icon));
                {
                    wixel.addChild(new Item(gs(R.string.xbridge_compatible), DexCollectionType.DexbridgeWixel, R.drawable.wixel_icon));
                    wixel.addChild(new Item(gs(R.string.classic_simple), DexCollectionType.BluetoothWixel, R.drawable.wixel_icon));
                }

                g4.addChild(new Item(gs(R.string.g4_share_receiver), DexCollectionType.DexcomShare, R.drawable.g4_share_icon));
                g4.addChild(new Item(gs(R.string.parakeet_wifi), DexCollectionType.WifiWixel, R.drawable.jamorham_parakeet_marker));
            }
            g5g6.addChild(new Item("G5", DexCollectionType.DexcomG5, R.drawable.g5_icon));
            g5g6.addChild(new Item("G6, G7, 1", DexCollectionType.DexcomG6, R.drawable.g6_icon));
        }

        Tree<Item> libre = root.addChild(new Item(gs(R.string.libre), gs(R.string.what_type_of_libre_bridge_device_do_you_use), R.drawable.libre_icon_image));
        {
            libre.addChild(new Item(gs(R.string.bluetooth_bridge_device_blucon_limitter_bluereader_tomato_etc), DexCollectionType.LimiTTer, R.drawable.bluereader_icon));
            libre.addChild(new Item(gs(R.string.librealarm_app_using_sony_smartwatch), DexCollectionType.LibreAlarm, R.drawable.ic_watch_grey600_48dp));
            libre.addChild(new Item(gs(R.string.libre_patched), DexCollectionType.LibreReceiver, R.drawable.libre_icon_image));

        }
        Tree<Item> other = root.addChild(new Item(gs(R.string.other), gs(R.string.which_type_of_device), R.drawable.wikimedia_question_mark));
        {
            other.addChild(new Item("640G / 670G", DexCollectionType.NSEmulator, R.drawable.mm600_series));
            other.addChild(new Item("CareSens Air", DexCollectionType.NSEmulator, R.drawable.caresens_air_icon_image));
            other.addChild(new Item("Medtrum A6 / S7", DexCollectionType.Medtrum, R.drawable.a6_icon));
            other.addChild(new Item("Nightscout Follower", DexCollectionType.NSFollow, R.drawable.nsfollow_icon));
            other.addChild(new Item("Dex Share Follower", DexCollectionType.SHFollow, R.drawable.nsfollow_icon));
            //
            other.addChild(new Item("EverSense", DexCollectionType.NSEmulator, R.drawable.wikimedia_eversense_icon_pbroks13));
        }
    }
~~~

#### getTreeDialog() :
DexCollectionType.setDexCollectionType를 통해 수집할 대상 CGM 센서를 선택 후, DexCollectionHelper.assistance를 통해 Helper을 이용하여 해당 센서 수집함.
~~~
        // leaf node
        } else {
            final Tree<Item> fbranch = branch;
            final Item item = branch.data;
            builder.setTitle(R.string.are_you_sure);
            builder.setMessage(activity.getString(R.string.set_data_source_to, item.name));
            builder.setPositiveButton(R.string.yes, new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    DexCollectionType.setDexCollectionType(item.getCollectionType());
                    dismissDialog();
                    DexCollectionHelper.assistance(activity, item.getCollectionType());
                    sw = null; // work here is done
                }
            });
~~~

## com/eveningoutpost/dexdrip/g5model/

### Ob1G5StateMachine.java
OB1 G5 communication 로직을 다루는 코드  

#### doGetData() : 

데이터 요청
~~~
connection.writeCharacteristic(Control, nn(use_g5_internal_alg ? (getEGlucose(parent) ? new EGlucoseTxMessage(shortTxId()).byteSequence : new GlucoseTxMessage().byteSequence) : new SensorTxMessage().byteSequence))
        .subscribe(
                characteristicValue -> {
                    if (d)
                        UserError.Log.d(TAG, "Wrote SensorTxMessage request");
                }, throwable -> {
                    UserError.Log.e(TAG, "Failed to write SensorTxMessage: " + throwable);
                    if (throwable instanceof BleGattCharacteristicException) {
                        final int status = ((BleGattCharacteristicException) throwable).getStatus();
                        UserError.Log.e(TAG, "Got status message: " + getStatusName(status));
                        if (status == 8) {
                            UserError.Log.e(TAG, "Request rejected due to Insufficient Authorization failure!");
                            parent.authResult(false);
                        }
                    }
                });
~~~

데이터 수신 및 처리
~~~
flatMap(notificationObservable -> notificationObservable)
        .timeout(6, TimeUnit.SECONDS)
        .subscribe(bytes -> {
            UserError.Log.d(TAG, "Received indication bytes: " + bytesToHex(bytes));
            final PacketShop data_packet = classifyPacket(bytes);
            switch (data_packet.type) {
                ...
~~~
Dexcom G7의 경우, EGlucoseRxMessage2 유형 메시지 수신을 통해 glucose 확인  
EGlucoseRxMessage2.java 코드의 EGlucoseRxMessage2 클래스 활용  
성공적으로 glucose 수치를 얻어내면 "Status: Got glucose"의 로그 확인 가능  
glucose 데이터 수신 후 glucoseRxCommon 메서드 호출  

~~~
case EGlucoseRxMessage2:
    val eglucose2 = (EGlucoseRxMessage2) data_packet.msg;
    UserError.Log.d(TAG, "EG2 Debug: " + eglucose2);
    if (eglucose2.isValid()) {
        parent.processCalibrationState(eglucose2.calibrationState());
        DexTimeKeeper.updateAge(getTransmitterID(), (int) eglucose2.timestamp);
        DexSessionKeeper.setStart(eglucose2.getRealSessionStartTime());
        if (eglucose2.usable()) {
            parent.msg("Got glucose");
        } else {
            parent.msg("Got data");
        }
        glucoseRxCommon(eglucose2, parent, connection);
        parent.saveTransmitterMac();
    } else {
        parent.msg("Invalid Glucose");
    }
    break;
~~~

#### glucoseRxCommon() : 
1. Raw 데이터 요청  
~~~
if (JoH.ratelimit("ob1-g5-also-read-raw", 20)) {
    if (FirmwareCapability.isTransmitterRawCapable(getTransmitterID())) {
        enqueueUniqueCommand(new SensorTxMessage(), "Also read raw");
    } else {
        handleNonSensorRxState(parent, connection);
    }
}
~~~

* JoH.ratelimit("ob1-g5-also-read-raw", 20): 20초 간격으로 rate limit을 확인
* FirmwareCapability.isTransmitterRawCapable(getTransmitterID()): 현재 트랜스미터가 raw 데이터를 지원하는지 확인
    * 지원하면 SensorTxMessage를 큐에 추가하여 raw 데이터를 요청
    * 지원하지 않으면 handleNonSensorRxState 함수를 호출하여 다른 상태를 처리

2. Time 메시지 요청  
~~~
if (JoH.pratelimit("g5-tx-time-since", 7200)
        || glucose.calibrationState().warmingUp()
        || !DexSessionKeeper.isStarted()) {
    if (JoH.ratelimit("g5-tx-time-governer", 30)) {
        if (getTransmitterID().length() > 4) {
            enqueueUniqueCommand(new TimeTxMessage(), "Periodic Query Time");
        }
    }
}
~~~
* JoH.pratelimit("g5-tx-time-since", 7200): 2시간 간격으로 rate limit을 확인
* glucose.calibrationState().warmingUp(): 글루코스 데이터가 웜업 상태인지 확인
* !DexSessionKeeper.isStarted(): 세션이 시작되지 않았는지 확인
* 위 조건 중 하나라도 충족하면, 추가로 30초 rate limit을 확인
* 트랜스미터 ID 길이가 4보다 크면, TimeTxMessage를 큐에 추가하여 주기적인 시간을 요청
3. 백필(Backfill) 요청 및 데이터 처리  
~~~
if (glucose.calibrationState().readyForBackfill() && !parent.getBatteryStatusNow) {
    backFillIfNeeded(parent, connection);
}
processGlucoseRxMessage(parent, glucose);
parent.updateLast(tsl());
parent.clearErrors();
~~~
* glucose.calibrationState().readyForBackfill() && !parent.getBatteryStatusNow: 백필이 필요하고 배터리 상태를 확인 중이지 않은 경우.
    * backFillIfNeeded(parent, connection): 필요시 백필을 수행
* processGlucoseRxMessage(parent, glucose): 수신된 글루코스 메시지를 처리
* parent.updateLast(tsl()): 마지막 업데이트 시간을 갱신
* parent.clearErrors(): 에러를 초기화

#### processGlucoseRxMessage() : 
**초기 검사 및 설정**  
1. 초기 설정  
~~~
lastGlucosePacket = tsl();
DexTimeKeeper.updateAge(getTransmitterID(), glucose.timestamp);
~~~
* lastGlucosePacket을 현재 시간으로 설정합니다.
* DexTimeKeeper를 사용해 트랜스미터 ID와 함께 glucose의 타임스탬프를 업데이트합니다.

2. 센서 활성화 검사  
~~~
if (glucose.calibrationState().warmingUp()) {
    checkAndActivateSensor();
}
~~~

3. 센서 활성화 검사  
~~~
if (glucose.calibrationState().warmingUp()) {
    checkAndActivateSensor();
}
~~~
* 글루코스 데이터가 웜업 상태인지 확인하고, 웜업 중이면 센서를 활성화합니다.

**Usable 데이터 처리**  

4. Usable 데이터 확인  
~~~
if (glucose.usable() || (glucose.insufficient() && Pref.getBoolean("ob1_g5_use_insufficiently_calibrated", true))) {
    UserError.Log.d(TAG, "Got usable glucose data from transmitter!!");
    final long rxtimestamp = glucose.getRealTimestamp();
    checkAndActivateSensor();
~~~
* 글루코스 데이터가 사용 가능하거나, 불충분하지만 설정에서 불충분한 데이터 사용을 허용하면 다음을 수행합니다:
    * 로그를 기록합니다.
    * rxtimestamp에 실제 타임스탬프를 저장합니다.
    * 센서를 활성화합니다.

5. 데이터 저장 및 처리  
~~~
DexSyncKeeper.store(getTransmitterID(), rxtimestamp, parent.static_last_connected, lastGlucosePacket);
final BgReading bgReading = BgReading.bgReadingInsertFromG5(glucose.glucose, rxtimestamp);
~~~
* DexSyncKeeper에 데이터를 저장합니다.
* BgReading 객체를 생성하고, 글루코스 값을 삽입합니다.

**BgReading 객체 처리**  
6. BgReading 객체 추가 설정  
~~~
if (bgReading != null) {
    try {
        bgReading.calculated_value_slope = glucose.getTrend() / Constants.MINUTE_IN_MS; // note this is different to the typical calculated slope, (normally delta)
        if (bgReading.calculated_value_slope == Double.NaN) {
            bgReading.hide_slope = true;
        }
    } catch (Exception e) {
        // not a good number - does this exception ever actually fire?
    }
    if (!FirmwareCapability.isTransmitterRawCapable(getTransmitterID())) {
        bgReading.noRawWillBeAvailable();
    }
    if (glucose.insufficient()) {
        bgReading.appendSourceInfo("Insufficient").save();
    }
} else {
    UserError.Log.wtf(TAG, "New BgReading was null in processGlucoseRxMessage!");
}
~~~
* bgReading이 null이 아니면 다음을 수행합니다:
    * 트렌드 값을 계산하여 calculated_value_slope에 저장합니다.
    * NaN 값인지 확인하고, NaN이면 hide_slope를 true로 설정합니다.
    * 트랜스미터가 raw 데이터를 지원하지 않으면 noRawWillBeAvailable 메서드를 호출합니다.
    * 불충분한 데이터이면 appendSourceInfo("Insufficient")를 호출하고 저장합니다.
* bgReading이 null이면 에러 로그를 기록합니다.

7. 마지막 글루코스 패킷 업데이트  


## com/eveningoutpost/dexdrip/models/

### Sensor.java

### LibreBluetooth.java
LibreBluetooth 클래스 정의, Libre 블루투스 연결 코드  

### BgReading.java
#### create() : 
새로운 BgReading 객체를 생성하고, 이를 데이터베이스에 저장하는 역할. 구체적으로, create 메소드는 다음과 같은 단계를 수행함.  
- 1. 새로운 BgReading 객체 생성: 주어진 원시 데이터(raw_data), 필터링된 데이터(filtered_data), 타임스탬프(timestamp) 등을 사용하여 새로운 BgReading 객체를 생성.
- 2. 필드 설정: 생성된 객체의 필드들을 설정. 여기에는 센서 정보, 보정 정보, 나이 조정된 원시 값 계산 등이 포함됨.
- 3. 데이터베이스에 저장:
    설정된 BgReading 객체를 데이터베이스에 저장하기 위해 save() 메소드를 호출. 이 메소드는 ActiveAndroid의 Model 클래스에 정의되어 있으며, 객체의 현재 상태를 데이터베이스에 저장.


## com/eveningoutpost/dexdrip/services/
### DexCollectionService.java
LibreBluetooth 모델을 이용하여 블루투스로 Libre 데이터를 수집하는 서비스 코드  


### Ob1G5CollectionService.java
state는 현재 프로세스가 어떤 상태에 놓이는지에 대한 변수이다.  
1. SCAN
2. CONNECT_NOW
3. DISCOVER
4. CHECK_AUTH
5. GET_DATA
- Ob1G5StateMachine.java의 `doGetData` 메서드 호출
6. CLOSE



### G5CollectionService.java
com.eveningoutpost.dexdrip.g5model 내의 모델을 이용하여 블루투스로 Dexcom 데이터를 수집하는 서비스 코드

#### onStartCommand() : 
서비스 시작과 끝이 onCreate()와 onDestroy 였다면, 서비스의 활성 수명은 onStartCommand()와 onBind()에서 시작함. 블루투스 LE 기기와의 상호작용을 위해 필요한 초기화 작업, 실행 조건 확인, 블루투스 설정 등을 수행. 또한, 서비스가 중복으로 실행되지 않도록 상태를 관리하고, 필요한 경우 서비스가 다시 시작되도록 설정함. WakeLock을 통해 서비스 실행 중 기기가 절전 모드로 들어가지 않도록 보호함.
- 1. Context 및 데이터베이스 초기화
    ~~~
    Context context = getApplicationContext();
    xdrip.checkAppContext(context);
    Sensor.InitDb(context); // ensure db is initialized
    ~~~
- 2. WakeLock 설정 : WakeLock을 설정하여 서비스가 실행되는 동안 기기가 잠자기 모드로 들어가지 않도록 함.
    ~~~
    final PowerManager.WakeLock wl = JoH.getWakeLock("g5-start-service", 120000);
    ~~~
- 3. 서비스 실행 상태 확인 및 초기 설정 : 서비스가 실행 중이지 않고, keep_running 플래그가 설정되어 있는 경우 초기 설정을 수행하고 로그를 기록.
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
- 4. 서비스 실행 조건 확인 : 서비스가 실행될 조건을 확인하고, 조건이 만족되지 않으면 서비스를 중지.
    ~~~
    if (!shouldServiceRun()) {
        Log.e(TAG,"Shutting down as no longer using G5 data source");
        service_running = false;
        keep_running = false;
        stopSelf();
        return START_NOT_STICKY;
    }
    ~~~
- 5. 블루투스 초기화 및 스캔 설정 : 블루투스 매니저와 어댑터를 초기화하고, 이전에 연결된 GATT 객체가 있으면 닫음. 활성화된 센서가 있는지 확인하고, 센서가 활성화되어 있으면 블루투스를 설정.
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
- 6. 서비스 상태 갱신 및 종료 처리 : 서비스의 실행 상태를 갱신하고, START_STICKY를 반환하여 서비스가 강제 종료된 후에도 재시작되도록 설정합니다.
    ~~~
    service_running = false;
    return START_STICKY;
    ~~~
- 7. WakeLock 해제 : 마지막으로 WakeLock을 해제하여, 기기가 잠자기 모드로 들어갈 수 있도록 함.
    ~~~
    } finally {
        JoH.releaseWakeLock(wl);
    }
    ~~~

## com/eveningoutpost/dexdrip/utils/

### DexCollectionHelper.java
#### assistance() : 
- Dexcom G6, G7, 1 / G5에 해당하는 경우
~~~
// g6 is currently a pseudo type which enables required g6 settings and then sets g5
case DexcomG6:
    Ob1G5CollectionService.setG6Defaults();

    DexCollectionType.setDexCollectionType(DexCollectionType.DexcomG5);
    // intentional fall thru

case DexcomG5:
    final String pref = "dex_txid";
    textSettingDialog(activity,
            pref, activity.getString(R.string.dexcom_transmitter_id),
            activity.getString(R.string.enter_your_transmitter_id_exactly),
            InputType.TYPE_TEXT_FLAG_NO_SUGGESTIONS | InputType.TYPE_TEXT_FLAG_CAP_CHARACTERS | InputType.TYPE_TEXT_VARIATION_VISIBLE_PASSWORD,
            new Runnable() {
                @Override
                public void run() {
                    // InputType.TYPE_TEXT_FLAG_CAP_CHARACTERS does not seem functional here
                    Pref.setString(pref, Pref.getString(pref, "").toUpperCase());
                    if (!Dialog.askIfNeeded(activity, Pref.getString(pref, ""))) {
                        Home.staticRefreshBGCharts();
                    }
                    CollectionServiceStarter.restartCollectionServiceBackground();
                }
            });
    break;
~~~
- CareSens Air에 해당하는 경우
~~~
TODO : DexCollectionType.NSEmulator에 대해서는 어떻게 처리하는가? case 문에서는 확인안됨
~~~
- Libre Patched에 해당하는 경우
~~~
case LibreReceiver:
    Home.staticRefreshBGChartsOnIdle();
    break;
~~~