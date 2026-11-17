# Jarvis2.2
from pathlib import Path
import zipfile, textwrap, shutil

root = Path("/mnt/data/JARVIS_Gelistirilmis")
if root.exists():
    shutil.rmtree(root)
files = {
"settings.gradle.kts": """
pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories { google(); mavenCentral() }
}
rootProject.name = "Jarvis"
include(":app")
""",
"build.gradle.kts": """
plugins {
    id("com.android.application") version "8.7.3" apply false
    id("org.jetbrains.kotlin.android") version "2.0.21" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.0.21" apply false
}
""",
"gradle.properties": """
org.gradle.jvmargs=-Xmx2g -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
""",
"app/build.gradle.kts": """
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("org.jetbrains.kotlin.plugin.compose")
}
android {
    namespace = "com.example.jarvis"
    compileSdk = 35
    defaultConfig {
        applicationId = "com.example.jarvis"
        minSdk = 26
        targetSdk = 35
        versionCode = 2
        versionName = "2.0"
    }
}
dependencies {
    implementation(platform("androidx.compose:compose-bom:2024.12.01"))
    implementation("androidx.activity:activity-compose:1.10.0")
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.foundation:foundation")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")
    debugImplementation("androidx.compose.ui:ui-tooling")
}
""",
"app/src/main/AndroidManifest.xml": """
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission android:name="android.permission.INTERNET" />
    <application
        android:allowBackup="false"
        android:label="JARVIS"
        android:usesCleartextTraffic="false"
        android:theme="@android:style/Theme.Material.Light.NoActionBar"
        android:supportsRtl="true">
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
""",
"app/src/main/java/com/example/jarvis/MainActivity.kt": r"""
package com.example.jarvis

import android.Manifest
import android.content.Intent
import android.os.Bundle
import android.speech.RecognizerIntent
import android.speech.tts.TextToSpeech
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.shadow
import androidx.compose.ui.graphics.Brush
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import org.json.JSONObject
import java.net.HttpURLConnection
import java.net.URL
import java.util.Locale
import java.util.concurrent.Executors

private val Navy = Color(0xFF061321)
private val Cyan = Color(0xFF39D9FF)
private data class ChatMessage(val fromUser: Boolean, val text: String)

class MainActivity : ComponentActivity() {
    private var tts: TextToSpeech? = null
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        tts = TextToSpeech(this) { status ->
            if (status == TextToSpeech.SUCCESS) tts?.language = Locale("tr", "TR")
        }
        setContent {
            JarvisScreen(
                onSpeak = { say ->
                    tts?.language = Locale("tr", "TR")
                    tts?.speak(say, TextToSpeech.QUEUE_FLUSH, null, "jarvis_reply")
                }
            )
        }
    }
    override fun onDestroy() {
        tts?.stop()
        tts?.shutdown()
        super.onDestroy()
    }
}

@Composable
private fun JarvisScreen(onSpeak: (String) -> Unit) {
    var input by remember { mutableStateOf("") }
    var endpoint by remember { mutableStateOf("https://YOUR-SERVER.example.com/chat") }
    var status by remember { mutableStateOf("Sunucu adresini Ayarlar bölümüne girin") }
    var busy by remember { mutableStateOf(false) }
    var showSettings by remember { mutableStateOf(false) }
    val messages = remember {
        mutableStateListOf(ChatMessage(false, "Merhaba. Ben JARVIS. Türkçe sohbet için hazırım."))
    }
    val executor = remember { Executors.newSingleThreadExecutor() }

    fun askJarvis(text: String) {
        val question = text.trim()
        if (question.isEmpty() || busy) return
        messages.add(ChatMessage(true, question))
        input = ""
        busy = true
        status = "JARVIS düşünüyor..."
        executor.execute {
            try {
                val conn = (URL(endpoint.trim()).openConnection() as HttpURLConnection).apply {
                    requestMethod = "POST"
                    connectTimeout = 15000
                    readTimeout = 60000
                    doOutput = true
                    setRequestProperty("Content-Type", "application/json; charset=utf-8")
                }
                val payload = JSONObject().put("message", question).toString()
                conn.outputStream.use { it.write(payload.toByteArray(Charsets.UTF_8)) }
                val code = conn.responseCode
                val stream = if (code in 200..299) conn.inputStream else conn.errorStream
                val body = stream.bufferedReader().use { it.readText() }
                conn.disconnect()
                if (code !in 200..299) throw IllegalStateException("Sunucu hatası ($code): $body")
                val answer = JSONObject(body).optString("reply", "Sunucudan yanıt alınamadı.")
                android.os.Handler(mainLooper).post {
                    messages.add(ChatMessage(false, answer))
                    status = "Bağlantı başarılı"
                    busy = false
                    onSpeak(answer)
                }
            } catch (e: Exception) {
                android.os.Handler(mainLooper).post {
                    val error = "Bağlantı kurulamadı. Sunucu adresini ve internet bağlantısını kontrol edin."
                    messages.add(ChatMessage(false, error))
                    status = e.message ?: "Bağlantı hatası"
                    busy = false
                }
            }
        }
    }

    val speechLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        val spoken = result.data?.getStringArrayListExtra(RecognizerIntent.EXTRA_RESULTS)?.firstOrNull()
        if (!spoken.isNullOrBlank()) {
            val cleaned = spoken.trim()
            // Foreground voice button: saying “Jarvis” at the start acts as the wake phrase.
            val command = cleaned.replaceFirst(Regex("(?i)^jarvis[, ]*"), "").ifBlank { "Buradayım. Nasıl yardımcı olabilirim?" }
            askJarvis(command)
        }
    }
    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
                putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
                putExtra(RecognizerIntent.EXTRA_LANGUAGE, "tr-TR")
                putExtra(RecognizerIntent.EXTRA_PROMPT, "JARVIS'e sorunuzu söyleyin")
            }
            speechLauncher.launch(intent)
        } else status = "Sesli kullanım için mikrofon izni gerekli."
    }

    Box(Modifier.fillMaxSize().background(
        Brush.verticalGradient(listOf(Color(0xFF020912), Navy, Color(0xFF0A263B)))
    )) {
        Column(Modifier.fillMaxSize().padding(16.dp)) {
            Row(verticalAlignment = Alignment.CenterVertically) {
                Column(Modifier.weight(1f)) {
                    Text("J.A.R.V.I.S.", color = Cyan, fontSize = 27.sp, fontWeight = FontWeight.Bold)
                    Text("PERSONAL AI • TÜRKÇE", color = Color.LightGray, fontSize = 10.sp, letterSpacing = 2.sp)
                }
                TextButton(onClick = { showSettings = !showSettings }) { Text("⚙ Ayarlar", color = Cyan) }
            }
            if (showSettings) {
                OutlinedTextField(
                    value = endpoint, onValueChange = { endpoint = it },
                    modifier = Modifier.fillMaxWidth(),
                    label = { Text("Güvenli AI sunucu adresi", color = Color.LightGray) },
                    singleLine = true,
                    colors = OutlinedTextFieldDefaults.colors(
                        focusedTextColor = Color.White, unfocusedTextColor = Color.White,
                        focusedBorderColor = Cyan, unfocusedBorderColor = Color(0xFF31566A)
                    )
                )
                Text("API anahtarını buraya yazmayın. Anahtar yalnızca sunucuda tutulmalıdır.",
                    color = Color(0xFFFFD27D), fontSize = 11.sp)
            }
            Spacer(Modifier.height(14.dp))
            Box(Modifier.fillMaxWidth(), contentAlignment = Alignment.Center) { ArcReactor() }
            Text("SİSTEM DURUMU", color = Cyan, modifier = Modifier.align(Alignment.CenterHorizontally),
                fontSize = 11.sp, letterSpacing = 2.sp)
            Text(status, color = Color.LightGray, modifier = Modifier.align(Alignment.CenterHorizontally),
                fontSize = 10.sp)
            Spacer(Modifier.height(14.dp))
            Text("SOHBET", color = Cyan, fontWeight = FontWeight.Bold, letterSpacing = 2.sp)
            LazyColumn(
                modifier = Modifier.weight(1f).fillMaxWidth().padding(top = 8.dp),
                verticalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                items(messages) { msg ->
                    Row(Modifier.fillMaxWidth(), horizontalArrangement =
                        if (msg.fromUser) Arrangement.End else Arrangement.Start) {
                        Surface(
                            color = if (msg.fromUser) Color(0xFF12445C) else Color(0xFF102536),
                            shape = RoundedCornerShape(14.dp),
                            modifier = Modifier.widthIn(max = 310.dp)
                        ) { Text(msg.text, color = Color.White, modifier = Modifier.padding(12.dp)) }
                    }
                }
            }
            Row(verticalAlignment = Alignment.CenterVertically) {
                OutlinedTextField(
                    value = input, onValueChange = { input = it },
                    modifier = Modifier.weight(1f),
                    placeholder = { Text("JARVIS'e mesaj yaz...", color = Color.Gray) },
                    colors = OutlinedTextFieldDefaults.colors(
                        focusedTextColor = Color.White, unfocusedTextColor = Color.White,
                        focusedBorderColor = Cyan, unfocusedBorderColor = Color(0xFF31566A)
                    ),
                    shape = RoundedCornerShape(16.dp), maxLines = 3
                )
                Spacer(Modifier.width(6.dp))
                Button(
                    enabled = !busy,
                    onClick = { permissionLauncher.launch(Manifest.permission.RECORD_AUDIO) },
                    colors = ButtonDefaults.buttonColors(containerColor = Color(0xFF12445C))
                ) { Text("🎙") }
                Spacer(Modifier.width(4.dp))
                Button(enabled = !busy, onClick = { askJarvis(input) },
                    colors = ButtonDefaults.buttonColors(containerColor = Cyan)) {
                    Text(if (busy) "…" else "➤", color = Navy)
                }
            }
            Text("Mikrofon yalnızca düğmeye bastığınızda açılır. Uygulama arka planda dinlemez.",
                color = Color.Gray, fontSize = 10.sp, modifier = Modifier.padding(top = 7.dp))
        }
    }
}

@Composable
private fun ArcReactor() {
    Box(
        Modifier.size(150.dp).shadow(22.dp, CircleShape).background(
            Brush.radialGradient(listOf(Color(0xFF8AF2FF), Color(0xFF087EA5), Color(0xFF061321))),
            CircleShape
        ).padding(12.dp),
        contentAlignment = Alignment.Center
    ) {
        Box(Modifier.fillMaxSize().background(Color(0xFF061321), CircleShape).padding(8.dp)
            .background(Brush.radialGradient(listOf(Color(0xFF35DFFF), Color(0xFF075071))), CircleShape),
            contentAlignment = Alignment.Center) {
            Text("✦", color = Color.White, fontSize = 62.sp, fontWeight = FontWeight.Light)
        }
    }
}
""",
"backend/main.py": """
import os
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from openai import OpenAI

app = FastAPI(title="JARVIS AI backend")
client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
MODEL = os.environ.get("OPENAI_MODEL", "gpt-4.1-mini")

class ChatRequest(BaseModel):
    message: str

@app.post("/chat")
def chat(req: ChatRequest):
    if not os.environ.get("OPENAI_API_KEY"):
        raise HTTPException(status_code=500, detail="OPENAI_API_KEY sunucuda ayarlanmamış")
    try:
        response = client.responses.create(
            model=MODEL,
            instructions=(
                "Sen JARVIS adında nazik, zeki, kısa ve net konuşan bir Türkçe yapay zekâ "
                "asistanısın. Kullanıcıyla Türkçe konuş. Kendini gerçek bir insan gibi tanıtma."
            ),
            input=req.message,
        )
        return {"reply": response.output_text}
    except Exception:
        raise HTTPException(status_code=502, detail="AI servisine ulaşılamadı")
""",
"backend/requirements.txt": """
fastapi
uvicorn[standard]
openai
""",
"backend/README_TR.md": """
# JARVIS AI sunucusu

Android uygulaması API anahtarını içermez. Bu küçük sunucu, anahtarı sunucu tarafında tutarak OpenAI API'ye istek gönderir.

## Kurulum
Python 3.10+ gerekir.

```bash
python -m venv .venv
# Windows:
.venv\\Scripts\\activate
# macOS/Linux:
source .venv/bin/activate
pip install -r requirements.txt
```

Ortam değişkenlerini ayarlayın:

```bash
# macOS/Linux
export OPENAI_API_KEY="API_ANAHTARINIZ"
export OPENAI_MODEL="gpt-4.1-mini"
uvicorn main:app --host 0.0.0.0 --port 8000
```

Windows PowerShell:
```powershell
$env:OPENAI_API_KEY="API_ANAHTARINIZ"
$env:OPENAI_MODEL="gpt-4.1-mini"
uvicorn main:app --host 0.0.0.0 --port 8000
```

Sunucuyu internete açacaksanız HTTPS, erişim kimlik doğrulaması, istek sınırı ve maliyet limitleri ekleyin. API anahtarını Android uygulamasına veya herkese açık depoya koymayın.

Android uygulamasındaki Ayarlar alanına `https://sunucu-adresiniz/chat` adresini girin. `http://` yerel test adresleri Android güvenlik ayarları nedeniyle çalışmayabilir; HTTPS kullanın.
""",
"README_TR.md": """
# JARVIS Android – Geliştirilmiş başlangıç sürümü

## Neler eklendi?
- Iron Man esintili koyu mavi / camgöbeği arayüz ve Ark Reaktörü
- Türkçe yazılı sohbet
- Android konuşma tanıma ile Türkçe sesli giriş
- “Jarvis” kelimesiyle başlayan sesli komutu ayıklama (mikrofon düğmesiyle, uygulama açıkken)
- Android Türkçe metin okuma (TTS)
- Ayarlar alanından kendi HTTPS AI sunucu adresini girme
- Gerçek yapay zekâ için ayrı backend örneği

## Android Studio'da çalıştırma
1. ZIP'i çıkarın ve `JARVIS_Gelistirilmis` klasörünü Android Studio'da açın.
2. Gradle eşitlemesini tamamlayın.
3. Android cihazı USB hata ayıklama ile bağlayın veya emülatör açın.
4. Run'a basın; mikrofon özelliği için uygulama izin istediğinde izin verin.
5. AI bağlantısı için `backend/README_TR.md` dosyasındaki sunucuyu kurun.
6. Uygulamada ⚙ Ayarlar'a basıp sunucu adresinizi `https://.../chat` biçiminde girin.

## Önemli sınırlar
- Bu ZIP derlenebilir bir Android Studio kaynak projesidir; önceden derlenmiş APK değildir.
- Gerçek AI yanıtları için sizin kuracağınız backend ve geçerli API erişimi gerekir. Kullanım ücretleri oluşabilir.
- Filmdeki ses birebir kopyalanmaz; cihazın Türkçe TTS sesi kullanılır.
- Sürekli arka plan mikrofonu / gizli dinleme yoktur. Mikrofon yalnızca kullanıcı ses düğmesine bastığında, Android'in izin verdiği görünür oturumda kullanılır.
- “Jarvis” uyandırma ifadesi, o sesli komut oturumunda tanınan metnin başında algılanır; uygulama kapalıyken veya arka planda sürekli dinlemez.
- Android Studio ve Android SDK 35 kullanılması önerilir.
"""
}
for rel, content in files.items():
    p = root / rel
    p.parent.mkdir(parents=True, exist_ok=True)
    p.write_text(textwrap.dedent(content).lstrip(), encoding="utf-8")

zip_path = Path("/mnt/data/JARVIS_Gelistirilmis_Android_Projesi.zip")
with zipfile.ZipFile(zip_path, "w", zipfile.ZIP_DEFLATED) as z:
    for p in root.rglob("*"):
        if p.is_file():
            z.write(p, p.relative_to(root.parent))
print(f"Geliştirilmiş ZIP oluşturuldu: {zip_path} ({zip_path.stat().st_size:,} bayt)")
