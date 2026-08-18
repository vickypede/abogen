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
form_data = {"pending_id": pending_id, "voice": "af_heart", "speed": "1.0"}
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

## Step 5: Download output
When done, find output files in the 📁 Files sidebar.
Look in the output folder or check the server log for the output path.

## Notes
- Select **T4 GPU** runtime: Runtime → Change runtime type → T4 GPU
- ~900K characters ≈ 30 min on T4 GPU vs 6+ hours on CPU
- Download files BEFORE closing the tab — Colab deletes everything on disconnect
