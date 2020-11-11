### 1. 找到 Settings 里面 Build number 定义的 xml 文件
* `packages/apps/Settings/res/values/strings.xml`
```
<!-- About phone screen,  setting option name  [CHAR LIMIT=40] -->
<string name="build_number">Build number</string>
```
* `packages/apps/Settings/res/xml/device_info_settings.xml`
```
        <!-- Device Kernel version -->
        <com.android.settings.DividerPreference
                android:key="kernel_version"
                android:enabled="false"
                android:shouldDisableView="false"
                android:selectable="false"
                android:title="@string/kernel_version"
                android:summary="@string/device_info_default"
                settings:allowDividerAbove="true"
                settings:allowDividerBelow="true"/>

        <!-- Detailed build version -->
        <Preference android:key="build_number"
                android:enabled="false"
                android:shouldDisableView="false"
                android:title="@string/build_number"
                android:summary="@string/device_info_default"/>
```

### 2. java 里面的调用在下面这个位置
* `packages/apps/Settings/src/com/android/settings/DeviceInfoSettings.java`
```
import android.os.Build;

public class DeviceInfoSettings extends SettingsPreferenceFragment implements Indexable {
// ... ...
    private static final String LOG_TAG = "DeviceInfoSettings";

    private static final String KEY_MANUAL = "manual";
    private static final String KEY_REGULATORY_INFO = "regulatory_info";
    private static final String KEY_SYSTEM_UPDATE_SETTINGS = "system_update_settings";
    private static final String PROPERTY_URL_SAFETYLEGAL = "ro.url.safetylegal";    private static final String PROPERTY_SELINUX_STATUS = "ro.build.selinux";
    private static final String KEY_KERNEL_VERSION = "kernel_version";
    private static final String KEY_BUILD_NUMBER = "build_number";
    private static final String KEY_DEVICE_MODEL = "device_model";
    private static final String KEY_SELINUX_STATUS = "selinux_status";
// ... ...
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        // .. ...
        // 这里的这个 Build 就是 frameworks/base/core/java/android/os/Build.java 里面的 Build
        // 所以 Build.DISPLAY == getString("ro.build.display.id")
        setStringSummary(KEY_BUILD_NUMBER, Build.DISPLAY);
        findPreference(KEY_BUILD_NUMBER).setEnabled(true);
        // .. ...
    }
    // ... ...
    private void setStringSummary(String preference, String value) {
        try {
            findPreference(preference).setSummary(value);
        } catch (RuntimeException e) {
            findPreference(preference).setSummary(
                getResources().getString(R.string.device_info_default));
        }
    }

// ... ... 
}
```
* `frameworks/base/core/java/android/os/Build.java`
```
public class Build {
    // ... ...
        /** Either a changelist number, or a label like "M4-rc20". */
    public static final String ID = getString("ro.build.id");

    /** A build ID string meant for displaying to the user */
    public static final String DISPLAY = getString("ro.build.display.id");
    // ... ...
    }
```

### 3. 编译脚本里面生成的值
* `core/version_defaults.mk`
```
ifeq "" "$(BUILD_NUMBER)"
  BUILD_NUMBER := eng.$(USER).$(shell date +%Y%m%d.%H%M%S)
ifeq ($(TARGET_BUILD_VARIANT),user)
  BUILD_NUMBER := user.$(USER).$(shell date +%Y%m%d.%H%M%S)
endif
```
* `build/core/main.mk`
```
$(shell mkdir -p $(OUT_DIR) && \
    echo -n $(BUILD_NUMBER) > $(OUT_DIR)/build_number.txt && \
    echo -n $(BUILD_DATETIME) > $(OUT_DIR)/build_date.txt)
BUILD_NUMBER_FROM_FILE := $$(cat $(OUT_DIR)/build_number.txt)
BUILD_DATETIME_FROM_FILE := $$(cat $(OUT_DIR)/build_date.txt)
```

* `build/core/Makefile`
```
ifeq ($(TARGET_BUILD_VARIANT),user)
  # User builds should show:
  # release build number or branch.buld_number non-release builds

  # Dev. branches should have DISPLAY_BUILD_NUMBER set
  ifeq "true" "$(DISPLAY_BUILD_NUMBER)"
    BUILD_DISPLAY_ID := $(BUILD_ID).$(BUILD_NUMBER_FROM_FILE) $(BUILD_KEYS)
  else
    BUILD_DISPLAY_ID := $(BUILD_ID) $(BUILD_KEYS)
  endif
else
  # Non-user builds should show detailed build information
  BUILD_DISPLAY_ID := $(build_desc)
endif
```
* `tools/buildinfo.sh`
```
echo "ro.build.id=$BUILD_ID"
echo "ro.build.display.id=$BUILD_DISPLAY_ID"
echo "ro.build.version.incremental=$BUILD_NUMBER"
```

### 4. 增加自己个性化的 Version
* `packages/apps/Settings/res/xml/device_info_settings.xml`
```
        <!-- Detailed build version -->
        <Preference android:key="build_number"
                android:enabled="false"
                android:shouldDisableView="false"
                android:title="@string/build_number"
                android:summary="@string/device_info_default"/>

        <!-- Detailed Aplex version -->
        <Preference android:key="aplex_version"
                android:enabled="false"
                android:shouldDisableView="false"
                android:title="@string/aplex_version"
                android:summary="@string/device_info_default"/>
```
* `packages/apps/Settings/res/values/strings.xml`
```
    <string name="build_number">Build number</string>
    <string name="aplex_version">Aplex Version</string>
```
* `packages/apps/Settings/src/com/android/settings/DeviceInfoSettings.java`
```
public class DeviceInfoSettings extends SettingsPreferenceFragment implements Indexable {
    // ... ...
    private static final String KEY_APLEX_VERSION = "aplex_version";
    // ... ...
        public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        // ... ...
        setStringSummary(KEY_BUILD_NUMBER, Build.DISPLAY);
        findPreference(KEY_BUILD_NUMBER).setEnabled(true);

        setStringSummary(KEY_APLEX_VERSION, Build.APLEX_VERSION);
        // ... ...
        }
    // ... ...
    }
```
* `frameworks/base/core/java/android/os/Build.java`
```
public class Build {
    // ... ...
    public static final String DISPLAY = getString("ro.build.display.id");

    public static final String APLEX_VERSION = getString("ro.aplex.version");
    // ... ...
    }
```
* `build/tools/buildinfo.sh`
```
echo "ro.build.display.id=$BUILD_DISPLAY_ID"
echo "ro.aplex.version=CMIAR157R100_A7.1200923_Release"
```
* 修改完上面的信息之后，需要执行 `make update-api` 更新应用层的信息，不然会编译不通过。