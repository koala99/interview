  <h1>app 实现多进程</h1>
  <p>
 Android中，默认一个APK包就对应一个进程。
Android平台对每个进程有内存限制，如果一個app有多个进程，那么总的内存就是所有进程的内存的总和，使用多进程，可以提高我们APP占用的最高内存。

<p>
实现多进程可以通过设置service、broadcast、activity的标签android:process来实现。
一般情况下启动这些组件默认是在同一个进程里运行的，如果设置了android:process标签，则会运行在其他进程里。
如果android:process的value不是”:”开头，则系统里有同样名字的进程的话，会放到已存在的同名进程里运行，这样能减小消耗。
如果android:process的value是以”:”开头，则启动一个名字为value的进程。
比如:仅仅需要在AndroidManifest.xml里面注册时，加上android:process！
    
    <service
            android:name=".service.AbleService"
            android:enabled="true"
            android:exported="true"
            android:process="com.fingerth.able.service"/>
