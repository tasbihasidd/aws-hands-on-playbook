# Complete Nginx Static Server & Reverse Proxy Practical Lab & Notes (Ubuntu / AWS EC2)

In this practical lab, we configure Nginx to act like a **"Traffic Police"** or **"Receptionist"** that handles two different tasks using a single Public IP address and Port `80`:

1. **Static Content (`/`):** Serves a static HTML file directly from `/var/www/html` when someone visits the main root URL.
2. **Reverse Proxy (`/api`):** Forwards requests starting with `/api` internally to a backend application running on port `3000`.

---

## Step 0: Launch AWS EC2 Instance & Connect via SSH

Before installing Nginx, we need to create a cloud server (EC2 instance) on AWS and connect to it from our local computer.

### 1. Launch an EC2 Instance (Free Tier)

1. Log in to your **AWS Management Console** and go to the **EC2 Dashboard**.
2. Click **Launch Instance** and configure the following settings:
   * **Name:** Give your server a name (e.g., `nginx-lab-server`).
   * **Application and OS Images (AMI):** Select **Ubuntu** (Ubuntu Server 22.04 or 24.04 LTS — *Free tier eligible*).
   * **Instance Type:** Select **`t3.micro`** (or `t2.micro` depending on region — *Free tier eligible*).
   * **Key Pair (login):** Click **Create new key pair**, give it a name (e.g., `my-aws-key`), keep the format as **`.pem`**, and click **Create key pair**. The `.pem` file will automatically download to your computer—keep it safe!
   * **Network Settings & Storage:** Keep the default **Free Tier** settings (make sure **Allow SSH traffic from Anywhere** is checked so you can connect to your server).
3. Click **Launch Instance** and wait a minute until the **Instance State** turns to **Running**. Copy your instance's **Public IPv4 address**.

### 2. Connect to the Server via SSH (and Fix Key Permission Error)

1. Open your **Terminal** (or Command Prompt / Git Bash) inside the **exact folder/directory where your `.pem` key pair file is saved** (for example, `Downloads`):

   ```bash
   cd ~/Downloads
   ```

2. Run the SSH command to connect to your Ubuntu server (replace `my-aws-key.pem` with your key name and `YOUR_SERVER_PUBLIC_IP` with your EC2 Public IP):

   ```bash
   ssh -i "my-aws-key.pem" ubuntu@YOUR_SERVER_PUBLIC_IP
   ```

   *(Type `yes` and press `Enter` if it asks about host authenticity).*

3. **⚠️ Expected Permission Error (`UNPROTECTED PRIVATE KEY FILE!`):**
   * When you run the `ssh` command for the first time on Linux/Mac, it will **fail with a permission error** (`Permissions 0644 for 'my-aws-key.pem' are too open`).
   * **Why it happens:** SSH refuses to use a private key file if other users on your computer can read it.
   * **How to fix (`chmod`):** Restrict the key file permissions so only you (the owner) can read it:

     ```bash
     chmod 400 my-aws-key.pem
     ```

4. **Connect Again:** Now run the exact same `ssh` command again:

   ```bash
   ssh -i "my-aws-key.pem" ubuntu@YOUR_SERVER_PUBLIC_IP
   ```

> **✅ Test & Verify (Step 0):**
> Your terminal prompt should change from your local machine name to `ubuntu@ip-172-xx-xx-xx:~$`. This confirms you are successfully logged into your cloud server.

---

## Step 1: Install Nginx & Allow Port 80 in AWS Security Group

Update your system's package list, install Nginx, and make sure it starts automatically on boot:

