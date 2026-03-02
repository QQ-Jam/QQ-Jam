# ============ 配置文件 config.py ============

# 指定的 Host 文件夹路径
HOST_PATH = "/Users/你的用户名/文稿/Coco/Qianqian/Live/ABC123_小明"

# 扫描间隔（分钟）
SCAN_INTERVAL_MINUTES = 30

# 只处理创建时间在多少小时内的 LiveDate 文件夹
FOLDER_AGE_HOURS = 6

# Whisper 模型选择: tiny, small, medium, large-v3
WHISPER_MODEL = "medium"

# 语言
LANGUAGE = "zh"

# 数据库路径（和脚本放同一目录）
DATABASE_PATH = "Transcription.db"


# ============ transcriber.py（主程序）============
import os
import sys
import time
import sqlite3
from datetime import datetime, timedelta
from pathlib import Path

# 设置 Hugging Face 镜像（国内加速）
os.environ['HF_ENDPOINT'] = 'https://hf-mirror.com'

from faster_whisper import WhisperModel
from config import (
    HOST_PATH,
    SCAN_INTERVAL_MINUTES,
    FOLDER_AGE_HOURS,
    WHISPER_MODEL,
    LANGUAGE,
    DATABASE_PATH
)


def init_database():
    """初始化数据库和表"""
    conn = sqlite3.connect(DATABASE_PATH)
    cursor = conn.cursor()
    
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS Transcription_Record (
            Host TEXT NOT NULL,
            HostName TEXT NOT NULL,
            LiveDate TEXT NOT NULL,
            filename TEXT NOT NULL,
            Duration INTEGER,
            Flag INTEGER,
            PRIMARY KEY (Host, LiveDate, filename)
        )
    ''')
    
    conn.commit()
    conn.close()
    print("✅ 数据库初始化完成")


def parse_host_folder(host_path):
    """从路径解析 Host 和 HostName"""
    folder_name = os.path.basename(host_path.rstrip('/'))
    parts = folder_name.split('_', 1)
    host = parts[0]
    host_name = parts[1] if len(parts) > 1 else ""
    return host, host_name


def get_video_duration_minutes(file_path):
    """获取视频时长（分钟，四舍五入）"""
    try:
        from moviepy.editor import VideoFileClip
        with VideoFileClip(file_path) as clip:
            duration_seconds = clip.duration
            duration_minutes = round(duration_seconds / 60)
            return max(1, duration_minutes)  # 至少1分钟
    except Exception as e:
        print(f"⚠️ 无法获取视频时长: {e}")
        return 0


def is_folder_recent(folder_path, hours):
    """检查文件夹创建时间是否在指定小时内"""
    try:
        stat = os.stat(folder_path)
        create_time = datetime.fromtimestamp(stat.st_birthtime)
        cutoff_time = datetime.now() - timedelta(hours=hours)
        return create_time > cutoff_time
    except AttributeError:
        mtime = datetime.fromtimestamp(os.path.getmtime(folder_path))
        cutoff_time = datetime.now() - timedelta(hours=hours)
        return mtime > cutoff_time


def is_already_processed(host, live_date, filename):
    """检查是否已经处理过"""
    conn = sqlite3.connect(DATABASE_PATH)
    cursor = conn.cursor()
    
    cursor.execute('''
        SELECT 1 FROM Transcription_Record 
        WHERE Host = ? AND LiveDate = ? AND filename = ?
    ''', (host, live_date, filename))
    
    result = cursor.fetchone()
    conn.close()
    
    return result is not None


def save_record(host, host_name, live_date, filename, duration):
    """保存处理记录到数据库"""
    conn = sqlite3.connect(DATABASE_PATH)
    cursor = conn.cursor()
    
    cursor.execute('''
        INSERT INTO Transcription_Record 
        (Host, HostName, LiveDate, filename, Duration, Flag)
        VALUES (?, ?, ?, ?, ?, 1)
    ''', (host, host_name, live_date, filename, duration))
    
    conn.commit()
    conn.close()


def transcribe_video(model, video_path, output_path):
    """转录视频并保存文字"""
    print(f"🎬 开始转录: {os.path.basename(video_path)}")
    start_time = time.time()
    
    segments, info = model.transcribe(video_path, language=LANGUAGE)
    
    with open(output_path, 'w', encoding='utf-8') as f:
        for segment in segments:
            start_min = int(segment.start // 60)
            start_sec = int(segment.start % 60)
            timestamp = f"[{start_min:02d}:{start_sec:02d}]"
            f.write(f"{timestamp} {segment.text}\n")
    
    elapsed = round((time.time() - start_time) / 60, 1)
    print(f"✅ 转录完成: {os.path.basename(output_path)} (耗时 {elapsed} 分钟)")


def scan_and_process(model):
    """扫描并处理新视频"""
    host, host_name = parse_host_folder(HOST_PATH)
    
    print(f"\n{'='*50}")
    print(f"🔍 扫描中... Host: {host}, HostName: {host_name}")
    print(f"📁 路径: {HOST_PATH}")
    print(f"⏰ 时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'='*50}")
    
    if not os.path.exists(HOST_PATH):
        print(f"❌ 路径不存在: {HOST_PATH}")
        return
    
    processed_count = 0
    skipped_count = 0
    
    # 遍历 LiveDate 文件夹
    for live_date in os.listdir(HOST_PATH):
        live_date_path = os.path.join(HOST_PATH, live_date)
        
        # 跳过非文件夹
        if not os.path.isdir(live_date_path):
            continue
        
        # 检查文件夹是否在指定时间内创建
        if not is_folder_recent(live_date_path, FOLDER_AGE_HOURS):
            continue
        
        print(f"\n📂 发现新文件夹: {live_date}")
        
        # 遍历视频文件
        for filename in os.listdir(live_date_path):
            # 只处理 mp4 和 ts 文件
            if not filename.lower().endswith(('.mp4', '.ts')):
                continue
            
            video_path = os.path.join(live_date_path, filename)
            
            # 检查是否已处理过
            if is_already_processed(host, live_date, filename):
                print(f"   ⏭️ 已处理过，跳过: {filename}")
                skipped_count += 1
                continue
            
            # 生成输出文件路径
            base_name = os.path.splitext(filename)[0]
            output_path = os




pip install faster-whisper moviepy

# 启动服务
python transcriber.py

# 停止服务
按 Ctrl + C

# 后台运行，日志保存到 log.txt
nohup python transcriber.py > log.txt 2>&1 &

# 查看日志
tail -f log.txt

# 停止
pkill -f transcriber.py

首先我们有一个sqlite的DB叫Transcription.db
table name:Transcription_Record
column:
Host, 30位字符串，不能为空，Primary Key
HostName，50位字符串，不能为空
LiveDate，8位字符串，不能为空，Primary Key
filename,30位字符串，不能为空，Primary Key
Duration， integer类型
Flag，一位integer类型

我会指定一个文件夹：比如 文稿/Coco/Qianqian/Live/Host 这个路径,这个路径里面会有很多LiveDate的日期的文件夹，比较20260206或20260302等，你就扫这个文件夹的创建日期在6小时以为的文件夹就行，因为老早的你可能都已经处理过了，然后这个文件夹里面就会有视频文件，可能是.mp4或可能是.ts文件。
如果有那么就开始执行文字导出，导出后放到与视频同路径下，文件名也一样，后缀是txt或者是其他文本格式的文件就行。
最后再去Transcription_Record 这个table插入一笔记录，
Host就是“文稿/Coco/Qianqian/Live/Host”这个路径里面的Host文件夹名称的 _ 前面的字符串，
HostName就是“文稿/Coco/Qianqian/Live/Host”这个路径里面的Host文件夹名称中 _ 后面的字符串，
LiveDate就是“路径里面会有很多LiveDate的日期的文件夹”的文件夹名称，
filename就是你处理的视频文件的文件名，
Duration就是这个视频的长度（分钟），
Flag你就默认写1 

运行的机制：那么为了避免你重复的去处理同一个视频，每一次你可以先与table中的数据做比较有没有处理过，如果没有处理过是新的文件，你就可以开始工作了。

所以就是：每半小时（可配置）你就会跑起来，然后会去扫我指定的文件夹里面的创建日期为6小时以内的文件夹（可配置），如果里面有视频文件，那就先到Transcription_Record 这里看看你之前有没有处理过，
没有记录的话你就可以开始工作啦。
以上就是我的需求，你先正确理解一下，有什么不明确的你先问我，然后再生成代码。

补充：
1：Host 文件夹的格式，一定是 XXX_YYY 这种格式吗？， 是的一定是的，不用做exception处理。
2：扫描范围，你说扫描"6小时以内的文件夹"，是指 LiveDate 文件夹（如 20260206）的创建时间，对吗？ 是的！！！
3：多个 Host 文件夹，我会指定一个host路径，（如 文稿/Coco/Qianqian/Live/ABC123_小明）
4：视频时长单位， 四舍五入
5：运行方式 ，你这个问题太专业了我不懂，我只希望能让他稳定的一直跑下去，技术性的解决方案你想
6：模型选择， 默认用 medium 模型可以吗？ 可以的咱们先试试
        
        # 检查文件夹是否
