### 1. 准备一个关机图标的资源
* 放在Android7.1 源码的如下位置：
* 下面的图片放在了 Resource 文件夹里面
* `frameworks/base/packages/SystemUI/res/drawable-nodpi/ic_sysbar_power.png`
* `frameworks/base/packages/SystemUI/res/layout/power.xml`
* 增加按键布局
```
<com.android.systemui.statusbar.policy.KeyButtonView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:systemui="http://schemas.android.com/apk/res-auto"
    android:id="@+id/power"
    android:layout_width="@dimen/navigation_key_width"
    android:layout_height="match_parent"
    android:layout_weight="0"
    android:src="@drawable/ic_sysbar_power"
    systemui:keyCode="0"
    android:scaleType="center"
    android:contentDescription="@string/accessibility_power"
    android:paddingStart="@dimen/navigation_key_padding"
    android:paddingEnd="@dimen/navigation_key_padding"
    />
```
* `frameworks/base/packages/SystemUI/res/values/strings.xml`
* 注册这个power资源可访问
```
<string name="accessibility_power" translatable="false">Power</string>
```

### 2. 设置 power 按键可见
* `frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarInflaterView.java`
```
    public static final String VOLUME_SUB = "volume_sub";

    public static final String POWER = "power";

    public static final String GRAVITY_SEPARATOR = ";";

    // ... ...
    protected View inflateButton(String buttonSpec, ViewGroup parent, boolean landscape) {
    // .. ...
        } else if (VOLUME_SUB.equals(button)) {
            v = inflater.inflate(R.layout.volume_sub, parent, false);
            if (landscape && isSw600Dp()) {
                setupLandButton(v);
            }
        } else if (POWER.equals(button)) {
            v = inflater.inflate(R.layout.power, parent, false);
            if (landscape && isSw600Dp()) {
                setupLandButton(v);
            }
    // ... ...
    }
```
* 设置 power的顺序
* `frameworks/base/packages/SystemUI/res/values-sw600dp/config.xml`
```
    <string name="config_navBarLayout" translatable="false">space;volume_sub,back,home,recent,volume_add,screenshot,power;menu_ime</string>
```
* `frameworks/base/packages/SystemUI/res/values-sw900dp/config.xml`
```
    <string name="config_navBarLayout" translatable="false">space;volume_sub,back,home,recent,volume_add,screenshot,power;menu_ime</string>
```
* `frameworks/base/packages/SystemUI/res/values/config.xml`
```
    <string name="config_navBarLayout" translatable="false">space;volume_sub,back,home,recent,volume_add,screenshot,power;menu_ime</string>
```
* 设置 power 按键状态, 设置按键触发 onclick：
* `frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/PhoneStatusBar.java`
```
    private void awakenDreams() {
        if (mDreamManager != null) {
            try {
                mDreamManager.awaken();
            } catch (RemoteException e) {
                // fine, stay asleep then
            }
        }
    }

    // Add by chen
    // 触发函数
    private View.OnClickListener mPowerClickListener = new View.OnClickListener(){
        public void onClick(View v){
            Intent intent = new Intent("android.intent.action.POWER_MENU");
            mContext.sendBroadcast(intent);
        }
    };

    boolean isShow=Settings.System.getInt(mContext.getContentResolver(), Settings.System.SCREENSHOT_BUTTON_SHOW, 1)==1;
    if(isShow){
        screenshotButton.setVisibility(View.VISIBLE);
    }else{
        screenshotButton.setVisibility(View.GONE);
    }

    // Add by Frey_chen 20200918
    // Add power button function ---
    ButtonDispatcher powerButton = mNavigationBarView.getPowerButton();
    powerButton.setOnClickListener(mPowerClickListener);
    //powerButton.setOnTouchListener(mPowerTouchListener);
    powerButton.setVisibility(View.VISIBLE);
    // ---

    ButtonDispatcher volumeAddButton=mNavigationBarView.getVolumeAddButton();
    ButtonDispatcher volumeSubButton=mNavigationBarView.getVolumeSubButton();
```

### 3. 设置广播监听：
* `frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/NavigationBarView.java`
```
    public NavigationBarView(Context context, AttributeSet attrs) {
        super(context, attrs);
        // ... ... 
        mButtonDisatchers.put(R.id.power, new ButtonDispatcher(R.id.power));
        mButtonDisatchers.put(R.id.volume_add, new ButtonDispatcher(R.id.volume_add));
        mButtonDisatchers.put(R.id.volume_sub, new ButtonDispatcher(R.id.volume_sub));
    }

    public ButtonDispatcher getPowerButton() {
        return mButtonDisatchers.get(R.id.power);
    }
```
* 广播监听，当监听到 onclick ， 执行下面的功能。
* `frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java`
```
        // register for multiuser-relevant broadcasts
        filter = new IntentFilter(Intent.ACTION_USER_SWITCHED);
        context.registerReceiver(mMultiuserReceiver, filter);

        // Add by Frey_chen
        // Add monitor for POWER_MENU
        IntentFilter ifPower = new IntentFilter("android.intent.action.POWER_MENU");
        context.registerReceiver(new BroadcastReceiver(){
            @Override
            public void onReceive(Context context, Intent intent) {
                //show global actions dialog
                showGlobalActionsInternal();
            }
        }, ifPower);
```