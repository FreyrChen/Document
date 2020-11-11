### 1. 默认时区修改
* device/rockchip/rk3399/rk3399_all/system.prop
* 这是是修改时区配置的
```
persist.sys.timezone=Asia/Shanghai
```

* frameworks/base/packages/SettingsProvider/res/values/defaults.xml
* 这个是通过网络自动获取时区的配置
```
<bool name="def_auto_time_zone">false</bool>
```

### 2. 网络时间获取异常
* 系统上电之后出现了无法同步网络时间的情况。
* 参考链接; https://blog.csdn.net/weixin_39966398/article/details/88904344
* 修改的文件
* frameworks/base/core/java/android/util/NtpTrustedTime.java
```
    // 修改为可修改的 mServer
    private String mServer;
    private final long mTimeout;

    private ConnectivityManager mCM;

    private boolean mHasCache;
    private long mCachedNtpTime;
    private long mCachedNtpElapsedRealtime;
    private long mCachedNtpCertainty;

    // 添加网络时间服务器网址数组
    String[] backupNtpServers = new String[]{
        "tw.pool.ntp.org",
        "time.nist.gov",
        "time-a.nist.gov"
    };
    int index = -1;
    
    // ... ...
            if (LOGD) Log.d(TAG, "forceRefresh() from cache miss");
        final SntpClient client = new SntpClient();

        // 去掉了之前的时间同步
        // 添加新的时间同步操作
        boolean result = false;
        while(!(result = client.requestTime(mServer, (int)mTimeout))
                && index < (backupNtpServers.length-1)) {
            index++;
            mServer = backupNtpServers[index];
        }

        index = -1;
        Resources res = sContext.getResources();
        String defaultServer = res.getString(
                com.android.internal.R.string.config_ntpServer);
        String secureServer = Settings.Global.getString(
                sContext.getContentResolver(), Settings.Global.NTP_SERVER);

        mServer  = secureServer != null ? secureServer : defaultServer;

        if(result) {
            mHasCache = true;
            mCachedNtpTime = client.getNtpTime();
            mCachedNtpElapsedRealtime = client.getNtpTimeReference();
            mCachedNtpCertainty = client.getRoundTripTime() / 2;
        }

        return result;
```

* git diff NtpTrustedTime.java
```
diff --git a/core/java/android/util/NtpTrustedTime.java b/core/java/android/util/NtpTrustedTime.java
index ed2d3c6..f114074 100644
--- a/core/java/android/util/NtpTrustedTime.java
+++ b/core/java/android/util/NtpTrustedTime.java
@@ -39,7 +39,7 @@ public class NtpTrustedTime implements TrustedTime {
     private static NtpTrustedTime sSingleton;
     private static Context sContext;

-    private final String mServer;
+    private String mServer;
     private final long mTimeout;

     private ConnectivityManager mCM;
@@ -49,6 +49,13 @@ public class NtpTrustedTime implements TrustedTime {
     private long mCachedNtpElapsedRealtime;
     private long mCachedNtpCertainty;

+    String[] backupNtpServers = new String[]{
+        "tw.pool.ntp.org",
+        "time.nist.gov",
+        "time-a.nist.gov"
+    };
+    int index = -1;
+
     private NtpTrustedTime(String server, long timeout) {
         if (LOGD) Log.d(TAG, "creating NtpTrustedTime using " + server);
         mServer = server;
@@ -101,15 +108,31 @@ public class NtpTrustedTime implements TrustedTime {

         if (LOGD) Log.d(TAG, "forceRefresh() from cache miss");
         final SntpClient client = new SntpClient();
-        if (client.requestTime(mServer, (int) mTimeout)) {
+
+        boolean result = false;
+        while(!(result = client.requestTime(mServer, (int)mTimeout))
+                && index < (backupNtpServers.length-1)) {
+            index++;
+            mServer = backupNtpServers[index];
+        }
+
+        index = -1;
+        Resources res = sContext.getResources();
+        String defaultServer = res.getString(
+                com.android.internal.R.string.config_ntpServer);
+        String secureServer = Settings.Global.getString(
+                sContext.getContentResolver(), Settings.Global.NTP_SERVER);
+
+        mServer  = secureServer != null ? secureServer : defaultServer;
+
+        if(result) {
             mHasCache = true;
             mCachedNtpTime = client.getNtpTime();
             mCachedNtpElapsedRealtime = client.getNtpTimeReference();
             mCachedNtpCertainty = client.getRoundTripTime() / 2;
-            return true;
-        } else {
-            return false;
         }
+
+        return result;
     }

     @Override
```