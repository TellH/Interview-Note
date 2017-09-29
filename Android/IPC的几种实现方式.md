## Bundle

Android中Activity，Service，Receiver都支持在Intent中传递Bundle数据，由于Bundle实现了Parcelable接口，可以轻松地在进程间传送。基本类型，实现了Parcelable接口的对象，实现了Serializable接口的对象可以放入Bundle中，这样Bundle就成了进程间通信的载体。



## Messenger

Messenger是一种轻量级的IPC方案，底层实现是AIDL。

- 在服务端Service实例化一个Handler，再通过Handler实例化Messenger，通过Service的onBind返回给客户端.

```java
public class MessengerService extends Service {
    private final Messenger mMessenger = new Messenger(new MessengerHandler());

    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
          ...
        }
    }

    public MessengerService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
```

- 客户端获取Messenger

```java
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mMessenger = new Messenger(service);
        }
```



- 服务端和客户端通过Messenger为信使，Message为信息的载体来实现IPC

```java
        Message message = Message.obtain(null, MessengerService.MSG_FROM_CLIENT);
        Bundle data = new Bundle();
        data.putString("msg", "Hello, this is client.");
        message.setData(data);
        message.replyTo = mGetReplyMessenger;
        mMessenger.send(message);

        private Messenger mGetReplyMessenger = new Messenger(new Handler(){
            @Override
            public void handleMessage(Message msg) {
            switch (msg.what) {
                case MessengerService.MSG_FROM_CLIENT:
                    Log.i(TAG, "receive msg from server:" + msg.getData().getString("reply"));
                    break;
                default:
                    super.handleMessage(msg);
            }
            }
        });
```

server端

```java
                    Messenger client = msg.replyTo;
                    Message replyMsg = Message.obtain(null, MSG_FROM_CLIENT);
                    Bundle data = new Bundle();
                    data.putString("reply", "Hello, this is server, I know you.");
                    replyMsg.setData(data);
                    try {
                        client.send(replyMsg);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
```

在Messenger中进行数据传递必须将数据放入Message中，而Message中能使用的载体只有int what，int arg1，int arg2，Bundle以及Messenger replyTo。在Android 2.2以后，obj字段支持实现Parcelable接口的对象通过它来传输。





## ContentProvider

ContentProvider除了onCreate由系统回掉并运行在主线程中，其他五个方法均由外界回掉并运行在Binder线程池中。

```java
public class BookProvider extends ContentProvider {
    private static final String TAG = BookProvider.class.getSimpleName();
    public static final String AUTHORITY = "tellh.com.androidart.ipc.BookProvider";
    public static final Uri BOOK_CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/book");
    public static final Uri URSER_CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/user");
    public static final int BOOK_URI_CODE = 0;
    public static final int USER_URI_CODE = 1;
    private static final UriMatcher sUriMatcher = new UriMatcher(UriMatcher.NO_MATCH);

    static {
        // 将uri与code关联
        sUriMatcher.addURI(AUTHORITY, "book", BOOK_URI_CODE);
        sUriMatcher.addURI(AUTHORITY, "book", USER_URI_CODE);
    }

    private Context mContext;
    private SQLiteDatabase mDb;

    @Override
    public boolean onCreate() {
        Logger.init(TAG);
        Logger.d("onCreate");
        mContext = getContext();
        initData();
        return true;
    }

    private void initData() {
        mDb = new BookDbOpenHelper(mContext).getWritableDatabase();
        mDb.execSQL("delete from " + BookDbOpenHelper.BOOK_TABLE_NAME);
        mDb.execSQL("delete from " + BookDbOpenHelper.USER_TABLE_NAME);
        mDb.execSQL("insert into book values(3,'Android');");
        mDb.execSQL("insert into book values(4,'iOS');");
        mDb.execSQL("insert into book values(5,'Html5');");
        mDb.execSQL("insert into user values(1,'jack',1);");
        mDb.execSQL("insert into user values(2,'rose',0);");
    }

    private String getTableName(Uri uri) {
        String tableNmae = null;
        switch (sUriMatcher.match(uri)) {
            case BOOK_URI_CODE:
                tableNmae = BookDbOpenHelper.BOOK_TABLE_NAME;
                break;
            case USER_URI_CODE:
                tableNmae = BookDbOpenHelper.USER_TABLE_NAME;
                break;
            default:
                break;
        }
        return tableNmae;
    }

    /**
     * @param uri           查询资源的定位符
     * @param projection    投影，具体要查的列
     * @param selection     条件
     * @param selectionArgs 条件中占位符的值
     * @param sortOrder     排序
     * @return 查询结果的游标
     */
    @Nullable
    @Override
    public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder) {
        Logger.d("query");
        String tableName = getTableName(uri);
        if (tableName == null)
            throw new IllegalArgumentException("Invalid URI");
        return mDb.query(tableName, projection, selection, selectionArgs, null, null, sortOrder, null);
    }

    @Nullable
    @Override
    public String getType(Uri uri) {
        Logger.d("getType");
        return null;
    }

    @Nullable
    @Override
    public Uri insert(Uri uri, ContentValues values) {
        Logger.d("insert");
        String tableName = getTableName(uri);
        if (tableName == null)
            throw new IllegalArgumentException("Invalid URI");
        mDb.insert(tableName, null, values);
        mContext.getContentResolver().notifyChange(uri, null);
        return uri;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        Logger.d("delete");
        String tableName = getTableName(uri);
        if (tableName == null)
            throw new IllegalArgumentException("Invalid URI");
        int count = mDb.delete(tableName, selection, selectionArgs);
        if (count > 0)
            mContext.getContentResolver().notifyChange(uri, null);
        return count;
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs) {
        Logger.d("update");
        String tableName = getTableName(uri);
        if (tableName == null)
            throw new IllegalArgumentException("Invalid URI");
        int rows = mDb.update(tableName, values, selection, selectionArgs);
        if (rows > 0)
            mContext.getContentResolver().notifyChange(uri, null);
        return 0;
    }
}
```

```xml
        <provider
            android:name=".ipc.BookProvider"
            android:authorities="tellh.com.androidart.ipc.BookProvider"
            android:permission="tellh.com.androidart.PROVIDER"
            android:process=":provider" />
```



客户端调用ContentProvider暴露的方法

```java
        Uri uri = Uri.parse("content://tellh.com.androidart.ipc.BookProvider/book");
        ContentValues contentValues = new ContentValues();
        contentValues.put("bid", 6);
        contentValues.put("name", "深入理解Java虚拟机");
        getContentResolver().insert(uri, contentValues);
        Cursor cursor = getContentResolver().query(uri, new String[]{"bid", "name"}, null, null, null);
```



## Socket 网络编程

## Broadcast

