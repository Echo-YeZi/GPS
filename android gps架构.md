## 1. android gps架构

### 1.1android gps实现方案

#### 1.1.1流程

![img](https://img-blog.csdn.net/20170521153133588?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQzOTQxNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

#### 1.1.2高通定位方案架构

![img](https://img-blog.csdn.net/20170521153230312?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQzOTQxNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

> `GPS Application`(各种GPS定位的apk)都通过android系统的LocationManager对GPS进行打开/关闭/启动等操作
>
> 然后等待数据的上报。所以架构中有2个流程,由上往下的控制流,由下往上的数据流。



> 1. `GPS Application`和`LocationManagerService`所在进程通过Binder机制进行跨进程调用。
>
> 2. `GpsLocationProvider`和`com_android_server_location_GpsLocationProvide`以及`com_android_server_location_GpsLocationProvide`和`gps.so`库相互之间都是通过==数据结构==进行回调。
>
>    > 其实, `com_android_server_location_GpsLocationProvide`只是==Framework和HAL之间的一个桥梁==。
>
>    HAL的gps.so库和Modem通过高通的QMI机制进行数据传输,并且数据的格式是==NMEA==类型



### 1.2GPS数据结构

> `\hardware\libhardware\include\hardware\gps.h`，定义了GPS底层相关的结构体和接口



**GpsLocation 包含经纬度海拔,速度,定位的精度等卫星信息**

```c++
typedef struct {
    size_t          size;
    uint16_t        flags;
    double          latitude;
    double          longitude;
    double          altitude;
    float           speed;
    float           bearing;
    float           accuracy;
    GpsUtcTime    timestamp;
} GpsLocation;
```



**GpsStatus：GPS的状态**

```c++
typedef struct {
    size_t          size;
    GpsStatusValue status;
} GpsStatus;
```

>  status有未知、正在定位、停止定位、启动未定义、未启动五种状态
>
> typedef uint16_t GpsStatusValue;
> #define GPS_STATUS_NONE             0
> #define GPS_STATUS_SESSION_BEGIN    1
> #define GPS_STATUS_SESSION_END      2
> #define GPS_STATUS_ENGINE_ON        3
> #define GPS_STATUS_ENGINE_OFF       4


**GpsSvInfo 包含卫星ID、信噪比等信息,定义如下,**

```c++
typedef struct {
    size_t          size;
    int     prn;
    float   snr;
    float   elevation;
    float   azimuth;
} GpsSvInfo;
```



**GpsSvStatus 包含可视卫星数、星历时间、年历时间等信息**

```c++
typedef struct {
    size_t          size;
    int         num_svs;
    GpsSvInfo   sv_list[GPS_MAX_SVS];
    uint32_t    ephemeris_mask;
    uint32_t    almanac_mask;
    uint32_t    used_in_fix_mask;
} GpsSvStatus;
```



**GpsCallbacks对应JNI的回调方法**

```c++
typedef struct {
    size_t      size;
    gps_location_callback location_cb;
    gps_status_callback status_cb;
    gps_sv_status_callback sv_status_cb;
    gps_nmea_callback nmea_cb;
    gps_set_capabilities set_capabilities_cb;
    gps_acquire_wakelock acquire_wakelock_cb;
    gps_release_wakelock release_wakelock_cb;
    gps_create_thread create_thread_cb;
    gps_request_utc_time request_utc_time_cb;
} GpsCallbacks;
```



**GpsInterface：JNI层通过此接口与HAL进行交互**

```c++
typedef struct {
    size_t          size;
    int   (*init)( GpsCallbacks* callbacks );
    int   (*start)( void );
    int   (*stop)( void );
    void  (*cleanup)( void );
    int   (*inject_time)(GpsUtcTime time, int64_t timeReference,
                         int uncertainty);
    int  (*inject_location)(double latitude, double longitude, float accuracy);
    void  (*delete_aiding_data)(GpsAidingData flags);
    int   (*set_position_mode)(GpsPositionMode mode, GpsPositionRecurrence recurrence,
            uint32_t min_interval, uint32_t preferred_accuracy, uint32_t preferred_time);
    const void* (*get_extension)(const char* name);
} GpsInterface;
```



**gps_device_t:GPS设备结构体，继承自hw_device_t，硬件适配接口，向上层提供了get_gps_interface接口**

```c++
struct gps_device_t {
    struct hw_device_t common;
    const GpsInterface* (*get_gps_interface)(struct gps_device_t* dev);
};
```



## 2.GPS服务启动过程

### 2.1概述

> 在android系统中,GPS对应的系统服务为==LocationManagerService==

#### 2.1.1 启动流程

1. SystemServer.java的startOtherServices方法中添加LocationManagerService方法

   ```java
   location = new LocationManagerService(context);
   ServiceManager.addService(Context.LOCATION_SERVICE, location);
   ```

2. apk中获取gps服务代理

   ```java
   mLocationManager = (LocationManager)getSystemService(Context.LOCATION_SERVICE);
   ```

3. SystemServer.java的startOtherServices方法中启动服务

   ```java
   locationF.systemRunning();
   ```



### 2.2 回调

> 在调用systemRunning之前,在添加到系统过程中,会调用==LocationManagerService==的构造方法

- systemRunning方法中会调用loadProvidersLocked方法

  ```java
  loadProvidersLocked();
  updateProvidersLocked();
  ```

  - loadProvidersLocked方法主要是添加设备上支持的GPS定位Provider,核心代码如下

    ```java
    if (GpsLocationProvider.isSupported()) {
                // Create a gps location provider
                GpsLocationProvider gpsProvider = new GpsLocationProvider(mContext, this,
                        mLocationHandler.getLooper());
                mGpsStatusProvider = gpsProvider.getGpsStatusProvider();
                mNetInitiatedListener = gpsProvider.getNetInitiatedListener();
                addProviderLocked(gpsProvider);
                mRealProviders.put(LocationManager.GPS_PROVIDER, gpsProvider);
                mGpsMeasurementsProvider = gpsProvider.getGpsMeasurementsProvider();
                mGpsNavigationMessageProvider = gpsProvider.getGpsNavigationMessageProvider();
                mGpsGeofenceProxy = gpsProvider.getGpsGeofenceProxy();
            }
    ```

    > 备如果支持GpsLocationProvider,就会新建GpsLocationProvider对象,然后添加到mProviders和mProvidersByName等list中

  - GpsLocationProvider的构造方法和isSupported之前,会调用class_init_native方法

    > ```static { class_init_native(); }```
    >
    > 该方法是一个native方法,
    >
    > ```java
    > private static native void class_init_native();
    > ```

  - GpsLocationProvider.java对应的C/C++文件为```com_android_server_location_GpsLocationProvider.cpp```

    ==在该文件中设置了大量回调函数==



#### 2.2.1 Framework调用JNI层方法

**GpsLocationProvider.java中调用的native方法较多,部分如下**

![img](https://img-blog.csdn.net/20170521155345809?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQzOTQxNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

**com_android_server_location_GpsLocationProvider.cpp的sMethods中详细的列举了对应的native方法,部分如下**

```java
{"class_init_native", "()V", (void *)android_location_GpsLocationProvider_class_init_native},
{"native_is_supported", "()Z", (void*)android_location_GpsLocationProvider_is_supported},
{"native_is_agps_ril_supported", "()Z", (void*)android_location_GpsLocationProvider_is_agps_ril_supported},
•••
```



#### 2.2.2 JNI层调用Framework方法

**com_android_server_location_GpsLocationProvider还定义着对GpsLocationProvide.java的回调方法。部分定义如下,**

```java
static jmethodID method_reportLocation;
static jmethodID method_reportStatus;
static jmethodID method_reportSvStatus;
•••
```

**这些回调方法都在android_location_GpsLocationProvider_class_init_native方法中进行赋值,部分代码如下,**

```java
method_reportLocation = env->GetMethodID(clazz, "reportLocation", "(IDDDFFFJ)V");
method_reportStatus = env->GetMethodID(clazz, "reportStatus", "(I)V");
method_reportSvStatus = env->GetMethodID(clazz, "reportSvStatus", "()V");
•••
```



#### 2.2.3 JNI层调用gps库方法

**com_android_server_location_GpsLocationProvider的class_init_native方法中获取gps.so库中的sGpsInterface结构代码如下**,

```c+
hw_module_t* module;
err = hw_get_module(GPS_HARDWARE_MODULE_ID, (hw_module_t const**)&module);
    if (err == 0) {
        hw_device_t* device;
        err = module->methods->open(module, GPS_HARDWARE_MODULE_ID, &device);
        if (err == 0) {
            gps_device_t* gps_device = (gps_device_t *)device;
            sGpsInterface = gps_device->get_gps_interface(gps_device);
        }
    }
```

1. 调用hw_get_module方法加载gps相关的so库,获取hw_module_t结构体

2. 利用该结构体打开so库,获取hw_device_t结构体

3. 利用hw_device_t结构体获取GpsInterface结构体

   > **GpsInterface结构体指向```hardware\qcom\gps\loc_api\libloc_api_50001```中loc.cpp的sLocEngInterface**



**其他数据结构**

```c++
static const GpsInterface* sGpsInterface = NULL;
static const GpsXtraInterface* sGpsXtraInterface = NULL;
static const AGpsInterface* sAGpsInterface = NULL;
static const GpsNiInterface* sGpsNiInterface = NULL;
static const GpsDebugInterface* sGpsDebugInterface = NULL;
static const AGpsRilInterface* sAGpsRilInterface = NULL;
static const GpsGeofencingInterface* sGpsGeofencingInterface = NULL;
static const GpsMeasurementInterface* sGpsMeasurementInterface = NULL;
static const GpsNavigationMessageInterface* sGpsNavigationMessageInterface = NULL;
static const GnssConfigurationInterface* sGnssConfigurationInterface = NULL;
```

- 这些结构体都通过GpsInterface的get_extension方法获取,不同方法只是以字符区分

  ```java
  sGpsXtraInterface = (const GpsXtraInterface*)sGpsInterface->get_extension(GPS_XTRA_INTERFACE);
  •••
  ```

- 这些字符都定义在gps.h文件中

  ```c++
  #define GPS_XTRA_INTERFACE      "gps-xtra"
  #define GPS_DEBUG_INTERFACE      "gps-debug"
  #define AGPS_INTERFACE      "agps"
  •••
  ```

- loc.cpp的loc_get_extension方法如下

  ```c++
  const void* loc_get_extension(const char* name)
  {
      ENTRY_LOG();
      const void* ret_val = NULL;
   
     LOC_LOGD("%s:%d] For Interface = %s\n",__func__, __LINE__, name);
     if (strcmp(name, GPS_XTRA_INTERFACE) == 0)
     {
         ret_val = &sLocEngXTRAInterface;
     }
     else if (strcmp(name, AGPS_INTERFACE) == 0)
     {
         ret_val = &sLocEngAGpsInterface;
     }
  •••
  EXIT_LOG(%p, ret_val);
      return ret_val;
  }
  ```

  

#### 2.3.4 *gps库调用JNI层方法*

**com_android_server_location_GpsLocationProvider.cpp中定义了一些gps库调回调的结构体**

如sGpsCallbacks,主要是gps库中上传数据的信息,定义如下

```c++
GpsCallbacks sGpsCallbacks = {
    sizeof(GpsCallbacks),
    location_callback,
    status_callback,
    sv_status_callback,
    nmea_callback,
    set_capabilities_callback,
    acquire_wakelock_callback,
    release_wakelock_callback,
    create_thread_callback,
    request_utc_time_callback,
};
```



## 3.GPS的打开/初始化

> 在Java层打开gps,其实对于gps库来说,就是执行初始化过程

### 3.1 Java层分析

- android系统中打开GPS的方法往数据库里面写值

  ```java
  private void enableGps(boolean enable) {
  try {
  	Settings.Secure.setLocationProviderEnabled(getContentResolver(),
  					LocationManager.GPS_PROVIDER, enable);
  		} catch (Exception e) {
  		}
  	}
  ```

- 往setting数据库中的location_providers_allowed字段写值

  ```java
  putStringForUser(cr, Settings.Secure.LOCATION_PROVIDERS_ALLOWED, provider, userId);
  ```

- LocationManagerService的systemRunning方法中会监听setting数据库中的location_providers_allowed字段值的变化

  ```java
  mContext.getContentResolver().registerContentObserver(
      Settings.Secure.getUriFor(Settings.Secure.LOCATION_PROVIDERS_ALLOWED), true,
                  new ContentObserver(mLocationHandler) {
                      @Override
                      public void onChange(boolean selfChange) {
                          synchronized (mLock) {
                              updateProvidersLocked();
                          }
                      }
                  }, UserHandle.USER_ALL);
  ```

  如果该值有变化,则调用updateProvidersLocked方法。主要的流程图如下

  ![img](https://img-blog.csdn.net/20170521191506403?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQzOTQxNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

  - updateProvidersLocked方法中,如果支持gps,调用updateProviderListenersLocked方法,

    ```html
    updateProviderListenersLocked(name, true);
    ```

  - updateProviderListenersLocked主要方法如下,

    ```java
    •••••
    boolean shouldBeEnabled = isAllowedByCurrentUserSettingsLocked(name);
    ••••
    if (enabled) {
                p.enable();
                if (listeners > 0) {
                    applyRequirementsLocked(provider);
                }
            } else {
                p.disable();
            }
    ```

  - 调用isAllowedByCurrentUserSettingsLocked方法读取数据库字段值,然后根据该值进行相应的打开/关闭GPS操作

    ```java
    private boolean isAllowedByCurrentUserSettingsLocked(String provider) {
            if (mEnabledProviders.contains(provider)) {
                return true;
            }
            if (mDisabledProviders.contains(provider)) {
                return false;
            }
            // Use system settings
            ContentResolver resolver = mContext.getContentResolver();
     
            return Settings.Secure.isLocationProviderEnabledForUser(resolver, provider, mCurrentUserId);
        }
    ```

  - Settings中的isLocationProviderEnabledForUser方法如下

    ```java
    public static final boolean isLocationProviderEnabledForUser(ContentResolver cr, String provider, int userId) {
                String allowedProviders = Settings.Secure.getStringForUser(cr,
                        LOCATION_PROVIDERS_ALLOWED, userId);
                return TextUtils.delimitedStringContains(allowedProviders, ',', provider);
            }
    ```

  - GpsLocationProvider的enable方法如下,

    ```java
    @Override
        public void enable() {
            synchronized (mLock) {
                if (mEnabled) return;
                mEnabled = true;
            }
     
            sendMessage(ENABLE, 1, null);
        }
    ```

  - ProviderHandler对ENABLE消息处理如下

    ```java
    case ENABLE:
          if (msg.arg1 == 1) {
              handleEnable();
           } else {
              handleDisable();
          }
         break;
    ```

  - handleEnable方法如下

    ```java
    private void handleEnable() {
            if (DEBUG) Log.d(TAG, "handleEnable");
            boolean enabled = native_init(); //初始化GPS HAL层模块
            if (enabled) {
                mSupportsXtra = native_supports_xtra();//判断GPS模块是否支持Xtra
                // TODO: remove the following native calls if we can make sure they are redundant.
                if (mSuplServerHost != null) {// 设置SUPL服务器地址和端口
                    native_set_agps_server(AGPS_TYPE_SUPL, mSuplServerHost, mSuplServerPort);
                }
                if (mC2KServerHost != null) { //设置CDMA2000
                    native_set_agps_server(AGPS_TYPE_C2K, mC2KServerHost, mC2KServerPort);
                }
                mGpsMeasurementsProvider.onGpsEnabledChanged();
                mGpsNavigationMessageProvider.onGpsEnabledChanged();
            } else {
                synchronized (mLock) {
                    mEnabled = false;
                }
                Log.w(TAG, "Failed to enable location provider");
            }
        }
    ```



### 3.2 HAL初始化

- com_android_server_location_GpsLocationProvider的android_location_GpsLocationProvider_init方法如下

  ```c++
  static jboolean android_location_GpsLocationProvider_init(JNIEnv* env, jobject obj)
  {
      // this must be set before calling into the HAL library
      if (!mCallbacksObj)
          mCallbacksObj = env->NewGlobalRef(obj);
   
      // fail if the main interface fails to initialize
      if (!sGpsInterface || sGpsInterface->init(&sGpsCallbacks) != 0)
          return JNI_FALSE;
   
      // if XTRA initialization fails we will disable it by sGpsXtraInterface to NULL,
      // but continue to allow the rest of the GPS interface to work.
      if (sGpsXtraInterface && sGpsXtraInterface->init(&sGpsXtraCallbacks) != 0)
          sGpsXtraInterface = NULL;
      if (sAGpsInterface)
          sAGpsInterface->init(&sAGpsCallbacks);
      if (sGpsNiInterface)
          sGpsNiInterface->init(&sGpsNiCallbacks);
      if (sAGpsRilInterface)
          sAGpsRilInterface->init(&sAGpsRilCallbacks);
      if (sGpsGeofencingInterface)
          sGpsGeofencingInterface->init(&sGpsGeofenceCallbacks);
   
      return JNI_TRUE;
  }
  ```

- sGpsCallbacks的定义如下

  ```c++
  GpsCallbacks sGpsCallbacks = {
      sizeof(GpsCallbacks),
      location_callback,
      status_callback,
      sv_status_callback,
      nmea_callback,
      set_capabilities_callback,
      acquire_wakelock_callback,
      release_wakelock_callback,
      create_thread_callback,
      request_utc_time_callback,
  };
  ```

  

### 3.3.小结

`com_android_server_location_GpsLocationProvider`只是Java和HAL库之间的一个==纽带==。

1. Java中加载HAL库,并且对HAL库进行初始化。然后就可以对HAL的GPS发出指令,例如打开,关闭GPS等。

   > 这时数据流的方向为
   >
   > Java-->JNI--> GPS库

2. GPS库中执行打开/关闭动作,向Java上报卫星以及定位消息。

   >  这时数据流的方向为
   >
   > GPS库-->JNI--> Java



**GPS库中对应的init方法为, loc.cpp 的loc_init方法**

```c++
static int loc_init(GpsCallbacks* callbacks)
{
    int retVal = -1;
    ENTRY_LOG();
    LOC_API_ADAPTER_EVENT_MASK_T event;
    if (NULL == callbacks) {
        LOC_LOGE("loc_init failed. cb = NULL\n");
        EXIT_LOG(%d, retVal);
        return retVal;
    }
 
    event = LOC_API_ADAPTER_BIT_PARSED_POSITION_REPORT |
            LOC_API_ADAPTER_BIT_SATELLITE_REPORT |
            LOC_API_ADAPTER_BIT_LOCATION_SERVER_REQUEST |
            LOC_API_ADAPTER_BIT_ASSISTANCE_DATA_REQUEST |
            LOC_API_ADAPTER_BIT_IOCTL_REPORT |
            LOC_API_ADAPTER_BIT_STATUS_REPORT |
            LOC_API_ADAPTER_BIT_NMEA_1HZ_REPORT |
            LOC_API_ADAPTER_BIT_NI_NOTIFY_VERIFY_REQUEST;
            // 继续注册回调
    LocCallbacks clientCallbacks = {local_loc_cb, /* location_cb */
                                    callbacks->status_cb, /* status_cb */
                                    local_sv_cb, /* sv_status_cb */
                                    callbacks->nmea_cb, /* nmea_cb */
                                    callbacks->set_capabilities_cb, /* set_capabilities_cb */
                                    callbacks->acquire_wakelock_cb, /* acquire_wakelock_cb */
                                    callbacks->release_wakelock_cb, /* release_wakelock_cb */
                                    callbacks->create_thread_cb, /* create_thread_cb */
                                    NULL, /* location_ext_parser */
                                    NULL, /* sv_ext_parser */
                                    callbacks->request_utc_time_cb, /* request_utc_time_cb */
                                    };
 
    gps_loc_cb = callbacks->location_cb;
    gps_sv_cb = callbacks->sv_status_cb;
          // loc_eng_init以及里面的方法较复杂,暂不论述。
    retVal = loc_eng_init(loc_afw_data, &clientCallbacks, event, NULL);
    loc_afw_data.adapter->mSupportsAgpsRequests = !loc_afw_data.adapter->hasAgpsExtendedCapabilities();
    loc_afw_data.adapter->mSupportsPositionInjection = !loc_afw_data.adapter->hasCPIExtendedCapabilities();
    loc_afw_data.adapter->mSupportsTimeInjection = !loc_afw_data.adapter->hasCPIExtendedCapabilities();
    loc_afw_data.adapter->setGpsLockMsg(0);
    loc_afw_data.adapter->requestUlp(getCarrierCapabilities());
    loc_afw_data.adapter->setXtraUserAgent();
 
    if(retVal) {
        LOC_LOGE("loc_eng_init() fail!");
        goto err;
    }
 
    loc_afw_data.adapter->setPowerVoteRight(loc_get_target() == TARGET_QCA1530);
    loc_afw_data.adapter->setPowerVote(true);
 
    LOC_LOGD("loc_eng_init() success!");
 
err:
    EXIT_LOG(%d, retVal);
    return retVal;
}
```



**gps.h中有对应的GpsCallbacks结构,**

```c++
typedef struct {
    /** set to sizeof(GpsCallbacks) */
    size_t      size;
    gps_location_callback location_cb;
    gps_status_callback status_cb;
    gps_sv_status_callback sv_status_cb;
    gps_nmea_callback nmea_cb;
    gps_set_capabilities set_capabilities_cb;
    gps_acquire_wakelock acquire_wakelock_cb;
    gps_release_wakelock release_wakelock_cb;
    gps_create_thread create_thread_cb;
    gps_request_utc_time request_utc_time_cb;
} GpsCallbacks;
```



## 4.启动GPS

### 4.1.Java层启动

- 在gps定位的apk中,启动GPS

  ```java
  mLocationManager.requestLocationUpdates(provider, 500, 0, mLocationListener);
  ```

  **调用流程图如下**

  ![img](https://img-blog.csdn.net/20170521192749979?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjQzOTQxNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

- 调用native_set_position_mode方法进行设置。然后调用native_start方法启动GPS

  - 在startNavigating方法中,首先获取系统支持的GPS工作模式,主要有三种

  ```java
  private static final int GPS_POSITION_MODE_STANDALONE = 0;//仅GPS工作
  private static final int GPS_POSITION_MODE_MS_BASED = 1;// AGPS MSB模式
  private static final int GPS_POSITION_MODE_MS_ASSISTED = 2;// AGPS MSA模式
  ```

  - 在gps.h也有对应的定义,定义如下

  ```c++
  #define GPS_POSITION_MODE_STANDALONE    0
  #define GPS_POSITION_MODE_MS_BASED      1
  #define GPS_POSITION_MODE_MS_ASSISTED   2
  ```



### 4.2.HAL 层启动GPS

