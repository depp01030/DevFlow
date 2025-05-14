# 🔥 那 FastAPI 呢？

✅ FastAPI 本身是 Application，不是 Server。

- FastAPI 是幫你定義好：「如果有人來問 GET / 我要回什麼」
- FastAPI 不負責打開 port，不負責監聽 request

---

所以，如果你直接執行：

```bash
python my_fastapi.py
```

會發生什麼？

- FastAPI 只會建立一個 `app` 物件，定義好請求處理的綱要。
- 然後 Python 程式會順順的執行完，結束。
- 沒有打開網路 port，沒有連線，沒有等待 request，也沒有持續上線。

因為：

> 只是一段执行完就結束的程式，沒有一個會一直等待連線的 server 在跟你合作。

一個 Web Server 最基本要求：不能执行完就關掉，而是要持續上線，監聽上百上千的 request，等有人來確認連線。

---

# 🔥 那誰來幫 FastAPI 把門打開，聽外界的 request？

✅ 答案是：**Uvicorn！**

- Uvicorn 是 ASGI Server
- Uvicorn 負責：
  - 打開 socket (IP + port)
  - 持續監聽這個小門
  - 收到 request 時，把 request 上交給 FastAPI app
  - FastAPI app 處理後，回應給 Uvicorn，再由 Uvicorn 送回 client (e.g. 瀏覽器)

所以：

> **FastAPI 是「大腦」，Uvicorn 是「耳朵跟嘴巴」。**

---

# 整體流程概覽

1. 啟動 Uvicorn：
```bash
uvicorn my_fastapi:app --host 0.0.0.0 --port 8000
```
2. Uvicorn 打開一個 TCP socket，監聽 8000 port
3. 有 HTTP request 進來，Uvicorn 接收
4. Uvicorn 將 request 轉交給 FastAPI app
5. FastAPI app 處理完，回應給 Uvicorn
6. Uvicorn 再把回應給 client

---

# 🌟 小結

- **FastAPI 是處理規則，不是 server，本身不會打開 port 或等待連線**
- **需要 Uvicorn 作為 server，打開門，一直等待 request 來點開 FastAPI app**

