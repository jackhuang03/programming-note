### Log Error from catalina.out of Tomcat (Tomcat catalina.out 日誌錯誤訊息)
```
23-Jun-2021 10:59:08.574 SEVERE [main] org.apache.catalina.session.StandardManager.startInternal Exception loading sessions from persistent storage
     java.lang.ClassCastException: cannot assign instance of java.lang.StackTraceElement to field
      tw.com.xxxx.bean.Xxxxxx.field of type java.lang.String in instance of tw.com.xxxx.bean.Xxxxxx
	at java.io.ObjectStreamClass$FieldReflector.setObjFieldValues(ObjectStreamClass.java:2301)
	at java.io.ObjectStreamClass.setObjFieldValues(ObjectStreamClass.java:1431)
	at java.io.ObjectInputStream.defaultReadFields(ObjectInputStream.java:2371)
	at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:2289)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2147)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1646)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:482)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:440)
	**at org.apache.catalina.session.StandardSession.doReadObject(StandardSession.java:1587)**
	at org.apache.catalina.session.StandardSession.readObjectData(StandardSession.java:1040)
	at org.apache.catalina.session.StandardManager.doLoad(StandardManager.java:218)
	at org.apache.catalina.session.StandardManager.load(StandardManager.java:162)
	at org.apache.catalina.session.StandardManager.startInternal(StandardManager.java:354)
```

### Cause (原因)
```
Tomcat 重啟時，Tomcat 嘗試要將 AP 存放 Session 中的資料物件反序列化時發生錯誤。這個案例發生的主因為 Session 中的物件沒有實作序列化的處理。

Error happened When Tomcat server restarted and then tried to deserialize session data. Originally, the session object from application did not implment Serializale.java 
which caused invalid process of deserialization.
```
[Session persistence problem with devtools' autorestart · Issue #4939 · spring-projects/spring-boot (github.com)](https://github.com/spring-projects/spring-boot/issues/4939)

### Solution (處理方式)
1. Failed handling (失敗處理)
```
處理 Tomcat 的設定檔，關閉 session 資料的儲存，不要讓 Tomcat 啟動時去讀 session 暫存

Configuring Catalina_Home/conf/server.xml to stop reading back temporary session data when starts(restarts) off.
```
[java - Running Tomcat in Eclipse and getting "Exception loading sessions from persistent storage" - Stack Overflow](https://stackoverflow.com/questions/21181964/running-tomcat-in-eclipse-and-getting-exception-loading-sessions-from-persisten)

2. Successful handling (成功處理)
```java
/*
 * 資料物件實作 java.io.Serializable
 * Data Object implements java.io.Serializable
 */
public class Foo implements Serializable {
  //default
	private static final long serialVersionUID = 1L;
	private String id;
}
```

### Verification (驗證)
```java
@Test
public void testIsSerializedObject() throws IOException, ClassNotFoundException {

        Foo target = new Foo();
        target.setId("1");

        //驗證一 : 確認是否為 java.io.Serializable 類 (大部分情境適用)
        //verification 1 : check if subclass of java.io.Serializable.java (most condition handled)
        assertTrue(target instanceof Serializable);

        //驗證二 : 模擬序列化、反序列化 (嚴謹測試)
        //成功 : System.out.println(o.toString()) 輸出物件
        //失敗 : 拋出例外 java.io.NotSerializableException: com.xxxx.pojo.Foo
        //verification 2 : mocking serialzation and deserialzation
        //success : System.out.println(o.toString()) output Object in String
        //fail : java.io.NotSerializableException: com.xxxx.pojo.Foo thrown
        ByteArrayOutputStream bf = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bf);
        oos.writeObject(target);
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bf.toByteArray()));
        Object o = ois.readObject();
        System.out.println(o.toString());
}
```

### Thinking more (延伸)
```java
/**
為什麼 : 平常我們使用 restful client 或是 restful server 使用 DTO 物件時並未實作 java.io.Serializable ？
回答 : 因為一般使用 Spring 框架或是 Json 套件時，最後都轉型為 java.lang.String，而 String 本身有序列化，所以 http 傳送時轉成 byte 資料的物件沒有異常。
*/

@Test
public void testStringIsSerializedObject() throws IOException, ClassNotFoundException {

        String target = new String (“test”);

        //驗證一 : 確認是否為 java.io.Serializable 類 (大部分情境適用)
        assertTrue(target instanceof Serializable);

        //驗證二 : 模擬 byte 資料傳送 (嚴謹測試)
        //成功 : System.out.println(o.toString()) 輸出物件
        ByteArrayOutputStream bf = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bf);
        oos.writeObject(target);
        oos.close();

        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bf.toByteArray()));
        Object o = ois.readObject();
        System.out.println(o.toString());
}
```

### Reference (參考)
[深入理解 Java 序列化 | JAVACORE (dunwu.github.io)](https://dunwu.github.io/javacore/io/java-serialization.html#_6-java-%E5%BA%8F%E5%88%97%E5%8C%96%E7%9A%84%E7%BC%BA%E9%99%B7)

[JAVA：說說你對序列化的理解_osc_22rhv8iu - MdEditor (gushiciku.cn)](https://www.gushiciku.cn/pl/gpvd/zh-tw)

[Introduction to Java Serialization | Baeldung](https://www.baeldung.com/java-serialization)

[Java - Serialization - Tutorialspoint](https://www.tutorialspoint.com/java/java_serialization.htm)
