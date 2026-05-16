
https://chat.z.ai/c/ab6106b6-813a-4562-9a2e-178fe0f4c52e


<project>

<doc>
# Grocery Finance Tracker - Native Android Dev Context

## Developer Profile
- **Background:** Experienced Fullstack WebDev (React, Node, REST).
- **Environment:** Mac, VSCode, pure headless CLI development (NO Android Studio GUI).
- **Tools:** Physical Android phone for testing, `adb`, `gradlew`.
- **Paradigm Focus:** Strictly modern Android (Kotlin, Jetpack Compose, Coroutines).

## Project Setup & State
- **Project Name:** GroceryTracker
- **Root Directory:** `~/Desktop/AndroidNativeProjects/GroceryTracker/`
- **Package:** `com.example.grocerytracker`
- **Gradle:** Version 8.5. `gradle wrapper` was used to bootstrap `gradlew` manually. 
- **Gradle Config:** Had to manually create `gradle.properties` with `android.useAndroidX=true`, `android.enableJetifier=true`, and critically `org.gradle.jvmargs=-Xmx4g` (to solve OOM errors when importing extended Material Icons).

## Current Architecture & What's Built
1. **UI (Jetpack Compose):** Single Activity (`MainActivity`). Simple state-driven navigation (`UiState` sealed class in `AppViewModel`) to switch between Input Screen and Debug Screen.
2. **OS Contracts:** Implemented native `ActivityResultContracts` for Camera (`TakePicturePreview`) and Photo Picker (`PickVisualMedia`). Camera permissions handled correctly.
3. **Threading:** Successfully wrapped Google ML Kit callbacks into Kotlin `suspendCancellableCoroutine` and dispatched to `Dispatchers.Default` to prevent UI freezing.
4. **Database (Room):** Just added. Relational schema: `Receipt` (1-to-Many) `Item`. Uses `ForeignKey` with `onDelete = CASCADE`. 
5. **Migrations:** Database is at Version 2. Explicit `Migration(1, 2)` object implemented in `AppDatabase.kt` (added 'category' column to Items table).
6. **Background Jobs (WorkManager):** Just added. `OcrWorker` takes an array of image URIs via `Data`, processes them, saves to DB, and triggers a native `NotificationManager` alert upon success.

## The Next Big Task: Advanced OCR Pipeline (ONNX)
We decided against `llama.cpp` (too heavy for OS-sanctioned background WorkManager limits) and Google ML Kit (too chaotic raw text output). 
We are migrating to **PaddleOCR via ONNX Runtime** (`com.microsoft.onnxruntime:onnxruntime-android:1.16.3` is in dependencies).

### Current State of ONNX Integration:
- A dummy `OnnxOcrEngine.kt` class exists in the `ocr` package.
- It returns mock data so the `WorkManager` -> `Room DB` -> `Notification` pipeline could be tested end-to-end.
- **TODO for next session:** Replace the mock with real ONNX logic.

### ONNX Implementation Plan (For Next Session):
1. Download PaddleOCR `.onnx` models (det/text recognition) and place in `app/src/main/assets/`.
2. In `OnnxOcrEngine`: Initialize `OrtEnvironment` and `OrtSession`.
3. Pre-processing: Take the Android `Bitmap` (from the URI), resize it to 640x640, normalize the pixels (0-255 to 0.0-1.0), and create an `OnnxTensor`.
4. Inference: `session.run()`.
5. Post-processing: Parse the raw float array output tensors back into Bounding Boxes and Strings.
6. Create a "Review & Edit" UI state to let the user fix OCR mistakes *before* the `WorkManager` saves to the Room DB.

## Key "Web to Native" Lessons Learned So Far
- Android Studio hides `gradle.properties` configs; headless requires manual creation.
- `Icons.Default` only has basic icons; extended ones require `material-icons-extended` (which is huge and causes OOM without `-Xmx4g`).
- Variables in Compose must be declared top-to-bottom (lambda captures fail otherwise).
- The OS *will* kill background processes; you cannot run HTTP servers or heavy LLMs in `WorkManager` without hitting strict 10-minute timeouts. Fast, native C++/ONNX bindings are required for high-throughput background ML.
</doc>

<doc desc="snippet of original project proposal during ideation phase">


Interesting! Suppose I am a student in this course and for my final project, I would like to create a "Grocery Finance tracker" app. Idea: daily life shopping results in lots of paper receipt, I want to be able to scan it using my phone, then it automatically OCR. It records the shopping record (where, items + the spending amount), and then downstream do the usual spending tracking stuff with simple dashboard and analytics. (one eg of not too trivial: most frequent repeat purchase item) Would this app be able to effectively leverage the knowledges in the learning modules above?

...

ok a few things: - how about the edit screen after scanning and OCR'ed (you did mention this above) => I am thinking about not just allowing manual editing of the detected items (standard web style form), but also allow user to manually adjust the detected bounding box using dragging touch screen gesture (assume that some advanced OCR pipeline is available in future iteration) - mass import: select a folder then work on all image files in it in background one by one? (app notification for long running job) - allow both user select photo as well as live camera => UX/flow challenge? also select file means cross app interaction? (open the OS native file picker) - allow "share analytic chart" to other apps / export chart image - data backup: allow export of app local data in portable format. lol now the app no longer look like "a simple tutorial app"...

...
### 1. Adjustable Bounding Boxes (The Edit Screen)
* **What it teaches:** Custom Gesture Handling & Coordinate Math.
* **The Web vs. Native shift:** On the web, you'd probably use a library like Fabric.js or Konva.js to handle dragging DOM elements over an image. In native, there is no DOM. You are literally calculating touch coordinates (`CGPoint` / `Offset`), running hit-tests to see if a finger touched a box corner, and updating the box's frame state. 
* **The Tech:** SwiftUI `DragGesture` with `.gesture()` modifiers; Compose `pointerInput` and `detectDragGestures`. You also have to learn how to map image coordinates (which might be 3000x4000 pixels) to screen coordinates (which might be 390x844 points) without distortion. It’s a fantastic math/logic puzzle.

### 2. Mass Import & Background Processing + Notification
* **What it teaches:** App Lifecycles, Background Tasks, and OS Constraints.
* **The Web vs. Native shift:** This is the **most important lesson** in all of mobile dev. On the web, if a user starts a massive loop, it runs until the tab is closed. On mobile, **the OS hates you.** If you try to OCR 50 images in the background, iOS or Android will literally murder your app process to save battery after about 30 seconds.
* **The Tech:** You have to use *OS-sanctioned* background schedulers: `WorkManager` (Android) and `BGProcessingTask` (iOS). You will learn how to chunk a queue of images, process one, save to DB, and check if the OS is about to kill you. Then, you trigger a **Local Notification** (`UNUserNotificationCenter` on iOS, `NotificationManager` on Android) to ping the user when the queue finishes, even if the app is closed. This is elite-level mobile architecture.

### 3. Multi-Source Input (Camera vs. Photo vs. File Picker)
* **What it teaches:** OS Contracts / Delegates & Cross-App Communication.
* **The Web vs. Native shift:** On the web, `<input type="file">` handles everything. In native, different inputs require entirely different OS pipelines. 
* **The UX/Flow Solution:** You don't build separate screens for this. You use a native **Action Sheet** (a menu that slides up from the bottom of the screen: "Take Photo", "Choose from Library", "Choose from Files").
* **The Tech:** 
    * *Camera:* Requires building a custom `UIViewControllerRepresentable` (iOS) or `AndroidView` (Android) wrapping `AVCaptureSession` / `CameraX`.
    * *Photo Picker:* Native `PhotosUI` (iOS) / `PhotoPicker` contract (Android).
    * *File Picker:* `UIDocumentPickerViewController` (iOS) / `OpenDocumentTree` contract (Android). Notice the word "Contract/Delegate"? That’s how OS apps talk to each other securely without sharing memory.

### 4. Share Analytic Chart Image
* **What it teaches:** OS Interoperability & UI Rendering.
* **The Web vs. Native shift:** On the web, `html2canvas` is notoriously janky. In native, rendering a view to an image is a first-class citizen, but sharing it requires invoking the OS "Share Sheet".
* **The Tech:** First, you snapshot your chart view (`ImageRenderer` in SwiftUI, `drawToBitmap` in Compose). Then, you hand that image to the OS Share Sheet (`UIActivityViewController` on iOS, `Intent.ACTION_SEND` on Android). Suddenly, your app can send the chart to WhatsApp, iMessage, Slack, or save it to Files—without you writing a single line of networking code.

### 5. Data Backup Export
* **What it teaches:** File System Access & Data Serialization.
* **The Web vs. Native shift:** Browsers sandbox downloads to a specific folder. Native apps have to request secure access to user directories.
* **The Tech:** You pull your relational data from SwiftData/Room, serialize it into a portable format (JSON is easy, CSV is better for Excel users), write it to a temporary file in the app's `tmp` directory, and then use the OS File Picker (in "Export" mode) to let the user choose where to save it (iCloud Drive, Google Drive, local "Downloads" folder).

---

### The Professor's Warning: How not to fail the class

This app is now a 3-month project for a single developer. If you try to build all of this in the last two weeks of the semester, you will fail. 

To get an A+, you need to present a **Phased Delivery Plan** on day one:

* **Phase 1 (The Pass):** Photo Picker only -> Basic OCR -> Text Review Screen (just edit text, no boxes) -> Save to DB -> Basic Dashboard.
* **Phase 2 (The Grade Bump):** Add Live Camera. Add Data Export (JSON). Add the Share Sheet for charts.
* **Phase 3 (The Demo Day Flex):** Add Bounding Box adjustment. Add Mass Folder Import with Background Tasks & Notifications.

If you walk into the final presentation having fully nailed Phase 1 and 2, and you *show a video* of Phase 3 working (even if it's a little buggy), any engineering manager interviewing you would hire you on the spot. 

You've identified exactly what makes mobile hard, and exactly what makes it powerful. Stick to this plan!

</doc>

<codebase>

<file>
package com.example.grocerytracker.data

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import androidx.room.migration.Migration
import androidx.sqlite.db.SupportSQLiteDatabase

@Database(entities = [Receipt::class, Item::class], version = 2) // Version 2!
abstract class AppDatabase : RoomDatabase() {
    abstract fun receiptDao(): ReceiptDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "grocery_database"
                )
                // THE NATIVE SHIFT: Explicit Schema Migrations
                // If we add a column in version 2, we must tell Room how to alter the table.
                .addMigrations(MIGRATION_1_2)
                .build()
                INSTANCE = instance
                instance
            }
        }

        // Example Migration: Let's pretend in Version 2 we added the 'category' column to Items
        private val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("ALTER TABLE items ADD COLUMN category TEXT NOT NULL DEFAULT 'Unknown'")
            }
        }
    }
}
</file>
<file>
package com.example.grocerytracker.data

import androidx.room.Entity
import androidx.room.ForeignKey
import androidx.room.PrimaryKey
import java.util.Date

@Entity(tableName = "receipts")
data class Receipt(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val storeName: String,
    val date: Long = System.currentTimeMillis(),
    val total: Double
)

@Entity(
    tableName = "items",
    foreignKeys = [ForeignKey(
        entity = Receipt::class,
        parentColumns = ["id"],
        childColumns = ["receiptId"],
        onDelete = ForeignKey.CASCADE // Native shift: If receipt deletes, items auto-delete!
    )]
)
data class Item(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val receiptId: Int,
    val name: String,
    val price: Double,
    val category: String = "Unknown" // Placeholder for future schema migration
)
</file>
<file>
package com.example.grocerytracker.data

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query

@Dao
interface ReceiptDao {
    @Insert
    suspend fun insertReceipt(receipt: Receipt): Long

    @Insert
    suspend fun insertItems(items: List<Item>)

    @Query("SELECT * FROM receipts ORDER BY date DESC")
    fun getAllReceipts(): kotlinx.coroutines.flow.Flow<List<Receipt>>
}
</file>

<file>
package com.example.grocerytracker.ocr

import android.content.Context
import com.example.grocerytracker.data.Item
import com.example.grocerytracker.data.Receipt
import kotlinx.coroutines.delay

data class ParsedReceipt(
    val receipt: Receipt,
    val items: List<Item>
)

class OnnxOcrEngine(private val context: Context) {
    
    // MOCK IMPLEMENTATION
    // In the next session, you will:
    // 1. Initialize OrtEnvironment and OrtSession here with your .onnx file
    // 2. Convert the Uri to a Bitmap
    // 3. Resize Bitmap to 640x640
    // 4. Convert to FloatBuffer (Tensor)
    // 5. Run session.run()
    // 6. Parse output tensor bounding boxes back into text
    suspend fun processImage(imageUri: String): ParsedReceipt {
        delay(1000) // Simulate processing time
        
        return ParsedReceipt(
            receipt = Receipt(storeName = "Mock Store", total = 15.50),
            items = listOf(
                Item(receiptId = 0, name = "Mock Apples", price = 5.50, category = "Fruit"),
                Item(receiptId = 0, name = "Mock Bread", price = 10.00, category = "Bakery")
            )
        )
    }
}
</file>
<file>
package com.example.grocerytracker.worker

import android.app.NotificationChannel
import android.app.NotificationManager
import android.content.Context
import android.os.Build
import androidx.core.app.NotificationCompat
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters
import androidx.work.Data
import com.example.grocerytracker.R
import com.example.grocerytracker.data.AppDatabase
import com.example.grocerytracker.data.Item
import com.example.grocerytracker.data.Receipt
import com.example.grocerytracker.ocr.OnnxOcrEngine

