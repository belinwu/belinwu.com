---
layout: post
title: 小心 getLaunchIntentForPackage() 方法
date: 2019-01-10 15:44
categories: Android
---

> - 应用中响应 `android.intent.action.MAIN` 和 `android.intent.category.LAUNCHER` 在本文中称为主界面。
> - 本文基于 Android O


## 问题现象

用PackageInstaller安装应用，在安装完成界面里点击打开，应用闪屏页打开后，按Home键回到桌面，点击桌面里的应用图标。

问题点：再打开一个闪屏页。

## 问题原因

应用中启动别的应用，以上问题场景使用的是 `PackageManager#getLaunchIntentForPackage()` 这个API，它的实现是：

```java
// frameworks\base\core\java\android\app\ApplicationPackageManager.java
@Override
public Intent getLaunchIntentForPackage(String packageName) {
    // First see if the package has an INFO activity; the existence of
    // such an activity is implied to be the desired front-door for the
    // overall package (such as if it has multiple launcher entries).
    Intent intentToResolve = new Intent(Intent.ACTION_MAIN);
    intentToResolve.addCategory(Intent.CATEGORY_INFO);
    intentToResolve.setPackage(packageName);
    List<ResolveInfo> ris = queryIntentActivities(intentToResolve, 0);

    // Otherwise, try to find a main launcher activity.
    if (ris == null || ris.size() <= 0) {
        // reuse the intent instance
        intentToResolve.removeCategory(Intent.CATEGORY_INFO);
        intentToResolve.addCategory(Intent.CATEGORY_LAUNCHER);
        intentToResolve.setPackage(packageName); // <- 这里
        ris = queryIntentActivities(intentToResolve, 0);
    }
    if (ris == null || ris.size() <= 0) {
        return null;
    }
    Intent intent = new Intent(intentToResolve);
    intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
    intent.setClassName(ris.get(0).activityInfo.packageName,
            ris.get(0).activityInfo.name);
    return intent;
}
```

正常桌面启动某个应用的实现如下：

```java
// Launcher3\src\com\android\launcher3\AppInfo.java
public static Intent makeLaunchIntent(Context context, LauncherActivityInfoCompat info,
        UserHandleCompat user) {
    long serialNumber = UserManagerCompat.getInstance(context).getSerialNumberForUser(user);
    return new Intent(Intent.ACTION_MAIN)
        .addCategory(Intent.CATEGORY_LAUNCHER)
        .setComponent(info.getComponentName())
        .setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_RESET_TASK_IF_NEEDED)
        .putExtra(EXTRA_PROFILE, serialNumber);
}
```

对比以上两种启动另一个应用的代码实现，可以发现： `PackageManager#getLaunchIntentForPackage()` 这个API 多了 `intentToResolve.setPackage(packageName); `。

起始应用A直接使用该方法返回的 `Intent` 对象去启动目标应用B，该intent会被AMS增加一个flag：FLAG_ACTIVITY_BROUGHT_TO_FRONT，代码如下：

```java
// frameworks\base\services\core\java\com\android\server\am\ActivityStarter.java
/**
 * Figure out which task and activity to bring to front when we have found an existing matching
 * activity record in history. May also clear the task if needed.
 * @param intentActivity Existing matching activity.
 * @return {@link ActivityRecord} brought to front.
 */
private ActivityRecord setTargetStackAndMoveToFrontIfNeeded(ActivityRecord intentActivity) {
    mTargetStack = intentActivity.getStack();
    mTargetStack.mLastPausedActivity = null;
    // If the target task is not in the front, then we need to bring it to the front...
    // except...  well, with SINGLE_TASK_LAUNCH it's not entirely clear. We'd like to have
    // the same behavior as if a new instance was being started, which means not bringing it
    // to the front if the caller is not itself in the front.
    final ActivityStack focusStack = mSupervisor.getFocusedStack();
    ActivityRecord curTop = (focusStack == null)
            ? null : focusStack.topRunningNonDelayedActivityLocked(mNotTop);

    final TaskRecord topTask = curTop != null ? curTop.getTask() : null;
    if (topTask != null
            && (topTask != intentActivity.getTask() || topTask != focusStack.topTask())
            && !mAvoidMoveToFront) {
        mStartActivity.intent.addFlags(Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT); // <- 这里
        if (mSourceRecord == null || (mSourceStack.topActivity() != null &&
                mSourceStack.topActivity().getTask() == mSourceRecord.getTask())) {
            // We really do want to push this one into the user's face, right now.
            if (mLaunchTaskBehind && mSourceRecord != null) {
                intentActivity.setTaskToAffiliateWith(mSourceRecord.getTask());
            }
            mMovedOtherTask = true;
// ...
}
```

如果该B应用启动后置后台，那么会根据B应用的主界面的lauchMode创建或复用任务栈里的对象，会有意想不到的结果。比如：如果B应用的主界面launchMode是standard，那么会有第二个主界面被创建在任务栈里。

更详细原因分析请参考文章：[关于Android应用回到桌面会重复打开闪屏页](https://www.jianshu.com/p/b202690b7d96)

## 解决方案

### 第一种

在起始应用A里发起跳转时：

```java
Intent intent = context.getPackageManager().getLaunchIntentForPackage(packageName);
intent.setPackage(null); // 加上这句代码
context.startActivity(intent);
```

### 第二种

在目标应用B的主界面`onCreate`里，添加：

```java
super.onCreate(savedInstanceState);
if ((getIntent().getFlags() & Intent.FLAG_ACTIVITY_BROUGHT_TO_FRONT)> 0) {
    /**为了防止重复启动多个闪屏页面**/
    finish();
    return;
}
```