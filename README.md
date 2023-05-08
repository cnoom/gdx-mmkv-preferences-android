# gdx-mmkv-preferences

基于 Libgdx [`com.badlogic.gdx.Preferences`](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/Preferences.java) 和 [`MMKV`](https://github.com/Tencent/MMKV) 的 key-value 持久化

## 使用

在 Android 程序入口(继承了 [`com.badlogic.gdx.backends.android.AndroidApplication`](https://github.com/libgdx/libgdx/blob/master/backends/gdx-backend-android/src/com/badlogic/gdx/backends/android/AndroidApplication.java) 的类) 中重写其 [`getPreferences`](https://github.com/libgdx/libgdx/blob/master/backends/gdx-backend-android/src/com/badlogic/gdx/backends/android/AndroidApplication.java#L327) 方法

```java
public class AndroidLauncher extends AndroidApplication {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        //...
        initializeMMKV();
    }

    private void initializeMMKV() {
        MMKV.initialize(this);
    }

    @Override
    public Preferences getPreferences(String name) {
        return new MMKVPreferences(name);
    }
}

```



```java
public class AndroidApplication extends Activity implements AndroidApplicationBase {
    	private void init (ApplicationListener listener, AndroidApplicationConfiguration config, boolean isForView) {
            //...
            Gdx.app = this;
            //...
        }
}
```

## 实现

支持 [`Preferences.put()`](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/Preferences.java#L51)：

```java
public class GdxMMKVPreferences implements Preferences {
    //...
    @Override
    public Preferences put(Map<String, ?> vals) {
        for (Map.Entry<String, ?> val : vals.entrySet()) {
            if (val.getValue() instanceof Boolean) {
                putBoolean(val.getKey(), (Boolean) val.getValue());
            } else if (val.getValue() instanceof Integer) {
                putInteger(val.getKey(), (Integer) val.getValue());
            } else if (val.getValue() instanceof Long) {
                putLong(val.getKey(), (Long) val.getValue());
            } else if (val.getValue() instanceof String) {
                putString(val.getKey(), (String) val.getValue());
            } else if (val.getValue() instanceof Float) {
                putFloat(val.getKey(), (Float) val.getValue());
            }
        }
        return this;
    }
    //...
}
```

不支持 [`Preferences.get()`](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/Preferences.java#L74)：

```java
public class GdxMMKVPreferences implements Preferences {
    //...
    /**
     * @deprecated Intentionally Not Supported. <br>
     * Use allKeys() instead, getAll() not implement because type-erasure inside mmkv
     */
    @Deprecated
    @Override
    public Map<String, ?> get() {
        return mmkv.getAll();
    }
    //...
}
```

不需要 [`Preferences.flush()`](https://github.com/libgdx/libgdx/blob/master/gdx/src/com/badlogic/gdx/Preferences.java#L83) ：

```java
public class GdxMMKVPreferences implements Preferences {    
    /**
     * @deprecated MMKV don't need to flush
     */
    @Deprecated
    @Override
    public void flush() {
    }
}
```

## 详细

使用 [`MMKV`](https://github.com/Tencent/MMKV) 时需要先初始化：

```java
public class MMKV implements SharedPreferences, SharedPreferences.Editor {
    //...
    public static String initialize(Context context) {
        String root = context.getFilesDir().getAbsolutePath() + "/mmkv";
        MMKVLogLevel logLevel = MMKVLogLevel.LevelInfo;
        return initialize(context, root, (LibLoader)null, logLevel, (MMKVHandler)null);
    }
    // initialize 有许多其他的重载方法
    //...
}
```

[`MMKV`](https://github.com/Tencent/MMKV) **不支持**直接获取所有键值对：

```java
public class MMKV implements SharedPreferences, SharedPreferences.Editor {
    //...
    public Map<String, ?> getAll() {
        throw new UnsupportedOperationException(
            "Intentionally Not Supported. Use allKeys() instead, getAll() not implement because type-erasure inside mmkv"
        );
    }
    //...
}
```

这是因为MMKV的存储方式不同于 [`SharedPreferences`](https://developer.android.com/reference/android/content/SharedPreferences) ，它使用了一种类似于内存映射的技术，将数据保存在一个连续的内存区域中。因此，获取所有键值对需要将整个内存区域读取出来，这可能会导致性能问题和内存消耗。因此，MMKV没有提供 `getAll()` 方法，而是推荐使用其他方式来管理和操作数据。

例如，可以使用 `keys()` 方法获取所有键的列表，然后遍历列表来逐一获取对应的值：

```java
MMKV mmkv = MMKV.defaultMMKV();
String[] keys = mmkv.allKeys();
if (keys == null) {
    return;
}
for (String key : keys) {
    if (key.equals("key1")) {
        String value1 = mmkv.getString(key, "");
        // do something with value1
    } else if (key.equals("key2")) {
        int value2 = mmkv.getInt(key, 0);
        // do something with value2
    }
}
```

或者，可以为每个键值对分别保存一个版本号，在需要获取所有值时，检查每个键的版本号是否发生了变化，如果发生了变化，则表示值已经更新，需要重新获取：

```java
MMKV mmkv = MMKV.defaultMMKV();
mmkv.encode("version", 1L);
long oldVersion = mmkv.decodeLong("version");
log("oldVersion", String.valueOf(oldVersion));
// do something
long version = 2L;
mmkv.encode("version", version);
long newVersion = mmkv.decodeLong("version");
if (newVersion != oldVersion) {
    // do something
}
```



## 另请参阅

[外观模式](https://en.wikipedia.org/wiki/Facade_pattern)

[MMKV 原理](https://github.com/Tencent/MMKV/wiki/design)
