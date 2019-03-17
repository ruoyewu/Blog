---
title: Android IPC 方案
date: 2018-03-18 16:16
tags:
	- android
---

`IPC(Inter-Process Communication): 进程间通信`是所有系统中不可忽略的一部分，在 Android 中，我们知道系统会为每个 APP 分配一个 DVM 实例，而每个 DVM 实例都处于一个单独的进程中，也就是说，一般情况下每个 APP 都处于一个单独的进程，而 APP 之间的通信肯定是在所难免的，所以就需要了解一下 Android 上的 IPC 机制。

一般情况下，进程间的通信可以使用一下几个方法，

1.  Bundle
2.  共享文件
3.  Messenger
4.  AIDL
5.  ContentProvider
6.  Socket

## Bundle

Bundle 实现了 Parcelable 序列化，能够比较方便地在进程间传递信息，Android 中的三大组件（ Activity 、 Service 、 Receiver ）都支持在 Intent 中使用 Bundle 传递数据，所以一般使用 Bundle 传递数据的场景为**三大组件之间通信**，这些组件可以在一个进程中，也可以不在一个进程中，都可以使用一个携带着 Bundle 的 Intent 。

## 共享文件

因为在 Android 系统中，多个进程可以同时对一个文件进行读写，那么就可以通过共享同一个文件的方式交换数据，数据的发送方可以事先将需要发送的数据写在一个特定的文件中，数据的接收方就可以直接读取这个文件得到数据，完成进程之间的信息传递，但是由于这种方式需要同时读写文件，可能会导致文件的内容出错，另外，数据接收方并不知道发送方何时刚好将数据写进文件，所以不能完成实时通信。

## Messenger

Messenger 是一个轻量级的 IPC 方案，底层实现是 AIDL ，因为它内部包含的还是一个使用 AIDL 生成的接口，接口拥有一个方法`sendMessage(Message)`，同时当我们调用`Messenger.send(Message)`方法的时候，在 Messenger 内部调用的就是这个接口中的`sendMessage(Message)`方法，只不过 Messenger 把这一部分代码封装了一下。

Messenger 提供的只是一个单向数据传递的功能，如果想要客户端能够与服务端交流，就需要两个 Messenger 分别负责其中一个单向通信，对于从客户端向服务端传递数据的这个过程，

1.  继承 Service 的自定义 Service ，在其中定义一个自定义的 Handler 类重写`handleMessage(Message)`方法用于在 Service 中处理由 客户端传过来的 Message ，然后由这个自定义的 Handler 的实例构建一个 Messenger 实例，同时在 Service 的`onBind(Intent)`方法中返回由方法`Messenger.getBinder()`得到的 Binder 实例。
2.  在客户端，如一个 Activity 中，通过 ServiceConnection 类与对应的 Service 绑定，同时在`onServiceConnected(ComponentName, IBinder)`方法中，根据传递进来的 IBinder 构建一个 Messenger 实例
3.  在需要客户端发送信息给 Service 的时候，只需要调用`Messenger.send(Message)`方法，就可以将这个 Message 传递给 Service 处理

由上述整个过程可以知道，使用 Messenger 的时候每次数据交流的载体只能是 Message ，同时一个 Messenger 只负责一个方向的信息传递，每次只处理一个 Message ，不需要考虑并发请求，也不适用于交流比较频繁的场景。

示例：

`MessengerService.java`

```java
public class MessengerService extends Service {

    private Messenger mMessenger = new Messenger(new MyHandler());

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }

    static class MyHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            // 获取 Bundle
            Bundle bundle = msg.getData();
        }
    }
}
```

`MainActivity.java`

```java
public class MainActivity extends AppCompatActivity {

    private Messenger mMessenger;

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mMessenger = new Messenger(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            mMessenger = null;
        }
    };

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);

        try {
            sendMessage();
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }
    
    private void sendMessage() throws RemoteException {
        Bundle bundle = new Bundle();
        // 为 Bundle 添加数据
        Message message = new Message();
        message.setData(bundle);
        mMessenger.send(message);
    }

    @Override
    protected void onDestroy() {
        unbindService(mConnection);
        super.onDestroy();
    }
}
```



