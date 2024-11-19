# index-of-website-download-image-url

```
import os
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin
from concurrent.futures import ThreadPoolExecutor
import queue
import json

# 设置初始URL和下载路径
base_url = "https://eigaland.com/wordpress/wp-content/uploads/"
download_path = "crunchyroll-vertrieb_uploads"

# 创建下载文件夹
if not os.path.exists(download_path):
    os.makedirs(download_path)

# 读取或初始化进度文件
progress_file = 'download_progress.json'
if os.path.exists(progress_file):
    with open(progress_file, 'r') as f:
        progress = json.load(f)
else:
    progress = {'last_processed_id': None}

# 定义全局队列
download_queue = queue.Queue()

def download_file(url, filepath):
    try:
        print(filepath)
        response = requests.get(url, stream=True)
        response.raise_for_status()
        with open(filepath, "wb") as f:
            for chunk in response.iter_content(chunk_size=8192):
                f.write(chunk)
    except requests.RequestException as e:
        print(f"Error downloading {url}: {e}")

def download_folders(url, path):
    try:
        response = requests.get(url)
        response.raise_for_status()
    except requests.RequestException as e:
        print(f"Error fetching {url}: {e}")
        return

    soup = BeautifulSoup(response.content, 'html.parser')
    for link in soup.find_all('a'):
        href = link.get('href')

        if href is None:
            continue
        if href.startswith('/'):
            continue
        if href.endswith('/') and href != "":
            # 如果是文件夹，递归下载
            new_url = urljoin(url, href)
            new_path = os.path.join(path, href.strip('/'))
            if not os.path.exists(new_path):
                os.makedirs(new_path)
            download_folders(new_url, new_path)
        else:
            print(href)
            # 下载文件
            new_url = urljoin(url, href)
            filepath = os.path.join(path, href)
            print(filepath)
            download_queue.put((new_url, filepath))

def worker():
    while True:
        try:
            url, filepath = download_queue.get(timeout=1)
            download_file(url, filepath)
            download_queue.task_done()
        except queue.Empty:
            break

# 填充队列
download_folders(base_url, download_path)

# 捕获KeyboardInterrupt异常
try:
    with ThreadPoolExecutor(max_workers=128) as executor:
        for _ in range(128):
            executor.submit(worker)
    download_queue.join()
except KeyboardInterrupt:
    print("Download paused, saving progress...")
    with open(progress_file, 'w') as f:
        json.dump({'last_processed_id': 'some_unique_identifier'}, f)
    raise

# 下载完成后清除进度文件
os.remove(progress_file)

```
