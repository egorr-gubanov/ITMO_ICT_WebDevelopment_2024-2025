# Задание 4: Многопользовательский Чат

## Практическое задание

Реализовать двухпользовательский или многопользовательский чат. Для максимального количества баллов реализуйте многопользовательский чат.

### Требования:
- Обязательно использовать библиотеку `socket`
- Для многопользовательского чата необходимо использовать библиотеку `threading`

### Реализация:
- **Протокол TCP:** 100% баллов
- **Протокол UDP:** 80% баллов
- Для UDP используйте threading для получения сообщений на клиенте
- Для TCP запустите клиентские подключения и обработку сообщений от всех пользователей в потоках. Не забудьте сохранять пользователей, чтобы отправлять им сообщения

### Полезные ссылки:
- [Документация Python: threading](https://docs.python.org/3/library/threading.html)
- [WebDevBlog: Введение в потоки Python](https://webdevblog.ru/vvedenie-v-potoki-python/)

---

## Реализация

### Сервер (task4/server.py)

```python
import socket
import threading
import time
from datetime import datetime

# Список активных клиентов
clients = []
clients_lock = threading.Lock()

def broadcast_message(message, sender_conn=None):
    """Отправляет сообщение всем клиентам кроме отправителя"""
    with clients_lock:
        for conn, name in clients:
            if conn != sender_conn:
                try:
                    conn.sendall(message.encode('utf-8'))
                except:
                    # Если не удалось отправить, удаляем клиента
                    clients.remove((conn, name))

def handle_client(conn, addr):
    """Обрабатывает подключение клиента"""
    global clients
    
    try:
        # Получаем имя клиента
        name = conn.recv(1024).decode('utf-8')
        if not name:
            return
            
        with clients_lock:
            clients.append((conn, name))
        
        print(f"Пользователь {name} присоединился к чату")
        broadcast_message(f"🔵 {name} присоединился к чату", conn)
        
        while True:
            try:
                message = conn.recv(1024).decode('utf-8')
                if not message:
                    break
                    
                if message == '/quit':
                    break
                
                # Форматируем сообщение с временной меткой
                timestamp = datetime.now().strftime('%H:%M:%S')
                full_message = f"[{timestamp}] {name}: {message}"
                
                print(full_message)
                broadcast_message(full_message, conn)
                
            except ConnectionResetError:
                break
            except Exception as e:
                print(f"Ошибка при получении сообщения от {name}: {e}")
                break
                
    except Exception as e:
        print(f"Ошибка обработки клиента {addr}: {e}")
    finally:
        # Удаляем клиента из списка
        with clients_lock:
            for i, (client_conn, client_name) in enumerate(clients):
                if client_conn == conn:
                    clients.pop(i)
                    print(f"Пользователь {client_name} покинул чат")
                    broadcast_message(f"🔴 {client_name} покинул чат", conn)
                    break
        
        conn.close()

def run_server():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server_socket:
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        server_socket.bind(('127.0.0.1', 8004))
        server_socket.listen(5)
        
        print("Чат-сервер")
        print("=" * 20)
        print(f"Сервер запущен на порту 8004...")
        print(f"Протокол: TCP")
        print(f"Многопоточность: Включена")
        print("\nОжидание подключений...")
        print("Нажмите Ctrl+C для остановки...")
        
        try:
            while True:
                conn, addr = server_socket.accept()
                print(f"Новое подключение от {addr}")
                
                # Создаем поток для обработки клиента
                client_thread = threading.Thread(
                    target=handle_client, 
                    args=(conn, addr),
                    daemon=True
                )
                client_thread.start()
                
        except KeyboardInterrupt:
            print("\nОстановка сервера...")
            with clients_lock:
                for conn, name in clients:
                    conn.close()
            print("Сервер остановлен")

if __name__ == "__main__":
    run_server()
```

### Клиент (task4/client.py)

```python
import socket
import threading
import sys

def receive_messages(client_socket):
    """Получает сообщения от сервера в отдельном потоке"""
    while True:
        try:
            message = client_socket.recv(1024).decode('utf-8')
            if not message:
                break
            print(f"\n{message}")
            print("Введите сообщение (или /quit для выхода): ", end="", flush=True)
        except ConnectionResetError:
            print("\nСоединение с сервером разорвано")
            break
        except Exception as e:
            print(f"\nОшибка получения сообщения: {e}")
            break

def run_client():
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as client_socket:
            # Подключаемся к серверу
            client_socket.connect(('127.0.0.1', 8004))
            
            # Получаем имя пользователя
            name = input("Введите ваше имя: ").strip()
            if not name:
                name = "Аноним"
            
            # Отправляем имя серверу
            client_socket.sendall(name.encode('utf-8'))
            
            print(f"Добро пожаловать в чат, {name}!")
            print("Введите сообщение (или /quit для выхода): ", end="", flush=True)
            
            # Запускаем поток для получения сообщений
            receive_thread = threading.Thread(
                target=receive_messages, 
                args=(client_socket,),
                daemon=True
            )
            receive_thread.start()
            
            # Основной цикл отправки сообщений
            while True:
                try:
                    message = input()
                    
                    if message == '/quit':
                        client_socket.sendall(message.encode('utf-8'))
                        break
                    
                    if message.strip():
                        client_socket.sendall(message.encode('utf-8'))
                        
                except KeyboardInterrupt:
                    print("\nВыход...")
                    client_socket.sendall('/quit'.encode('utf-8'))
                    break
                except Exception as e:
                    print(f"Ошибка отправки сообщения: {e}")
                    break
                    
    except ConnectionRefusedError:
        print("Ошибка: не удалось подключиться к серверу")
        print("Убедитесь, что сервер запущен")
    except Exception as e:
        print(f"Ошибка: {e}")

if __name__ == "__main__":
    run_client()
```

## Запуск

1. **Запустите сервер:**
   ```bash
   python task4/server.py
   ```

2. **Запустите несколько клиентов в разных терминалах:**
   ```bash
   python task4/client.py
   ```

3. **В каждом клиенте:**
   - Введите ваше имя
   - Начинайте общаться!

## Результат

**Сервер:**
```
Чат-сервер
====================
Сервер запущен на порту 8004...
Протокол: TCP
Многопоточность: Включена

Ожидание подключений...
Новое подключение от ('127.0.0.1', 54321)
Пользователь Алиса присоединился к чату
Новое подключение от ('127.0.0.1', 54322)
Пользователь Боб присоединился к чату
[14:30:15] Алиса: Привет всем!
[14:30:18] Боб: Привет, Алиса!
```

**Клиент 1 (Алиса):**
```
Введите ваше имя: Алиса
Добро пожаловать в чат, Алиса!
Введите сообщение (или /quit для выхода): Привет всем!

[14:30:18] Боб: Привет, Алиса!
```

**Клиент 2 (Боб):**
```
Введите ваше имя: Боб
Добро пожаловать в чат, Боб!

[14:30:15] Алиса: Привет всем!
Введите сообщение (или /quit для выхода): Привет, Алиса!
```

## Выводы

1. Реализован многопользовательский чат с поддержкой TCP протокола
2. Использована многопоточность для обработки множественных клиентов
3. Добавлена система имен пользователей и временных меток
4. Реализована трансляция сообщений всем участникам чата
5. Добавлена корректная обработка отключений клиентов
6. Чат поддерживает неограниченное количество участников