class OcrWorker(
    context: Context,
    workerParams: WorkerParameters
) : CoroutineWorker(context, workerParams) {

    private val db = AppDatabase.getDatabase(context)
    private val ocrEngine = OnnxOcrEngine(context) // Will use our Mock for now

    override suspend fun doWork(): Result {
        val imageUris = inputData.getStringArray("IMAGE_URIS") ?: return Result.failure()

        try {
            imageUris.forEach { uriString ->
                // 1. Run OCR (currently mocked)
                val parsedReceipt = ocrEngine.processImage(uriString)
                
                // 2. Save to DB
                val receiptId = db.receiptDao().insertReceipt(parsedReceipt.receipt)
                val itemsWithForeignId = parsedReceipt.items.map { it.copy(receiptId = receiptId.toInt()) }
                db.receiptDao().insertItems(itemsWithForeignId)
            }

            // 3. Send Notification
            showSuccessNotification()
            return Result.success()
        } catch (e: Exception) {
            return Result.failure()
        }
    }

    private fun showSuccessNotification() {
        val channelId = "ocr_channel"
        val notificationManager = applicationContext.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(channelId, "OCR Processing", NotificationManager.IMPORTANCE_DEFAULT)
            notificationManager.createNotificationChannel(channel)
        }

        val notification = NotificationCompat.Builder(applicationContext, channelId)
            .setSmallIcon(android.R.drawable.ic_menu_gallery)
            .setContentTitle("Grocery Tracker")
            .setContentText("Finished processing ${inputData.getStringArray("IMAGE_URIS")?.size} receipts!")
            .setPriority(NotificationCompat.PRIORITY_DEFAULT)
            .build()

        notificationManager.notify(1001, notification)
    }
}
</file>

<file>
package com.example.grocerytracker.ui

import android.graphics.Bitmap
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.google.mlkit.vision.common.InputImage
import com.google.mlkit.vision.text.TextRecognition
import com.google.mlkit.vision.text.latin.TextRecognizerOptions
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch
import kotlinx.coroutines.suspendCancellableCoroutine
import kotlin.coroutines.resume
import kotlin.coroutines.resumeWithException

// State representation (like Redux state)
sealed class UiState {
    object Idle : UiState()
    object Loading : UiState()
    data class Success(val image: Bitmap, val rawText: String) : UiState()
    data class Error(val message: String) : UiState()
}

class AppViewModel : ViewModel() {

    private val _uiState = MutableStateFlow<UiState>(UiState.Idle)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    // The Core Logic: Run OCR on a background thread
    fun processImage(bitmap: Bitmap) {
        viewModelScope.launch(Dispatchers.Default) { // Background thread!
            _uiState.value = UiState.Loading
            try {
                val text = runMlKitOcr(bitmap)
                _uiState.value = UiState.Success(bitmap, text)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.localizedMessage ?: "Unknown error")
            }
        }
    }

    // THE MAGIC: Turning ML Kit's callback into a Kotlin Coroutine (async/await)
    private suspend fun runMlKitOcr(bitmap: Bitmap): String = suspendCancellableCoroutine { cont ->
        val image = InputImage.fromBitmap(bitmap, 0)
        val recognizer = TextRecognition.getClient(TextRecognizerOptions.DEFAULT_OPTIONS)

        recognizer.process(image)
            .addOnSuccessListener { visionText ->
                cont.resume(visionText.text) // Resolve the "Promise"
            }
            .addOnFailureListener { e ->
                cont.resumeWithException(e) // Reject the "Promise"
            }
    }

    fun resetState() {
        _uiState.value = UiState.Idle
    }
}
</file>
<file>
package com.example.grocerytracker.ui

import androidx.compose.foundation.Image
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ArrowBack
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.asImageBitmap
import androidx.compose.ui.unit.dp

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun DebugScreen(image: android.graphics.Bitmap, rawText: String, onBackClicked: () -> Unit) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("OCR Debug Screen") },
                navigationIcon = {
                    IconButton(onClick = onBackClicked) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "Back")
                    }
                }
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(16.dp)
        ) {
            Text("Scanned Image:", style = MaterialTheme.typography.titleMedium)
            Spacer(modifier = Modifier.height(8.dp))
            
            // Display the image
            Image(
                bitmap = image.asImageBitmap(),
                contentDescription = "Scanned Receipt",
                modifier = Modifier
                    .fillMaxWidth()
                    .height(250.dp)
            )

            Spacer(modifier = Modifier.height(24.dp))
            
            Text("Raw ML Kit Output:", style = MaterialTheme.typography.titleMedium)
            Spacer(modifier = Modifier.height(8.dp))
            
            // Display the raw text in a scrollable box
            Card(modifier = Modifier.fillMaxSize()) {
                Text(
                    text = rawText,
                    modifier = Modifier
                        .padding(16.dp)
                        .verticalScroll(rememberScrollState()) // Native scroll view!
                )
            }
        }
    }
}

</file>
<file>
package com.example.grocerytracker.ui

import android.Manifest
import android.content.Context
import android.graphics.Bitmap
import android.graphics.ImageDecoder
import android.net.Uri
import android.os.Build
import android.provider.MediaStore
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.PickVisualMediaRequest
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.*
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.CameraAlt
import androidx.compose.material.icons.filled.Folder
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.asImageBitmap
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp

import androidx.work.*
import androidx.work.Data
import android.widget.Toast

import com.example.grocerytracker.worker.OcrWorker

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MainScreen(viewModel: AppViewModel) {
    // Observe State (Recomposition happens automatically, like React re-rendering)
    val uiState by viewModel.uiState.collectAsState()
    val context = LocalContext.current

    // Get instance
    val workManager = WorkManager.getInstance(context)

    // --- NATIVE OS CONTRACTS ---
    
    // 2. Camera Contract (TakePicturePreview gives us a Bitmap directly, no file URI management needed!)
    val cameraLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.TakePicturePreview()
    ) { bitmap: Bitmap? ->
        bitmap?.let { viewModel.processImage(it) }
    }
    
    // 1. Camera Permission Contract
    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) cameraLauncher.launch(null) // Launch camera if permitted
    }

    // 3. Photo Picker Contract (Modern Android replacement for File Picker for images)
    val photoPickerLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri: Uri? ->
        uri?.let {
            //val bitmap = uriToBitmap(context, it)
            //viewModel.processImage(bitmap)

            // Create WorkRequest
            val inputData = Data.Builder().putStringArray("IMAGE_URIS", arrayOf(it.toString())).build()
            val ocrWorkRequest = OneTimeWorkRequestBuilder<OcrWorker>()
                .setInputData(inputData)
                .build()
                
            // Enqueue!
            workManager.enqueue(ocrWorkRequest)
            
            // Show a snackbar to user
            Toast.makeText(context, "Added to background queue!", Toast.LENGTH_SHORT).show()
        }
    }
    // --------------------------------

    // Simple "Navigation" based on State
    when (val state = uiState) {
        is UiState.Idle -> {
            Scaffold(
                topBar = { TopAppBar(title = { Text("Grocery Tracker") }) }
            ) { padding ->
                Column(
                    modifier = Modifier
                        .fillMaxSize()
                        .padding(padding)
                        .padding(16.dp),
                    verticalArrangement = Arrangement.Center,
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Button(
                        onClick = { 
                            // Check/Request permission natively before launching camera
                            permissionLauncher.launch(Manifest.permission.CAMERA) 
                        },
                        modifier = Modifier.fillMaxWidth().padding(bottom = 16.dp)
                    ) {
                        Icon(Icons.Default.CameraAlt, contentDescription = null)
                        Spacer(modifier = Modifier.width(8.dp))
                        Text("Scan with Camera")
                    }

                    OutlinedButton(
                        onClick = { 
                            photoPickerLauncher.launch(
                                PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
                            ) 
                        },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Icon(Icons.Default.Folder, contentDescription = null)
                        Spacer(modifier = Modifier.width(8.dp))
                        Text("Choose from Gallery")
                    }
                }
            }
        }

        is UiState.Loading -> {
            Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
                Text("Running OCR on background thread...")
            }
        }

        is UiState.Success -> {
            DebugScreen(
                image = state.image,
                rawText = state.rawText,
                onBackClicked = { viewModel.resetState() }
            )
        }

        is UiState.Error -> {
            Column(
                modifier = Modifier.fillMaxSize().padding(16.dp),
                verticalArrangement = Arrangement.Center,
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text("Error: ${state.message}", color = MaterialTheme.colorScheme.error)
                Spacer(modifier = Modifier.height(16.dp))
                Button(onClick = { viewModel.resetState() }) { Text("Go Back") }
            }
        }
    }
}

// Native shift: Converting a content:// URI to a Bitmap requires Context & ImageDecoder
private fun uriToBitmap(context: Context, uri: Uri): Bitmap {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
        ImageDecoder.decodeBitmap(ImageDecoder.createSource(context.contentResolver, uri))
    } else {
        @Suppress("DEPRECATION")
        MediaStore.Images.Media.getBitmap(context.contentResolver, uri)
    }
}

</file>
</codebase>

<notes>
Theme and main activity source code is omitted.
We are currently on the bgProcessing branch.
</notes>
</project>

Good morning!

Let's continue the project. Current wishlist:

- implement just enough UI (and potentially data schema) to be able to see detailed result of background job
- fix: currently can only select one file? Want: can select multiple files, and/or select folder to automatically process all images files inside the folder (but recursive or not?). Maybe a dialog window asking for confirmation and show the stat of how many files in total across how many folders, before proceeding.
- Integrate PaddleOCR (lite version?)

You may choose which to work on first. Work iteratively and break down into unit of work that is "right-sized" for your capability level. Thanks!


----

Good morning! Looking at the wishlist, I'll tackle **multiple file/folder selection first** because:
1. It's self-contained and doesn't require schema changes
2. It'll make testing the future OCR pipeline much more efficient
3. It's a good "web to native" lesson in OS contracts

Let me break this into a focused unit of work:

## Unit of Work: Multiple File & Folder Selection

