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

#### setupCharts() : 
차트 셋업 코드  
아래처럼 setLineChartData 메서드를 통해 시각화할 데이터를 입력
~~~
chart.setLineChartData(bgGraphBuilder.lineData());
~~~
utilityModels/BgGraphBuilder.java에 정의된 BgGraphBuilder는 수집되는 데이터를 차트에 반영할 수 있도록 함.  

## com/eveningoutpost/dexdrip/utilitymodels
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