# 自架NAS 網頁骨架網頁     創作者:謝秉棋

## 主要學習:
1. 基礎檔案架構
2. 後端資料庫處理
3. 程式用到套件
4. 程式完成打包成Docker鏡像
5. 後端程式講解


## 基礎檔案架構:
1. 完成版辨識網頁 (https://identify.qi0527.com)
2. 檔案資料架構
3. 讓google搜尋到自己的網頁Search console(https://search.google.com/search-console/about)(像是在google搜尋謝秉棋或qi0527 都會有我的個人網頁:https://qi0527.com/)
```
├── app.py
├── templates/
│   ├── index.html    #登陸會員首頁
│   ├── member.html   #辨識圖片與上傳圖片網頁
│   └── error.html    #網頁錯誤網頁
├── data/
│   ├── data1.xml     #臉部辨識所需xml
│   └── eye.xml       #臉部辨識所需xml
├── static/
    └── 1.jpg         #網頁背景圖片 
    
```

## 程式用到套件:
1. 網頁伺服器(Flask)
2. 辨識需要(cv2,numpy,mediaipie)
3. 資料庫MongoDB 需要的套件(MongoDB,GridFs,ObjectId)
4. 其他套件(圖片編碼base64, 密碼哈希flask_bcrypt)

## 程式完成打包成Docker鏡像:
1. 一定要先下載docker
2. 先在nas docker安裝python 3.9.9 slim 或更高版本但要slim  在linux cmd 輸入:docker pull python:3.9-slim-bullseye (slim輕量級適合小型應用程序或微服務)
3. 創建tmp 資料夾 mkdir /home/qi/tmp   (依照自己的路徑隨意創建)
4. cd /home/qi/tmp  
5. 先在tmp資料夾內上傳(app.py,templates,data,static)所有資料夾
6. nano Dockerfile
7. 在dockerfile內上傳 (Ctrl+o 儲存ctrl+x退出)
```
ROM python:3.9.9

RUN pip3 install numpy
RUN pip3 install Flask
RUN pip3 install mediapipe
RUN pip3 install Flask-Bcrypt
RUN pip3 install pymongo
RUN apt-get update && apt-get install -y \
    libgl1-mesa-glx \
    libsm6 \
    libxext6 \
    libxrender-dev
RUN pip3 install opencv-python-headless
RUN mkdir -p /identify/
COPY ./app.py /identify
COPY ./templates /identify/templates
COPY ./static /identify/static
COPY ./data /identify/data
WORKDIR /identify
CMD [ "python", "/identify/app.py"]

```
8. 打包成 identifypython:v01 鏡像

```
sudo docker image build -t identifypython:v01 .   
```

9. 運行

```
sudo docker container run --rm -it -p 192.168.0.227:7500:7500 identifypython:v01
```
10. 完整上架網頁要到portainer 上架網頁並放入你的鏡像與端口

![contents](/images/portainer2.png)

## 後端程式講解

### 資料庫處理
```
from bson import ObjectId
from gridfs import GridFS
from pymongo import MongoClient
import base64

# 連接到 MongoDB
client = MongoClient("填入自己的")
db = client.skeleton_code

# 初始化 GridFS 用於 MongoDB 圖像存儲
fs = GridFS(db)

@app.route("/signin", methods=["POST"])
def signin():
    name = request.form["name"]
    password = request.form["password"]

    user = db.users.find_one({"name": name})

    if user:
        if bcrypt.check_password_hash(user['password'], password):
            session.clear()  # 清除所有會話變量
            session['username'] = name
            session[name + '_logged_in'] = True
            return jsonify(success=True)
        else:
            return jsonify(success=False, message="密碼錯誤")
    else:
        hashed_password = bcrypt.generate_password_hash(password).decode('utf-8')
        user_data = {"name": name, "password": hashed_password, "image_count": 0, "original_image_ids": [], "processed_image_ids": []}
        db.users.insert_one(user_data)

        session.clear()  # 清除所有會話變量
        session['username'] = name
        session[name + '_logged_in'] = True
        return jsonify(success=True)
```

## 辨識方式要用物件導向

```
# Mediapipe 姿勢檢測類
class PoseDetector:
    def __init__(self):
        self.mp_pose = mp.solutions.pose
        self.pose = self.mp_pose.Pose(min_detection_confidence=0.5, min_tracking_confidence=0.5)
        self.face_cascade = cv2.CascadeClassifier(cv2.data.haarcascades + 'haarcascade_frontalface_default.xml')

    def process_image(self, image_data):
        try:
            # 解碼圖像數據為 OpenCV 圖像格式
            image = cv2.imdecode(np.frombuffer(image_data, np.uint8), cv2.IMREAD_COLOR)
            if image is None:
                raise Exception("無法加載圖像。")
        except Exception as e:
            return None, str(e)

        try:
            # 將圖像轉換為 RGB 格式
            image_rgb = cv2.cvtColor(image, cv2.COLOR_BGR2RGB)

            height, width, _ = image_rgb.shape
            image_resized = cv2.resize(image_rgb, (width, height))

            # 使用 Mediapipe 進行姿勢檢測
            with self.mp_pose.Pose(min_detection_confidence=0.5, min_tracking_confidence=0.5) as pose:
                results = pose.process(image_resized)

                if results.pose_landmarks:
                    # 在圖像上繪製檢測到的姿勢標誌
                    mp.solutions.drawing_utils.draw_landmarks(
                        image_resized,
                        results.pose_landmarks,
                        self.mp_pose.POSE_CONNECTIONS,
                        landmark_drawing_spec=mp.solutions.drawing_styles.get_default_pose_landmarks_style()
                    )

            # 如有需要，將圖像轉回 BGR 格式
            processed_image = cv2.cvtColor(image_resized, cv2.COLOR_RGB2BGR)

            # 人臉檢測
            gray_image = cv2.cvtColor(processed_image, cv2.COLOR_BGR2GRAY)
            faces = self.face_cascade.detectMultiScale(gray_image, scaleFactor=1.3, minNeighbors=5, minSize=(30, 30))

            for (x, y, w, h) in faces:
                cv2.rectangle(processed_image, (x, y), (x + w, y + h), (255, 0, 0), 2)


        except Exception as e:
            return None, str(e)

        return processed_image, None
```