## AIDL

`AIDL(Android Interface Define Language) : Android 接口定义语言`

AIDL 跨进程通信是 Android 使用最广泛的一种，能够处理高并发的数据请求（相比于 Messenger ）。使用 AIDL 完成进程间通信的时候，一般需要编写一个`.aidl`文件用于生成对应的接口文件，这个`.aidl`文件的编写也就是类似于 Java 中的接口，编写接口完之后通过`sync project`这个按钮构建一下，就可以得到由这个`.aidl`文件生成的对应的`.java`接口文件，这个接口文件的子类 Stub 类是 Binder 的子类，实现这个类中的方法，完成数据的操作，最后在 Service  的`onBind()`方法中返回上述的 Stub 类的实例，将这个用于传递信息的 Binder 传递出去。客户端就可以使用这个 Binder 通过`Stub.asInterface(IBinder)`方法得到上面的那个`.java`接口文件，得到之后就可以尽情地使用这个接口完成与 Service 的交互了。

另外，如果在客户端调用了 AIDL 的接口之后，调用这个接口的线程就会进入阻塞状态，直到这个接口返回结果，所以如果在服务端进行的是耗时操作的并且调用端是主线程话，可能会导致 ANR 错误，此时应该将调用接口方法的过程放到子线程进行，然后得到结果之后再使用 Handler 回到主线程。

另外，如果需要给 Service 设置监听，就必然会使用到注册监听器和解注册监听器，同时这个监听器也会在进程中传递，但是平常使用的方法都是直接在 Service 中设置一个存放监听器的 List ，然后添加监听和解除监听的时候直接调用 List 的`add(Object)`和`remove(Object)`方法进行管理，但是当使用进程传输的时候，我们知道，它并不是把对应的实例直接进行传递，而是通过 Parcelable 传输实例的内容，然后通过进程传递数据之后再重新构造一个对应的实例，所以进程两端的根本就不是同一个对象，不能通过`remove(Object)`移除真正需要移除的对象。而有一个类，RemoteCallbackList 能够完成这个工作，RemoteCallbackList 并不是一个 List ，它内部是通过一个`ArrayMap<IBinder, Callback>`实现对监听器的保存的，可以看到这里使用 IBinder 作为键，通过 Map 得到对应的 Callback ，因为 IBinder 是能够跨进程的，所以对于 IBinder 来说，它是确定的，于是即便是 Callback 是一个不确定的对象，也可以通过确定的 IBinder 找到准确的 Callback 。

一般一个应用使用 AIDL 传递数据的时候，都会将 AIDL 接口放在 Service 中供其调用以提供服务，有时一个应用可能会提供多种服务，而如果每一种服务对应的 AIDL 接口都需要使用一个 Service 的话，就会造成 Service 数量的显著增长，从而增加应用的开销，这种时候可以使用 Binder 连接池的方式，实现一个 Service 提供多种 AIDL 服务的功能。当有多种 AIDL 服务的时候，可以建立一个用于获取对应 AIDL 接口的 AIDL 接口，即实现`IBinder queryBinder(int binderCode)`方法，使用时可以通过 binderCode 得到具体的 IBinder 对象，真正在 Service 中`onBinder(Intent)`中返回的是这个接口的实例，然后再通过调用这个 Binder 对应的接口的方法`queryBinder(int)`获取实际提供服务的接口对应的 IBinder 从而构建真正提供服务的接口。这样一个流程就可以只需要一个 Service 提供多种服务并且可以根据条件得到对应的服务的接口。

### 示例：

`IBookManager.aidl`

```java
package com.wuruoye.aidl;

import com.wuruoye.aidl.Book;

// Declare any non-default types here with import statements

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```

