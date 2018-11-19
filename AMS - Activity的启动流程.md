# AMS #

## 1. Acitivity 中Intent启动 ##
    
### android.app.Activity ###
 startActivity（Intent）-> startActivity（Intent,  Bundle options）-> startActivityForResult(Intent intent, int requestCode,Bundle options) 

requestCode 为-1；
startActivityForResult方法中 会判断mParent 是否为空。
mParent ： 有attach 方法传入进来。 类型也为Activity。
mParent 假如不为空，则走的父类的 
-> startActivityFromChild(@NonNull Activity child, @RequiresPermission Intent intent,int requestCode, @Nullable Bundle options)

如果为空，则走Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);


mParent == null；
Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);

mMainThread： 有attach 方法传入， 为 ActivityThread 类型。   mMainThread.getApplicationThread()则为 返回ApplicationThread 。ActivityThread中有一个 ApplicationThread 的对象， mAppThread。

mToken： IBinder类型， 有attach 方法传入。

ar 为执行startActivity的结果。


mParent != null ；
走的 父累的mParent.startActivityFromChild(this, intent, requestCode, options)。也是Activity 的一个方法。而 这个方法 和mParent 的流程类似。也是走的 Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, child,
                intent, requestCode, options);
只是其中一个参数是传的child 也是当前Activity。


mInstrumentation ：  Instrumentation类型，也是有attach 传入。 Instrumentation这个类相当于操作监控类，所有的操作都会经过此类。这也是android测试框架的核心类。


### android.app.Instrumentation ###

-> ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) 


contextThread 即 mMainThread.getApplicationThread()传入的ApplicationThread.其也是一个继承IBinder 的类。

分析 ：
IApplicationThread whoThread = (IApplicationThread) contextThread;

whoThread强制转化为IApplicationThread， ApplicationThread 继承与IApplicationThread.Stub。

> 疑问： IApplicationThread.Stub在哪里？？？？
>         IApplicationThread whoThread = (IApplicationThread) contextThread; IBinder 强转成 IApplicationThread？？？



-> int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);


### android.app.ActivityManager ###


-> getService（） 得带IActivityManager对象。

-> IActivityManagerSingleton.get();
    
IActivityManagerSingleton 内部类。 

 private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };

Singleton 为一个单利的工厂类。

public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}


通过create 来生产出 单利。

onCreate 实现 

 final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;



ServiceManager 为Service的管理类，通过tag 获得对应的服务。ServiceManager获得了Activity_service的IBinder 对象。通过 IActivityManager.Stub.asInterface(b) 封装IBinder，转为 IActivityManager接口对象。

IActivityManager 为 ActivityService的IBinder的代理类。

> 疑问： IActivityManager.Stub.asInterface(b) 没找到？？？在哪里。


得带ActivityService 的代理类后：

-> startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options)

whoThread : 即 IBinder ， ApplicationThread
who.getBasePackageName() ： 包名
token ：activity的token
mEmbeddedID 为id，有attach 传入。


startActivity 是通过IBinder调用了ActivityService的startActivity 方法。
###  com.android.server.am.ActivityManagerService ###

-> startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions)


caller : IApplicationThread 
callingPackage : 包名
intent ： 信息
resultTo ： Ibinder 即为activity的token
resultWho： 为id；
startFlags ： 为 0
profilerInfo ： 为null；

-> startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId)


userId : UserHandle.getCallingUserId(); calling 者的id， 

然后往下：

userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
                userId, false, ALLOW_FULL_ONLY, "startActivity", null);

mUserController : ActivityManagerService 创建的时候 随之创建的， 把 ActivityManagerService传入了 UserController。

UserController(ActivityManagerService service) {
    mService = service;
    mHandler = mService.mHandler;
    // User 0 is the first and only user that runs at boot.
    final UserState uss = new UserState(UserHandle.SYSTEM);
    mStartedUsers.put(UserHandle.USER_SYSTEM, uss);
    mUserLru.add(UserHandle.USER_SYSTEM);
    mLockPatternUtils = new LockPatternUtils(mService.mContext);
    updateStartedUserArrayLocked();
}


UserState ： UserController 创建而创建。 用户的状态，传入的使用户UserHandle的单利。

mStartedUsers ： 运作的用户记录。

mUserLru： 用户缓存记录

### com.android.server.am.UserController  ###
-> int handleIncomingUser(int callingPid, int callingUid, int userId, boolean allowAll,
            int allowMode, String name, String callerPackage) 

callingPid : 呼叫者的pid
callingUid： 。。。 uid
userId ： 呼叫者的userId 通过一定计算得到的。UserHandle.getUserId 。
allowAll ： false；
allowMode： ALLOW_FULL_ONLY = 2
name： "startActivity"
callerPackage： null