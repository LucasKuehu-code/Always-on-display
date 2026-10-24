name: Build Always On Display APK

on:
  push:
    branches: [main, master]
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.9'

      - name: Generate Android project
        run: |
          mkdir -p app
          cat > app/build.gradle <<'EOF'
          plugins {
              id 'com.android.application'
          }

          android {
              namespace 'com.lucas.aod'
              compileSdk 34

              defaultConfig {
                  applicationId 'com.lucas.aod'
                  minSdk 27
                  targetSdk 34
                  versionCode 1
                  versionName '1.0'
              }

              compileOptions {
                  sourceCompatibility JavaVersion.VERSION_17
                  targetCompatibility JavaVersion.VERSION_17
              }
          }
          EOF
          mkdir -p app/src/main
          cat > app/src/main/AndroidManifest.xml <<'EOF'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
              <uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission android:name="android.permission.USE_FULL_SCREEN_INTENT" />
              <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
              <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
              <uses-permission android:name="android.permission.QUERY_ALL_PACKAGES" />

              <application
                  android:allowBackup="false"
                  android:label="Always On Display"
                  android:theme="@android:style/Theme.DeviceDefault">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <activity
                      android:name=".AodActivity"
                      android:exported="false"
                      android:showWhenLocked="true"
                      android:turnScreenOn="true"
                      android:excludeFromRecents="true"
                      android:launchMode="singleTask"
                      android:taskAffinity=""
                      android:screenOrientation="portrait"
                      android:configChanges="orientation|screenSize|keyboardHidden"
                      android:theme="@android:style/Theme.DeviceDefault.NoActionBar" />

                  <service
                      android:name=".AodService"
                      android:exported="false"
                      android:foregroundServiceType="specialUse">
                      <property
                          android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
                          android:value="always_on_display" />
                  </service>

                  <service
                      android:name=".NotifListener"
                      android:exported="true"
                      android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
                      <intent-filter>
                          <action android:name="android.service.notification.NotificationListenerService" />
                      </intent-filter>
                  </service>

                  <receiver
                      android:name=".BootReceiver"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.BOOT_COMPLETED" />
                      </intent-filter>
                  </receiver>
              </application>
          </manifest>
          EOF
          mkdir -p app/src/main/java/com/lucas/aod
          cat > app/src/main/java/com/lucas/aod/AodActivity.java <<'EOF'
          package com.lucas.aod;

          import android.app.Activity;
          import android.app.NotificationManager;
          import android.content.Context;
          import android.content.Intent;
          import android.content.IntentFilter;
          import android.graphics.Canvas;
          import android.graphics.Color;
          import android.graphics.Paint;
          import android.graphics.Typeface;
          import android.graphics.drawable.Drawable;
          import android.os.BatteryManager;
          import android.os.Build;
          import android.os.Bundle;
          import android.os.Handler;
          import android.os.Looper;
          import android.text.TextUtils;
          import android.util.TypedValue;
          import android.view.Gravity;
          import android.view.View;
          import android.view.Window;
          import android.view.WindowInsets;
          import android.view.WindowInsetsController;
          import android.view.WindowManager;
          import android.widget.FrameLayout;
          import android.widget.ImageView;
          import android.widget.LinearLayout;
          import android.widget.TextView;

          import java.text.SimpleDateFormat;
          import java.util.Calendar;
          import java.util.Date;
          import java.util.Locale;
          import java.util.Random;

          public class AodActivity extends Activity {
              static volatile AodActivity current;

              private final Handler h = new Handler(Looper.getMainLooper());
              private final Random rnd = new Random();
              private LinearLayout box, notesBox;
              private TextView time, date, batt;
              private AnalogView analog;
              private int style, nMode, nMax, nDetail;

              private int dp(int v) { return (int) (v * getResources().getDisplayMetrics().density); }

              private TextView tv(int color, int sp, String family) {
                  TextView t = new TextView(this);
                  t.setTextColor(color);
                  t.setTextSize(TypedValue.COMPLEX_UNIT_SP, sp);
                  t.setTypeface(Typeface.create(family, Typeface.NORMAL));
                  t.setGravity(Gravity.CENTER);
                  t.setIncludeFontPadding(false);
                  return t;
              }

              static class AnalogView extends View {
                  private final Paint p = new Paint(Paint.ANTI_ALIAS_FLAG);
                  private final int col;

                  AnalogView(Context c, int col) {
                      super(c);
                      this.col = col;
                  }

                  private void hand(Canvas cv, float cx, float cy, float deg, float len, float w) {
                      double a = Math.toRadians(deg);
                      p.setStrokeWidth(w);
                      cv.drawLine(cx, cy, cx + (float) Math.sin(a) * len, cy - (float) Math.cos(a) * len, p);
                  }

                  @Override
                  protected void onDraw(Canvas cv) {
                      float cx = getWidth() / 2f, cy = getHeight() / 2f;
                      float r = Math.min(cx, cy) - 6;
                      p.setColor(col);
                      p.setStyle(Paint.Style.STROKE);
                      p.setStrokeCap(Paint.Cap.ROUND);
                      p.setStrokeWidth(3);
                      cv.drawCircle(cx, cy, r, p);
                      for (int i = 0; i < 12; i++) {
                          double a = Math.toRadians(i * 30);
                          float in = r * (i % 3 == 0 ? 0.80f : 0.90f);
                          cv.drawLine(cx + (float) Math.sin(a) * in, cy - (float) Math.cos(a) * in,
                                  cx + (float) Math.sin(a) * r * 0.96f, cy - (float) Math.cos(a) * r * 0.96f, p);
                      }
                      Calendar c = Calendar.getInstance();
                      float m = c.get(Calendar.MINUTE);
                      float hr = (c.get(Calendar.HOUR) % 12) + m / 60f;
                      hand(cv, cx, cy, hr * 30, r * 0.5f, 9);
                      hand(cv, cx, cy, m * 6, r * 0.78f, 5);
                  }
              }

              private final Runnable tick = new Runnable() {
                  @Override
                  public void run() {
                      refresh();
                      // Pixel shift every minute to avoid burn-in.
                      box.setTranslationX(rnd.nextInt(61) - 30);
                      box.setTranslationY(rnd.nextInt(81) - 40);
                      long now = System.currentTimeMillis();
                      h.postDelayed(this, 60000 - (now % 60000) + 50);
                  }
              };

              @Override
              protected void onCreate(Bundle b) {
                  super.onCreate(b);
                  setShowWhenLocked(true);
                  setTurnScreenOn(true);

                  Window w = getWindow();
                  w.addFlags(WindowManager.LayoutParams.FLAG_KEEP_SCREEN_ON);
                  WindowManager.LayoutParams lp = w.getAttributes();
                  lp.screenBrightness = 0.01f;
                  lp.layoutInDisplayCutoutMode =
                          WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
                  w.setAttributes(lp);
                  w.setStatusBarColor(Color.BLACK);
                  w.setNavigationBarColor(Color.BLACK);

                  if (Build.VERSION.SDK_INT >= 30) {
                      w.setDecorFitsSystemWindows(false);
                      WindowInsetsController c = w.getInsetsController();
                      if (c != null) {
                          c.hide(WindowInsets.Type.systemBars());
                          c.setSystemBarsBehavior(
                                  WindowInsetsController.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE);
                      }
                  } else {
                      w.getDecorView().setSystemUiVisibility(
                              View.SYSTEM_UI_FLAG_FULLSCREEN
                                      | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                                      | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY
                                      | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                                      | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                                      | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
                  }

                  style = Prefs.geti(this, "style", 0);
                  nMode = Prefs.geti(this, "nMode", 0);
                  nMax = Prefs.geti(this, "nMax", 5);
                  nDetail = Prefs.geti(this, "nDetail", 1);
                  float sc = Prefs.geti(this, "size", 100) / 100f;
                  int pos = Prefs.geti(this, "pos", 1);
                  int col = Prefs.color(this);
                  int dim = (col & 0x00FFFFFF) | 0xB0000000;

                  FrameLayout root = new FrameLayout(this);
                  root.setBackgroundColor(Color.BLACK);
                  root.setClickable(true);

                  box = new LinearLayout(this);
                  box.setOrientation(LinearLayout.VERTICAL);
                  box.setGravity(Gravity.CENTER_HORIZONTAL);
                  box.setPadding(dp(24), 0, dp(24), 0);

                  if (style == 3) {
                      analog = new AnalogView(this, col);
                      int s = (int) (dp(240) * sc);
                      box.addView(analog, new LinearLayout.LayoutParams(s, s));
                  } else {
                      float sz = 96;
                      String fam = "sans-serif-thin";
                      if (style == 1) { sz = 80; fam = "sans-serif-black"; }
                      if (style == 2) { sz = 130; }
                      if (style == 4) { sz = 72; fam = "monospace"; }
                      time = tv(col, (int) (sz * sc), fam);
                      if (style == 2) time.setLineSpacing(0, 0.85f);
                      box.addView(time);
                  }

                  date = tv(dim, 18, "sans-serif");
                  date.setPadding(0, dp(8), 0, 0);
                  batt = tv(dim, 16, "sans-serif");
                  batt.setPadding(0, dp(6), 0, 0);
                  date.setVisibility(Prefs.showDate(this) ? View.VISIBLE : View.GONE);
                  batt.setVisibility(Prefs.showBatt(this) ? View.VISIBLE : View.GONE);
                  box.addView(date);
                  box.addView(batt);

                  notesBox = new LinearLayout(this);
                  notesBox.setOrientation(nMode == 1 ? LinearLayout.HORIZONTAL : LinearLayout.VERTICAL);
                  notesBox.setGravity(Gravity.CENTER);
                  notesBox.setPadding(0, dp(24), 0, 0);
                  box.addView(notesBox, new LinearLayout.LayoutParams(
                          LinearLayout.LayoutParams.WRAP_CONTENT, LinearLayout.LayoutParams.WRAP_CONTENT));

                  FrameLayout.LayoutParams fp = new FrameLayout.LayoutParams(
                          FrameLayout.LayoutParams.MATCH_PARENT,
                          FrameLayout.LayoutParams.WRAP_CONTENT,
                          pos == 0 ? (Gravity.TOP | Gravity.CENTER_HORIZONTAL)
                                  : pos == 2 ? (Gravity.BOTTOM | Gravity.CENTER_HORIZONTAL)
                                  : Gravity.CENTER);
                  if (pos == 0) fp.topMargin = dp(100);
                  if (pos == 2) fp.bottomMargin = dp(100);
                  root.addView(box, fp);

                  // Tap: go to the normal lockscreen. Long-press: AOD off until next unlock.
                  root.setOnClickListener(v -> finish());
                  root.setOnLongClickListener(v -> {
                      AodService.dismissed = true;
                      finish();
                      return true;
                  });
                  setContentView(root);

                  getSystemService(NotificationManager.class).cancel(2);
                  current = this;
                  h.post(tick);

                  int m = Prefs.timeout(this);
                  if (m > 0) {
                      h.postDelayed(() -> {
                          AodService.dismissed = true;
                          finish();
                      }, m * 60000L);
                  }
              }

              @Override
              protected void onNewIntent(Intent intent) {
                  super.onNewIntent(intent);
                  setShowWhenLocked(true);
                  setTurnScreenOn(true);
                  refresh();
              }

              void refresh() {
                  if (box == null) return;
                  Date d = new Date();
                  if (time != null) {
                      Calendar c = Calendar.getInstance();
                      String hh = new SimpleDateFormat(Prefs.h24(this) ? "HH" : "h", Locale.getDefault()).format(d);
                      String mm = new SimpleDateFormat("mm", Locale.getDefault()).format(d);
                      time.setText(style == 2 ? hh + "\n" + mm : hh + ":" + mm);
                  }
                  if (analog != null) analog.invalidate();
                  date.setText(new SimpleDateFormat("EEEE, d MMMM", Locale.getDefault()).format(d));

                  Intent bi = registerReceiver(null, new IntentFilter(Intent.ACTION_BATTERY_CHANGED));
                  if (bi != null) {
                      int lv = bi.getIntExtra(BatteryManager.EXTRA_LEVEL, -1);
                      int sc = bi.getIntExtra(BatteryManager.EXTRA_SCALE, 100);
                      int st = bi.getIntExtra(BatteryManager.EXTRA_STATUS, -1);
                      boolean ch = st == BatteryManager.BATTERY_STATUS_CHARGING
                              || st == BatteryManager.BATTERY_STATUS_FULL;
                      if (lv >= 0 && sc > 0) {
                          batt.setText((lv * 100 / sc) + "%" + (ch ? "  charging" : ""));
                      }
                  }

                  notesBox.removeAllViews();
                  if (nMode == 2) return;
                  int n = 0;
                  for (NotifListener.Item it : NotifListener.items) {
                      if (n >= nMax) break;
                      if (nMode == 1) {
                          if (it.icon == null) continue;
                          try {
                              Drawable dr = it.icon.loadDrawable(this);
                              if (dr == null) continue;
                              ImageView iv = new ImageView(this);
                              iv.setImageDrawable(dr);
                              iv.setColorFilter(0xFFCCCCCC);
                              LinearLayout.LayoutParams lp = new LinearLayout.LayoutParams(dp(28), dp(28));
                              lp.setMargins(dp(6), 0, dp(6), 0);
                              notesBox.addView(iv, lp);
                              n++;
                          } catch (Exception ignored) { }
                      } else {
                          String s = it.app;
                          if (nDetail >= 1 && it.title != null) s += ": " + it.title;
                          if (nDetail >= 2 && it.text != null) {
                              s += " - " + (it.text.length() > 40 ? it.text.substring(0, 40) + "..." : it.text);
                          }
                          TextView t = tv(0xFF8A8A8A, 14, "sans-serif");
                          t.setSingleLine(true);
                          t.setEllipsize(TextUtils.TruncateAt.END);
                          t.setText(s);
                          t.setPadding(0, dp(3), 0, dp(3));
                          notesBox.addView(t);
                          n++;
                      }
                  }
              }

              @Override
              protected void onDestroy() {
                  current = null;
                  h.removeCallbacksAndMessages(null);
                  super.onDestroy();
              }
          }
          EOF
          mkdir -p app/src/main/java/com/lucas/aod
          cat > app/src/main/java/com/lucas/aod/AodService.java <<'EOF'
          package com.lucas.aod;

          import android.app.Notification;
          import android.app.NotificationChannel;
          import android.app.NotificationManager;
          import android.app.PendingIntent;
          import android.app.Service;
          import android.content.BroadcastReceiver;
          import android.content.Context;
          import android.content.Intent;
          import android.content.IntentFilter;
          import android.content.pm.ServiceInfo;
          import android.os.Build;
          import android.os.Handler;
          import android.os.IBinder;
          import android.os.Looper;

          public class AodService extends Service {
              static volatile boolean dismissed = false;
              static volatile boolean running = false;

              private final BroadcastReceiver rx = new BroadcastReceiver() {
                  @Override
                  public void onReceive(Context c, Intent i) {
                      String a = i.getAction();
                      if (Intent.ACTION_USER_PRESENT.equals(a)) {
                          dismissed = false;
                      } else if (Intent.ACTION_SCREEN_OFF.equals(a)) {
                          if (Prefs.enabled(c) && !dismissed) launch(c);
                      }
                  }
              };

              static void launch(final Context c) {
                  final Intent i = new Intent(c, AodActivity.class);
                  i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                          | Intent.FLAG_ACTIVITY_CLEAR_TOP
                          | Intent.FLAG_ACTIVITY_NO_ANIMATION);
                  try {
                      c.startActivity(i);
                  } catch (Exception ignored) { }
                  // Fallback: if the direct start was blocked, use a full-screen notification.
                  new Handler(Looper.getMainLooper()).postDelayed(new Runnable() {
                      @Override
                      public void run() {
                          if (AodActivity.current == null && Prefs.enabled(c) && !dismissed) {
                              try {
                                  PendingIntent pi = PendingIntent.getActivity(c, 0, i,
                                          PendingIntent.FLAG_IMMUTABLE | PendingIntent.FLAG_UPDATE_CURRENT);
                                  Notification n = new Notification.Builder(c, "aod_launch")
                                          .setSmallIcon(android.R.drawable.ic_lock_idle_lock)
                                          .setContentTitle("Always On Display")
                                          .setCategory(Notification.CATEGORY_ALARM)
                                          .setFullScreenIntent(pi, true)
                                          .setAutoCancel(true)
                                          .build();
                                  c.getSystemService(NotificationManager.class).notify(2, n);
                              } catch (Exception ignored) { }
                          }
                      }
                  }, 900);
              }

              @Override
              public void onCreate() {
                  super.onCreate();
                  NotificationManager nm = getSystemService(NotificationManager.class);
                  nm.createNotificationChannel(new NotificationChannel(
                          "aod_svc", "AOD service", NotificationManager.IMPORTANCE_MIN));
                  nm.createNotificationChannel(new NotificationChannel(
                          "aod_launch", "AOD launch", NotificationManager.IMPORTANCE_HIGH));
                  Notification n = new Notification.Builder(this, "aod_svc")
                          .setSmallIcon(android.R.drawable.ic_lock_idle_lock)
                          .setContentTitle("Always On Display active")
                          .setOngoing(true)
                          .build();
                  if (Build.VERSION.SDK_INT >= 34) {
                      startForeground(1, n, ServiceInfo.FOREGROUND_SERVICE_TYPE_SPECIAL_USE);
                  } else {
                      startForeground(1, n);
                  }
                  IntentFilter f = new IntentFilter();
                  f.addAction(Intent.ACTION_SCREEN_OFF);
                  f.addAction(Intent.ACTION_USER_PRESENT);
                  if (Build.VERSION.SDK_INT >= 33) {
                      registerReceiver(rx, f, Context.RECEIVER_NOT_EXPORTED);
                  } else {
                      registerReceiver(rx, f);
                  }
                  running = true;
              }

              @Override
              public int onStartCommand(Intent intent, int flags, int startId) {
                  return START_STICKY;
              }

              @Override
              public void onDestroy() {
                  running = false;
                  try { unregisterReceiver(rx); } catch (Exception ignored) { }
                  super.onDestroy();
              }

              @Override
              public IBinder onBind(Intent intent) { return null; }
          }
          EOF
          mkdir -p app/src/main/java/com/lucas/aod
          cat > app/src/main/java/com/lucas/aod/BootReceiver.java <<'EOF'
          package com.lucas.aod;

          import android.content.BroadcastReceiver;
          import android.content.Context;
          import android.content.Intent;

          public class BootReceiver extends BroadcastReceiver {
              @Override
              public void onReceive(Context c, Intent i) {
                  if (Prefs.enabled(c)) {
                      try {
                          c.startForegroundService(new Intent(c, AodService.class));
                      } catch (Exception ignored) { }
                  }
              }
          }
          EOF
          mkdir -p app/src/main/java/com/lucas/aod
          cat > app/src/main/java/com/lucas/aod/MainActivity.java <<'EOF'
          package com.lucas.aod;

          import android.app.Activity;
          import android.content.Intent;
          import android.content.pm.PackageManager;
          import android.graphics.Color;
          import android.net.Uri;
          import android.os.Build;
          import android.os.Bundle;
          import android.provider.Settings;
          import android.view.View;
          import android.widget.AdapterView;
          import android.widget.ArrayAdapter;
          import android.widget.Button;
          import android.widget.CompoundButton;
          import android.widget.LinearLayout;
          import android.widget.ScrollView;
          import android.widget.Spinner;
          import android.widget.Switch;
          import android.widget.TextView;
          import android.widget.Toast;

          public class MainActivity extends Activity {
              private TextView status;

              private int dp(int v) {
                  return (int) (v * getResources().getDisplayMetrics().density);
              }

              private TextView label(String s, int sp, int color) {
                  TextView t = new TextView(this);
                  t.setText(s);
                  t.setTextSize(sp);
                  t.setTextColor(color);
                  t.setPadding(0, dp(12), 0, dp(4));
                  return t;
              }

              private Button btn(String s, View.OnClickListener c) {
                  Button b = new Button(this);
                  b.setText(s);
                  b.setAllCaps(false);
                  b.setOnClickListener(c);
                  return b;
              }

              private Switch sw(String s, boolean on, CompoundButton.OnCheckedChangeListener c) {
                  Switch x = new Switch(this);
                  x.setText(s);
                  x.setChecked(on);
                  x.setPadding(0, dp(8), 0, dp(8));
                  x.setOnCheckedChangeListener(c);
                  return x;
              }

              private void spin(LinearLayout l, String title, String[] names, final int[] vals,
                                final String key, int def) {
                  l.addView(label(title, 14, 0xFFAAAAAA));
                  Spinner sp = new Spinner(this);
                  sp.setAdapter(new ArrayAdapter<>(this,
                          android.R.layout.simple_spinner_dropdown_item, names));
                  int cur = Prefs.geti(this, key, def);
                  for (int k = 0; k < vals.length; k++) if (vals[k] == cur) sp.setSelection(k);
                  sp.setOnItemSelectedListener(new AdapterView.OnItemSelectedListener() {
                      @Override
                      public void onItemSelected(AdapterView<?> parent, View v, int pos, long id) {
                          Prefs.sp(MainActivity.this).edit().putInt(key, vals[pos]).apply();
                      }
                      @Override
                      public void onNothingSelected(AdapterView<?> parent) { }
                  });
                  l.addView(sp);
              }

              private void open(Intent i) {
                  try {
                      startActivity(i);
                  } catch (Exception e) {
                      Toast.makeText(this, "Can't open that setting", Toast.LENGTH_SHORT).show();
                  }
              }

              private void syncService() {
                  Intent i = new Intent(this, AodService.class);
                  if (Prefs.enabled(this)) {
                      startForegroundService(i);
                  } else {
                      stopService(i);
                  }
              }

              @Override
              protected void onCreate(Bundle b) {
                  super.onCreate(b);
                  if (Build.VERSION.SDK_INT >= 33
                          && checkSelfPermission(android.Manifest.permission.POST_NOTIFICATIONS)
                          != PackageManager.PERMISSION_GRANTED) {
                      requestPermissions(new String[]{android.Manifest.permission.POST_NOTIFICATIONS}, 1);
                  }

                  ScrollView sv = new ScrollView(this);
                  LinearLayout l = new LinearLayout(this);
                  l.setOrientation(LinearLayout.VERTICAL);
                  int p = dp(20);
                  l.setPadding(p, p, p, p);
                  sv.addView(l);

                  l.addView(label("Always On Display", 26, Color.WHITE));
                  status = label("", 14, 0xFFAAAAAA);
                  l.addView(status);

                  l.addView(sw("Enable always-on display", Prefs.enabled(this), (v, on) -> {
                      Prefs.sp(this).edit().putBoolean("enabled", on).apply();
                      syncService();
                  }));

                  l.addView(label("CLOCK", 13, 0xFF69F0AE));
                  spin(l, "Clock style",
                          new String[]{"Thin", "Bold", "Stacked", "Analog", "Mono"},
                          new int[]{0, 1, 2, 3, 4}, "style", 0);
                  spin(l, "Clock size",
                          new String[]{"Small", "Medium", "Large", "Huge"},
                          new int[]{70, 100, 130, 160}, "size", 100);
                  spin(l, "Position on screen",
                          new String[]{"Top", "Center", "Bottom"},
                          new int[]{0, 1, 2}, "pos", 1);
                  l.addView(sw("24-hour clock", Prefs.h24(this), (v, on) ->
                          Prefs.sp(this).edit().putBoolean("h24", on).apply()));
                  l.addView(sw("Show date", Prefs.showDate(this), (v, on) ->
                          Prefs.sp(this).edit().putBoolean("showDate", on).apply()));
                  l.addView(sw("Show battery", Prefs.showBatt(this), (v, on) ->
                          Prefs.sp(this).edit().putBoolean("showBatt", on).apply()));

                  l.addView(label("Color", 14, 0xFFAAAAAA));
                  LinearLayout row = new LinearLayout(this);
                  int[] cols = {0xFFFFFFFF, 0xFF00E5FF, 0xFF69F0AE, 0xFFFFC400,
                          0xFFFF4081, 0xFFB388FF, 0xFFFF6E40, 0xFF40C4FF};
                  for (final int c : cols) {
                      Button x = new Button(this);
                      x.setBackgroundColor(c);
                      x.setOnClickListener(v -> {
                          Prefs.sp(this).edit().putInt("color", c).apply();
                          Toast.makeText(this, "Color saved", Toast.LENGTH_SHORT).show();
                      });
                      LinearLayout.LayoutParams lp = new LinearLayout.LayoutParams(0, dp(40), 1f);
                      lp.setMargins(dp(2), dp(4), dp(2), dp(4));
                      row.addView(x, lp);
                  }
                  l.addView(row);

                  l.addView(label("NOTIFICATIONS", 13, 0xFF69F0AE));
                  spin(l, "Show as",
                          new String[]{"Text list", "Icons only", "Off"},
                          new int[]{0, 1, 2}, "nMode", 0);
                  spin(l, "How many",
                          new String[]{"1", "3", "5", "8"},
                          new int[]{1, 3, 5, 8}, "nMax", 5);
                  spin(l, "Text list shows",
                          new String[]{"App name only", "App + title", "App + title + message"},
                          new int[]{0, 1, 2}, "nDetail", 1);

                  l.addView(label("TIMER", 13, 0xFF69F0AE));
                  spin(l, "Stay on for",
                          new String[]{"5 min", "10 min", "30 min", "1 hour", "Until I stop it"},
                          new int[]{5, 10, 30, 60, 0}, "timeout", 10);

                  l.addView(label("SETUP (do all 4 once)", 13, 0xFF69F0AE));
                  l.addView(btn("1. Allow display over other apps", v -> open(new Intent(
                          Settings.ACTION_MANAGE_OVERLAY_PERMISSION,
                          Uri.parse("package:" + getPackageName())))));
                  l.addView(btn("2. Allow full-screen alerts", v -> {
                      if (Build.VERSION.SDK_INT >= 34) {
                          open(new Intent(Settings.ACTION_MANAGE_APP_USE_FULL_SCREEN_INTENT,
                                  Uri.parse("package:" + getPackageName())));
                      } else {
                          Toast.makeText(this, "Not needed on this Android version",
                                  Toast.LENGTH_SHORT).show();
                      }
                  }));
                  l.addView(btn("3. Allow notification access", v ->
                          open(new Intent(Settings.ACTION_NOTIFICATION_LISTENER_SETTINGS))));
                  l.addView(btn("4. Battery: set to Unrestricted", v -> open(new Intent(
                          Settings.ACTION_APPLICATION_DETAILS_SETTINGS,
                          Uri.parse("package:" + getPackageName())))));

                  l.addView(label("PREVIEW", 13, 0xFF69F0AE));
                  l.addView(btn("Show AOD now", v ->
                          startActivity(new Intent(this, AodActivity.class))));

                  l.addView(label("On the AOD: tap = go to lockscreen. Long-press = AOD off "
                          + "until you unlock next.", 13, 0xFF888888));

                  setContentView(sv);
                  if (Prefs.enabled(this)) syncService();
              }

              @Override
              protected void onResume() {
                  super.onResume();
                  boolean ov = Settings.canDrawOverlays(this);
                  String ln = Settings.Secure.getString(getContentResolver(),
                          "enabled_notification_listeners");
                  boolean na = ln != null && ln.contains(getPackageName());
                  status.setText((ov ? "[x]" : "[ ]") + " Display over other apps\n"
                          + (na ? "[x]" : "[ ]") + " Notification access\n"
                          + (AodService.running ? "[x]" : "[ ]") + " Service running");
              }
          }
          EOF
          mkdir -p app/src/main/java/com/lucas/aod
          cat > app/src/main/java/com/lucas/aod/NotifListener.java <<'EOF'
          package com.lucas.aod;

          import android.app.Notification;
          import android.content.pm.PackageManager;
          import android.graphics.drawable.Icon;
          import android.service.notification.NotificationListenerService;
          import android.service.notification.StatusBarNotification;

          import java.util.ArrayList;
          import java.util.List;
          import java.util.concurrent.CopyOnWriteArrayList;

          public class NotifListener extends NotificationListenerService {
              static class Item {
                  String app, title, text;
                  Icon icon;
              }

              static final List<Item> items = new CopyOnWriteArrayList<>();

              @Override
              public void onListenerConnected() { rebuild(); }

              @Override
              public void onNotificationPosted(StatusBarNotification s) { rebuild(); }

              @Override
              public void onNotificationRemoved(StatusBarNotification s) { rebuild(); }

              private void rebuild() {
                  List<Item> out = new ArrayList<>();
                  try {
                      StatusBarNotification[] all = getActiveNotifications();
                      PackageManager pm = getPackageManager();
                      for (StatusBarNotification s : all) {
                          Notification n = s.getNotification();
                          if (s.isOngoing() || s.getPackageName().equals(getPackageName())) continue;
                          if ((n.flags & Notification.FLAG_GROUP_SUMMARY) != 0) continue;
                          Item it = new Item();
                          try {
                              it.app = pm.getApplicationLabel(pm.getApplicationInfo(s.getPackageName(), 0)).toString();
                          } catch (Exception e) {
                              it.app = s.getPackageName();
                          }
                          CharSequence t = n.extras.getCharSequence(Notification.EXTRA_TITLE);
                          CharSequence x = n.extras.getCharSequence(Notification.EXTRA_TEXT);
                          it.title = t != null ? t.toString() : null;
                          it.text = x != null ? x.toString() : null;
                          it.icon = n.getSmallIcon();
                          out.add(it);
                          if (out.size() >= 8) break;
                      }
                  } catch (Exception ignored) { }
                  items.clear();
                  items.addAll(out);
                  final AodActivity cur = AodActivity.current;
                  if (cur != null) cur.runOnUiThread(cur::refresh);
              }
          }
          EOF
          mkdir -p app/src/main/java/com/lucas/aod
          cat > app/src/main/java/com/lucas/aod/Prefs.java <<'EOF'
          package com.lucas.aod;

          import android.content.Context;
          import android.content.SharedPreferences;

          final class Prefs {
              static SharedPreferences sp(Context c) {
                  return c.getSharedPreferences("aod", Context.MODE_PRIVATE);
              }
              static boolean enabled(Context c) { return sp(c).getBoolean("enabled", false); }
              static boolean h24(Context c) { return sp(c).getBoolean("h24", false); }
              static boolean showDate(Context c) { return sp(c).getBoolean("showDate", true); }
              static boolean showBatt(Context c) { return sp(c).getBoolean("showBatt", true); }
              static int color(Context c) { return sp(c).getInt("color", 0xFFFFFFFF); }
              static int timeout(Context c) { return sp(c).getInt("timeout", 10); }
              static int geti(Context c, String k, int d) { return sp(c).getInt(k, d); }
          }
          EOF
          cat > build.gradle <<'EOF'
          plugins {
              id 'com.android.application' version '8.5.2' apply false
          }
          EOF
          cat > gradle.properties <<'EOF'
          org.gradle.jvmargs=-Xmx2g
          android.nonTransitiveRClass=true
          EOF
          cat > settings.gradle <<'EOF'
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "AlwaysOnDisplay"
          include ':app'
          EOF

      - name: Build APK
        run: gradle assembleDebug --no-daemon

      - name: Publish APK to Releases
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          cp app/build/outputs/apk/debug/app-debug.apk AlwaysOnDisplay.apk
          gh release delete latest --yes --cleanup-tag || true

          gh release create latest AlwaysOnDisplay.apk --title "Latest build" --notes "Download AlwaysOnDisplay.apk and install it."# Always-on-display
