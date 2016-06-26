java.lang.IllegalAccessError: Class ref in pre-verified class resolved to unexpected implementation


startActivity(new Intent(getApplicationContext(), classLoader.loadClass("com.github.lzyzsd.bundle1.Bundle1Activity")));

java.lang.RuntimeException: Unable to instantiate activity ComponentInfo{com.github.lzyzsd.androidplugin/com.github.lzyzsd.bundle1.Bundle1Activity}: java.lang.ClassNotFoundException: Didn't find class "com.github.lzyzsd.bundle1.Bundle1Activity" on path: DexPathList[[zip file "/data/data/com.github.lzyzsd.androidplugin/app_dex/hackdex.jar", zip file "/data/app/com.github.lzyzsd.androidplugin-2.apk"],nativeLibraryDirectories=[/data/app-lib/com.github.lzyzsd.androidplugin-2, /vendor/lib, /system/lib]]
at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2190)
at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2313)


ActivityThread

private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
  java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
  activity = mInstrumentation.newActivity(
          cl, component.getClassName(), r.intent);
}


android.util.SuperNotCalledException: Activity {com.github.lzyzsd.androidplugin/com.github.lzyzsd.androidplugin.MainActivity} did not call through to super.onCreate()


12-18 15:26:21.540 6051-6051/com.github.lzyzsd.androidplugin W/System.err: Caused by: android.content.ActivityNotFoundException: Unable to find explicit activity class {com.github.lzyzsd.androidplugin/com.github.lzyzsd.bundle1.Bundle1Activity}; have you declared this activity in your AndroidManifest.xml?

public static void checkStartActivityResult(int res, Object intent) {
        if (res >= ActivityManager.START_SUCCESS) {
            return;
        }

        switch (res) {
            case ActivityManager.START_INTENT_NOT_RESOLVED:
            case ActivityManager.START_CLASS_NOT_FOUND:
                if (intent instanceof Intent && ((Intent)intent).getComponent() != null)
                    throw new ActivityNotFoundException(
                            "Unable to find explicit activity class "
                            + ((Intent)intent).getComponent().toShortString()
                            + "; have you declared this activity in your AndroidManifest.xml?");
                throw new ActivityNotFoundException(
                        "No Activity found to handle " + intent);


ActivityStackSupervisor
startActivityMayWait ->resolveActivity

ActivityInfo resolveActivity(Intent intent, String resolvedType, int startFlags,
            ProfilerInfo profilerInfo, int userId) {
        // Collect information about the target of the Intent.
        ActivityInfo aInfo;
        try {
            ResolveInfo rInfo =
                AppGlobals.getPackageManager().resolveIntent(
                        intent, resolvedType,
                        PackageManager.MATCH_DEFAULT_ONLY
                                    | ActivityManagerService.STOCK_PM_FLAGS, userId);
            aInfo = rInfo != null ? rInfo.activityInfo : null;
        } catch (RemoteException e) {
            aInfo = null;
        }

ActivityStackSupervisor.startActivityLocked
if (err == ActivityManager.START_SUCCESS && aInfo == null) {
            // We couldn't find the specific class specified in the Intent.
            // Also the end of the line.
            err = ActivityManager.START_CLASS_NOT_FOUND;
        }



ActivityThread.java

public static IPackageManager getPackageManager() {
        if (sPackageManager != null) {
            //Slog.v("PackageManager", "returning cur default = " + sPackageManager);
            return sPackageManager;
        }
        IBinder b = ServiceManager.getService("package");
        //Slog.v("PackageManager", "default service binder = " + b);
        sPackageManager = IPackageManager.Stub.asInterface(b);
        //Slog.v("PackageManager", "default service = " + sPackageManager);
        return sPackageManager;
    }



ContextImpl.java
    @Override
        public PackageManager getPackageManager() {
            if (mPackageManager != null) {
                return mPackageManager;
            }

            IPackageManager pm = ActivityThread.getPackageManager();
            if (pm != null) {
                // Doesn't matter if we make more than one instance.
                return (mPackageManager = new ApplicationPackageManager(this, pm));
            }

            return null;
        }
