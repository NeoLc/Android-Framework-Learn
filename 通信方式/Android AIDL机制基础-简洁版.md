
## Android AIDL机制基础-简洁版

### 接口定义
>最简单和常见的AIDL代码示例

aidl/com/test/dcc/IDataService.aidl
```aidl
package com.test.dcc;
import com.test.dcc.IDataCallback;

interface IDataService {
    void registerCallback(int type, int id, IDataCallback callback);
    void unregisterCallback(int type, int id, IDataCallback callback);

    int publishCommand(int type, int id, in byte[] data);
}
```
aidl/com/test/dcc/IDataCallback.aidl
```aidl
package com.test.dcc;

interface IDataCallback {
    void onEvent(int type, int id, in byte[] data);
}
```

### 接口编译
编译方式多种多样, 只要能把AIDL接口**添加到android编译系统**即可
+ 方式1: 单独的编译出c++或java的后端
```mk
aidl_interface {
    name: "dataservice_aidl",
    unstable: true,
    local_include_dir: "aidl",
    srcs: [
        "aidl/com/test/dcc/IDataCallback.aidl",
        "aidl/com/test/dcc/IDataService.aidl",
    ],
    backend: {
        java: {
            enabled: true,
        },
        cpp: {
            enabled: true,
        },
    },
}
```

+ 方式2: 放置在系统框架(`frameworks/base/core/java/android`)中编译, 更加方便调用, 适用于系统接口.
```mk
java_defaults {
	name: "framework-defaults",
	...
	srcs: [
		...
		"core/java/android/dcc/IDataCallback.aidl",
		"core/java/android/dcc/IDataService.aidl",
		...
	]
}
```

+ 方式3: 直接在调用侧和java一起混编
```mk
java_library {
    name: "DataServiceSDK",
    installable: true,
    srcs: [
        "src/**/*.java",
        "src/**/*.aidl",
    ],
    platform_apis: true,
}
```

### 接口实现
>接口的实现可以是java或C++, 并将实现对象注册到`SystemManager`中方便调用.

+ C++实现
需继承Bnxxx, 并实现相关的接口.
```c++
#include <com/test/dcc/IDataServiceCallback.h>
#include <com/test/dcc/BnDataService.h>

class DataService : public BnDataService {
    void registerCallback(int type, int id, IDataServiceCallback callback) { ... }
    void unregisterCallback(int type, int id, IDataServiceCallback callback) { ... }

    int publishCommand(int type, int id, char data[]) { ... }
};
```

将实现类注册到`systemManager`中
```c++

#define NATIVESERVICE_NAME "com.test.dcc.IDataService"

int main(int argc, char **argv) {
    signal(SIGPIPE, SIG_IGN);

    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm = defaultServiceManager();

    sp<DataService> mService = new DataService();
    auto status = sm->addService(String16(NATIVESERVICE_NAME), mService, false);

    ProcessState::self()->startThreadPool();
    IPCThreadState::self()->joinThreadPool();
    return 0;
}
```

+ java实现
需继承`Stub`, , 并实现相关的接口.
```java
import com.test.dcc.IDataService.Stub;

public final class DataServiceImpl extends Stub {
    public void registerCallback(int type, int id, IDataServiceCallback callback) { ... }
    public void unregisterCallback(int type, int id, IDataServiceCallback callback) { ... }

    public int publishCommand(int type, int id, in byte[] data) { ... }
}
```
将实现类注册到`systemManager`中
```java
public final class DataService extends SystemService {
	private final DataServiceImpl mStub;
	
	public DataService(Context context) {
            mStub = new DataServiceImpl(mContext);
	}
	
	@Override
	public void onBootPhase(int phase) { ... }
	
	@Override
	public void onStart() {
            publishBinderService("service.name", mStub);
	}
}
```

### 接口使用
>`AIDL`个人理解本质是一个`RPC`机制, 是Binder接口调用的一种封装, 更加便于使用.

+ java调用AIDL接口
过程很简单, 就是通过`ServiceManager`获取服务对象即可.
```java
import com.test.dcc.IDataServiceCallback;
import com.test.dcc.IDataService;

public class DataServiceManager {
	private IDataService mService;
	
	public DataServiceManager() {
            getRemoteService();
	}
	
	private void getRemoteService() {
	    IBinder binder = null;
	    while(true) {
		binder = ServiceManager.getService(SERVICE_NAME);
	            if (binder != null) {
		    mService = ILumiDataService.Stub.asInterface(binder);
		    binder.linkToDeath(mDeathRecipient, 0);
	    	}
	    }
	}
	
	public void registerCallback(int type,int id, IDataCallback callback) {
	    mService.registerCallback(type, id, callback);
	}
	public void unregisterCallback(int type,int id, IDataCallback callback) {
            mService.unregisterCallback(type, id, callback);
	}
	
	public int publishCommand(int type,int id,byte[] data) {
	    mService.publishCommand(type, id, data);
	}
}
```
编译过程也简单, 链接aidl的android库
```mk
java_library {
    name: "DataServiceManager",
    srcs: [
        "src/**/*.java",
    ],
    static_libs: [
        "dataservice_aidl-java",
    ],
}
```
