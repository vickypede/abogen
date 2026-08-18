# Abogen Google Colab Runbook (T4 GPU)

## Step 1: Setup (one cell)
```python
!apt-get install -y espeak-ng > /dev/null 2>&1
!pip install abogen
```

## Step 2: Upload your PDF
Use the 📁 Files sidebar → ⬆️ upload button → upload your PDF to `/content/`

## Step 3: Start conversion (one cell)
Change the filename to match your uploaded file.
```python
import subprocess, time, requests

# Start server
server = subprocess.Popen(
    ['abogen-web'],
    stdout=open('/content/server.log', 'w'),
    stderr=subprocess.STDOUT
)
time.sleep(15)

# Upload file to server
FILENAME = "Cloud FinOps.pdf"  # ← CHANGE THIS to your file name

with open(f"/content/{FILENAME}", "rb") as f:
    resp = requests.post(
        "http://127.0.0.1:8808/wizard/upload",
        files={"file": (FILENAME, f)},
        allow_redirects=False
    )
pending_id = resp.headers.get("Location", "").split("pending_id=")[-1]
print(f"Pending ID: {pending_id}")

# Enable all chapters (use a number higher than your page count)
# We also force the output to save directly to /content/ so it's visible
form_data = {
    "pending_id": pending_id, 
    "voice": "af_heart", 
    "speed": "1.0",
    "save_mode": "custom",
    "output_folder": "/content"
}
for i in range(700):
    form_data[f"chapter-{i}-enabled"] = "on"

resp = requests.post(
    "http://127.0.0.1:8808/wizard/finish",
    data=form_data,
    headers={"X-Requested-With": "XMLHttpRequest"},
    allow_redirects=False
)
print(f"Status: {resp.status_code}")
print(resp.text[:300])
```

## Step 4: Monitor progress (new cell)
```python
import time
from IPython.display import clear_output

while True:
    clear_output(wait=True)
    with open('/content/server.log', 'r') as f:
        lines = f.readlines()
        for line in lines[-15:]:
            print(line.strip())
    time.sleep(10)
```

## Step 5: Download output (new cell)
Run this when the job finishes to automatically locate and download the generated file:
```python
import os
from google.colab import files

found_files = []
# Abogen usually saves it to /content/ or a hidden cache folder
for search_dir in ['/content', '/root/.cache/abogen']:
    for root, dirs, filenames in os.walk(search_dir):
        for filename in filenames:
            if filename.endswith(('.wav', '.m4b', '.m4a', '.mp3', '.epub')):
                # Ignore the original pdf input
                if "Cloud FinOps" in filename and filename.endswith('.pdf'): continue
                found_files.append(os.path.join(root, filename))

if found_files:
    # Get the most recently created audio file
    latest_file = max(found_files, key=os.path.getctime)
    
    # If it's hidden, move it to /content/ so you can see it in the sidebar
    if not latest_file.startswith('/content/'):
        new_path = os.path.join('/content', os.path.basename(latest_file))
        os.rename(latest_file, new_path)
        latest_file = new_path
        
    print(f"✅ Found output at: {latest_file}")
    print("⏳ Starting download now...")
    files.download(latest_file)
else:
    print("❌ Couldn't find the output file. Check the server.log for errors.")
```

## Notes
- Select **T4 GPU** runtime: Runtime → Change runtime type → T4 GPU
- ~900K characters ≈ 30 min on T4 GPU vs 6+ hours on CPU
- Download files BEFORE closing the tab — Colab deletes everything on disconnect

## For Next Time: Better Output Formats
By default, this script runs with standard settings (outputting WAV files). You can modify the `form_data` dictionary in **Step 3** to get much better formats:

### Option A: Single Audiobook File (M4B)
If you want a single, compact audiobook file that is natively supported by Apple Books and mobile player apps with working chapter markers:
```python
form_data = {
    "pending_id": pending_id, 
    "voice": "af_heart", 
    "speed": "1.0",
    "output_format": "m4b",
    "merge_chapters_at_end": "true",
    "save_chapters_separately": "false"
}
```

### Option B: Kindle-Style "Read-Along" Book (EPUB3)
If you want a format that contains **both the text and the audio synchronized together** (like Amazon Whispersync), Abogen can generate an EPUB3 file with "Media Overlays". This works perfectly in Apple Books and highlights the text as the audio plays:
```python
form_data = {
    "pending_id": pending_id, 
    "voice": "af_heart", 
    "speed": "1.0",
    "generate_epub3": "true"
}
```


