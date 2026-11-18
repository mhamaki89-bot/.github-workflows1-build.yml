name: Build Control Center APK

on:
  push:
    branches: [main, master]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Generate project files
        run: |
          mkdir -p app
          cat > app/build.gradle.kts <<'EOF_FILE'
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
          }

          android {
              namespace = "com.mohamed.cc"
              compileSdk = 34
              defaultConfig {
                  applicationId = "com.mohamed.cc"
                  minSdk = 26
                  targetSdk = 33
                  versionCode = 1
                  versionName = "1.0"
              }
              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }
              kotlinOptions { jvmTarget = "17" }
          }
          EOF_FILE
          mkdir -p app/src/main
          cat > app/src/main/AndroidManifest.xml <<'EOF_FILE'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
              <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
              <uses-permission android:name="android.permission.WRITE_SETTINGS" />

              <application
                  android:label="@string/app_name"
                  android:theme="@android:style/Theme.DeviceDefault.NoActionBar">

                  <activity android:name=".MainActivity" android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <service
                      android:name=".ControlService"
                      android:exported="false"
                      android:label="@string/app_name"
                      android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">
                      <intent-filter>
                          <action android:name="android.accessibilityservice.AccessibilityService" />
                      </intent-filter>
                      <meta-data
                          android:name="android.accessibilityservice"
                          android:resource="@xml/accessibility_config" />
                  </service>
              </application>
          </manifest>
          EOF_FILE
          mkdir -p app/src/main/java/com/mohamed/cc
          cat > app/src/main/java/com/mohamed/cc/ControlService.kt <<'EOF_FILE'
          package com.mohamed.cc

          import android.accessibilityservice.AccessibilityService
          import android.bluetooth.BluetoothManager
          import android.content.Context
          import android.content.Intent
          import android.graphics.Canvas
          import android.graphics.Color
          import android.graphics.Paint
          import android.graphics.Path
          import android.graphics.PixelFormat
          import android.graphics.RectF
          import android.graphics.drawable.GradientDrawable
          import android.hardware.camera2.CameraCharacteristics
          import android.hardware.camera2.CameraManager
          import android.media.AudioManager
          import android.net.ConnectivityManager
          import android.net.NetworkCapabilities
          import android.net.Uri
          import android.net.wifi.WifiManager
          import android.os.Build
          import android.provider.AlarmClock
          import android.provider.MediaStore
          import android.provider.Settings
          import android.view.Gravity
          import android.view.MotionEvent
          import android.view.View
          import android.view.WindowManager
          import android.view.accessibility.AccessibilityEvent
          import android.widget.FrameLayout
          import android.widget.LinearLayout
          import android.widget.TextView

          // ===== الألوان (عدّلها من هنا) =====
          object Colors {
              val DIM: Int = Color.parseColor("#99000000")
              val PANEL: Int = Color.parseColor("#EE2A0010")
              val CARD: Int = Color.parseColor("#CC881144")
              val ACTIVE: Int = Color.parseColor("#C2185B")
              val OFF: Int = Color.parseColor("#55FFFFFF")
              val ACCENT: Int = Color.parseColor("#F48FB1")
              val CREAM: Int = Color.parseColor("#FFE4D8")
          }

          class SliderView(
              ctx: Context,
              private val glyph: String,
              start: Float,
              private val onChange: (Float) -> Unit
          ) : View(ctx) {
              private var value = start.coerceIn(0f, 1f)
              private val bg = Paint(Paint.ANTI_ALIAS_FLAG).apply { color = Color.parseColor("#66FFFFFF") }
              private val fill = Paint(Paint.ANTI_ALIAS_FLAG).apply { color = Colors.CREAM }
              private val txt = Paint(Paint.ANTI_ALIAS_FLAG).apply { textAlign = Paint.Align.CENTER }
              private val path = Path()

              override fun onDraw(c: Canvas) {
                  val w = width.toFloat()
                  val h = height.toFloat()
                  val r = w / 2.2f
                  path.reset()
                  path.addRoundRect(RectF(0f, 0f, w, h), r, r, Path.Direction.CW)
                  c.save()
                  c.clipPath(path)
                  c.drawRect(0f, 0f, w, h, bg)
                  c.drawRect(0f, h * (1f - value), w, h, fill)
                  c.restore()
                  txt.textSize = w * 0.42f
                  c.drawText(glyph, w / 2f, h - w * 0.36f, txt)
              }

              override fun onTouchEvent(e: MotionEvent): Boolean {
                  parent?.requestDisallowInterceptTouchEvent(true)
                  if (e.actionMasked == MotionEvent.ACTION_DOWN || e.actionMasked == MotionEvent.ACTION_MOVE) {
                      value = (1f - e.y / height).coerceIn(0f, 1f)
                      invalidate()
                      onChange(value)
                  }
                  return true
              }
          }

          class ControlService : AccessibilityService() {
              private lateinit var wm: WindowManager
              private var handle: View? = null
              private var panel: View? = null
              private var shownAt = 0L
              private var torchOn = false
              private var torchId: String? = null
              private val torchCb = object : CameraManager.TorchCallback() {
                  override fun onTorchModeChanged(cameraId: String, enabled: Boolean) {
                      if (cameraId == torchId) torchOn = enabled
                  }
              }

              private fun dp(v: Int) = (v * resources.displayMetrics.density).toInt()

              private fun rr(color: Int, r: Float): GradientDrawable {
                  val g = GradientDrawable()
                  g.setColor(color)
                  g.cornerRadius = r
                  return g
              }

              override fun onServiceConnected() {
                  wm = getSystemService(Context.WINDOW_SERVICE) as WindowManager
                  try {
                      val cm = getSystemService(Context.CAMERA_SERVICE) as CameraManager
                      torchId = cm.cameraIdList.firstOrNull {
                          cm.getCameraCharacteristics(it).get(CameraCharacteristics.FLASH_INFO_AVAILABLE) == true
                      }
                      cm.registerTorchCallback(torchCb, null)
                  } catch (e: Exception) {
                  }
                  addHandle()
              }

              override fun onAccessibilityEvent(e: AccessibilityEvent) {
                  if (panel != null &&
                      e.eventType == AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED &&
                      e.packageName?.toString() != packageName &&
                      System.currentTimeMillis() - shownAt > 700
                  ) hidePanel()
              }

              override fun onInterrupt() {}

              override fun onDestroy() {
                  hidePanel()
                  handle?.let { try { wm.removeView(it) } catch (e: Exception) {} }
                  handle = null
                  try {
                      (getSystemService(Context.CAMERA_SERVICE) as CameraManager).unregisterTorchCallback(torchCb)
                  } catch (e: Exception) {
                  }
                  super.onDestroy()
              }

              // ===== منطقة السحب (أعلى يمين) =====
              private fun addHandle() {
                  val v = View(this)
                  var startY = 0f
                  var fired = false
                  v.setOnTouchListener { _, e ->
                      when (e.actionMasked) {
                          MotionEvent.ACTION_DOWN -> {
                              startY = e.rawY
                              fired = false
                          }
                          MotionEvent.ACTION_MOVE -> if (!fired && e.rawY - startY > dp(40)) {
                              fired = true
                              showPanel()
                          }
                      }
                      true
                  }
                  val lp = WindowManager.LayoutParams(
                      dp(110), dp(26),
                      WindowManager.LayoutParams.TYPE_ACCESSIBILITY_OVERLAY,
                      WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                          WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL or
                          WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN or
                          WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS,
                      PixelFormat.TRANSLUCENT
                  )
                  lp.gravity = Gravity.TOP or Gravity.END
                  wm.addView(v, lp)
                  handle = v
              }

              private fun launch(i: Intent) {
                  try {
                      i.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                      startActivity(i)
                  } catch (e: Exception) {
                  }
                  hidePanel()
              }

              private fun hidePanel() {
                  panel?.let { try { wm.removeView(it) } catch (e: Exception) {} }
                  panel = null
              }

              // ===== حالات الأزرار =====
              private fun wifiOn() = try {
                  (applicationContext.getSystemService(Context.WIFI_SERVICE) as WifiManager).isWifiEnabled
              } catch (e: Exception) { false }

              private fun btOn() = try {
                  (getSystemService(Context.BLUETOOTH_SERVICE) as BluetoothManager).adapter?.isEnabled == true
              } catch (e: Exception) { false }

              private fun airplaneOn() =
                  Settings.Global.getInt(contentResolver, Settings.Global.AIRPLANE_MODE_ON, 0) == 1

              private fun dataOn() = try {
                  val cm = getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
                  cm.getNetworkCapabilities(cm.activeNetwork)?.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR) == true
              } catch (e: Exception) { false }

              private fun panelIntent(action: String, fallback: String) =
                  Intent(if (Build.VERSION.SDK_INT >= 29) action else fallback)

              // ===== عناصر الواجهة =====
              private fun button(glyph: String, active: Boolean, radiusDp: Int, onClick: (TextView) -> Unit): TextView {
                  val t = TextView(this)
                  t.text = glyph
                  t.textSize = 22f
                  t.gravity = Gravity.CENTER
                  t.setTextColor(Colors.CREAM)
                  t.background = rr(if (active) Colors.ACTIVE else Colors.OFF, dp(radiusDp).toFloat())
                  t.setOnClickListener { onClick(t) }
                  return t
              }

              private fun lp(w: Int, h: Int, left: Int = 0, top: Int = 0): LinearLayout.LayoutParams {
                  val p = LinearLayout.LayoutParams(w, h)
                  p.leftMargin = left
                  p.topMargin = top
                  return p
              }

              private fun connectivityCard(): LinearLayout {
                  val card = LinearLayout(this)
                  card.orientation = LinearLayout.VERTICAL
                  card.gravity = Gravity.CENTER
                  card.background = rr(Colors.CARD, dp(26).toFloat())
                  card.setPadding(dp(6), dp(6), dp(6), dp(6))

                  val s = dp(54)
                  val m = dp(4)
                  val wifi = button("📶", wifiOn(), 40) {
                      launch(panelIntent("android.settings.panel.action.WIFI", Settings.ACTION_WIFI_SETTINGS))
                  }
                  val data = button("📡", dataOn(), 40) {
                      launch(panelIntent("android.settings.panel.action.INTERNET_CONNECTIVITY", Settings.ACTION_DATA_ROAMING_SETTINGS))
                  }
                  val bt = button("🔵", btOn(), 40) {
                      launch(Intent(Settings.ACTION_BLUETOOTH_SETTINGS))
                  }
                  val air = button("✈️", airplaneOn(), 40) {
                      launch(Intent(Settings.ACTION_AIRPLANE_MODE_SETTINGS))
                  }

                  val r1 = LinearLayout(this)
                  r1.addView(wifi, lp(s, s, m, m))
                  r1.addView(data, lp(s, s, m, m))
                  val r2 = LinearLayout(this)
                  r2.addView(bt, lp(s, s, m, m))
                  r2.addView(air, lp(s, s, m, m))
                  card.addView(r1)
                  card.addView(r2)
                  return card
              }

              private fun showPanel() {
                  if (panel != null) return
                  shownAt = System.currentTimeMillis()

                  val root = FrameLayout(this)
                  root.setBackgroundColor(Colors.DIM)
                  root.setOnClickListener { hidePanel() }

                  val col = LinearLayout(this)
                  col.orientation = LinearLayout.VERTICAL
                  col.setPadding(dp(16), dp(16), dp(16), dp(16))
                  col.background = rr(Colors.PANEL, dp(34).toFloat())
                  col.isClickable = true

                  // الصف الأول: الاتصالات + سطوع + صوت
                  val audio = getSystemService(Context.AUDIO_SERVICE) as AudioManager
                  val maxVol = audio.getStreamMaxVolume(AudioManager.STREAM_MUSIC).coerceAtLeast(1)
                  val curVol = audio.getStreamVolume(AudioManager.STREAM_MUSIC) / maxVol.toFloat()
                  val curBr = Settings.System.getInt(contentResolver, Settings.System.SCREEN_BRIGHTNESS, 128) / 255f

                  val brightness = SliderView(this, "☀️", curBr) { v ->
                      if (!Settings.System.canWrite(this)) {
                          launch(Intent(Settings.ACTION_MANAGE_WRITE_SETTINGS, Uri.parse("package:$packageName")))
                      } else {
                          Settings.System.putInt(
                              contentResolver, Settings.System.SCREEN_BRIGHTNESS_MODE,
                              Settings.System.SCREEN_BRIGHTNESS_MODE_MANUAL
                          )
                          Settings.System.putInt(contentResolver, Settings.System.SCREEN_BRIGHTNESS, (v * 254f).toInt() + 1)
                      }
                  }
                  val volume = SliderView(this, "🔊", curVol) { v ->
                      audio.setStreamVolume(AudioManager.STREAM_MUSIC, Math.round(v * maxVol), 0)
                  }

                  val row1 = LinearLayout(this)
                  row1.addView(connectivityCard(), lp(dp(140), dp(150)))
                  row1.addView(brightness, lp(dp(66), dp(150), dp(12)))
                  row1.addView(volume, lp(dp(66), dp(150), dp(12)))
                  col.addView(row1)

                  // الصف التاني: اختصارات
                  val row2 = LinearLayout(this)
                  val sc = dp(64)
                  val flash = button("🔦", torchOn, 20) { t ->
                      val id = torchId
                      if (id != null) {
                          try {
                              val cm = getSystemService(Context.CAMERA_SERVICE) as CameraManager
                              torchOn = !torchOn
                              cm.setTorchMode(id, torchOn)
                              t.background = rr(if (torchOn) Colors.ACTIVE else Colors.OFF, dp(20).toFloat())
                          } catch (e: Exception) {
                          }
                      }
                  }
                  val cam = button("📷", false, 20) { launch(Intent(MediaStore.INTENT_ACTION_STILL_IMAGE_CAMERA)) }
                  val calc = button("🧮", false, 20) {
                      launch(Intent.makeMainSelectorActivity(Intent.ACTION_MAIN, Intent.CATEGORY_APP_CALCULATOR))
                  }
                  val alarm = button("⏰", false, 20) { launch(Intent(AlarmClock.ACTION_SHOW_ALARMS)) }
                  row2.addView(flash, lp(0, sc).also { it.weight = 1f })
                  row2.addView(cam, lp(0, sc, dp(10)).also { it.weight = 1f })
                  row2.addView(calc, lp(0, sc, dp(10)).also { it.weight = 1f })
                  row2.addView(alarm, lp(0, sc, dp(10)).also { it.weight = 1f })
                  col.addView(row2, lp(LinearLayout.LayoutParams.MATCH_PARENT, sc, 0, dp(12)))

                  val fl = FrameLayout.LayoutParams(dp(330), FrameLayout.LayoutParams.WRAP_CONTENT, Gravity.TOP or Gravity.CENTER_HORIZONTAL)
                  fl.topMargin = dp(48)
                  root.addView(col, fl)

                  val params = WindowManager.LayoutParams(
                      WindowManager.LayoutParams.MATCH_PARENT,
                      WindowManager.LayoutParams.MATCH_PARENT,
                      WindowManager.LayoutParams.TYPE_ACCESSIBILITY_OVERLAY,
                      WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE or
                          WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN or
                          WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS,
                      PixelFormat.TRANSLUCENT
                  )
                  wm.addView(root, params)
                  panel = root
              }
          }
          EOF_FILE
          mkdir -p app/src/main/java/com/mohamed/cc
          cat > app/src/main/java/com/mohamed/cc/MainActivity.kt <<'EOF_FILE'
          package com.mohamed.cc

          import android.app.Activity
          import android.content.Intent
          import android.graphics.Color
          import android.graphics.drawable.GradientDrawable
          import android.net.Uri
          import android.os.Bundle
          import android.provider.Settings
          import android.view.Gravity
          import android.view.View
          import android.widget.Button
          import android.widget.LinearLayout
          import android.widget.ScrollView
          import android.widget.TextView

          class MainActivity : Activity() {
              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  val d = resources.displayMetrics.density
                  fun dp(v: Int) = (v * d).toInt()

                  val col = LinearLayout(this)
                  col.orientation = LinearLayout.VERTICAL
                  col.setPadding(dp(24), dp(48), dp(24), dp(24))
                  col.setBackgroundColor(Color.parseColor("#2A0010"))
                  col.layoutDirection = View.LAYOUT_DIRECTION_RTL

                  fun text(s: String, size: Float, color: String, bottom: Int) {
                      val t = TextView(this)
                      t.text = s
                      t.textSize = size
                      t.setTextColor(Color.parseColor(color))
                      t.textDirection = View.TEXT_DIRECTION_RTL
                      t.gravity = Gravity.START
                      t.setPadding(0, 0, 0, dp(bottom))
                      t.setLineSpacing(0f, 1.25f)
                      col.addView(t)
                  }

                  fun button(s: String, onClick: () -> Unit) {
                      val b = Button(this)
                      b.text = s
                      b.setTextColor(Color.parseColor("#FFE4D8"))
                      val bg = GradientDrawable()
                      bg.setColor(Color.parseColor("#C2185B"))
                      bg.cornerRadius = dp(18).toFloat()
                      b.background = bg
                      b.setOnClickListener { onClick() }
                      val lp = LinearLayout.LayoutParams(LinearLayout.LayoutParams.MATCH_PARENT, dp(56))
                      lp.bottomMargin = dp(12)
                      col.addView(b, lp)
                  }

                  text("Control Center", 28f, "#FFE4D8", 8)
                  text("اتبع الخطوات دي بالترتيب:", 16f, "#F48FB1", 16)

                  text("1) فعّل الخدمة: دوس الزرار، ودوّر على Control Center في (التطبيقات المثبتة / Installed apps) وشغّله.", 15f, "#FFE4D8", 8)
                  button("تفعيل الخدمة") {
                      startActivity(Intent(Settings.ACTION_ACCESSIBILITY_SETTINGS))
                  }
                  text("لو ظهرت رسالة \"إعداد مقيّد\": روح Settings ← Apps ← Control Center ← النقط التلاتة فوق ← Allow restricted settings، وارجع فعّل الخدمة.", 13f, "#CCFFE4D8", 20)

                  text("2) صلاحية السطوع: عشان سلايدر السطوع يشتغل.", 15f, "#FFE4D8", 8)
                  button("صلاحية السطوع") {
                      startActivity(
                          Intent(Settings.ACTION_MANAGE_WRITE_SETTINGS, Uri.parse("package:$packageName"))
                      )
                  }

                  text("3) الاستخدام: اسحب لتحت من أعلى الشاشة ناحية اليمين (المنطقة الصغيرة جنب الساعة والبطارية). الضغط برّه اللوحة بيقفلها.", 15f, "#FFE4D8", 8)

                  val scroll = ScrollView(this)
                  scroll.addView(col)
                  setContentView(scroll)
              }
          }
          EOF_FILE
          mkdir -p app/src/main/res/values
          cat > app/src/main/res/values/strings.xml <<'EOF_FILE'
          <?xml version="1.0" encoding="utf-8"?>
          <resources>
              <string name="app_name">Control Center</string>
              <string name="service_desc">يعرض Control Center بتاعك فوق أي تطبيق عند السحب لتحت من أعلى يمين الشاشة.</string>
          </resources>
          EOF_FILE
          mkdir -p app/src/main/res/xml
          cat > app/src/main/res/xml/accessibility_config.xml <<'EOF_FILE'
          <?xml version="1.0" encoding="utf-8"?>
          <accessibility-service xmlns:android="http://schemas.android.com/apk/res/android"
              android:accessibilityEventTypes="typeWindowStateChanged"
              android:accessibilityFeedbackType="feedbackGeneric"
              android:notificationTimeout="100"
              android:canRetrieveWindowContent="false"
              android:description="@string/service_desc" />
          EOF_FILE
          cat > build.gradle.kts <<'EOF_FILE'
          plugins {
              id("com.android.application") version "8.5.2" apply false
              id("org.jetbrains.kotlin.android") version "1.9.24" apply false
          }
          EOF_FILE
          cat > gradle.properties <<'EOF_FILE'
          org.gradle.jvmargs=-Xmx2g
          android.useAndroidX=true
          EOF_FILE
          cat > settings.gradle.kts <<'EOF_FILE'
          pluginManagement {
              repositories { google(); mavenCentral(); gradlePluginPortal() }
          }
          dependencyResolutionManagement {
              repositories { google(); mavenCentral() }
          }
          rootProject.name = "ControlCenter"
          include(":app")
          EOF_FILE

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: 17

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: "8.7"

      - name: Build APK
        run: gradle assembleDebug --no-daemon

      - uses: actions/upload-artifact@v4
        with:
          name: ControlCenter-APK
          path: app/build/outputs/apk/debug/app-debug.apk
