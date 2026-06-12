# Install redis stack to run redis

### **Option 1: Docker ke zariye (Sabse aasan aur recommended)**
Agar aapke system mein **Docker** installed hai, toh aap apne terminal ya command prompt par sirf yeh ek command chalayein. Yeh automatically download aur start ho jayega:

```bash
docker run -d --name redis-stack -p 6379:6379 -p 8001:8001 redis/redis-stack:latest
```
*   **Port 6379:** Hamare Python code ko connect karne ke liye.
*   **Port 8001:** Redis Insight (ek software hai jise aap browser mein `http://localhost:8001` khol kar apna saara data visually dekh sakte hain).

---

### **Option 2: macOS par (Homebrew ke sath)**
Agar aap Mac use kar rahe hain aur Docker nahi chalana chahte, toh terminal par yeh commands run karein:

```bash
# 1. Redis Stack repository ko tap karein
brew tap redis-stack/redis-stack

# 2. Redis Stack install karein
brew install redis-stack

# 3. Server ko start karein
redis-stack-server
```

---

### **Option 3: Windows par (WSL2 ke bina ya Docker ke bina)**
Windows par natively Redis run karna thoda mushkil hota hai. 
* Agar aapke paas **Docker** hai, toh **Option 1** best hai.
* Agar Docker nahi hai, toh aapko Windows par WSL (Windows Subsystem for Linux) active karke uske andar Ubuntu terminal par Redis Stack install karna hoga. 

*(Agar aap Windows par hain aur Docker nahi chal raha, toh mujhe batayein, main aapko WSL ke direct commands de dunga).*

---

### **Check Kaise Karein Ki Redis Chal Raha Hai?**
Jab aap upar diye gaye tarike se server start kar dein, toh terminal par command chalayein:
```bash
redis-cli ping
```
Agar response mein **`PONG`** aata hai, iska matlab aapka server successfully active ho chuka hai.

Iske baad aap apna Python script phir se run karke dekhiye. Woh bina kisi error ke chalega aur aapko cache aur chat history ka output dikhayega. 

Jab yeh chal jaye, toh mujhe batayiye, phir hum **Module 2 (Vector Database)** par badhenge!
---

### **Pre-requisites (Setup)**
Pehle apne system mein Redis library install kar lijiye:
```bash
pip install redis
```
*(Maand kar chalte hain ki aapke system par Redis server chal raha hai, local port `6379` par).*

---

### **Module 1: Practical Code Implementation**

Hum ek Python script likhenge jo in teen cheezon ko manage karegi:
1. **Simple Prompt Cache** (Using Strings with TTL)
2. **User LLM Settings** (Using Hashes)
3. **Chat History Memory Window** (Using Lists with Limit and TTL)

Yahan hai iska clean aur simple code:

