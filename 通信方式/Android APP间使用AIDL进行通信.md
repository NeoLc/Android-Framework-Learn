## Android APP间使用AIDL进行通信
PS: 待进行测试，仅供参考。

### APP-S定义AIDL接口
> AS中: 右键NEW -> AIDL -> AIDL File
> 定义完成后, build生成中间文件.
```aidl
package com.example.aidl_test_server.service;

// Declare any non-default types here with import statements

interface INeoService {
    String getSomeDrug(int type);
}
```

### APP-S实现AIDL接口
```java
package com.example.aidl_test_server.service;

public class NeoService  extends Service {

    private final static String TAG = "NeoService";

    private final INeoService.Stub binder = new INeoService.Stub() {
        @Override
        public String getSomeDrug(int type) {
            Log.d(TAG, "getSomeDrug..");
            return "wow~";
        }
    };

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```

### APP-S发布AIDL接口
>`android:exported`表示`暴露`接口给其他app
```xml
    <application
        <service android:name=".service.NeoService"
            android:exported="true">
            <intent-filter>
                <action android:name="com.example.aidl_test_server.service.NeoService" />
            </intent-filter>
        </service>
    </application>
```

### APP-C调用AIDL接口
>引用的方法可以是导入APP-S对应AIDL接口编译出的`JAR包`, 或者直接把生成的`INeoService.java`拷贝过来使用
```java
protected void onCreate(Bundle savedInstanceState) {
	super.onCreate(savedInstanceState);
	setContentView(R.layout.activity_main);
	aidl_test();
}

private ServiceConnection mServiceConnection = new ServiceConnection() {
	@Override
	public void onServiceConnected(ComponentName name, IBinder service) {
		mNeoService = INeoService.Stub.asInterface(service);
		mIsBound = true;
		try {
			String result = mNeoService.getSomeDrug(0);
			Log.d(TAG, "Result from AIDL: " + result);
		} catch (RemoteException e) {
			e.printStackTrace();
		}
	}

	@Override
	public void onServiceDisconnected(ComponentName name) {
		mIsBound = false;
		mNeoService = null;
	}
};

private void aidl_test() {
	Log.d(TAG, "aidl_test..");
	Intent intent = new Intent();
	intent.setAction("com.example.aidl_test_server.service.NeoService");
	intent.setPackage("com.example.aidl_test_server"); // 服务端应用包名
	bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
}

@Override
protected void onDestroy() {
	super.onDestroy();
	if (mIsBound) {
		unbindService(mServiceConnection);
		mIsBound = false;
	}
}
```