这里面定义了一个 IBookManager 接口，同时有两个方法`getBookList()`和`addBook(in Book)`。 AIDL 属于跨进程通信，所以能够支持直接传输的数据类型只有六种，分别是`int boolean long float double String`，如果需要传输其他的类型，就需要先定义一个数据类型，使之实现 Parcelable 接口，完成 Parcelable 序列化，然后再编写一个与这个数据类型同名的`.aidl`文件：

`Book.aidl`

```java
package com.wuruoye.aidl;

// Declare any non-default types here with import statements

parcelable Book;
```

这里对应的 Book 为在应用包里面定义的 Book 类：

`Book.java`

```java
public class Book implements Parcelable {
    public int id;
    public String name;

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(this.id);
        dest.writeString(this.name);
    }

    public Book(int id, String name) {
        this.id = id;
        this.name = name;
    }

    public Book() {
    }

    protected Book(Parcel in) {
        this.id = in.readInt();
        this.name = in.readString();
    }

    public static final Parcelable.Creator<Book> CREATOR = new Parcelable.Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel source) {
            return new Book(source);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };
}
```

这里需要明确一下这三个文件的位置，`.aidl`文件会有一个单独的文件夹`aidl/`，在这个文件夹中有一个包，包里放的就是所有的`.aidl`文件，然后如果`.aidl`文件需要引用自定义的数据类型，就需要引用另一个文件夹`java/`文件夹下的包里面的文件，如下：

![](http://blog-1251826226.coscd.myqcloud.com/Snip20180318_1.png)



## ContentProvider

1ContentProvider 是 Android 的四大组件之一，它被被创造的初衷就是为了完成应用间数据的共享，所以，它肯定是能够跨进程通信的，同时，ContentProvider 底层实现的方式也是 Binder ，与 Messenger 类似，对 Binder 进行了一定的封装。使用这个类进行跨进程通信的时候，首先需要在服务端构建一个继承 ContentProvider 的类，实现它的增删改查以及`getType(Uri)`这个方法，这里需要实现的方法就是当前应用对外提供数据的方式，同时使用 ContentProvider 的方式类似于使用数据库，然后在客户端获取数据的时候使用的是 ContentResolver 类，这个类提供了对应的增删改查的方法，当用户调用 ContentResolver 的方法的时候，它做的工作将就是根据条件找到所有注册在案的 ContentProvider ，然后调用对应的 ContentProvider 的数据操作的方法完成数据共享，然后通过 Binder 的方式将这些数据跨进程传递给客户端，完成一次数据的传递。

另外，除了 ContentResolver 默认提供的增删改查对应的方法之外，还提供了一个更通用的`call(Uri, String, String, Bundle)`方法，这个方法最终会调用 ContentProvider 的`call(String, String, Bundle)`方法，并返回一个 Bundle 对象用于存放数据，使用这个方法，就可以不仅仅局限于那几个方法，自由度更高。

## Socket

套接字，一般用于网络通信，使用 Socket 分别构建服务器和客户端，能够实现客户端向服务器发送请求并得到服务器的返回结果的功能。既然能够实现多设备之间的通信，那进程间通信必然也不在话下，只需要在提供数据服务的进程中使用 Socket 建立一个服务器，其他任何想访问这个进程数据的进程实现客户端并发起请求即可。

另外，如果需要使用 Socket 通信，需要程序有网络权限，即`<uses-permission android:name="android.permission.INTERNET"/>`，然后， Android 规定在 4.0 及以上的系统中不允许在主线程中发起网络请求，因为在发起请求到接受结果的这一段时间里，线程是处于阻塞状态的，所以为了防止 ANR ，需要新开辟一个线程发起请求，得到结果之后使用 Handler 转到主线程就好了。

## 总结

在 Android 系统中，能够用于进程之间通信的方式多种多样，不同的方法都有其特点，具体地可根据不同的要求选取不同的效果，但是由上面的一系列的关于 IPC 方法的探究上发现，AIDL 、 Messenger 、 ContentProvider 这三种进程间通信的方法的底层实现都使用到了 IBinder ，可见 IBinder 在 IPC 中的重要性。

## 参考

《Android 开发艺术探索》