```python
import redis
import json
import time

# 1. Redis connection establish karna
# 'decode_responses=True' ka use karne se hume bytes ki jagah direct string data milta hai.
r = redis.Redis(host='localhost', port=6379, decode_responses=True)

# Test Connection
try:
    if r.ping():
        print("Connected to Redis successfully!\n")
except redis.ConnectionError:
    print("Error: Redis server is not running.")
    exit()


# ==========================================
# PATTERN 1: Prompt Caching (Using Strings + TTL)
# ==========================================
# Agar user ne same prompt pucha, toh LLM API call na karke cached response return karenge.

def get_llm_response(prompt: str) -> str:
    # 1. Pehle cache check karo
    cache_key = f"cache:{prompt.lower().strip()}"
    cached_response = r.get(cache_key)
    
    if cached_response:
        print("[CACHE HIT] Response loaded from Redis.")
        return cached_response
    
    # 2. Agar cache mein nahi mila, toh model call karo (Simulated LLM call)
    print("[CACHE MISS] Calling LLM API (takes time & cost)...")
    time.sleep(2)  # Delay simulate karne ke liye
    actual_response = f"Simulated LLM response for: '{prompt}'"
    
    # 3. Response ko cache mein save karo 60 seconds (TTL) ke liye
    # SETEX command = SET + EXPIRE in one atomic step
    r.setex(cache_key, 60, actual_response)
    print("[CACHE SAVE] Response cached for 60 seconds.")
    
    return actual_response


# ==========================================
# PATTERN 2: LLM Configuration Management (Using Hashes)
# ==========================================
# Har user ki specific LLM temperature, model name, aur max_tokens save karna.

def save_user_llm_config(user_id: str, model: str, temp: float, max_tokens: int):
    config_key = f"user:{user_id}:config"
    # HSET se hum poora dictionary ek sath save kar sakte hain
    r.hset(config_key, mapping={
        "model_name": model,
        "temperature": str(temp),
        "max_tokens": str(max_tokens)
    })
    print(f"[CONFIG SAVE] Saved settings for User {user_id}")

def get_user_llm_config(user_id: str) -> dict:
    config_key = f"user:{user_id}:config"
    # HGETALL saare field-value pairs return karega
    return r.hgetall(config_key)


# ==========================================
# PATTERN 3: Chat History with Context Window (Using Lists + LTRIM)
# ==========================================
# Chat history ko limit mein rakhna taaki LLM ki token limit exceed na ho.

def add_message_to_history(session_id: str, role: str, content: str, max_turns: int = 5):
    chat_key = f"chat:session:{session_id}"
    
    # Message ko JSON object ki tarah serialize karke save karte hain
    message_data = json.dumps({"role": role, "content": content})
    
    # RPUSH se message list ke end (right) mein push hoga
    r.rpush(chat_key, message_data)
    
    # LTRIM se list ko limit karenge taaki sirf last 'max_turns' messages hi rahein
    # -max_turns se lekar -1 tak (pichle kuch messages) bachenge, baaki automatic pop ho jayenge
    r.ltrim(chat_key, -max_turns, -1)
    
    # Chat session ko 30 minutes (1800 sec) ka TTL de dete hain
    r.expire(chat_key, 1800)

def get_chat_history(session_id: str) -> list:
    chat_key = f"chat:session:{session_id}"
    # LRANGE 0 to -1 matlab complete list fetch karna
    raw_history = r.lrange(chat_key, 0, -1)
    # JSON strings ko vapas python dictionaries mein convert karna
    return [json.loads(msg) for msg in raw_history]


# ==========================================
# Execution / Testing the Functions
# ==========================================

if __name__ == "__main__":
    print("--- 1. Testing Prompt Cache ---")
    # First time: Cache Miss hoga
    print(get_llm_response("What is Machine Learning?"))
    # Second time: Cache Hit hoga (Sub-millisecond latency)
    print(get_llm_response("What is Machine Learning?"))
    
    print("\n--- 2. Testing User Configuration ---")
    save_user_llm_config("user_abc", "gpt-4o", 0.2, 512)
    config = get_user_llm_config("user_abc")
    print(f"User Config fetched: {config}")
    
    print("\n--- 3. Testing Chat History Window ---")
    # Hum 4 messages bhejenge par limit sirf 3 rakhenge to check if LTRIM works
    add_message_to_history("session_1", "user", "Hi", max_turns=3)
    add_message_to_history("session_1", "assistant", "Hello! How are you?", max_turns=3)
    add_message_to_history("session_1", "user", "What is Redis?", max_turns=3)
    add_message_to_history("session_1", "assistant", "Redis is fast.", max_turns=3)
    
    history = get_chat_history("session_1")
    print("Latest Chat History (Max 3 messages kept):")
    for msg in history:
        print(f"  {msg['role']}: {msg['content']}")
```

---

### **Explanation of Key Commands Used:**
1. **`setex(key, time, value)`**: Yeh standard caching ke liye perfect hai. Ek hi call mein data store bhi karta hai aur timer (expiry) bhi start kar deta hai.
2. **`hset(key, mapping={...})`**: Complex data ya configurations ko single key ke andar fields ki tarah structure karne ke liye use hota hai.
3. **`ltrim(key, start, end)`**: Yeh list ko specific range ke bahar se trim (delete) kar deta hai. Isse hamara memory usage control mein rehta hai aur chat history kabhi bohot badi nahi hoti.

Agar aapke paas local computer mein Python hai, toh aap ise turant chala kar test kar sakte hain. 

Jab aap is code se comfortable ho jayein, toh batayein. Hum seedhe **Module 2: Redis as a Vector Database (RAG & Semantic Search)** par chalenge, jahan hum embeddings generate karenge, Redis mein index banayenge aur vector queries chala kar dekhenge.
