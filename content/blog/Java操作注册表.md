---
external: false
title: Java操作注册表
date: 2020-09-17 02:23:23
tags:
- 后端
- Java
- blog
---
# 一、引子

在我们开发某些软件的过程中，常常需要对windows系统的一些系统设置做一些修改

- 比如我们开发一个代理软件，希望开启代理的时候把系统的代理修改为代理软件的ip和端口，又或者我们希望自己开发的软件，可以在安装的时候就加入到开机自启动中，这时候就需要通过修改注册表的方式修改系统的设置
- 如想要获取IE代理的IP地址，我们就需要拿到下面的注册表条目信息

```
\HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ProxyServer
```

![](/images/Pasted%20image%2020230320231542.png)


# 二、如何修改注册表

## 2.1、命令行

windows命令行本身就提供了很多处理注册表的命令，通过`REG /?`可以查看相关命令的使用方法

![](/images/Pasted%20image%2020230320231552.png)

有了命令我们自然就可以直接调用执行来处理注册表了，示例代码如下

![](/images/Pasted%20image%2020230320231602.png)

上述代码运行之后，注册表就多了一条新的记录

![](/images/Pasted%20image%2020230320231608.png)

## 2.2、工具jar包

有很多通过JNI的方式实现的修改注册表第三方工具jar包可以使用，比较有名的就是 [com.ice.jni.registry](http://www.trustice.com/java/jnireg/index.shtml)，点击链接可以直接下载源码，引用jar包和dll后，就可以通过调用它的工具类操作注册表了

- 不过官网的版本dll还是是32位的，这个年代已经看不到32位的机器了，所以我自己处理了一个，[点击下载](https://download.csdn.net/download/weixin_43753950/86541638)
![](/images/Pasted%20image%2020230320231638.png)

引入后，就可通过如下的方式操作注册表了

![](/images/Pasted%20image%2020230320231644.png)


## 2.3、反射调用java原生native方法

- 上面两个方法一个使用命令不好获取结果，执行复杂命令时也会比较麻烦
- 另外一个使用JNI的方式处理，需要额外处理dll文件

而这个方式

- 不使用 JNI。
- 不使用任何第三方工具
- 不直接使用 WINDOWS API
- 纯JAVA

当然这个也有一些问题，我们后面再说
我们知道`com.ice.jni.registry`是自己写了一个dll，在启动的时候在静态代码块中用`System.loadLibrary()`方法加载了一下dll，而java在1.4提供的`WindowsPreferences`类中其实本身就实现了一些操作注册表的native方法，如下

![](/images/Pasted%20image%2020230320231651.png)

不过这些方法都是私有方法没法直接使用，但是通过反射，我们依然可以利用这些方法处理注册表，下面给出工具类的完整代码，以及示例

该方式来源于apache的一个开源项目:[https://github.com/apache/npanday/tree/trunk/components/dotnet-registry/src/main/java/npanday/registry](https://github.com/apache/npanday/tree/trunk/components/dotnet-registry/src/main/java/npanday/registry)

```java
package me.ezerror; 
/**
 * Pure Java Windows Registry access.
 * Modified by petrucio@stackoverflow(828681) to add support for
 * reading (and writing but not creating/deleting keys) the 32-bits
 * registry view from a 64-bits JVM (KEY_WOW64_32KEY)
 * and 64-bits view from a 32-bits JVM (KEY_WOW64_64KEY).
 *****************************************************************************/

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;
import java.util.ArrayList;
import java.util.List;
import java.util.prefs.Preferences;

public class WinRegistry {
  public static final int HKEY_CURRENT_USER = 0x80000001;
  public static final int HKEY_LOCAL_MACHINE = 0x80000002;
  public static final int REG_SUCCESS = 0;
  public static final int REG_NOTFOUND = 2;
  public static final int REG_ACCESSDENIED = 5;

  public static final int KEY_WOW64_32KEY = 0x0200;
  public static final int KEY_WOW64_64KEY = 0x0100;

  private static final int KEY_ALL_ACCESS = 0xf003f;
  private static final int KEY_READ = 0x20019;
  private static Preferences userRoot = Preferences.userRoot();
  private static Preferences systemRoot = Preferences.systemRoot();
  private static Class<? extends Preferences> userClass = userRoot.getClass();
  private static Method regOpenKey = null;
  private static Method regCloseKey = null;
  private static Method regQueryValueEx = null;
  private static Method regEnumValue = null;
  private static Method regQueryInfoKey = null;
  private static Method regEnumKeyEx = null;
  private static Method regCreateKeyEx = null;
  private static Method regSetValueEx = null;
  private static Method regDeleteKey = null;
  private static Method regDeleteValue = null;

  static {
    try {
      regOpenKey     = userClass.getDeclaredMethod("WindowsRegOpenKey",     new Class[] { int.class, byte[].class, int.class });
      regOpenKey.setAccessible(true);
      regCloseKey    = userClass.getDeclaredMethod("WindowsRegCloseKey",    new Class[] { int.class });
      regCloseKey.setAccessible(true);
      regQueryValueEx= userClass.getDeclaredMethod("WindowsRegQueryValueEx",new Class[] { int.class, byte[].class });
      regQueryValueEx.setAccessible(true);
      regEnumValue   = userClass.getDeclaredMethod("WindowsRegEnumValue",   new Class[] { int.class, int.class, int.class });
      regEnumValue.setAccessible(true);
      regQueryInfoKey=userClass.getDeclaredMethod("WindowsRegQueryInfoKey1",new Class[] { int.class });
      regQueryInfoKey.setAccessible(true);
      regEnumKeyEx   = userClass.getDeclaredMethod("WindowsRegEnumKeyEx",   new Class[] { int.class, int.class, int.class });  
      regEnumKeyEx.setAccessible(true);
      regCreateKeyEx = userClass.getDeclaredMethod("WindowsRegCreateKeyEx", new Class[] { int.class, byte[].class });
      regCreateKeyEx.setAccessible(true);  
      regSetValueEx  = userClass.getDeclaredMethod("WindowsRegSetValueEx",  new Class[] { int.class, byte[].class, byte[].class });  
      regSetValueEx.setAccessible(true); 
      regDeleteValue = userClass.getDeclaredMethod("WindowsRegDeleteValue", new Class[] { int.class, byte[].class });  
      regDeleteValue.setAccessible(true); 
      regDeleteKey   = userClass.getDeclaredMethod("WindowsRegDeleteKey",   new Class[] { int.class, byte[].class });  
      regDeleteKey.setAccessible(true); 
    }
    catch (Exception e) {
      e.printStackTrace();
    }
  }

  private WinRegistry() {  }

  /**
   * Read a value from key and value name
   * @param hkey   HKEY_CURRENT_USER/HKEY_LOCAL_MACHINE
   * @param key
   * @param valueName
   * @param wow64  0 for standard registry access (32-bits for 32-bit app, 64-bits for 64-bits app)
   *               or KEY_WOW64_32KEY to force access to 32-bit registry view,
   *               or KEY_WOW64_64KEY to force access to 64-bit registry view
   * @return the value
   * @throws IllegalArgumentException
   * @throws IllegalAccessException
   * @throws InvocationTargetException
   */
  public static String readString(int hkey, String key, String valueName, int wow64) 
    throws IllegalArgumentException, IllegalAccessException,
    InvocationTargetException 
  {
    if (hkey == HKEY_LOCAL_MACHINE) {
      return readString(systemRoot, hkey, key, valueName, wow64);
    }
    else if (hkey == HKEY_CURRENT_USER) {
      return readString(userRoot, hkey, key, valueName, wow64);
    }
    else {
      throw new IllegalArgumentException("hkey=" + hkey);
    }
  }

  /**
   * Read value(s) and value name(s) form given key 
   * @param hkey  HKEY_CURRENT_USER/HKEY_LOCAL_MACHINE
   * @param key
   * @param wow64  0 for standard registry access (32-bits for 32-bit app, 64-bits for 64-bits app)
   *               or KEY_WOW64_32KEY to force access to 32-bit registry view,
   *               or KEY_WOW64_64KEY to force access to 64-bit registry view
   * @return the value name(s) plus the value(s)
   * @throws IllegalArgumentException
   * @throws IllegalAccessException
   * @throws InvocationTargetException
   */
  public static Map<String, String> readStringValues(int hkey, String key, int wow64) 
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    if (hkey == HKEY_LOCAL_MACHINE) {
      return readStringValues(systemRoot, hkey, key, wow64);
    }
    else if (hkey == HKEY_CURRENT_USER) {
      return readStringValues(userRoot, hkey, key, wow64);
    }
    else {
      throw new IllegalArgumentException("hkey=" + hkey);
    }
  }

  /**
   * Read the value name(s) from a given key
   * @param hkey  HKEY_CURRENT_USER/HKEY_LOCAL_MACHINE
   * @param key
   * @param wow64  0 for standard registry access (32-bits for 32-bit app, 64-bits for 64-bits app)
   *               or KEY_WOW64_32KEY to force access to 32-bit registry view,
   *               or KEY_WOW64_64KEY to force access to 64-bit registry view
   * @return the value name(s)
   * @throws IllegalArgumentException
   * @throws IllegalAccessException
   * @throws InvocationTargetException
   */
  public static List<String> readStringSubKeys(int hkey, String key, int wow64) 
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    if (hkey == HKEY_LOCAL_MACHINE) {
      return readStringSubKeys(systemRoot, hkey, key, wow64);
    }
    else if (hkey == HKEY_CURRENT_USER) {
      return readStringSubKeys(userRoot, hkey, key, wow64);
    }
    else {
      throw new IllegalArgumentException("hkey=" + hkey);
    }
  }

  /**
   * Create a key
   * @param hkey  HKEY_CURRENT_USER/HKEY_LOCAL_MACHINE
   * @param key
   * @throws IllegalArgumentException
   * @throws IllegalAccessException
   * @throws InvocationTargetException
   */
  public static void createKey(int hkey, String key) 
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    int [] ret;
    if (hkey == HKEY_LOCAL_MACHINE) {
      ret = createKey(systemRoot, hkey, key);
      regCloseKey.invoke(systemRoot, new Object[] { new Integer(ret[0]) });
    }
    else if (hkey == HKEY_CURRENT_USER) {
      ret = createKey(userRoot, hkey, key);
      regCloseKey.invoke(userRoot, new Object[] { new Integer(ret[0]) });
    }
    else {
      throw new IllegalArgumentException("hkey=" + hkey);
    }
    if (ret[1] != REG_SUCCESS) {
      throw new IllegalArgumentException("rc=" + ret[1] + "  key=" + key);
    }
  }

  /**
   * Write a value in a given key/value name
   * @param hkey
   * @param key
   * @param valueName
   * @param value
   * @param wow64  0 for standard registry access (32-bits for 32-bit app, 64-bits for 64-bits app)
   *               or KEY_WOW64_32KEY to force access to 32-bit registry view,
   *               or KEY_WOW64_64KEY to force access to 64-bit registry view
   * @throws IllegalArgumentException
   * @throws IllegalAccessException
   * @throws InvocationTargetException
   */
  public static void writeStringValue
    (int hkey, String key, String valueName, String value, int wow64) 
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    if (hkey == HKEY_LOCAL_MACHINE) {
      writeStringValue(systemRoot, hkey, key, valueName, value, wow64);
    }
    else if (hkey == HKEY_CURRENT_USER) {
      writeStringValue(userRoot, hkey, key, valueName, value, wow64);
    }
    else {
      throw new IllegalArgumentException("hkey=" + hkey);
    }
  }

  /**
   * Delete a given key
   * @param hkey
   * @param key
   * @throws IllegalArgumentException
   * @throws IllegalAccessException
   * @throws InvocationTargetException
   */
  public static void deleteKey(int hkey, String key) 
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    int rc = -1;
    if (hkey == HKEY_LOCAL_MACHINE) {
      rc = deleteKey(systemRoot, hkey, key);
    }
    else if (hkey == HKEY_CURRENT_USER) {
      rc = deleteKey(userRoot, hkey, key);
    }
    if (rc != REG_SUCCESS) {
      throw new IllegalArgumentException("rc=" + rc + "  key=" + key);
    }
  }

  /**
   * delete a value from a given key/value name
   * @param hkey
   * @param key
   * @param value
   * @param wow64  0 for standard registry access (32-bits for 32-bit app, 64-bits for 64-bits app)
   *               or KEY_WOW64_32KEY to force access to 32-bit registry view,
   *               or KEY_WOW64_64KEY to force access to 64-bit registry view
   * @throws IllegalArgumentException
   * @throws IllegalAccessException
   * @throws InvocationTargetException
   */
  public static void deleteValue(int hkey, String key, String value, int wow64) 
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    int rc = -1;
    if (hkey == HKEY_LOCAL_MACHINE) {
      rc = deleteValue(systemRoot, hkey, key, value, wow64);
    }
    else if (hkey == HKEY_CURRENT_USER) {
      rc = deleteValue(userRoot, hkey, key, value, wow64);
    }
    if (rc != REG_SUCCESS) {
      throw new IllegalArgumentException("rc=" + rc + "  key=" + key + "  value=" + value);
    }
  }

  //========================================================================
  private static int deleteValue(Preferences root, int hkey, String key, String value, int wow64)
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    int[] handles = (int[]) regOpenKey.invoke(root, new Object[] {
        new Integer(hkey), toCstr(key), new Integer(KEY_ALL_ACCESS | wow64)
    });
    if (handles[1] != REG_SUCCESS) {
      return handles[1];  // can be REG_NOTFOUND, REG_ACCESSDENIED
    }
    int rc =((Integer) regDeleteValue.invoke(root, new Object[] { 
          new Integer(handles[0]), toCstr(value) 
          })).intValue();
    regCloseKey.invoke(root, new Object[] { new Integer(handles[0]) });
    return rc;
  }

  //========================================================================
  private static int deleteKey(Preferences root, int hkey, String key) 
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    int rc =((Integer) regDeleteKey.invoke(root, new Object[] {
        new Integer(hkey), toCstr(key)
    })).intValue();
    return rc;  // can REG_NOTFOUND, REG_ACCESSDENIED, REG_SUCCESS
  }

  //========================================================================
  private static String readString(Preferences root, int hkey, String key, String value, int wow64)
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    int[] handles = (int[]) regOpenKey.invoke(root, new Object[] {
        new Integer(hkey), toCstr(key), new Integer(KEY_READ | wow64)
    });
    if (handles[1] != REG_SUCCESS) {
      return null; 
    }
    byte[] valb = (byte[]) regQueryValueEx.invoke(root, new Object[] {
        new Integer(handles[0]), toCstr(value)
    });
    regCloseKey.invoke(root, new Object[] { new Integer(handles[0]) });
    return (valb != null ? new String(valb).trim() : null);
  }

  //========================================================================
  private static Map<String,String> readStringValues(Preferences root, int hkey, String key, int wow64)
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    HashMap<String, String> results = new HashMap<String,String>();
    int[] handles = (int[]) regOpenKey.invoke(root, new Object[] {
        new Integer(hkey), toCstr(key), new Integer(KEY_READ | wow64)
    });
    if (handles[1] != REG_SUCCESS) {
      return null;
    }
    int[] info = (int[]) regQueryInfoKey.invoke(root, new Object[] {
        new Integer(handles[0])
    });

    int count  = info[2]; // count  
    int maxlen = info[3]; // value length max
    for(int index=0; index<count; index++)  {
      byte[] name = (byte[]) regEnumValue.invoke(root, new Object[] {
          new Integer(handles[0]), new Integer(index), new Integer(maxlen + 1)
      });
      String value = readString(hkey, key, new String(name), wow64);
      results.put(new String(name).trim(), value);
    }
    regCloseKey.invoke(root, new Object[] { new Integer(handles[0]) });
    return results;
  }

  //========================================================================
  private static List<String> readStringSubKeys(Preferences root, int hkey, String key, int wow64)
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    List<String> results = new ArrayList<String>();
    int[] handles = (int[]) regOpenKey.invoke(root, new Object[] {
        new Integer(hkey), toCstr(key), new Integer(KEY_READ | wow64) 
        });
    if (handles[1] != REG_SUCCESS) {
      return null;
    }
    int[] info = (int[]) regQueryInfoKey.invoke(root, new Object[] {
        new Integer(handles[0])
    });

    int count  = info[0]; // Fix: info[2] was being used here with wrong results. Suggested by davenpcj, confirmed by Petrucio
    int maxlen = info[3]; // value length max
    for(int index=0; index<count; index++)  {
      byte[] name = (byte[]) regEnumKeyEx.invoke(root, new Object[] {
          new Integer(handles[0]), new Integer(index), new Integer(maxlen + 1)
          });
      results.add(new String(name).trim());
    }
    regCloseKey.invoke(root, new Object[] { new Integer(handles[0]) });
    return results;
  }

  //========================================================================
  private static int [] createKey(Preferences root, int hkey, String key)
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    return (int[]) regCreateKeyEx.invoke(root, new Object[] {
      new Integer(hkey), toCstr(key)
    });
  }

  //========================================================================
  private static void writeStringValue(Preferences root, int hkey, String key, String valueName, String value, int wow64)
    throws IllegalArgumentException, IllegalAccessException, InvocationTargetException 
  {
    int[] handles = (int[]) regOpenKey.invoke(root, new Object[] {
        new Integer(hkey), toCstr(key), new Integer(KEY_ALL_ACCESS | wow64)
    });
    regSetValueEx.invoke(root, new Object[] { 
          new Integer(handles[0]), toCstr(valueName), toCstr(value) 
          }); 
    regCloseKey.invoke(root, new Object[] { new Integer(handles[0]) });
  }

  //========================================================================
  // utility
  private static byte[] toCstr(String str) {
    byte[] result = new byte[str.length() + 1];

    for (int i = 0; i < str.length(); i++) {
      result[i] = (byte) str.charAt(i);
    }
    result[str.length()] = 0;
    return result;
  }

  public static void main(String[] args) throws InvocationTargetException, IllegalAccessException {
  	// test by ezerror 
    String value = readString(HKEY_CURRENT_USER, "Software\\Microsoft\\Windows\\CurrentVersion\\Internet Settings", "ProxyServer", 0);
    System.out.println(value);
  }
}

```

![](/images/Pasted%20image%2020230320231704.png)

这种方式虽然直接避开了windows api以及第三方的工具，但是这个方式并不是稳妥的，一方面这种方式本身存在很多限制，或许你期待的操作根本无法通过这种方式实现，另外如果你的项目如果后续可能升级java版本，也许这块代码直接就失效了，所以最好不要将这段代码放在会持续迭代的项目里