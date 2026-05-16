Greetings! We've previously discussed various aspects of native Android app dev throuhout multiple sessions.

<context>
Interesting! Suppose I am a student in this course and for my final project, I would like to create a "Grocery Finance tracker" app. Idea: daily life shopping results in lots of paper receipt, I want to be able to scan it using my phone, then it automatically OCR. It records the shopping record (where, items + the spending amount), and then downstream do the usual spending tracking stuff with simple dashboard and analytics. (one eg of not too trivial: most frequent repeat purchase item) Would this app be able to effectively leverage the knowledges in the learning modules above?

----

This is a **10/10 final project idea**. In fact, it’s almost perfectly tailored to stress-test the exact paradigm shifts a web developer needs to internalize when moving to native mobile. 

It avoids the trap of just being a "web app wrapped in native UI" because it fundamentally requires deep integration with hardware, strict local data architecture, and heavy background processing.

Here is exactly how your "Grocery Finance Tracker" maps to the syllabus, and why it's such a great project for a fullstack web dev:

### How it leverages the syllabus (Module by Module)

*   **Module 2 (UI) & 9 (Animations):** Web devs are used to HTML tables or simple CSS grids for data. In this app, you’ll need to build complex, scrollable lists of receipts, nested lists of items within those receipts, and a dashboard. You'll get to play with native charts (or build custom bar charts using Compose/SwiftUI shapes) and micro-animations (e.g., a scanning line overlay on the camera, or a checkmark animating in when a receipt is successfully saved).
*   **Module 3 (State) & 5 (Concurrency):** **This is where the project shines.** OCR (Optical Character Recognition) is *heavy*. If you run it on the Main/UI thread, the app will freeze and the OS will kill it (a concept web devs rarely deal with since browsers handle JS threading for them). You will have to use `async/await` (Swift) or Coroutines (Kotlin) to show a "Scanning..." spinner (State), kick off the OCR in the background (Concurrency), and update the UI when it's done.
*   **Module 4 (Navigation):** You’ll need a flow like: *Home Tab -> Camera/Upload Screen -> (OCR Processing) -> "Review Extracted Data" Screen -> Save -> Dashboard Tab*. Passing the extracted string of text safely between these screens requires understanding native navigation stacks.
*   **Module 6 (Local Storage) & 7 (Architecture):** **The biggest trap for a web dev here.** A web dev might be tempted to just store the extracted receipt as a giant JSON blob in local storage. *Don't do that.* To get "most frequent repeat purchase item," you need a **relational database schema**. 
    *   `Receipt` table (ID, Store Name, Date, Total)
    *   `Item` table (ID, Receipt_ID, Name, Price, Category)
    *   You'll use SwiftData (iOS) or Room (Android) to write SQL-like queries (e.g., `SELECT name, COUNT(name) FROM items GROUP BY name ORDER BY COUNT(name) DESC`). This perfectly tests MVVM and Repository patterns.
*   **Module 8 (Native Capabilities):** You need the Camera. But more importantly, you need **Camera Permissions**. Handling the OS prompt "App wants to access the camera," and gracefully handling it if the user clicks "Deny," is a core native skill.

### The Tech Stack You'd Use (Fully Modern)

*   **iOS:** SwiftUI for UI, Swift Concurrency for OCR, **Vision framework** (Apple's built-in, on-device OCR—it is incredibly good and free, no API keys needed), and SwiftData for the database.
*   **Android:** Jetpack Compose for UI, Kotlin Coroutines for OCR, **Google ML Kit (Text Recognition)** (on-device, free), and Room Database.

### 3 Crucial Pieces of Advice for Your Project

Since I'm pretending to be your professor grading this, here is what I'd warn you about going in:

**1. OCR is never 100% perfect.**
Paper receipts are crinkled, faded, use weird fonts, and have thermal paper smudges. Your app *will* misread "Apple" as "Appl€" or miss a decimal point. 
*   *The Fix:* You **must** build a "Review & Edit" screen between the camera and the database. The OCR spits out raw text; you parse it into a list of (Item, Price), show it to the user, let them fix the typos, and *then* save it to the DB. This is a fantastic UX challenge.

**2. Parsing receipt text is an algorithmic nightmare.**
There is no standard receipt format. Target puts the total at the bottom; CVS puts it in the middle. Some use tab-separated values, some use spaces. 
*   *The Fix:* Don't try to build a perfect regex on day one. Start by using a heuristic: look for lines that end in a pattern that looks like currency (e.g., `$X.XX` or `X,XX €`). Assume the *last* currency match is the Total. 

**3. Scope creep with the Camera.**
Accessing a live camera feed, drawing a viewfinder box, and cropping the image is surprisingly complex (especially handling different phone aspect ratios). 
*   *The Fix:* For your MVP (Minimum Viable Product), **skip the live camera entirely**. Just use the native photo picker (`PhotosUI` on iOS / `Photo Picker` on Android) to let the user select an existing picture of their receipt from their camera roll. Once the OCR and DB parts work, add the live camera as a "Phase 2" feature.

**Verdict:** If you build this, you will walk out of the class not just knowing how to write native UI, but actually understanding *why* native apps are built differently than web apps. Go for it!

----
...

### 3. Multi-Source Input (Camera vs. Photo vs. File Picker)
* **What it teaches:** OS Contracts / Delegates & Cross-App Communication.
* **The Web vs. Native shift:** On the web, `<input type="file">` handles everything. In native, different inputs require entirely different OS pipelines. 
* **The UX/Flow Solution:** You don't build separate screens for this. You use a native **Action Sheet** (a menu that slides up from the bottom of the screen: "Take Photo", "Choose from Library", "Choose from Files").
* **The Tech:** 
    * *Camera:* Requires building a custom `UIViewControllerRepresentable` (iOS) or `AndroidView` (Android) wrapping `AVCaptureSession` / `CameraX`.
    * *Photo Picker:* Native `PhotosUI` (iOS) / `PhotoPicker` contract (Android).
    * *File Picker:* `UIDocumentPickerViewController` (iOS) / `OpenDocumentTree` contract (Android). Notice the word "Contract/Delegate"? That’s how OS apps talk to each other securely without sharing memory.

</context>

<context>
Developer profile: Experienced Fullstack WebDev (ReactJS, RESTful API web framework...), but learning native Mobile app dev.

Dev Machine: Mac with Android Studio installed, adb and gradle available but not sdkmanager nor avdmanager on system path. Has a physical mobile phone for dev.

Preference: Fully modern practise - kotlin, jetpack compose for UI. Prefer headless mode dev - that is, not using the Android Studio GUI, instead operates using terminal/command line manually (and edit source code with VSCode)
</context>

Okay, now, let's begin dev. I want to study the multi-source input => OCR using Google ML Kit. So user story/journey: main screen has two button (input via camera or filepicker), then native OS integration, then a screen that display the image and the raw detected text (a native UI "debug" screen).

Can you show me the source code? Also for project scaffolding, is there a way to do it without going throuh Android Studio, or I should just create folders and files manually? Many thanks!