### What we're building:
- Replace single-photo picker with multi-photo picker
- Add folder picker using `OpenDocumentTree` contract
- Add confirmation dialog showing image count before processing
- Background folder scanning (don't block UI thread)

### The "Native Shift" lessons here:
- `PickVisualMedia` only does single selection; need `GetMultipleContents` for multi
- Folder access uses SAF (Storage Access Framework) - completely different from web `<input type="file" webkitdirectory>`
- Tree URIs need `DocumentsContract` APIs to traverse - no simple `fs.readdir()`

---

### File 1: Create `FolderScanner.kt`

```kotlin
// app/src/main/java/com/example/grocerytracker/util/FolderScanner.kt
package com.example.grocerytracker.util

import android.content.Context
import android.net.Uri
import android.provider.DocumentsContract
import android.util.Log

/**
 * Result of scanning a folder for images.
 * imageUris can be passed directly to WorkManager.
 */
data class FolderScanResult(
    val imageUris: List<Uri>,
    val totalFiles: Int,
    val totalFolders: Int  // Subfolders found (not included in imageUris unless recursive=true)
)

/**
 * Utility to scan a SAF (Storage Access Framework) tree URI for image files.
 * 
 * Native Shift: On web, you'd use the File System Access API or a simple readdir.
 * On Android, folder access goes through DocumentsContract which has a very
 * different API surface - you query children via ContentResolver, not file paths.
 */
object FolderScanner {
    private const val TAG = "FolderScanner"
    
    // Common image extensions as fallback when MIME type isn't available
    private val IMAGE_EXTENSIONS = setOf("jpg", "jpeg", "png", "webp", "bmp")
    
    /**
     * Scans a folder tree for images.
     * @param context Application context for ContentResolver
     * @param treeUri The URI returned by OpenDocumentTree contract
     * @param recursive If true, scans subfolders too (slower)
     */
    fun scanFolder(context: Context, treeUri: Uri, recursive: Boolean = false): FolderScanResult {
        val imageUris = mutableListOf<Uri>()
        var folderCount = 0
        
        fun traverse(uri: Uri) {
            try {
                // Build the children query URI from the tree URI
                val childrenUri = DocumentsContract.buildChildDocumentsUriUsingTree(
                    treeUri,
                    DocumentsContract.getDocumentId(uri)
                )
                
                val projection = arrayOf(
                    DocumentsContract.Document.COLUMN_DOCUMENT_ID,
                    DocumentsContract.Document.COLUMN_MIME_TYPE,
                    DocumentsContract.Document.COLUMN_DISPLAY_NAME
                )
                
                // Query children - this is the native equivalent of fs.readdir()
                context.contentResolver.query(childrenUri, projection, null, null, null)?.use { cursor ->
                    while (cursor.moveToNext()) {
                        val docId = cursor.getString(0)
                        val mimeType = cursor.getString(1)
                        val displayName = cursor.getString(2)
                        val childUri = DocumentsContract.buildDocumentUriUsingTree(treeUri, docId)
                        
                        // Check if it's a directory
                        if (mimeType == DocumentsContract.Document.MIME_TYPE_DIR) {
                            folderCount++
                            if (recursive) {
                                traverse(childUri)
                            }
                        } else {
                            // Dual check: MIME type OR file extension
                            // Some providers don't report MIME types accurately
                            val isImageByMime = mimeType?.startsWith("image/") == true
                            val extension = displayName?.substringAfterLast('.')?.lowercase()
                            val isImageByExt = !extension.isNullOrEmpty() && IMAGE_EXTENSIONS.contains(extension)
                            
                            if (isImageByMime || isImageByExt) {
                                imageUris.add(childUri)
                            }
                        }
                    }
                }
            } catch (e: Exception) {
                // Some document providers may reject certain queries
                // Log but don't crash - just skip that folder
                Log.e(TAG, "Error traversing: ${DocumentsContract.getDocumentId(uri)}", e)
            }
        }
        
        // Start traversal from the root tree URI
        traverse(treeUri)
        
        return FolderScanResult(
            imageUris = imageUris,
            totalFiles = imageUris.size,
            totalFolders = folderCount
        )
    }
}
```

---

### File 2: Update `MainScreen.kt`

```kotlin
// app/src/main/java/com/example/grocerytracker/ui/MainScreen.kt
package com.example.grocerytracker.ui

import android.Manifest
import android.content.Context
import android.graphics.Bitmap
import android.graphics.ImageDecoder
import android.net.Uri
import android.os.Build
import android.provider.MediaStore
import android.widget.Toast
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.layout.width
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.CameraAlt
import androidx.compose.material.icons.filled.Folder
import androidx.compose.material.icons.filled.Image
import androidx.compose.material3.AlertDialog
import androidx.compose.material3.Button
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.ExperimentalMaterial3Api
import androidx.compose.material3.FilledTonalButton
import androidx.compose.material3.Icon
import androidx.compose.material3.IconButton
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.OutlinedButton
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.material3.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.rememberCoroutineScope
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.work.Data
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import com.example.grocerytracker.util.FolderScanResult
import com.example.grocerytracker.util.FolderScanner
import com.example.grocerytracker.worker.OcrWorker
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MainScreen(viewModel: AppViewModel) {
    val uiState by viewModel.uiState.collectAsState()
    val context = LocalContext.current
    val workManager = WorkManager.getInstance(context)
    val coroutineScope = rememberCoroutineScope()
    
    // --- LOCAL DIALOG STATES ---
    // These don't need to be in ViewModel - they're ephemeral UI state
    var showSelectionDialog by remember { mutableStateOf(false) }
    var folderScanResult by remember { mutableStateOf<FolderScanResult?>(null) }
    var isScanningFolder by remember { mutableStateOf(false) }

    // --- OS CONTRACTS (LAUNCHERS) ---
    
    // 1. Camera: TakePicturePreview returns Bitmap directly (no file management)
    val cameraLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.TakePicturePreview()
    ) { bitmap: Bitmap? ->
        bitmap?.let { viewModel.processImage(it) }
    }
    
    // 2. Camera permission request
    val permissionLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) cameraLauncher.launch(null)
    }

    // 3. Multiple photo picker
    // Native Shift: PickVisualMedia only does SINGLE selection.
    // GetMultipleContents uses ACTION_GET_CONTENT with EXTRA_ALLOW_MULTIPLE=true
    // Works on all API levels, not just Android 13+
    val multiplePhotoLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.GetMultipleContents()
    ) { uris: List<Uri> ->
        if (uris.isNotEmpty()) {
            enqueueOcrWork(workManager, context, uris)
        }
    }

    // 4. Folder picker
    // Native Shift: OpenDocumentTree grants persistent URI permissions automatically.
    // The returned URI is a "tree" URI, not a file URI - needs DocumentsContract APIs.
    val folderPickerLauncher = rememberLauncherForActivityResult(
        ActivityResultContracts.OpenDocumentTree()
    ) { treeUri: Uri? ->
        treeUri?.let {
            // Run folder scanning on IO dispatcher - don't block UI!
            coroutineScope.launch {
                isScanningFolder = true
                val result = withContext(Dispatchers.IO) {
                    FolderScanner.scanFolder(context, it, recursive = false)
                }
                isScanningFolder = false
                
                if (result.imageUris.isNotEmpty()) {
                    folderScanResult = result
                } else {
                    Toast.makeText(
                        context, 
                        "No images found in selected folder", 
                        Toast.LENGTH_SHORT
                    ).show()
                }
            }
        }
    }
    // --------------------------------

    // --- DIALOGS ---
    
    // Confirmation dialog after folder scan
    folderScanResult?.let { result ->
        AlertDialog(
            onDismissRequest = { folderScanResult = null },
            title = { Text("Confirm Import") },
            text = {
                val subfolderNote = if (result.totalFolders > 0) {
                    " (${result.totalFolders} subfolder(s) found, not included)"
                } else ""
                Text("Found ${result.totalFiles} image(s) in the selected folder$subfolderNote.\n\nProceed with OCR processing?")
            },
            confirmButton = {
                TextButton(
                    onClick = {
                        enqueueOcrWork(workManager, context, result.imageUris)
                        folderScanResult = null
                    }
                ) {
                    Text("Process All")
                }
            },
            dismissButton = {
                TextButton(onClick = { folderScanResult = null }) {
                    Text("Cancel")
                }
            }
        )
    }

    // Loading dialog while scanning folder
    if (isScanningFolder) {
        AlertDialog(
            onDismissRequest = { /* Don't allow dismiss during scan */ },
            title = { Text("Scanning Folder") },
            text = {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    CircularProgressIndicator(modifier = Modifier.size(24.dp))
                    Spacer(modifier = Modifier.width(16.dp))
                    Text("Looking for images...")
                }
            },
            confirmButton = {}  // No buttons - user must wait
        )
    }

    // Selection method dialog (replaces the old single picker)
    if (showSelectionDialog) {
        AlertDialog(
            onDismissRequest = { showSelectionDialog = false },
            title = { Text("Import Receipts") },
            text = {
                Column {
                    Text("Choose how to import your receipts:")
                    Spacer(modifier = Modifier.height(16.dp))
                    
                    OutlinedButton(
                        onClick = {
                            showSelectionDialog = false
                            multiplePhotoLauncher.launch("image/*")
                        },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Icon(Icons.Default.Image, contentDescription = null)
                        Spacer(modifier = Modifier.width(8.dp))
                        Text("Select Photos")
                    }
                    
                    Spacer(modifier = Modifier.height(8.dp))
                    
                    OutlinedButton(
                        onClick = {
                            showSelectionDialog = false
                            folderPickerLauncher.launch(null)
                        },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Icon(Icons.Default.Folder, contentDescription = null)
                        Spacer(modifier = Modifier.width(8.dp))
                        Text("Select Folder")
                    }
                }
            },
            confirmButton = {},
            dismissButton = {
                TextButton(onClick = { showSelectionDialog = false }) {
                    Text("Cancel")
                }
            }
        )
    }
    // -------------------

    // --- MAIN CONTENT (State-driven "navigation") ---
    when (val state = uiState) {
        is UiState.Idle -> {
            Scaffold(
                topBar = { TopAppBar(title = { Text("Grocery Tracker") }) }
            ) { padding ->
                Column(
                    modifier = Modifier
                        .fillMaxSize()
                        .padding(padding)
                        .padding(16.dp),
                    verticalArrangement = Arrangement.Center,
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    // Camera option (immediate ML Kit processing)
                    Button(
                        onClick = { 
                            permissionLauncher.launch(Manifest.permission.CAMERA) 
                        },
                        modifier = Modifier.fillMaxWidth().padding(bottom = 16.dp)
                    ) {
                        Icon(Icons.Default.CameraAlt, contentDescription = null)
                        Spacer(modifier = Modifier.width(8.dp))
                        Text("Scan with Camera")
                    }

                    // Gallery/Folder option (goes through WorkManager)
                    FilledTonalButton(
                        onClick = { showSelectionDialog = true },
                        modifier = Modifier.fillMaxWidth()
                    ) {
                        Icon(Icons.Default.Image, contentDescription = null)
                        Spacer(modifier = Modifier.width(8.dp))
                        Text("Import from Gallery / Folder")
                    }
                }
            }
        }

        is UiState.Loading -> {
            Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
                Text("Running OCR on background thread...")
            }
        }

        is UiState.Success -> {
            DebugScreen(
                image = state.image,
                rawText = state.rawText,
                onBackClicked = { viewModel.resetState() }
            )
        }

        is UiState.Error -> {
            Column(
                modifier = Modifier.fillMaxSize().padding(16.dp),
                verticalArrangement = Arrangement.Center,
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Text("Error: ${state.message}", color = MaterialTheme.colorScheme.error)
                Spacer(modifier = Modifier.height(16.dp))
                Button(onClick = { viewModel.resetState() }) { Text("Go Back") }
            }
        }
    }
}

/**
 * Enqueues OCR processing work for a list of image URIs.
 * Works with both gallery URIs (content://media/...) and 
 * folder tree URIs (content://com.android.providers.../tree/...)
 */
private fun enqueueOcrWork(workManager: WorkManager, context: Context, uris: List<Uri>) {
    val inputData = Data.Builder()
        .putStringArray("IMAGE_URIS", uris.map { it.toString() }.toTypedArray())
        .build()
    
    val ocrWorkRequest = OneTimeWorkRequestBuilder<OcrWorker>()
        .setInputData(inputData)
        .build()
    
    workManager.enqueue(ocrWorkRequest)
    Toast.makeText(context, "Added ${uris.size} image(s) to processing queue", Toast.LENGTH_SHORT).show()
}

// Keep this utility - we'll need it when implementing real OCR
private fun uriToBitmap(context: Context, uri: Uri): Bitmap {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
        ImageDecoder.decodeBitmap(ImageDecoder.createSource(context.contentResolver, uri))
    } else {
        @Suppress("DEPRECATION")
        MediaStore.Images.Media.getBitmap(context.contentResolver, uri)
    }
}
```

---

## Summary of changes:

| File | Action | What changed |
|------|--------|--------------|
| `util/FolderScanner.kt` | **NEW** | SAF folder traversal with image detection |
| `ui/MainScreen.kt` | **MODIFIED** | Multi-select picker, folder picker, confirmation dialogs |

## What you can test now:

1. **"Scan with Camera"** - Still uses immediate ML Kit (unchanged)
2. **"Import from Gallery / Folder"** - Opens dialog with two options:
   - **Select Photos** - System file picker, can multi-select images
   - **Select Folder** - Opens folder browser, scans non-recursive, shows confirmation

3. After confirming folder import, check:
   - Toast shows "Added X image(s) to processing queue"
   - Notification appears when WorkManager finishes (mock data in DB)

## Intentionally deferred:

- **Recursive folder scanning** - Added as parameter but not exposed in UI yet (the `recursive = false` is ready to flip)
- **Better error notification** - Worker still just shows success/fail silently

---

## Next unit of work options:

**Option A: View processed results UI** - Add a "History" screen to see receipts/items from DB (requires new DAO queries, new composable)

**Option B: Real PaddleOCR integration** - Replace mock in `OnnxOcrEngine.kt` with actual ONNX inference

Which would you like to tackle next?


----

quick debug round: build error:

> Task :app:compileDebugKotlin FAILED
ui/MainScreen.kt:64:38 Unresolved reference: collectAsState
ui/MainScreen.kt:169:17 Unresolved reference: Row
ui/MainScreen.kt:170:21 @Composable invocations can only happen from the context of a @Composable function
ui/MainScreen.kt:171:21 @Composable invocations can only happen from the context of a @Composable function
ui/MainScreen.kt:172:21 @Composable invocations can only happen from the context of a @Composable function


----

Classic cascading error! The root cause is just **two missing imports**:

1. `collectAsState` - lives in `androidx.compose.runtime`
2. `Row` - lives in `androidx.compose.foundation.layout`

The "Composable invocations" errors are fake - they cascade because when `Row` isn't recognized, Kotlin can't tell it's a composable lambda.

Add these to your imports in `MainScreen.kt`:

```kotlin
import androidx.compose.foundation.layout.Row
import androidx.compose.runtime.collectAsState
```

Your existing imports already have siblings of both (`Column`, `collectAsStateWithLifecycle`-adjacent stuff), so they just got dropped when I wrote the file.

Should compile clean after that.

----

Okay, works, but when proceeding via the custom select folder method, the app crash upon granting permission. However, I am in a time crunch and would like to focus on feature first. Would like to skip ahead a bit onto the PaddleOCR (lite?) integration (it's exciting but challenging). Can you show me how to do it? I can perform manual steps on your behalf. Thanks!

----

Understood - skipping the folder bug for now, let's get to the exciting part!

## PaddleOCR ONNX Integration

This is a multi-step process. I'll give you the code, you do the manual asset setup.

---

### MANUAL STEP 1: Download Models

Create this directory structure:
```
app/src/main/assets/ocr/
```

Download these 3 files and place them in that `ocr/` folder:

| File | Source (raw download links) |
|------|---------------------------|
| `det.onnx` | https://github.com/RapidAI/RapidOCR/raw/main/rapidocr_onnxruntime/models/ch_ppocr_mobile_v2.0_det_infer.onnx |
| `rec.onnx` | https://github.com/RapidAI/RapidOCR/raw/main/rapidocr_onnxruntime/models/ch_ppocr_mobile_v2.0_rec_infer.onnx |
| `keys.txt` | https://github.com/RapidAI/RapidOCR/raw/main/rapidocr_onnxruntime/models/ppocr_keys_v1.txt |

> **Note:** GitHub raw links sometimes redirect. If direct download fails, open the URL in browser, then use "Save As" or right-click → "Save link as". The `keys.txt` should be ~20KB with Chinese + English characters.

Verify your structure:
```
app/src/main/assets/ocr/
├── det.onnx   (~4MB)
├── rec.onnx   (~8MB)
└── keys.txt   (~20KB)
```

---

### MANUAL STEP 2: Verify Gradle Dependency

Your `build.gradle` should already have:
```kotlin
implementation("com.microsoft.onnxruntime:onnxruntime-android:1.16.3")
```

---

### CODE FILE 1: Create `OcrUtils.kt`

```kotlin
// app/src/main/java/com/example/grocerytracker/ocr/OcrUtils.kt
package com.example.grocerytracker.ocr

import android.graphics.Bitmap
import android.graphics.Color
import kotlin.math.max
import kotlin.math.sqrt

/**
 * Bounding box for detected text region.
 * Points are in original image coordinates.
 */
data class TextBox(
    val points: List<Pair<Float, Float>>,  // 4 corners: top-left, top-right, bottom-right, bottom-left
    val score: Float,
    val text: String = ""
) {
    /** Axis-aligned bounding box (simpler, good enough for receipts) */
    val bounds: Pair<Pair<Float, Float>, Pair<Float, Float>>
        get() {
            val minX = points.minOf { it.first }
            val minY = points.minOf { it.second }
            val maxX = points.maxOf { it.first }
            val maxY = points.maxOf { it.second }
            return (minX to minY) to (maxX to maxY)
        }
}

// ============================================================================
// DETECTION PREPROCESSING
// ============================================================================

data class ResizeResult(
    val bitmap: Bitmap,
    val ratioH: Float,
    val ratioW: Float
)

/**
 * Resize bitmap for detection model.
 * - Max dimension capped at 960px
 * - Resize to multiple of 32 (required by model)
 */
fun resizeForDetection(bitmap: Bitmap, maxSide: Int = 960): ResizeResult {
    val srcH = bitmap.height
    val srcW = bitmap.width
    val ratio = minOf(maxSide.toFloat() / maxOf(srcH, srcW), 1.0f)
    
    val resizeH = max(32, ((srcH * ratio).toInt() / 32) * 32)
    val resizeW = max(32, ((srcW * ratio).toInt() / 32) * 32)
    
    val resized = Bitmap.createScaledBitmap(bitmap, resizeW, resizeH, true)
    return ResizeResult(
        bitmap = resized,
        ratioH = srcH.toFloat() / resizeH,
        ratioW = srcW.toFloat() / resizeW
    )
}

/**
 * Convert bitmap to detection model input tensor.
 * Output shape: [1, 3, H, W] (NCHW format)
 * Normalization: (pixel - 127.5) / 127.5  => maps 0-255 to -1.0 to 1.0
 */
fun bitmapToDetTensor(bitmap: Bitmap): FloatArray {
    val h = bitmap.height
    val w = bitmap.width
    val pixels = IntArray(w * h)
    bitmap.getPixels(pixels, 0, w, 0, 0, w, h)
    
    // NCHW layout: [channel][row][col]
    val tensor = FloatArray(1 * 3 * h * w)
    var idx = 0
    
    for (c in 0 until 3) {           // Channel (BGR order for PaddleOCR!)
        for (y in 0 until h) {       // Height
            for (x in 0 until w) {   // Width
                val pixel = pixels[y * w + x]
                val value = when (c) {
                    0 -> Color.blue(pixel)    // B
                    1 -> Color.green(pixel)   // G
                    else -> Color.red(pixel)   // R
                }
                tensor[idx++] = (value - 127.5f) / 127.5f
            }
        }
    }
    
    return tensor
}

// ============================================================================
// DETECTION POSTPROCESSING (DB Algorithm - Simplified)
// ============================================================================

/**
 * Union-Find data structure for connected component analysis.
 * Used instead of OpenCV's findContours.
 */
private class UnionFind(size: Int) {
    private val parent = IntArray(size) { it }
    private val rank = IntArray(size)
    
    fun find(x: Int): Int {
        if (parent[x] != x) parent[x] = find(parent[x])
        return parent[x]
    }
    
    fun union(x: Int, y: Int) {
        val px = find(x)
        val py = find(y)
        if (px == py) return
        if (rank[px] < rank[py]) parent[px] = py
        else if (rank[px] > rank[py]) parent[py] = px
        else { parent[py] = px; rank[px]++ }
    }
}

/**
 * Extract bounding boxes from detection model output.
 * 
 * DB Post-processing (simplified):
 * 1. Threshold probability map at 0.3
 * 2. Find connected components via Union-Find (replaces OpenCV findContours)
 * 3. Compute bounding box for each component
 * 4. Scale boxes back to original image coordinates
 */
fun detTensorToBoxes(
    output: FloatArray,
    outputShape: LongArray,
    ratioH: Float,
    ratioW: Float,
    minBoxSize: Float = 3f,
    boxScoreThresh: Float = 0.3f,
    unclipRatio: Float = 1.5f
): List<TextBox> {
    // Output shape is [1, 1, H, W] - we need the HxW probability map
    val h = outputShape[2].toInt()
    val w = outputShape[3].toInt()
    
    // Step 1: Threshold to binary map
    val binaryMap = BooleanArray(h * w) { i ->
        output[i] >= boxScoreThresh
    }
    
    // Step 2: Find connected components using Union-Find
    val uf = UnionFind(h * w)
    
    // Connect horizontal neighbors
    for (y in 0 until h) {
        for (x in 1 until w) {
            val idx = y * w + x
            if (binaryMap[idx] && binaryMap[idx - 1]) {
                uf.union(idx, idx - 1)
            }
        }
    }
    
    // Connect vertical neighbors
    for (y in 1 until h) {
        for (x in 0 until w) {
            val idx = y * w + x
            if (binaryMap[idx] && binaryMap[(y - 1) * w + x]) {
                uf.union(idx, (y - 1) * w + x)
            }
        }
    }
    
    // Group pixels by component root
    val components = mutableMapOf<Int, MutableList<Pair<Int, Int>>>()
    for (y in 0 until h) {
        for (x in 0 until w) {
            val idx = y * w + x
            if (binaryMap[idx]) {
                val root = uf.find(idx)
                components.getOrPut(root) { mutableListOf() }.add(x to y)
            }
        }
    }
    
    // Step 3: Compute bounding boxes for each component
    val boxes = mutableListOf<TextBox>()
    
    for ((_, pixels) in components) {
        if (pixels.size < minBoxSize) continue
        
        var minX = Int.MAX_VALUE
        var minY = Int.MAX_VALUE
        var maxX = Int.MIN_VALUE
        var maxY = Int.MIN_VALUE
        var scoreSum = 0f
        
        for ((x, y) in pixels) {
            minX = minOf(minX, x)
            minY = minOf(minY, y)
            maxX = maxOf(maxX, x)
            maxY = maxOf(maxY, y)
            scoreSum += output[y * w + x]
        }
        
        val avgScore = scoreSum / pixels.size
        
        // Step 4: Unclip (expand box) - simplified version
        val boxW = (maxX - minX).toFloat()
        val boxH = (maxY - minY).toFloat()
        val clipDist = sqrt(boxW * boxH) * unclipRatio
        
        val expandX = clipDist * boxW / (boxW + boxH + 1e-6f)
        val expandY = clipDist * boxH / (boxW + boxH + 1e-6f)
        
        val x1 = (minX - expandX) * ratioW
        val y1 = (minY - expandY) * ratioH
        val x2 = (maxX + expandX) * ratioW
        val y2 = (maxY + expandY) * ratioH
        
        // Axis-aligned box as 4 corner points
        val points = listOf(
            x1 to y1,  // top-left
            x2 to y1,  // top-right
            x2 to y2,  // bottom-right
            x1 to y2   // bottom-left
        )
        
        boxes.add(TextBox(points, avgScore))
    }
    
    // Step 5: Sort by vertical position (top-to-bottom), then horizontal (left-to-right)
    return boxes.sortedWith(
        compareBy<TextBox> { it.points.minOf { p -> p.second } }
            .thenBy { it.points.minOf { p -> p.first } }
    )
}

// ============================================================================
// RECOGNITION PREPROCESSING
// ============================================================================

/**
 * Crop a region from bitmap and resize for recognition model.
 * - Target height: 48px
 * - Width: proportional, max 320px, multiple of 4
 * - Normalization: (pixel/255 - 0.5) / 0.5
 */
fun cropForRecognition(bitmap: Bitmap, box: TextBox): Bitmap {
    val ((x1, y1), (x2, y2)) = box.bounds
    
    // Clamp to bitmap bounds
    val left = maxOf(0, x1.toInt())
    val top = maxOf(0, y1.toInt())
    val right = minOf(bitmap.width, x2.toInt())
    val bottom = minOf(bitmap.height, y2.toInt())
    
    if (right <= left || bottom <= top) return Bitmap.createBitmap(1, 1, Bitmap.Config.ARGB_8888)
    
    val crop = Bitmap.createBitmap(bitmap, left, top, right - left, bottom - top)
    
    // Resize: height=48, width proportional (max 320), multiple of 4
    val targetH = 48
    val ratio = targetH.toFloat() / crop.height
    var targetW = (crop.width * ratio).toInt()
    targetW = minOf(targetW, 320)
    targetW = maxOf(4, (targetW / 4) * 4)
    
    return Bitmap.createScaledBitmap(crop, targetW, targetH, true)
}

/**
 * Convert bitmap to recognition model input tensor.
 * Output shape: [1, 3, 48, W]
 */
fun bitmapToRecTensor(bitmap: Bitmap): FloatArray {
    val h = bitmap.height
    val w = bitmap.width
    val pixels = IntArray(w * h)
    bitmap.getPixels(pixels, 0, w, 0, 0, w, h)
    
    val tensor = FloatArray(1 * 3 * h * w)
    var idx = 0
    
    for (c in 0 until 3) {
        for (y in 0 until h) {
            for (x in 0 until w) {
                val pixel = pixels[y * w + x]
                val value = when (c) {
                    0 -> Color.blue(pixel)
                    1 -> Color.green(pixel)
                    else -> Color.red(pixel)
                }
                // Normalize to [-1, 1]
                tensor[idx++] = (value / 255.0f - 0.5f) / 0.5f
            }
        }
    }
    
    return tensor
}

// ============================================================================
// RECOGNITION POSTPROCESSING (CTC Greedy Decode)
// ============================================================================

/**
 * CTC greedy decoding.
 * 
 * Output shape: [1, T, C] where T=sequence length, C=num classes
 * Rules:
 * - Class 0 is blank (skip)
 * - Remove consecutive duplicates
 * - Map remaining class indices to characters
 */
fun recTensorToText(output: FloatArray, outputShape: LongArray, keys: List<String>): String {
    val seqLen = outputShape[1].toInt()
    val numClasses = outputShape[2].toInt()
    
    val sb = StringBuilder()
    var lastCharIdx = -1
    
    for (t in 0 until seqLen) {
        // Find argmax for this timestep
        var maxIdx = 0
        var maxVal = Float.NEGATIVE_INFINITY
        
        for (c in 0 until numClasses) {
            val val_ = output[t * numClasses + c]
            if (val_ > maxVal) {
                maxVal = val_
                maxIdx = c
            }
        }
        
        // CTC rules
        if (maxIdx != 0 && maxIdx != lastCharIdx) {
            // Map to character (keys is 0-indexed but class 0 is blank, so keys[0] maps to class 1)
            if (maxIdx < keys.size) {
                sb.append(keys[maxIdx - 1])  // -1 because class 0 is blank
            }
        }
        lastCharIdx = maxIdx
    }
    
    return sb.toString()
}

/**
 * Load character dictionary from assets.
 * Format: one character per line (first line is usually the blank placeholder)
 */
fun loadDictionary(content: String): List<String> {
    return content.lines().filter { it.isNotEmpty() }
}
```

---

### CODE FILE 2: Rewrite `OnnxOcrEngine.kt`

```kotlin
// app/src/main/java/com/example/grocerytracker/ocr/OnnxOcrEngine.kt
package com.example.grocerytracker.ocr

import android.content.Context
import android.graphics.Bitmap
import android.graphics.ImageDecoder
import android.net.Uri
import android.os.Build
import android.provider.MediaStore
import android.util.Log
import com.example.grocerytracker.data.Item
import com.example.grocerytracker.data.Receipt
import ai.onnxruntime.OnnxTensor
import ai.onnxruntime.OrtEnvironment
import ai.onnxruntime.OrtException
import ai.onnxruntime.OrtSession
import java.io.BufferedReader
import java.io.InputStreamReader

data class ParsedReceipt(
    val receipt: Receipt,
    val items: List<Item>,
    val rawTextLines: List<String> = emptyList()  // For debugging
)

class OnnxOcrEngine(private val context: Context) {
    
    companion object {
        private const val TAG = "OnnxOcrEngine"
        private const val DET_MODEL = "ocr/det.onnx"
        private const val REC_MODEL = "ocr/rec.onnx"
        private const val KEYS_FILE = "ocr/keys.txt"
    }
    
    private var env: OrtEnvironment? = null
    private var detSession: OrtSession? = null
    private var recSession: OrtSession? = null
    private var keys: List<String> = emptyList()
    
    /**
     * Initialize ONNX runtime and load models.
     * Call this once before processing images.
     * Models are loaded from assets to internal storage (ONNX requires file path).
     */
    @Throws(OrtException::class)
    fun initialize() {
        if (env != null) return  // Already initialized
        
        Log.d(TAG, "Initializing ONNX Runtime...")
        
        env = OrtEnvironment.getEnvironment()
        
        // Copy models from assets to internal storage (ONNX needs a file path)
        val detPath = copyAssetToFile(DET_MODEL)
        val recPath = copyAssetToFile(REC_MODEL)
        
        // Load dictionary
        keys = loadDictionary(readAsset(KEYS_FILE))
        Log.d(TAG, "Loaded ${keys.size} character classes")
        
        // Create sessions
        val detOptions = OrtSession.SessionOptions().apply {
            setOptimizationLevel(OrtSession.SessionOptions.OptLevel.ALL_OPT)
            // Use NNAPI for hardware acceleration if available
            try { addNnapi() } catch (e: Exception) { Log.w(TAG, "NNAPI not available") }
        }
        
        val recOptions = OrtSession.SessionOptions().apply {
            setOptimizationLevel(OrtSession.SessionOptions.OptLevel.ALL_OPT)
            try { addNnapi() } catch (e: Exception) { Log.w(TAG, "NNAPI not available") }
        }
        
        detSession = env?.createSession(detPath, detOptions)
        recSession = env?.createSession(recPath, recOptions)
        
        Log.d(TAG, "ONNX sessions created successfully")
    }
    
    /**
     * Main OCR pipeline: URI -> ParsedReceipt
     */
    suspend fun processImage(imageUri: String): ParsedReceipt {
        try {
            initialize()
        } catch (e: OrtException) {
            Log.e(TAG, "Failed to initialize ONNX", e)
            return createFallbackReceipt("ONNX init failed: ${e.message}")
        }
        
        return try {
            // Step 1: Load bitmap from URI
            val bitmap = uriToBitmap(Uri.parse(imageUri))
            Log.d(TAG, "Loaded bitmap: ${bitmap.width}x${bitmap.height}")
            
            // Step 2: Run detection
            val textBoxes = runDetection(bitmap)
            Log.d(TAG, "Detected ${textBoxes.size} text regions")
            
            if (textBoxes.isEmpty()) {
                return createFallbackReceipt("No text detected")
            }
            
            // Step 3: Run recognition on each box
            val textLines = textBoxes.mapNotNull { box ->
                runRecognition(bitmap, box).also { text ->
                    Log.d(TAG, "Recognized: '$text' (score=${box.score})")
                }.ifBlank { null }
            }
            
            Log.d(TAG, "Final text lines: $textLines")
            
            // Step 4: Parse receipt structure from text lines
            parseReceiptFromText(textLines)
            
        } catch (e: Exception) {
            Log.e(TAG, "OCR processing failed", e)
            createFallbackReceipt("Processing error: ${e.message}")
        }
    }
    
    /**
     * Run text detection model.
     * Returns list of text bounding boxes.
     */
    private fun runDetection(bitmap: Bitmap): List<TextBox> {
        val session = detSession ?: return emptyList()
        val environment = env ?: return emptyList()
        
        // Preprocess
        val (resized, ratioH, ratioW) = resizeForDetection(bitmap)
        val tensorData = bitmapToDetTensor(resized)
        resized.recycle()
        
        val h = bitmap.height // Use original for ratio calc... wait, we already have ratios
        // Actually ratioH/W are already computed correctly in resizeForDetection
        
        // Create input tensor [1, 3, H, W]
        val shape = longArrayOf(1, 3, ratioH.toInt().toLong(), ratioW.toInt().toLong())
        // Wait, shape should use resized dimensions, not ratios
        // Let me fix: resizeForDetection returns the resized bitmap, we need its dimensions
        // Actually I made an error - let me recalculate
        
        // The tensor was created from resized bitmap, so shape should match
        // But I don't have resized dimensions here... let me fix the ResizeResult
        // TODO: Fix this properly
        
        // WORKAROUND: Re-derive from tensor size
        val tensorSize = tensorData.size
        val channels = 3
        val pixelCount = tensorSize / channels
        val resizedH = (bitmap.height.toFloat() / ratioH).toInt()
        val resizedW = (bitmap.width.toFloat() / ratioW).toInt()
        val correctShape = longArrayOf(1, 3, resizedH.toLong(), resizedW.toLong())
        
        return environment.createTensor(shape = correctShape, data = tensorData).use { inputTensor ->
            val inputs = mapOf(session.inputNames.first() to inputTensor)
            
            session.run(inputs).use { results ->
                val outputName = session.outputNames.first()
                val outputTensor = results[outputName] as? OnnxTensor ?: return emptyList()
                val outputData = outputTensor.floatBuffer.array()
                val outputShape = outputTensor.info.shape
                
                detTensorToBoxes(
                    output = outputData,
                    outputShape = outputShape,
                    ratioH = ratioH,
                    ratioW = ratioW
                )
            }
        }
    }
    
    /**
     * Run text recognition on a single cropped region.
     */
    private fun runRecognition(bitmap: Bitmap, box: TextBox): String {
        val session = recSession ?: return ""
        val environment = env ?: return ""
        if (keys.isEmpty()) return ""
        
        // Preprocess: crop and resize
        val crop = cropForRecognition(bitmap, box)
        val tensorData = bitmapToRecTensor(crop)
        
        val h = crop.height
        val w = crop.width
        val shape = longArrayOf(1, 3, h.toLong(), w.toLong())
        
        return environment.createTensor(shape = shape, data = tensorData).use { inputTensor ->
            val inputs = mapOf(session.inputNames.first() to inputTensor)
            
            session.run(inputs).use { results ->
                val outputName = session.outputNames.first()
                val outputTensor = results[outputName] as? OnnxTensor ?: return ""
                val outputData = outputTensor.floatBuffer.array()
                val outputShape = outputTensor.info.shape
                
                recTensorToText(outputData, outputShape, keys)
            }
        }.also { crop.recycle() }
    }
    
    // =========================================================================
    // RECEIPT PARSING (Heuristic-based)
    // =========================================================================
    
    /**
     * Parse detected text lines into structured Receipt + Items.
     * 
     * This is a simple heuristic parser. Real receipts vary wildly,
     * so this will have edge cases. The "Review & Edit" screen will
     * let users fix mistakes.
     */
    private fun parseReceiptFromText(lines: List<String>): ParsedReceipt {
        val items = mutableListOf<Item>()
        var storeName = "Unknown Store"
        var total = 0.0
        
        for (line in lines) {
            val trimmed = line.trim()
            if (trimmed.isBlank()) continue
            
            // Try to detect store name (usually first non-empty line)
            if (storeName == "Unknown Store" && !trimmed.matches(Regex("^\\d"))) {
                storeName = trimmed
                continue
            }
            
            // Try to detect total line
            if (trimmed.contains("total", ignoreCase = true) || 
                trimmed.contains("合计", ignoreCase = true) ||
                trimmed.contains("总计", ignoreCase = true)) {
                val priceMatch = Regex("""[\$¥€£]?\s*([\d,]+\.\d{2})""").find(trimmed)
                priceMatch?.let {
                    total = it.groupValues[1].replace(",", "").toDoubleOrNull() ?: 0.0
                }
                continue
            }
            
            // Try to parse as item line: "Item Name  12.99" or "Item Name x2  12.99"
            val itemRegex = Regex("""^(.+?)\s+(?:x\d+\s+)?[\$¥€£]?\s*([\d,]+\.\d{2})$""")
            val match = itemRegex.find(trimmed)
            if (match != null) {
                val name = match.groupValues[1].trim()
                val price = match.groupValues[2].replace(",", "").toDoubleOrNull()
                if (price != null && price > 0) {
                    items.add(Item(
                        receiptId = 0,  // Will be set by caller
                        name = name,
                        price = price,
                        category = guessCategory(name)
                    ))
                }
            }
        }
        
        // If no total found, sum up items
        if (total == 0.0 && items.isNotEmpty()) {
            total = items.sumOf { it.price }
        }
        
        return ParsedReceipt(
            receipt = Receipt(
                storeName = storeName,
                total = total
            ),
            items = items,
            rawTextLines = lines
        )
    }
    
    /**
     * Simple category guessing based on keywords.
     */
    private fun guessCategory(name: String): String {
        val lower = name.lowercase()
        return when {
            lower.contains("apple") || lower.contains("banana") || lower.contains("orange") ||
            lower.contains("fruit") || lower.contains("vegetable") || lower.contains("tomato") ||
            lower.contains("onion") || lower.contains("potato") -> "Produce"
            
            lower.contains("milk") || lower.contains("cheese") || lower.contains("yogurt") ||
            lower.contains("butter") || lower.contains("cream") -> "Dairy"
            
            lower.contains("bread") || lower.contains("cake") || lower.contains("cookie") ||
            lower.contains("pastry") || lower.contains("bakery") -> "Bakery"
            
            lower.contains("chicken") || lower.contains("beef") || lower.contains("pork") ||
            lower.contains("meat") || lower.contains("fish") || lower.contains("seafood") -> "Meat/Seafood"
            
            lower.contains("water") || lower.contains("juice") || lower.contains("soda") ||
            lower.contains("beer") || lower.contains("wine") || lower.contains("drink") -> "Beverages"
            
            lower.contains("chip") || lower.contains("snack") || lower.contains("cracker") ||
            lower.contains("nut") -> "Snacks"
            
            else -> "Other"
        }
    }
    
    // =========================================================================
    // UTILITIES
    // =========================================================================
    
    private fun createFallbackReceipt(reason: String): ParsedReceipt {
        Log.w(TAG, "Creating fallback receipt: $reason")
        return ParsedReceipt(
            receipt = Receipt(storeName = "OCR Error", total = 0.0),
            items = emptyList(),
            rawTextLines = listOf("[Error: $reason]")
        )
    }
    
    private fun uriToBitmap(uri: Uri): Bitmap {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            ImageDecoder.decodeBitmap(ImageDecoder.createSource(context.contentResolver, uri))
        } else {
            @Suppress("DEPRECATION")
            MediaStore.Images.Media.getBitmap(context.contentResolver, uri)
        }
    }
    
    private fun copyAssetToFile(assetPath: String): String {
        val file = File(context.filesDir, assetPath.substringAfterLast("/"))
        file.parentFile?.mkdirs()
        
        if (file.exists()) return file.absolutePath
        
        context.assets.open(assetPath).use { input ->
            file.outputStream().use { output ->
                input.copyTo(output)
            }
        }
        return file.absolutePath
    }
    
    private fun readAsset(assetPath: String): String {
        return context.assets.open(assetPath).bufferedReader().use { it.readText() }
    }
    
    private fun OrtEnvironment.createTensor(shape: LongArray, data: FloatArray): OnnxTensor {
        return OnnxTensor.createTensor(this, java.nio.FloatBuffer.wrap(data), shape)
    }
    
    /**
     * Clean up native resources.
     */
    fun close() {
        try {
            detSession?.close()
            recSession?.close()
            // Don't close env - it's a singleton
        } catch (e: OrtException) {
            Log.e(TAG, "Error closing sessions", e)
        }
        detSession = null
        recSession = null
    }
}
```

---

## Wait - There's a Bug I Spotted

In `runDetection()`, I have messy code trying to reconstruct the shape. Let me fix `OcrUtils.kt`'s `ResizeResult`:

```kotlin
// In OcrUtils.kt, update ResizeResult:
data class ResizeResult(
    val bitmap: Bitmap,
    val ratioH: Float,
    val ratioW: Float,
    val resizedHeight: Int,  // ADD THIS
    val resizedWidth: Int    // ADD THIS
)

// Update resizeForDetection to return these:
fun resizeForDetection(bitmap: Bitmap, maxSide: Int = 960): ResizeResult {
    val srcH = bitmap.height
    val srcW = bitmap.width
    val ratio = minOf(maxSide.toFloat() / maxOf(srcH, srcW), 1.0f)
    
    val resizeH = max(32, ((srcH * ratio).toInt() / 32) * 32)
    val resizeW = max(32, ((srcW * ratio).toInt() / 32) * 32)
    
    val resized = Bitmap.createScaledBitmap(bitmap, resizeW, resizeH, true)
    return ResizeResult(
        bitmap = resized,
        ratioH = srcH.toFloat() / resizeH,
        ratioW = srcW.toFloat() / resizeW,
        resizedHeight = resizeH,  // ADD
        resizedWidth = resizeW    // ADD
    )
}
```

Then fix `runDetection()` in `OnnxOcrEngine.kt` - replace the messy part with:

```kotlin
private fun runDetection(bitmap: Bitmap): List<TextBox> {
    val session = detSession ?: return emptyList()
    val environment = env ?: return emptyList()
    
    // Preprocess
    val (resized, ratioH, ratioW, resizedH, resizedW) = resizeForDetection(bitmap)
    val tensorData = bitmapToDetTensor(resized)
    resized.recycle()
    
    // Create input tensor [1, 3, H, W]
    val shape = longArrayOf(1, 3, resizedH.toLong(), resizedW.toLong())
    
    return environment.createTensor(shape = shape, data = tensorData).use { inputTensor ->
        val inputs = mapOf(session.inputNames.first() to inputTensor)
        
        session.run(inputs).use { results ->
            val outputName = session.outputNames.first()
            val outputTensor = results[outputName] as? OnnxTensor ?: return emptyList()
            val outputData = outputTensor.floatBuffer.array()
            val outputShape = outputTensor.info.shape
            
            detTensorToBoxes(
                output = outputData,
                outputShape = outputShape,
                ratioH = ratioH,
                ratioW = ratioW
            )
        }
    }
}
```

---

## Testing Checklist

After placing assets and building:

1. **Single image via Gallery** → Should trigger WorkManager → Notification → Check logcat for OCR output
2. **Logcat filter**: `adb logcat -s OnnxOcrEngine:D` to see detection/recognition logs
3. **Expected flow**:
   - "Initializing ONNX Runtime..."
   - "Loaded 6625 character classes"  
   - "Loaded bitmap: 3000x4000"
   - "Detected X text regions"
   - "Recognized: 'Some text' (score=0.95)"
   - "Final text lines: [...]"

## Likely Issues to Watch For:

| Symptom | Likely Cause |
|---------|--------------|
| `FileNotFoundException` on model load | Assets not in correct path, or filename mismatch |
| `OrtException: shape mismatch` | Model input name different from `inputNames.first()` |
| Empty detection results | Image too small, or threshold too high |
| Garbage text recognition | Wrong normalization (BGR vs RGB), or keys file encoding issue |

Let me know what happens when you run it!


----

> Task :app:kaptGenerateStubsDebugKotlin FAILED
ocr/OcrUtils.kt:249:10 Expecting a name
ocr/OcrUtils.kt:249:11 Expecting ')'
ocr/OcrUtils.kt:249:13 Unexpected tokens (use ';' to separate expressions on the same line)


> Task :app:compileDebugKotlin FAILED
ocr/OnnxOcrEngine.kt:307:20 Unresolved reference: File
ocr/OnnxOcrEngine.kt:313:39 Cannot infer a type for this parameter. Please specify it explicitly.

FAILURE: Build failed with an exception.

--

04-17 12:48:02.784 22423 24917 E OnnxOcrEngine: ai.onnxruntime.OrtException: Error code - ORT_FAIL - message: Load model from /data/user/0/com.example.grocerytracker/files/det.onnx failed:/onnxruntime_src/onnxruntime/core/graph/model.cc:150 onnxruntime::Model::Model(onnx::ModelProto &&, const onnxruntime::PathString &, const onnxruntime::IOnnxRuntimeOpSchemaRegistryList *, const logging::Logger &, const onnxruntime::ModelOptions &) Unsupported model IR version: 10, max supported IR version: 9
04-17 12:48:02.784 22423 24917 E OnnxOcrEngine: 
04-17 12:48:02.784 22423 24917 E OnnxOcrEngine:         at ai.onnxruntime.OrtSession.createSession(Native Method)

--

Okay this is getting really hard (which I totally expected - it seems edge/mobile AI inference is in general quite immature, poorly supported, etc). Summary of what happened:
- First I fixed some syntax error on your behalf (old Kotlin version doesn't seem to support nested destructuring bind, so I do it twice/three times; you forgot to import java File class)
- The model seems incompatible. But to be honest, the model links you provided didn't work, and it come as a surprise to me that PaddleOCR model comes it their own format, and ONNX exported format is almost non-existent on the web (!) (they say "please convert it yourself") (compare with eg LLM, there's tons of both community and official ONNX version of the models). I located one that a community member did the conversion (https://huggingface.co/marsena/paddleocr-onnx-models , PP-OCRv5_server_det_infer.onnx	and PP-OCRv5_server_rec_infer.onnx respectively).

Now what?

----

You're absolutely right - mobile ML is painful. The ecosystem is fragmented, docs are sparse, and version incompatibilities are everywhere. This is a real "native shift" that web devs don't expect.

## The Fix: Upgrade ONNX Runtime

The error is clear: your model is IR version 10, ORT 1.16.3 only supports up to 9.

**MANUAL STEP: Update `build.gradle`**

```kotlin
// Change this:
implementation("com.microsoft.onnxruntime:onnxruntime-android:1.16.3")

// To this:
implementation("com.microsoft.onnxruntime:onnxruntime-android:1.19.2")
```

Then sync gradle: `./gradlew --stop && ./gradlew assembleDebug`

---

## Code Fixes

### Fix 1: `OcrUtils.kt` - The destructuring issue

Find `recTensorToText` function and fix line ~249. Change:

```kotlin
// OLD (broken in older Kotlin):
val (x1, y1), (x2, y2) = box.bounds
```

To:
```kotlin
// NEW:
val topLeft = box.bounds.first
val bottomRight = box.bounds.second
val x1 = topLeft.first
val y1 = topLeft.second
val x2 = bottomRight.first
val y2 = bottomRight.second
```

### Fix 2: `OnnxOcrEngine.kt` - Missing import + destructuring

Add import at top:
```kotlin
import java.io.File
```

Fix `runDetection()` - change:
```kotlin
// OLD (nested destructuring doesn't work):
val (resized, ratioH, ratioW, resizedH, resizedW) = resizeForDetection(bitmap)
```

To:
```kotlin
// NEW:
val resizeResult = resizeForDetection(bitmap)
val resized = resizeResult.bitmap
val ratioH = resizeResult.ratioH
val ratioW = resizeResult.ratioW
val resizedH = resizeResult.resizedHeight
val resizedW = resizeResult.resizedWidth
```

---

## Potential Issue: PP-OCRv5 Server Model Specs

The "server" models you found are larger/slower than mobile variants, and **might have different input/output specs**. If you get a shape mismatch error after upgrading ORT, add this debug logging to `runDetection()` and `runRecognition()` right before creating the tensor:

```kotlin
// Add before environment.createTensor() in runDetection():
Log.d(TAG, "Det input name: ${session.inputNames.first()}")
Log.d(TAG, "Det input shape expected: [1, 3, $resizedH, $resizedW]")

// Add in runDetection() after session.run():
Log.d(TAG, "Det output name: $outputName")
Log.d(TAG, "Det output shape: ${outputShape.toList()}")

// Add before environment.createTensor() in runRecognition():
Log.d(TAG, "Rec input shape expected: [1, 3, ${crop.height}, ${crop.width}]")

// Add in runRecognition() after session.run():
Log.d(TAG, "Rec output shape: ${outputShape.toList()}")
```

Run it and share the logcat output if shapes are wrong. PP-OCRv5 might expect different dimensions (e.g., 640 instead of 960 for detection, or 64 height instead of 48 for recognition).

---

## If All Else Fails: Nuclear Option

If the v5 models have completely incompatible specs and you're spending too much time on this, there's an alternative path:

**Tesseract OCR** (`com.google.android.gms:play-services-mlkit-text-recognition` is ML Kit you already have, but raw `tess-two` library gives more control):

- Pros: Well-documented, stable, pure Java/C++ bindings, no ONNX versioning hell
- Cons: Older architecture, not as accurate as modern deep learning models

But let's try the ORT upgrade first. The version mismatch is almost certainly the only blocker.


----

D OnnxOcrEngine: Initializing ONNX Runtime...
D OnnxOcrEngine: Loaded 18426 character classes
D OnnxOcrEngine: ONNX sessions created successfully
D OnnxOcrEngine: Loaded bitmap: 1080x2400
E OnnxOcrEngine: OCR processing failed
E OnnxOcrEngine: java.lang.IllegalStateException: unable to getPixels(), pixel access is not supported on Config#HARDWARE bitmaps
E OnnxOcrEngine:         at android.graphics.Bitmap.checkHardware(Bitmap.java:444)
E OnnxOcrEngine:         at android.graphics.Bitmap.getPixels(Bitmap.java:2353)
E OnnxOcrEngine:         at com.example.grocerytracker.ocr.OcrUtilsKt.bitmapToDetTensor(OcrUtils.kt:73)
E OnnxOcrEngine:         at com.example.grocerytracker.ocr.OnnxOcrEngine.runDetection(OnnxOcrEngine.kt:133)
E OnnxOcrEngine:         at com.example.grocerytracker.ocr.OnnxOcrEngine.processImage(OnnxOcrEngine.kt:98)
E OnnxOcrEngine:         at com.example.grocerytracker.worker.OcrWorker.doWork(OcrWorker.kt:31)

huh...

----

Classic Android gotcha! Hardware bitmaps live in GPU memory and can't be read with `getPixels()`. Web devs never think about this because canvas/browser handles it transparently.

## The Fix

In `OnnxOcrEngine.kt`, update `uriToBitmap()` to force a software bitmap:

```kotlin
private fun uriToBitmap(uri: Uri): Bitmap {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
        ImageDecoder.decodeBitmap(
            ImageDecoder.createSource(context.contentResolver, uri)
        ) { decoder, _, _ ->
            // KEY FIX: Force software allocation
            // Hardware bitmaps (default) don't support getPixels()
            decoder.allocator = ImageDecoder.ALLOCATOR_SOFTWARE
        }
    } else {
        @Suppress("DEPRECATION")
        MediaStore.Images.Media.getBitmap(context.contentResolver, uri)
    }
}
```

The only change is adding that lambda with `decoder.allocator = ImageDecoder.ALLOCATOR_SOFTWARE`.

---

## Why This Happens (The "Native Shift" Lesson)

```
Web:     <img src="x"> → canvas.getImageData() → Just Works™
Android: URI → ImageDecoder → HARDWARE bitmap → getPixels() → 💥
```

Android 8+ defaults to `HARDWARE` bitmaps because they're faster for rendering (GPU-backed). But ML inference needs CPU-accessible memory. The `ALLOCATOR_SOFTWARE` flag tells Android to allocate in RAM instead of VRAM.

Trade-off: Software bitmaps use more RAM and are slightly slower to render, but we're not rendering them anyway—we're immediately converting to a float tensor and discarding the bitmap.

---

Should compile and run clean after this. You should start seeing actual OCR output in logcat!

----

Okay, the pipeline seems to be able to run to completion:

D OnnxOcrEngine: Loaded bitmap: 3809x3024
D OnnxOcrEngine: Detected 0 text regions
W OnnxOcrEngine: Creating fallback receipt: No text detected

Which is... frustrating... I mean, the "sales and marketing" material I saw for PaddleOCR sell it as being powerful, versatile, robust, and accurate. (Also, my receipt have Chinese character, have somewhat yellow lighting - but it's well within the range of "clean and easily recognizable" for a human)

Is there some way to visualize things to debug?

----

Yes! This is the right instinct. In ML pipelines, you need to visualize intermediate results. Let's add debug output that saves images you can pull with `adb`.

## Create Debug Utility

```kotlin
// app/src/main/java/com/example/grocerytracker/ocr/DebugUtils.kt
package com.example.grocerytracker.ocr

import android.content.Context
import android.graphics.Bitmap
import android.graphics.Color
import android.util.Log
import java.io.File

object DebugUtils {
    private const val TAG = "OCR_DEBUG"
    
    /**
     * Save bitmap to app's files directory for adb pull.
     * Usage: adb pull /data/user/0/com.example.grocerytracker/files/ocr_debug/
     */
    fun saveBitmap(context: Context, bitmap: Bitmap, filename: String) {
        try {
            val dir = File(context.filesDir, "ocr_debug")
            dir.mkdirs()
            val file = File(dir, filename)
            file.outputStream().use { out ->
                bitmap.compress(Bitmap.CompressFormat.PNG, 100, out)
            }
            Log.d(TAG, "Saved: ${file.absolutePath} (${bitmap.width}x${bitmap.height})")
        } catch (e: Exception) {
            Log.e(TAG, "Failed to save $filename", e)
        }
    }
    
    /**
     * Convert a float tensor (NCHW format) to a grayscale bitmap for visualization.
     * Useful for probability maps, heatmaps, etc.
     * 
     * @param channel Which channel to visualize (default 0)
     * @param clamp Min/max values to map to 0-255 (auto-calculated if null)
     */
    fun saveTensorAsImage(
        context: Context,
        data: FloatArray,
        shape: LongArray,
        filename: String,
        channel: Int = 0,
        clamp: Pair<Float, Float>? = null
    ) {
        // NCHW format
        val c = shape[1].toInt()
        val h = shape[2].toInt()
        val w = shape[3].toInt()
        
        if (channel >= c) {
            Log.w(TAG, "Channel $channel out of range (max $c)")
            return
        }
        
        // Find min/max if not specified
        val (minVal, maxVal) = clamp ?: run {
            var min = Float.MAX_VALUE
            var max = Float.MIN_VALUE
            for (y in 0 until h) {
                for (x in 0 until w) {
                    val idx = channel * h * w + y * w + x
                    val v = data[idx]
                    if (v < min) min = v
                    if (v > max) max = v
                }
            }
            Log.d(TAG, "$filename channel $channel: min=$min, max=$max")
            min to max
        }
        
        val range = maxVal - minVal
        val pixels = IntArray(h * w)
        
        for (y in 0 until h) {
            for (x in 0 until w) {
                val idx = channel * h * w + y * w + x
                val normalized = if (range > 0) (data[idx] - minVal) / range else 0f
                val gray = (normalized * 255).toInt().coerceIn(0, 255)
                pixels[y * w + x] = Color.rgb(gray, gray, gray)
            }
        }
        
        val bitmap = Bitmap.createBitmap(w, h, Bitmap.Config.ARGB_8888)
        bitmap.setPixels(pixels, 0, w, 0, 0, w, h)
        saveBitmap(context, bitmap, filename)
        bitmap.recycle()
    }
    
    /**
     * Save a binary map (BooleanArray) as black/white image.
     */
    fun saveBinaryMap(
        context: Context,
        binaryMap: BooleanArray,
        width: Int,
        height: Int,
        filename: String
    ) {
        val pixels = IntArray(width * height)
        for (i in binaryMap.indices) {
            pixels[i] = if (binaryMap[i]) Color.WHITE else Color.BLACK
        }
        val bitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888)
        bitmap.setPixels(pixels, 0, width, 0, 0, width, height)
        saveBitmap(context, bitmap, filename)
        bitmap.recycle()
    }
}
```

---

## Add Debug Calls to `OcrUtils.kt`

In `detTensorToBoxes()`, add a debug variant or insert calls. Here's the updated function:

```kotlin
fun detTensorToBoxes(
    output: FloatArray,
    outputShape: LongArray,
    ratioH: Float,
    ratioW: Float,
    minBoxSize: Float = 3f,
    boxScoreThresh: Float = 0.3f,
    unclipRatio: Float = 1.5f,
    // DEBUG PARAMS
    context: Context? = null,
    debugPrefix: String = "det"
): List<TextBox> {
    val h = outputShape[2].toInt()
    val w = outputShape[3].toInt()
    
    Log.d("OCR_DEBUG", "Detection output shape: ${outputShape.toList()}")
    Log.d("OCR_DEBUG", "Output size: ${output.size}, expected: ${h * w}")
    
    // DEBUG: Save raw probability map
    context?.let {
        DebugUtils.saveTensorAsImage(
            it, output, outputShape,
            "${debugPrefix}_1_raw_probmap.png",
            channel = 0
        )
    }
    
    // Step 1: Threshold to binary map
    val binaryMap = BooleanArray(h * w) { i ->
        output[i] >= boxScoreThresh
    }
    
    // DEBUG: Save binary map
    context?.let {
        DebugUtils.saveBinaryMap(it, binaryMap, w, h, "${debugPrefix}_2_binary.png")
    }
    
    // Count how many pixels passed threshold
    val trueCount = binaryMap.count { it }
    Log.d("OCR_DEBUG", "Binary map: $trueCount / ${h * w} pixels above threshold $boxScoreThresh")
    
    // ... rest of function unchanged ...
```

---

## Add Debug Calls to `OnnxOcrEngine.kt`

Update `runDetection()`:

```kotlin
private fun runDetection(bitmap: Bitmap): List<TextBox> {
    val session = detSession ?: return emptyList()
    val environment = env ?: return emptyList()
    
    // Preprocess
    val resizeResult = resizeForDetection(bitmap)
    val resized = resizeResult.bitmap
    val ratioH = resizeResult.ratioH
    val ratioW = resizeResult.ratioW
    val resizedH = resizeResult.resizedHeight
    val resizedW = resizeResult.resizedWidth
    
    // DEBUG: Save resized input
    DebugUtils.saveBitmap(context, resized, "det_0_resized_input.png")
    
    val tensorData = bitmapToDetTensor(resized)
    
    // DEBUG: Verify tensor values
    var tensorMin = Float.MAX_VALUE
    var tensorMax = Float.MIN_VALUE
    for (v in tensorData) {
        if (v < tensorMin) tensorMin = v
        if (v > tensorMax) tensorMax = v
    }
    Log.d("OCR_DEBUG", "Input tensor range: $tensorMin to $tensorMax (expected: -1.0 to 1.0)")
    
    resized.recycle()
    
    val shape = longArrayOf(1, 3, resizedH.toLong(), resizedW.toLong())
    
    Log.d("OCR_DEBUG", "Det model input name: ${session.inputNames.first()}")
    Log.d("OCR_DEBUG", "Det model input shape: ${shape.toList()}")
    
    return environment.createTensor(shape = shape, data = tensorData).use { inputTensor ->
        val inputs = mapOf(session.inputNames.first() to inputTensor)
        
        session.run(inputs).use { results ->
            val outputName = session.outputNames.first()
            Log.d("OCR_DEBUG", "Det model output name: $outputName")
            
            val outputTensor = results[outputName] as? OnnxTensor ?: return emptyList()
            val outputData = outputTensor.floatBuffer.array()
            val outputShape = outputTensor.info.shape
            
            detTensorToBoxes(
                output = outputData,
                outputShape = outputShape,
                ratioH = ratioH,
                ratioW = ratioW,
                context = context,  // DEBUG
                debugPrefix = "det"
            )
        }
    }
}
```

---

## Run and Pull Debug Images

```bash
# Build and install
./gradlew installDebug

# Trigger OCR (pick an image)

# Pull debug images
adb pull /data/user/0/com.example.grocerytracker/files/ocr_debug/ ./ocr_debug/

# List what you got
ls -la ./ocr_debug/
```

You should get:
```
det_0_resized_input.png    - What the model sees (resized, but NOT color-normalized)
det_1_raw_probmap.png      - Raw model output (grayscale heatmap)
det_2_binary.png           - After thresholding (black/white)
```

---

## What to Look For

| File | Expected | Problem Indicators |
|------|----------|-------------------|
| `resized_input` | Receipt visible, maybe squished | Completely black/white = normalization bug |
| `raw_probmap` | Bright spots where text is | All black = model outputting near-zero; All white = outputting near-one; Random noise = wrong model/input |
| `binary` | White blobs on text lines | All black = threshold too high or probmap all low |

Share what you see (or the min/max values from logcat) and we can diagnose from there!

----

Okay, I've got some permission error when trying to pull, workaround is adb shell "run-as com.example.grocerytracker cp ... /sdcard/Download/..." then pull from that instead. Result: resized_input look normal and legible. But the other two files does not exists.

Quick remark: I also added:
import android.util.Log
import android.content.Context
As missing import on OcrUtils.kt

Also, even with those added Log.d lines, how can I see them? adb logcat -s OnnxOcrEngine:D doesn't show... perhaps I need to change class name in the filter? It seems tedious (ya I know I am complaining...)

----

Two issues:

## 1. Logcat Filter

I used tag `OCR_DEBUG` but you're filtering for `OnnxOcrEngine`. Use:

```bash
adb logcat -s OCR_DEBUG:D OnnxOcrEngine:D
```

Or even simpler, just grep for anything we log:

```bash
adb logcat | grep -i "ocr\|debug\|det"
```

## 2. Missing Debug Images = Silent Crash

If `resized_input` saved but the others didn't, the code is probably throwing an exception inside `session.run()` or when accessing the output tensor, and it's being swallowed somewhere.

Let me add bulletproof logging. Update `runDetection()` in `OnnxOcrEngine.kt`:

```kotlin
private fun runDetection(bitmap: Bitmap): List<TextBox> {
    val session = detSession ?: run { Log.e(TAG, "detSession is null!"); return emptyList() }
    val environment = env ?: run { Log.e(TAG, "env is null!"); return emptyList() }
    
    val resizeResult = resizeForDetection(bitmap)
    val resized = resizeResult.bitmap
    val ratioH = resizeResult.ratioH
    val ratioW = resizeResult.ratioW
    val resizedH = resizeResult.resizedHeight
    val resizedW = resizeResult.resizedWidth
    
    Log.d(TAG, "=== DETECTION START ===")
    Log.d(TAG, "Input: ${resizedW}x${resizedH}, ratioH=$ratioH, ratioW=$ratioW")
    
    DebugUtils.saveBitmap(context, resized, "det_0_resized_input.png")
    
    val tensorData = bitmapToDetTensor(resized)
    resized.recycle()
    
    val shape = longArrayOf(1, 3, resizedH.toLong(), resizedW.toLong())
    
    Log.d(TAG, "Model input names: ${session.inputNames.toList()}")
    Log.d(TAG, "Model output names: ${session.outputNames.toList()}")
    
    try {
        val inputTensor = environment.createTensor(shape = shape, data = tensorData)
        val inputs = mapOf(session.inputNames.first() to inputTensor)
        
        Log.d(TAG, "Running inference...")
        val results = session.run(inputs)
        Log.d(TAG, "Inference complete!")
        
        val outputName = session.outputNames.first()
        Log.d(TAG, "Output name: $outputName")
        
        val rawOutput = results[outputName]
        Log.d(TAG, "Output type: ${rawOutput?.javaClass?.simpleName}")
        
        if (rawOutput == null) {
            Log.e(TAG, "Output tensor is null!")
            results.close()
            inputTensor.close()
            return emptyList()
        }
        
        val outputTensor = rawOutput as OnnxTensor
        val outputInfo = outputTensor.info
        Log.d(TAG, "Output shape: ${outputInfo.shape.toList()}")
        Log.d(TAG, "Output type: ${outputInfo.type}")
        
        val outputData = outputTensor.floatBuffer.array()
        Log.d(TAG, "Output data length: ${outputData.size}")
        
        // Quick sample of output values
        val sampleSize = minOf(10, outputData.size)
        val sample = outputData.take(sampleSize).map { "%.4f".format(it) }
        Log.d(TAG, "Output sample: $sample")
        
        // Save raw probmap regardless of shape
        DebugUtils.saveTensorAsImage(
            context, outputData, outputInfo.shape,
            "det_1_raw_probmap.png", channel = 0
        )
        
        val boxes = detTensorToBoxes(
            output = outputData,
            outputShape = outputInfo.shape,
            ratioH = ratioH,
            ratioW = ratioW,
            context = context
        )
        
        Log.d(TAG, "=== DETECTION END: ${boxes.size} boxes ===")
        
        results.close()
        inputTensor.close()
        return boxes
        
    } catch (e: Exception) {
        Log.e(TAG, "DETECTION FAILED", e)
        Log.e(TAG, "Exception: ${e.javaClass.simpleName}: ${e.message}")
        e.stackTrace?.take(5)?.forEach { Log.e(TAG, "  at $it") }
        return emptyList()
    }
}
```

---

Now run again and check logcat:

```bash
adb logcat | grep -E "(OCR_DEBUG|OnnxOcrEngine|DETECTION)"
```

The key lines to look for:
- `Output shape: [?, ?, ?, ?]` - tells us actual model output format
- `Output sample: [...]` - tells us if values are reasonable (0-1 range for probabilities)
- `DETECTION FAILED` - if there's an exception

Share those log lines!


----

Okay...

D OnnxOcrEngine: Initializing ONNX Runtime...
D OnnxOcrEngine: Loaded 18426 character classes
D OnnxOcrEngine: ONNX sessions created successfully
D OnnxOcrEngine: Loaded bitmap: 3809x3024
D OCR_DEBUG: Saved: /data/user/0/com.example.grocerytracker/files/ocr_debug/det_0_resized_input.png (960x736)
D OCR_DEBUG: Input tensor range: -1.0 to 1.0 (expected: -1.0 to 1.0)
D OCR_DEBUG: Det model input name: x
D OCR_DEBUG: Det model input shape: [1, 3, 736, 960]
D OnnxOcrEngine: Model input names: [x]
D OnnxOcrEngine: Model output names: [fetch_name_0]
D OnnxOcrEngine: Running inference...
D OnnxOcrEngine: Inference complete!
D OnnxOcrEngine: Output name: fetch_name_0
D OnnxOcrEngine: Output type: Optional
E OnnxOcrEngine: DETECTION FAILED
E OnnxOcrEngine: java.lang.ClassCastException: java.util.Optional cannot be cast to ai.onnxruntime.OnnxTensor
E OnnxOcrEngine:         at com.example.grocerytracker.ocr.OnnxOcrEngine.runDetection(OnnxOcrEngine.kt:201)
E OnnxOcrEngine:         at com.example.grocerytracker.ocr.OnnxOcrEngine.processImage(OnnxOcrEngine.kt:98)
E OnnxOcrEngine:         at com.example.grocerytracker.worker.OcrWorker.doWork(OcrWorker.kt:31)
E OnnxOcrEngine:         at androidx.work.CoroutineWorker$startWork$1.invokeSuspend(CoroutineWorker.kt:68)
E OnnxOcrEngine:         at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
E OnnxOcrEngine:         at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:106)
E OnnxOcrEngine:         at kotlinx.coroutines.scheduling.CoroutineScheduler.runSafely(CoroutineScheduler.kt:584)
E OnnxOcrEngine:         at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.executeTask(CoroutineScheduler.kt:793)
E OnnxOcrEngine:         at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.runWorker(CoroutineScheduler.kt:697)
E OnnxOcrEngine:         at kotlinx.coroutines.scheduling.CoroutineScheduler$Worker.run(CoroutineScheduler.kt:684)
E OnnxOcrEngine: Exception: ClassCastException: java.util.Optional cannot be cast to ai.onnxruntime.OnnxTensor
E OnnxOcrEngine:   at com.example.grocerytracker.ocr.OnnxOcrEngine.runDetection(OnnxOcrEngine.kt:201)
E OnnxOcrEngine:   at com.example.grocerytracker.ocr.OnnxOcrEngine.processImage(OnnxOcrEngine.kt:98)
E OnnxOcrEngine:   at com.example.grocerytracker.worker.OcrWorker.doWork(OcrWorker.kt:31)
E OnnxOcrEngine:   at androidx.work.CoroutineWorker$startWork$1.invokeSuspend(CoroutineWorker.kt:68)
E OnnxOcrEngine:   at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:33)
D OnnxOcrEngine: Detected 0 text regions
W OnnxOcrEngine: Creating fallback receipt: No text detected

(I didn't completely follow your debug log line source code 100%, so. But I think the log above should contain enough info?)


----

Found it! The output is wrapped in a `java.util.Optional`. This is a quirk of the ORT Java API - `results.get(name)` returns `Optional<OnnxValue>`, not the tensor directly.

## The Fix

In `runDetection()`, change this block:

```kotlin
val rawOutput = results[outputName]
Log.d(TAG, "Output type: ${rawOutput?.javaClass?.simpleName}")

if (rawOutput == null) {
    Log.e(TAG, "Output tensor is null!")
    results.close()
    inputTensor.close()
    return emptyList()
}

val outputTensor = rawOutput as OnnxTensor
```

To this:

```kotlin
// ORT Java API wraps outputs in Optional<OnnxValue>
val optionalOutput = results[outputName]
Log.d(TAG, "Output wrapper type: ${optionalOutput?.javaClass?.simpleName}")

val outputTensor = when (optionalOutput) {
    is java.util.Optional<*> -> {
        val unwrapped = optionalOutput.orElse(null)
        Log.d(TAG, "Unwrapped type: ${unwrapped?.javaClass?.simpleName}")
        unwrapped as? OnnxTensor
    }
    is OnnxTensor -> optionalOutput
    else -> null
}

if (outputTensor == null) {
    Log.e(TAG, "Failed to get output tensor!")
    results.close()
    inputTensor.close()
    return emptyList()
}
```

---

## Also Fix `runRecognition()`

Same issue will happen there. Find the similar block in `runRecognition()` and change:

```kotlin
val outputTensor = results[outputName] as? OnnxTensor ?: return ""
```

To:

```kotlin
val outputTensor = when (val opt = results[outputName]) {
    is java.util.Optional<*> -> opt.orElse(null) as? OnnxTensor ?: return ""
    is OnnxTensor -> opt
    else -> return ""
}
```

---

After this, you should finally see the probmap images and actual detection results!


----



D OnnxOcrEngine: Model input names: [x]
D OnnxOcrEngine: Model output names: [fetch_name_0]
D OnnxOcrEngine: Running inference...
D OnnxOcrEngine: Inference complete!
D OnnxOcrEngine: Output name: fetch_name_0
D OnnxOcrEngine: Output wrapper type: Optional
D OnnxOcrEngine: Unwrapped type: OnnxTensor
D OnnxOcrEngine: Output shape: [1, 1, 736, 960]
D OnnxOcrEngine: Output type: FLOAT
D OnnxOcrEngine: Output data length: 706560
D OnnxOcrEngine: Output sample: [0.0004, 0.0003, 0.0002, 0.0001, 0.0003, 0.0003, 0.0002, 0.0002, 0.0003, 0.0003]
D OCR_DEBUG: det_1_raw_probmap.png channel 0: min=-5.9604645E-8, max=0.99999994
D OCR_DEBUG: Saved: /data/user/0/com.example.grocerytracker/files/ocr_debug/det_1_raw_probmap.png (960x736)
D OCR_DEBUG: Detection output shape: [1, 1, 736, 960]
D OCR_DEBUG: Output size: 706560, expected: 706560
D OCR_DEBUG: det_1_raw_probmap.png channel 0: min=-5.9604645E-8, max=0.99999994
D OCR_DEBUG: Saved: /data/user/0/com.example.grocerytracker/files/ocr_debug/det_1_raw_probmap.png (960x736)
D OCR_DEBUG: Saved: /data/user/0/com.example.grocerytracker/files/ocr_debug/det_2_binary.png (960x736)
D OCR_DEBUG: Binary map: 94348 / 706560 pixels above threshold 0.3
D OnnxOcrEngine: === DETECTION END: 39 boxes ===
D OnnxOcrEngine: Detected 39 text regions
D OnnxOcrEngine: Recognized: '  - 量  - 啤  - 東  - 职' (score=0.98100305)
...

----

Finally! Seems detection phase passed. I also manually checked the debug images - it looks very accurate (segmentation approach and then threshold to generate bounding box regions?). However, the recognition part is utterly messed up (lots of unrecognisable chinese character, always interspersed with "-", etc). The test image is a Chinese grocery receipt. The HF model card page for the model I use explicitly said it should support English + Chinese. What gives? Suspect: maybe segmentation is accurate but bounding box is still messed up but I have no way to know now. Can you now link it up to the OCR debug screen (maybe reuse the one we haven't touched since project start back at the very beginning - the one used for the camera flow), and display the bbox visually?


(peak usage hour GLM4.7?)


----

Excellent progress! Detection working with 39 boxes is actually very good. The garbage recognition is likely caused by misaligned crops. Let's visualize the boxes to confirm.

## File 1: Add debug method to `OnnxOcrEngine.kt`

Add this method at the end of the class, before `close()`:

```kotlin
/**
 * Debug method: Run full OCR pipeline and return boxes with text.
 * Used for visual debugging of bounding box alignment.
 */
fun debugProcessImage(bitmap: Bitmap): List<TextBox> {
    try {
        initialize()
    } catch (e: OrtException) {
        Log.e(TAG, "Failed to initialize ONNX", e)
        return emptyList()
    }
    
    return try {
        val boxes = runDetection(bitmap)
        Log.d(TAG, "Debug: ${boxes.size} boxes detected")
        
        boxes.map { box ->
            val crop = cropForRecognition(bitmap, box)
            Log.d(TAG, "Crop for recognition: ${crop.width}x${crop.height}")
            
            // Save crop for debugging
            DebugUtils.saveBitmap(context, crop, "rec_crop_${box.hashCode()}.png")
            
            val text = runRecognition(bitmap, box)
            Log.d(TAG, "Box text: '$text'")
            box.copy(text = text)
        }
    } catch (e: Exception) {
        Log.e(TAG, "Debug processing failed", e)
        emptyList()
    }
}
```

---

## File 2: Update `UiState` in `AppViewModel.kt`

Add a new state class:

```kotlin
sealed class UiState {
    object Idle : UiState()
    object Loading : UiState()
    data class Success(val image: Bitmap, val rawText: String) : UiState()
    data class OcrDebug(val image: Bitmap, val textBoxes: List<com.example.grocerytracker.ocr.TextBox>) : UiState()
    data class Error(val message: String) : UiState()
}
```

Add new method in `AppViewModel` class:

```kotlin
fun processImageWithOnnx(context: android.content.Context, bitmap: Bitmap) {
    viewModelScope.launch(Dispatchers.Default) {
        _uiState.value = UiState.Loading
        try {
            val engine = com.example.grocerytracker.ocr.OnnxOcrEngine(context)
            val boxes = engine.debugProcessImage(bitmap)
            engine.close()
            _uiState.value = UiState.OcrDebug(bitmap, boxes)
        } catch (e: Exception) {
            _uiState.value = UiState.Error(e.localizedMessage ?: "Unknown error")
        }
    }
}
```

---

## File 3: Create `OcrDebugScreen.kt`

```kotlin
// app/src/main/java/com/example/grocerytracker/ui/OcrDebugScreen.kt
package com.example.grocerytracker.ui

import androidx.compose.foundation.Canvas
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ArrowBack
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.geometry.Size
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.asImageBitmap
import androidx.compose.ui.graphics.drawscope.Stroke
import androidx.compose.ui.unit.dp
import com.example.grocerytracker.ocr.TextBox

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun OcrDebugScreen(
    image: android.graphics.Bitmap,
    textBoxes: List<TextBox>,
    onBackClicked: () -> Unit
) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("OCR Debug (${textBoxes.size} boxes)") },
                navigationIcon = {
                    IconButton(onClick = onBackClicked) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "Back")
                    }
                }
            )
        }
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(8.dp)
        ) {
            // Image with overlaid bounding boxes
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .aspectRatio(image.width.toFloat() / image.height.toFloat())
            ) {
                // Background image
                androidx.compose.foundation.Image(
                    bitmap = image.asImageBitmap(),
                    contentDescription = "Receipt with boxes",
                    modifier = Modifier.fillMaxSize()
                )
                
                // Overlay bounding boxes
                Canvas(modifier = Modifier.fillMaxSize()) {
                    // Scale factors: image coords -> canvas coords
                    val scaleX = size.width / image.width.toFloat()
                    val scaleY = size.height / image.height.toFloat()
                    
                    textBoxes.forEachIndexed { index, box ->
                        val topLeft = box.bounds.first
                        val bottomRight = box.bounds.second
                        
                        val x = topLeft.first * scaleX
                        val y = topLeft.second * scaleY
                        val w = (bottomRight.first - topLeft.first) * scaleX
                        val h = (bottomRight.second - topLeft.second) * scaleY
                        
                        // Draw box
                        drawRect(
                            color = Color.Red,
                            topLeft = Offset(x, y),
                            size = Size(w, h),
                            style = Stroke(width = 2.dp.toPx())
                        )
                        
                        // Draw index number at top-left corner
                        // (Simple text drawing - may look rough but good enough for debug)
                    }
                }
            }

            Spacer(modifier = Modifier.height(12.dp))
            
            Text(
                "Recognized Text:",
                style = MaterialTheme.typography.titleMedium,
                modifier = Modifier.padding(horizontal = 8.dp)
            )
            Spacer(modifier = Modifier.height(8.dp))

            // Scrollable list of recognized text
            LazyColumn(
                modifier = Modifier.fillMaxSize(),
                contentPadding = PaddingValues(horizontal = 8.dp, vertical = 4.dp),
                verticalArrangement = Arrangement.spacedBy(4.dp)
            ) {
                items(textBoxes) { box ->
                    Card(modifier = Modifier.fillMaxWidth()) {
                        Column(modifier = Modifier.padding(8.dp)) {
                            Text(
                                text = box.text.ifBlank { "(empty)" },
                                style = MaterialTheme.typography.bodyLarge
                            )
                            Text(
                                text = "score: ${"%.3f".format(box.score)}",
                                style = MaterialTheme.typography.bodySmall,
                                color = MaterialTheme.colorScheme.onSurfaceVariant
                            )
                        }
                    }
                }
            }
        }
    }
}
```

---

## File 4: Update `MainScreen.kt`

Change the camera launcher to use the new ONNX debug flow:

```kotlin
// Change this:
val cameraLauncher = rememberLauncherForActivityResult(
    ActivityResultContracts.TakePicturePreview()
) { bitmap: Bitmap? ->
    bitmap?.let { viewModel.processImage(it) }
}

// To this:
val cameraLauncher = rememberLauncherForActivityResult(
    ActivityResultContracts.TakePicturePreview()
) { bitmap: Bitmap? ->
    bitmap?.let { viewModel.processImageWithOnnx(context, it) }
}
```

Add the new state handler in the `when` block (after `is UiState.Success`):

```kotlin
is UiState.OcrDebug -> {
    OcrDebugScreen(
        image = state.image,
        textBoxes = state.textBoxes,
        onBackClicked = { viewModel.resetState() }
    )
}
```

---

## Test It

```bash
./gradlew installDebug
# Use camera to scan receipt
# Pull debug crops if needed:
adb shell "run-as com.example.grocerytracker ls files/ocr_debug/"
adb shell "run-as com.example.grocerytracker cp files/ocr_debug/rec_crop_*.png /sdcard/Download/"
adb pull /sdcard/Download/rec_crop_*.png ./ocr_crops/
```

## What to Look For

| Observation | Diagnosis |
|-------------|-----------|
| Boxes perfectly aligned with text | Recognition issue is elsewhere (normalization, keys file) |
| Boxes shifted/offset | Ratio calculation bug |
| Boxes too small/narrow | Unclip ratio wrong |
| Boxes overlapping/multiple boxes per line | Detection threshold too low |
| Boxes missing text regions | Detection threshold too high |

Share a screenshot or describe what the boxes look like!


----

quick check:

ui/MainScreen.kt:231:5 'when' expression must be exhaustive, add necessary 'is OcrDebug' branch or 'else' branch instead
ui/MainScreen.kt:279:35 Unresolved reference: textBoxes

----

You probably need to add the `else` branch OR the `OcrDebug` branch isn't quite right. Here's the exact fix:

## Fix 1: Add the branch (or `else`)

Find the `when (val state = uiState)` block. The safest quick fix is to add `else` at the very end:

```kotlin
when (val state = uiState) {
    is UiState.Idle -> {
        // ... existing code ...
    }

    is UiState.Loading -> {
        // ... existing code ...
    }

    is UiState.Success -> {
        // ... existing code ...
    }
    
    // ADD THIS BLOCK:
    is UiState.OcrDebug -> {
        OcrDebugScreen(
            image = state.image,
            textBoxes = state.textBoxes,
            onBackClicked = { viewModel.resetState() }
        )
    }

    is UiState.Error -> {
        // ... existing code ...
    }
}
```

## Fix 2: Missing import

Add this import at the top of `MainScreen.kt`:

```kotlin
import com.example.grocerytracker.ocr.TextBox
```

(Though technically you shouldn't need this if you're accessing `state.textBoxes` directly... but add it just in case)

---

## If still stuck

Just replace the entire `when` block with:

```kotlin
when (val state = uiState) {
    is UiState.Idle -> {
        Scaffold(
            topBar = { TopAppBar(title = { Text("Grocery Tracker") }) }
        ) { padding ->
            Column(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(padding)
                    .padding(16.dp),
                verticalArrangement = Arrangement.Center,
                horizontalAlignment = Alignment.CenterHorizontally
            ) {
                Button(
                    onClick = { permissionLauncher.launch(Manifest.permission.CAMERA) },
                    modifier = Modifier.fillMaxWidth().padding(bottom = 16.dp)
                ) {
                    Icon(Icons.Default.CameraAlt, contentDescription = null)
                    Spacer(modifier = Modifier.width(8.dp))
                    Text("Scan with Camera")
                }

                FilledTonalButton(
                    onClick = { showSelectionDialog = true },
                    modifier = Modifier.fillMaxWidth()
                ) {
                    Icon(Icons.Default.Image, contentDescription = null)
                    Spacer(modifier = Modifier.width(8.dp))
                    Text("Import from Gallery / Folder")
                }
            }
        }
    }

    is UiState.Loading -> {
        Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
            CircularProgressIndicator()
            Text("Running OCR...")
        }
    }

    is UiState.Success -> {
        DebugScreen(
            image = state.image,
            rawText = state.rawText,
            onBackClicked = { viewModel.resetState() }
        )
    }

    is UiState.OcrDebug -> {
        OcrDebugScreen(
            image = state.image,
            textBoxes = state.textBoxes,
            onBackClicked = { viewModel.resetState() }
        )
    }

    is UiState.Error -> {
        Column(
            modifier = Modifier.fillMaxSize().padding(16.dp),
            verticalArrangement = Arrangement.Center,
            horizontalAlignment = Alignment.CenterHorizontally
        ) {
            Text("Error: ${state.message}", color = MaterialTheme.colorScheme.error)
            Spacer(modifier = Modifier.height(16.dp))
            Button(onClick = { viewModel.resetState() }) { Text("Go Back") }
        }
    }
}
```


----

Okay... I can see the bbox now. Generally reasonable, seems problem is really in the recog phase. Eg it seems it always judge as "- X", where X is some obscure chinese character, for every single character in the ground truth. This happen regardless of whether ground truth is eng or chinese.




