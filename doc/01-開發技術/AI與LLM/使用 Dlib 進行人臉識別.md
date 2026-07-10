
### 1. Dlib 使用環境建立
``` shell
# 安裝 opencv
sudo apt-get install python3-opencv  # Python版本
sudo apt-get install libopencv-dev   # C++ 開發套件

# 安裝套件
sudo apt-get install build-essential cmake libboost-all-dev

# 編譯及安裝 dlib
git clone https://github.com/davisking/dlib.git
cd dlib
mkdir build && cd build
cmake .. -DDLIB_USE_CUDA=0  # 禁用CUDA（Genio 350無NVIDIA GPU）
cmake --build . --config Release
sudo make install

# 安装 face_recognition（Python 套件）
pip3 install face_recognition
```

### 2. Demo 程式範例
``` python
import face_recognition
import cv2

video_capture = cv2.VideoCapture("/dev/video130")
known_image = face_recognition.load_image_file("known.jpg")
known_encoding = face_recognition.face_encodings(known_image)[0]

while True:
    ret, frame = video_capture.read()
    face_locations = face_recognition.face_locations(frame)
    face_encodings = face_recognition.face_encodings(frame, face_locations)

    for (top, right, bottom, left), face_encoding in zip(face_locations, face_encodings):
        match = face_recognition.compare_faces([known_encoding], face_encoding, tolerance=0.5)
        
        if match[0]:
            # 在辨識到的人臉周圍繪製矩形
            cv2.rectangle(frame, (left, top), (right, bottom), (0, 255, 0), 2)
            # 顯示辨識成功的訊息
            cv2.putText(frame, "Match Found!", (left, top - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.9, (0, 255, 0), 2)
        else:
            # 在未辨識到的人臉周圍繪製矩形
            cv2.rectangle(frame, (left, top), (right, bottom), (0, 0, 255), 2)
            # 顯示未辨識的訊息
            cv2.putText(frame, "Unknown", (left, top - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.9, (0, 0, 255), 2)

    cv2.imshow('Face Recognition', frame)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        break

video_capture.release()
cv2.destroyAllWindows()

```
