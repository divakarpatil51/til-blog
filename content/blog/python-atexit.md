+++
date = '2025-07-05T21:31:01+10:00'
draft = false
title = 'How to Use Python’s atexit Module for Cleanup in Networked Applications'
+++

I recently came across Python's [`atexit`](https://docs.python.org/3/library/atexit.html) module and became curious about practical use cases in real-world applications. To explore this, I created a simple client-server app: the client publishes data periodically, and the server consumes and stores it.

### A bit about `atexit` before we get into the app:

> The `atexit` module defines functions to register and unregister cleanup functions. Functions thus registered are automatically executed upon normal interpreter termination.

In short, you can register functions with `atexit` to perform cleanup tasks when your Python application exits normally.  

However, there's a caveat: these functions **won’t** run if a fatal internal error occurs or if you use [`os._exit()`](https://docs.python.org/3/library/os.html#os._exit). In such cases, the [`signal`](https://docs.python.org/3/library/signal.html) module can help — it allows you to handle termination signals like `SIGINT` or `SIGTERM` and run custom cleanup code before the process exits.

## Code:
#### Server:

The server listens on `0.0.0.0:12345` and uses `MyUDPHandler` to queue incoming data into `ingest_queue`. A separate daemon thread continuously processes this queue in the background. Because daemon threads don’t block the program from exiting, it’s important to use `atexit` to handle cleanup. The `shutdown_handler` function, registered with `atexit`, ensures that any remaining data still in the queue is properly flushed to the database when the server shuts down gracefully.

> Note: This example uses a shared in-memory SQLite database for testing purposes. In a real app, you’d want to use a persistent on-disk database or a shared DB connection pool.


```python
import atexit
import queue
import time
from typing import TypedDict
import threading
import json
import socketserver
from typing import override
import logging
import sqlite3

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(thread)d | %(message)s",
)
logger = logging.getLogger(__name__)


SENSOR_MEASUREMENT_TABLE = "sensor_measurements"


def get_connection():
    # Using an in-memory database for testing purposes
    return sqlite3.connect(
        "file:memdb1?mode=memory&cache=shared",
        uri=True,
        timeout=20,
        check_same_thread=False,
    )


def initialize_db():
    with get_connection() as conn:
        conn.execute(
            f"""
            CREATE TABLE IF NOT EXISTS {SENSOR_MEASUREMENT_TABLE} (
                sensor_id integer,
                temperature decimal,
                humidity decimal,
                ingested_at timestamp);
            """
        )


class SensorMetrics(TypedDict):
    sensor_id: int
    temperature: float
    humidity: float
    ingested_at: float


ingest_queue: queue.Queue[SensorMetrics] = queue.Queue()


def _persist_data(metrics: list[SensorMetrics], conn: sqlite3.Connection):
    buffer = [(*metric.values(),) for metric in metrics]
    try:
        conn.executemany(
            f"""
            INSERT INTO {SENSOR_MEASUREMENT_TABLE} 
            (sensor_id, temperature, humidity, ingested_at)
            values (?,?,?,?)""",
            buffer,
        )
        conn.commit()
    except Exception:
        logger.exception("Error occurred while persisting data")


class MyUDPHandler(socketserver.DatagramRequestHandler):
    @override
    def handle(self):
        logger.info("Processing request from %s", self.client_address)
        data: SensorMetrics = json.loads(self.request[0].strip().decode("utf-8"))
        ingest_queue.put(data)


def _process_queued_data():
    with get_connection() as conn:
        while True:
            # Emulating a delay to simulate real-time processing
            time.sleep(0.1)
            _persist_data([ingest_queue.get()], conn)


def shutdown_handler():
    logger.info("Shutting down server, persisting remaining data...")
    buffer = []
    while not ingest_queue.empty():
        buffer.append(ingest_queue.get())

    if buffer:
        with get_connection() as conn:
            _persist_data(buffer, conn)

            cursor = conn.execute("SELECT COUNT(*) FROM sensor_measurements")
            logger.info(
                "Total records persisted in shutdown_handler: %s", cursor.fetchone()[0]
            )


atexit.register(shutdown_handler)


def main() -> None:
    initialize_db()
    threading.Thread(target=_process_queued_data, daemon=True).start()
    with socketserver.ThreadingUDPServer(("0.0.0.0", 12345), MyUDPHandler) as server:
        logger.info("Server started at localhost:12345")
        server.serve_forever()


if __name__ == "__main__":
    main()
```

#### Client:

```python
import socket
import logging
import time
import random
import json

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(thread)d | %(message)s",
)
logger = logging.getLogger(__name__)


def main() -> None:
    client_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    addr = ("localhost", 12345)
    logger.info("Connected to server: %s", addr)

    try:
        while True:
            data = {
                "sensor_id": random.randint(10, 1000),
                "temperature": random.uniform(1, 35),
                "humidity": random.uniform(1, 12),
                "ingested_at": time.time(),
            }
            client_socket.sendto(json.dumps(data).encode(), addr)
            time.sleep(0.01)
    except KeyboardInterrupt:
        print("\nClient interrupted and shutting down.")
    finally:
        client_socket.close()


if __name__ == "__main__":
    main()
```

### Final Thoughts:
This small project helped me understand where `atexit` can fit in a typical app: cleaning up or persisting data before shutdown. In this case, it ensures any remaining items in the queue are flushed to the database when the server exits normally.

If you’re building something that needs graceful shutdown (like logging, data persistence, or closing open connections), `atexit` is a simple and elegant tool to consider.