```bash
sudo apt update -y
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Check if Nginx is active and running inside the server:

```bash
sudo systemctl status nginx
```

*(Press `q` on your keyboard to exit the status screen)*

> **✅ Test & Verify (Step 1 — Browser & Security Group Check):**
>
> 1. **First Attempt (Before Allowing Port 80):** Open your browser and visit `http://YOUR_SERVER_PUBLIC_IP` (make sure to use `http://`, **not** `https://`).
>    * *What happens:* The page **will fail to load** (*This site can't be reached / Connection timed out*) because AWS blocks all incoming web traffic on Port 80 by default.
> 2. **Fix in AWS Console:** Go to your EC2 instance $\rightarrow$ **Security** tab $\rightarrow$ click your **Security Group** $\rightarrow$ **Edit inbound rules** $\rightarrow$ **Add rule**:
>    * **Type:** `HTTP` (Port `80`)
>    * **Source:** `0.0.0.0/0` (Anywhere-IPv4)
>    * Click **Save rules**.
> 3. **Second Attempt (After Allowing Port 80):** Refresh `http://YOUR_SERVER_PUBLIC_IP` in your browser.
>    * *Expected Result:* You will see the default **"Welcome to nginx!"** page! This proves Nginx is running and accessible from the internet.

---

## Step 2: Prepare the Static File & Backend Service (Using `vim`)

### 1. Replace Default Welcome Page with Custom Static HTML (`/var/www/html`)

* **What we did & Why:** Static files (like HTML, CSS, and images) do not require any backend processing or database. Nginx serves files directly from `/var/www/html` by default. We will replace Nginx's default welcome page with our own custom message.

Open `index.html` using `vim`:

```bash
sudo mkdir -p /var/www/html
sudo vim /var/www/html/index.html
```

* Press **`i`** to enter `-- INSERT --` mode, delete the old default HTML text if present, and type or paste:

  ```html
  <h1>Directly served static file by Nginx</h1>
  ```

* Press **`Esc`**, type **`:wq`**, and press **`Enter`** to save and exit.

> **✅ Test & Verify (Static HTML Replacement):**
>
> 1. **Terminal check:** Run `cat /var/www/html/index.html` to verify your new HTML code is saved.
> 2. **Browser check:** Refresh `http://YOUR_SERVER_PUBLIC_IP/` in your browser (no Nginx restart needed for HTML changes).
>    * *Expected Result:* The default *"Welcome to nginx!"* message is gone, and you now see **"Directly served static file by Nginx"**.

### 2. Setup the Backend Service (Port 3000)

* **What we did & Why:** In real-world applications, backend services (Node.js, Python, Java, etc.) run on internal ports like `3000` or `5000`. We don't want users to type `:3000` in the browser URL, nor do we want to expose internal backend ports directly to the public internet.

Create a backend folder and make an `index.html` file inside it using `vim`:

```bash
mkdir -p ~/backend-demo
cd ~/backend-demo
vim index.html
```

* Press **`i`** and add the following code:

  ```html
  <h1>Hello from Backend App (Port 3000)!</h1>
  ```

* Press **`Esc`**, type **`:wq`**, and press **`Enter`** to save and exit.

Start a simple Python HTTP server in the background (`&`) on port `3000`:

```bash
python3 -m http.server 3000 &
```

*(Tip: Press `Enter` one more time after running this command to bring your terminal prompt back).*

> **✅ Test & Verify (Backend Service on Port 3000):**
> Run the following command inside your server terminal to test if the backend is responding locally:
>
> ```bash
> curl http://localhost:3000
> ```
>
> * *Expected Result:* The terminal outputs `<h1>Hello from Backend App (Port 3000)!</h1>`.
> * *(Note: If you try `http://YOUR_SERVER_PUBLIC_IP:3000` in your browser, it won't load because Port 3000 is closed in the Security Group—and that's good! Next, we will connect it through Nginx).*

---

## Step 3: Create Nginx Configuration (Reverse Proxy Setup)

Create a custom configuration file inside Nginx's `sites-available` directory:

```bash
sudo vim /etc/nginx/sites-available/my-server.conf
```

Press **`i`** and paste the following configuration:

```nginx
server {
    listen 80;
    server_name _;

    # 1. Direct Static File Serving
    root /var/www/html;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    # 2. Reverse Proxy for Backend
    # Requests starting with "/api" are forwarded to port 3000
    location /api {
        proxy_pass http://127.0.0.1:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

*(Note: Using `location /api` without a trailing slash ensures that both `/api` and `/api/` work smoothly).*

Save and exit the file: Press **`Esc`** $\rightarrow$ type **`:wq`** $\rightarrow$ press **`Enter`**.

> **✅ Test & Verify (Config File Check):**
> Run `cat /etc/nginx/sites-available/my-server.conf` to double-check that there are no extra brackets or typos in the `proxy_pass` line.
> *(Note: If you visit `http://YOUR_SERVER_PUBLIC_IP/api` right now, you will still get a `404 Not Found` error because we haven't enabled this configuration file yet in Step 4!).*

---

## Step 4: Enable Configuration (Create Symlink)

* **What we did & Why:** Putting a config file inside `sites-available` does not activate it. Nginx only reads configurations placed inside the `sites-enabled` folder. Therefore, we create a symbolic link (shortcut) to our file and remove the `default` configuration so they don't conflict with each other.

```bash
# Create the symbolic link (shortcut)
sudo ln -s /etc/nginx/sites-available/my-server.conf /etc/nginx/sites-enabled/

# Remove the default configuration to avoid conflicts
sudo rm -f /etc/nginx/sites-enabled/default
```

> **✅ Test & Verify (Symlink Check):**
> Run this command to check the contents of `sites-enabled`:
>
> ```bash
> ls -la /etc/nginx/sites-enabled/
> ```
>
> * *Expected Result:* You should see **only** `my-server.conf -> /etc/nginx/sites-available/my-server.conf` listed, and `default` should be completely gone.

---

## Step 5: Test Syntax & Reload Nginx

Always test your Nginx configuration for syntax errors before applying changes:

```bash
sudo nginx -t
```

> **✅ Test & Verify (Syntax Check):**
> * *Expected Output:*
>   * `nginx: the configuration file /etc/nginx/nginx.conf syntax is ok`
>   * `nginx: configuration file /etc/nginx/nginx.conf test is successful`

Now reload Nginx to apply the new settings without dropping active connections:

```bash
sudo systemctl reload nginx
```

---

## Step 6: Final End-to-End Browser Testing (Using Public IP)

Open your web browser and test both routes using your server's Public IP address (using `http://`):

1. **Test 1 — Static Content (`/`):**
   * **Visit:** `http://YOUR_SERVER_PUBLIC_IP/`
   * **Expected Result:** **Directly served static file by Nginx**
   * **What happened behind the scenes:** Nginx matched `location /` and served `/var/www/html/index.html` directly.

2. **Test 2 — Reverse Proxy (`/api` and `/api/`):**
   * **Visit:** `http://YOUR_SERVER_PUBLIC_IP/api` (or `http://YOUR_SERVER_PUBLIC_IP/api/`)
   * **Expected Result:** **Hello from Backend App (Port 3000)!**
   * **What happened behind the scenes:** Nginx matched `location /api`, secretly forwarded your request internally to `http://127.0.0.1:3000/`, grabbed the response from the Python backend server, and displayed it to you on Port 80!

---

## Step 7: Bind a Custom Domain Name to Your Nginx Server

* **What we did & Why:** Sharing a raw IP address (like `32.200.108.23`) is hard for users to remember, and if you ever restart or replace your EC2 instance, the IP might change. Binding a **custom domain name** (e.g., `example.com`) connects a human-readable web address to your EC2 server via DNS and tells Nginx to respond specifically to requests for that domain.

### 1. Point Your Domain to EC2 via DNS (`A` Record)

1. Log in to your **Domain Registrar** or DNS provider (such as **AWS Route 53**, **Namecheap**, **GoDaddy**, or **Cloudflare**).
2. Open the **DNS Management / DNS Records** settings for your domain.
3. Add two **`A` Records** pointing to your EC2 **Public IPv4 address**:
   * **Record 1 (Root Domain):**
     * **Type:** `A`
     * **Host / Name:** `@` *(represents your root domain, e.g., `example.com`)*
     * **Value / Points to:** `YOUR_SERVER_PUBLIC_IP`
     * **TTL:** `Automatic` or `300` seconds
   * **Record 2 (`www` Subdomain):**
     * **Type:** `A`
     * **Host / Name:** `www` *(represents `www.example.com`)*
     * **Value / Points to:** `YOUR_SERVER_PUBLIC_IP`
     * **TTL:** `Automatic` or `300` seconds
4. Save the DNS records. *(Note: DNS changes usually take 1–5 minutes to propagate).*

### 2. Update `server_name` in Nginx Configuration

Now tell Nginx to listen for your specific domain name instead of the wildcard `_`:

1. Open your Nginx configuration file using `vim`:

   ```bash
   sudo vim /etc/nginx/sites-available/my-server.conf
   ```

2. Press **`i`** to enter Insert mode, and update the `server_name` line (replace `example.com` with your actual domain):

   ```nginx
   server {
       listen 80;
       server_name example.com www.example.com;

       # 1. Direct Static File Serving
       root /var/www/html;
       index index.html index.htm;

       location / {
           try_files $uri $uri/ =404;
       }

       # 2. Reverse Proxy for Backend
       location /api {
           proxy_pass http://127.0.0.1:3000/;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

3. Press **`Esc`**, type **`:wq`**, and press **`Enter`** to save and exit.

4. Test the syntax and reload Nginx:

   ```bash
   sudo nginx -t && sudo systemctl reload nginx
   ```

> **✅ Test & Verify (Step 7 — Domain Binding Check):**
>
> 1. **DNS Propagation Check (Terminal):** Run `nslookup example.com` (or `ping -c 2 example.com`) in your terminal to verify that your domain resolves to `YOUR_SERVER_PUBLIC_IP`.
> 2. **Browser Check (Static Route):** Visit `http://example.com/` (and `http://www.example.com/`).
>    * *Expected Result:* **Directly served static file by Nginx**
> 3. **Browser Check (Reverse Proxy Route):** Visit `http://example.com/api`.
>    * *Expected Result:* **Hello from Backend App (Port 3000)!**

*(Optional Next Step for Production: Once a domain is bound, you can easily enable free **HTTPS / SSL** using **Certbot / Let's Encrypt** and opening **Port 443** in your AWS Security Group).*

---

## 🛠️ Troubleshooting & Errors Log (Issues Faced & How to Fix Them)

### 1. SSH Error: `Permissions 0644 for 'key.pem' are too open` (`UNPROTECTED PRIVATE KEY FILE!`)

* **Why it happened:** SSH requires your `.pem` private key file to be readable only by you. By default, downloaded files have open read permissions.
* **How to fix:** Run `chmod 400 your-key-name.pem` in the directory where the key is stored, then run the `ssh -i` command again.

### 2. Public IP Not Loading in Browser (Even After Allowing Port 80)

* **Why it happened:** When clicking "Open address" in the AWS Console, browsers often default to **`https://`** (Port 443), whereas we only configured **HTTP (Port 80)**.
* **How to fix:** Manually type **`http://`** in the browser's address bar (e.g., `http://32.200.108.23`). Also check if Ubuntu's internal firewall (`sudo ufw status`) is blocking traffic; if active, run `sudo ufw allow 'Nginx Full'`.

### 3. Vim Error: `E212: Can't open file for writing`

* **Why it happened:** System configuration files inside `/etc/nginx/` require root (administrator) permissions to modify. Opening the file with `vim` without `sudo` prevents saving.
* **How to fix:**
  * Always open system files using `sudo vim /etc/nginx/...`.
  * If you already typed your changes without `sudo`, save them without losing your work by pressing `Esc` and running:

    ```vim
    :w !sudo tee % > /dev/null
    ```

    Then type `:q!` and press `Enter` to exit.

### 4. `proxy_pass` Syntax Error (Markdown Link Pasted by Mistake)

* **Why it happened:** Copying directly from formatted lab notes pasted markdown link brackets into the config file: `proxy_pass [http://127.0.0.1:3000/](http://127.0.0.1:3000/);`, causing `nginx -t` to fail.
* **How to fix:** Remove all square brackets and parentheses so the line looks clean:

  ```nginx
  proxy_pass http://127.0.0.1:3000/;
  ```

### 5. Getting `404 Not Found` When Visiting `/api`

* **Why it happened (Two reasons):**
  1. **Missing Symlink:** The shortcut in `sites-enabled` was not created yet, and the `default` config file was still active, so Nginx ignored `my-server.conf`.
  2. **Trailing Slash (`/`) Mismatch:** If the config uses `location /api/` (with a trailing slash) and you visit `/api` (without a slash) in the browser, Nginx routes it to `location /` instead and returns a 404 error.
* **How to fix:**
  1. Create the symlink in `sites-enabled` and remove `default` (**Step 4**).
  2. Use `location /api` (without a trailing slash) in the config so both `/api` and `/api/` work properly.

### 6. Random Logs Like `"GET / HTTP/1.1" 200` or `code 400` Appearing in Terminal

* **Why it happened:** This is **not** an error. Because the Python server (`python3 -m http.server 3000 &`) is running in the background of your current terminal session, it prints live access logs whenever someone (or automated internet bots) hits port `3000`.
* **How to fix:** Your terminal is not frozen—simply press **`Enter`** on your keyboard to get your command prompt back.

---

## 📋 Quick `vim` Cheat Sheet (For Future Reference)

* **`i`** $\rightarrow$ Enter Insert mode (to start typing/editing)
* **`Esc`** $\rightarrow$ Return to Normal mode (to run commands)
* **`:wq`** $\rightarrow$ Write (save) changes and quit `vim`
* **`:q!`** $\rightarrow$ Quit without saving changes