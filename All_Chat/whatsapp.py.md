| `hashlib`  | Python module          | Provides hashing function like SHA-256                   |
| ---------- | ---------------------- | ---------------------------------------------------------|
| `hmac`     | module                 | Create secure message signature                          |
| `json`     | module                 | Work with JSON data                                      |
| `os`       | module                 | Work with operating system, files, environment variables |
| `requests` | module                 | Send HTTP request to website/API                         |
| `pathlib`  | module                 | Work with file and folder path                           |
| `Path`     | Class inside `pathlib` | Represent file/folder path                               |

---


| `key`                             | name of variable                                |
| --------------------------------- | ----------------------------------------------- |
| `default`                         | fallback value                                  |
| `__file__`                        | path of current Python file                     |
| `Path(__file__)`                  | Turn that path into Path object.                |
| `.resolve()`                      | Get absolute path                               |
| `.parent.parent`                  | go up 2 folder                                  |
|                                   | /home/furba/project/whatsapp/server.py          |
|                                   | /home/furba/project                             |
| `/ "whatsapp_server_config.json"` | /home/furba/project/whatsapp_server_config.json |
| `json.load(f)`                    | Convert Json.data --> python.data               |
| `OSError`                         | Problem opening/reading  file                   |
|                                   | Example: file doesn't exist                     |
| `ValueError`                      | Problem with JSON                               |
|                                   | Example: invalid JSON                           |
| json.dump(f)                      | python.data ---> json.data                      |
|                                   |                                                 |

---

| data[key] = value | `data = {"WHATSAPP_ACCESS_TOKEN": "old"}`      |
| ----------------- | ---------------------------------------------- |
|                   | key = "WHATSAPP_ACCESS_TOKEN"<br>value = "new" |
|                   | `data = {"WHATSAPP_ACCESS_TOKEN": "new"}`      |
