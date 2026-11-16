# ORB-SLAM2 CUDA on NVIDIA Jetson Nano (ROS Melodic + OpenCV 4.1.1 + USB Webcam)

Dokumentasi dan panduan instalasi lengkap (*end-to-end*) untuk membangun dan menjalankan **ORB_SLAM2_CUDA** dengan akselerasi GPU di **NVIDIA Jetson Nano** (Ubuntu 18.04 / ROS Melodic) menggunakan kamera **USB Webcam** secara *real-time*.

Panduan ini telah dilengkapi solusi permanen untuk mengatasi konflik versi library antara **ROS Melodic `cv_bridge` (OpenCV 3.2)** dan **Custom OpenCV 4.1.1 (CUDA)** yang umumnya menyebabkan *Segmentation Fault* (`exit code -11`).

---

## 📋 Spesifikasi Sistem yang Diuji

| Komponen | Versi / Spesifikasi |
| :--- | :--- |
| **Perangkat Keras** | NVIDIA Jetson Nano Developer Kit (4GB) |
| **Sistem Operasi** | Ubuntu 18.04 LTS (`aarch64` / JetPack 4.x) |
| **CUDA & OpenCV** | CUDA 10.2 & OpenCV 4.1.1 (Built with CUDA) |
| **ROS Distribution** | ROS Melodic Morenia (Python 2.7.17) |
| **Sensor Kamera** | USB Webcam (`/dev/video0`, 640×480 @ 30 FPS) |

---

## 🛠️ 1. Instalasi Dependensi Sistem

Perbarui sistem dan pasang seluruh pustaka dasar yang dibutuhkan oleh Pangolin, PCL, Eigen3, serta driver kamera ROS:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
    build-essential \
    cmake \
    git \
    libglew-dev \
    libpython2.7-dev \
    libeigen3-dev \
    libboost-all-dev \
    libpcl-dev \
    ros-melodic-pcl-ros \
    ros-melodic-usb-cam \
    ros-melodic-image-transport \
    ros-melodic-tf
```

*(Opsional)* Pastikan file swap aktif minimal 4 GB agar proses kompilasi di Jetson Nano tidak kehabisan RAM (*Out of Memory*).

---

## 🖥️ 2. Instalasi Pangolin (Visualisasi 3D)

Kloning dan kompilasi **Pangolin** (gunakan versi yang kompatibel dengan Ubuntu 18.04 / OpenGL):

```bash
cd ~
git clone --recursive https://github.com/stevenlovegrove/Pangolin.git
cd Pangolin
git checkout v0.6
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j2
sudo make install
sudo ldconfig
```

---

## 🚀 3. Kloning & Kompilasi Core Library `ORB_SLAM2_CUDA`

1. Kloning repositori `ORB_SLAM2_CUDA` ke direktori home (`~`):

```bash
cd ~
git clone https://github.com/tau-adl/ORB_SLAM2_CUDA.git
cd ~/ORB_SLAM2_CUDA
```

*(Ganti URL git di atas dengan URL repositori fork milik Anda jika sudah dipublikasikan).*

2. Bangun pustaka pihak ketiga (**DBoW2**, **g2o**), ekstrak **ORBvoc.txt**, dan kompilasi library inti `libORB_SLAM2_CUDA.so`:

```bash
chmod +x build.sh
./build.sh
```

> **Catatan Arsitektur GPU Jetson Nano:** Jika muncul error `nvcc fatal : Unsupported gpu architecture` saat menjalankan `./build.sh`, pastikan flag arsitektur CUDA di `CMakeLists.txt` diatur ke Maxwell (`-arch=sm_53` atau `-gencode arch=compute_53,code=sm_53`).

---

## 🔧 4. Mengatasi Konflik `cv_bridge` (OpenCV 3.2 vs OpenCV 4.1.1)

Secara bawaan, `ros-melodic-cv-bridge` menggunakan **OpenCV 3.2**, sedangkan `ORB_SLAM2_CUDA` menggunakan **OpenCV 4.1.1**. Untuk mencegah *Segmentation Fault* (`exit code -11`) saat node ROS menerima frame kamera, kompilasi ulang `cv_bridge` dalam workspace Catkin lokal:

### A. Kloning `vision_opencv` (Branch `noetic`) & Terapkan Patch Python 2.7
```bash
mkdir -p ~/cv_bridge_ws/src && cd ~/cv_bridge_ws/src
git clone -b noetic https://github.com/ros-perception/vision_opencv.git

# 1. Ubah target Boost dari python37 ke python (Python 2.7 di Ubuntu 18.04)
sed -i 's/python37/python/g' ~/cv_bridge_ws/src/vision_opencv/cv_bridge/CMakeLists.txt

# 2. Sesuaikan fungsi do_numpy_import() dengan makro import_array() NumPy Python 2.7
sed -i -E 's/static void\s*\*\s*do_numpy_import\s*\(\s*\)/static void do_numpy_import()/g' ~/cv_bridge_ws/src/vision_opencv/cv_bridge/src/module.hpp
sed -i '/return nullptr;/d' ~/cv_bridge_ws/src/vision_opencv/cv_bridge/src/module.hpp
sed -i 's/return do_numpy_import();/do_numpy_import();/g' ~/cv_bridge_ws/src/vision_opencv/cv_bridge/src/module.cpp
```

### B. Kompilasi Workspace `cv_bridge_ws`
```bash
cd ~/cv_bridge_ws
catkin_make -DCMAKE_BUILD_TYPE=Release -DOpenCV_DIR=/usr/local/lib/cmake/opencv4 -j2

# Aktifkan overlay workspace secara permanen di ~/.bashrc
grep -q "cv_bridge_ws" ~/.bashrc || echo "source ~/cv_bridge_ws/devel/setup.bash --extend" >> ~/.bashrc
source ~/.bashrc
```

---

## 🤖 5. Kompilasi Node ROS `ORB_SLAM2_CUDA`

Perbarui `CMakeLists.txt` pada folder ROS agar menggunakan Catkin modern dan menautkan (*link*) executable `Mono`, `Stereo`, serta `RGBD` ke `cv_bridge` OpenCV 4.1.1:

```bash
cd ~/ORB_SLAM2_CUDA/Examples/ROS/ORB_SLAM2_CUDA

cat << 'EOF' > CMakeLists.txt
cmake_minimum_required(VERSION 2.8.3)
project(ORB_SLAM2_CUDA)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -O3 -march=native")
add_definitions(-DCOMPILEDWITHC11)

find_package(catkin REQUIRED COMPONENTS
  roscpp
  tf
  sensor_msgs
  image_transport
  cv_bridge
  geometry_msgs
  nav_msgs
  pcl_ros
)

LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/../../../cmake_modules)

find_package(OpenCV 4 REQUIRED)
find_package(Eigen3 3.1.0 REQUIRED)
find_package(Pangolin REQUIRED)
find_package(CUDA REQUIRED)
find_package(PCL REQUIRED)

include_directories(
  ${catkin_INCLUDE_DIRS}
  ${PCL_INCLUDE_DIRS}
  ${PROJECT_SOURCE_DIR}
  ${PROJECT_SOURCE_DIR}/lib
  ${PROJECT_SOURCE_DIR}/../../../
  ${PROJECT_SOURCE_DIR}/../../../include
  ${Pangolin_INCLUDE_DIRS}
  ${EIGEN3_INCLUDE_DIR}
  ${CUDA_INCLUDE_DIRS}
)

link_directories(${PCL_LIBRARY_DIRS})
add_definitions(${PCL_DEFINITIONS})

set(LIBS
  ${catkin_LIBRARIES}
  ${OpenCV_LIBS}
  ${EIGEN3_LIBS}
  ${Pangolin_LIBRARIES}
  ${PCL_LIBRARIES}
  ${PROJECT_SOURCE_DIR}/../../../Thirdparty/DBoW2/lib/libDBoW2.so
  ${PROJECT_SOURCE_DIR}/../../../Thirdparty/g2o/lib/libg2o.so
  ${PROJECT_SOURCE_DIR}/../../../lib/libORB_SLAM2_CUDA.so
)

add_executable(Mono lib/SlamData.cc src/ros_mono.cc)
target_link_libraries(Mono ${LIBS})

add_executable(Stereo src/ros_stereo.cc)
target_link_libraries(Stereo ${LIBS})

add_executable(RGBD src/ros_rgbd.cc)
target_link_libraries(RGBD ${LIBS})
EOF

rm -rf build && mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j2
cp Mono Stereo RGBD ..
```

---

## 📷 6. Konfigurasi Jaringan Lokal & Launch File Webcam USB

1. Atur `ROS_MASTER_URI` dan `ROS_IP` ke antarmuka lokal (`127.0.0.1`) serta daftarkan `ROS_PACKAGE_PATH`:

```bash
sed -i '/ROS_IP/d;/ROS_HOSTNAME/d;/ROS_MASTER_URI/d;/ORB_SLAM2_CUDA\/Examples\/ROS/d' ~/.bashrc
printf '\nexport ROS_MASTER_URI=http://127.0.0.1:11311\nexport ROS_IP=127.0.0.1\nexport ROS_PACKAGE_PATH=${ROS_PACKAGE_PATH}:/home/nvidia/ORB_SLAM2_CUDA/Examples/ROS\n' >> ~/.bashrc
source ~/.bashrc
rospack profile
```

2. Buat file launch `ros_usb_cam.launch` untuk menghubungkan driver `usb_cam` (`/dev/video0`) ke topik `/camera/image_raw` dengan tampilan visualisasi Pangolin aktif (`bUseViewer=true`):

```bash
cat << 'EOF' > ~/ORB_SLAM2_CUDA/Examples/ROS/ORB_SLAM2_CUDA/launch/ros_usb_cam.launch
<?xml version="1.0"?>
<launch>
    <arg name="vocabulary_path" default="$(find ORB_SLAM2_CUDA)/../../../Vocabulary/ORBvoc.txt" />
    <arg name="camera_setting_path" default="$(find ORB_SLAM2_CUDA)/conf/IDScam.yaml" />
    <arg name="bUseViewer" default="true" />
    <arg name="bEnablePublishROSTopic" default="true" />

    <node name="usb_cam" pkg="usb_cam" type="usb_cam_node" output="screen">
        <param name="video_device" value="/dev/video0" />
        <param name="image_width" value="640" />
        <param name="image_height" value="480" />
        <param name="pixel_format" value="yuyv" />
        <param name="camera_frame_id" value="usb_cam" />
        <param name="io_method" value="mmap" />
        <remap from="/usb_cam/image_raw" to="/camera/image_raw" />
    </node>

    <node name="ORB_SLAM2_CUDA" pkg="ORB_SLAM2_CUDA" type="Mono" output="screen"
        args="$(arg vocabulary_path) $(arg camera_setting_path) $(arg bUseViewer) $(arg bEnablePublishROSTopic)">
    </node>
</launch>
EOF
```

---

## ▶️ 7. Menjalankan Visual SLAM (Real-Time)

Pastikan webcam USB sudah terpasang di `/dev/video0`, lalu maksimalkan *clock* CPU/GPU Jetson Nano dan jalankan `roslaunch`:

```bash
sudo jetson_clocks
roslaunch ORB_SLAM2_CUDA ros_usb_cam.launch
```

### Verifikasi Keberhasilan
* **Pemuatan Vocabulary:** `ORBvoc.txt` selesai dimuat dalam ±10–15 detik.
* **Tampilan Pangolin:** Jendela **ORB-SLAM2: Current Frame** menampilkan video langsung dari webcam dengan titik fitur ORB berwarna hijau, dan **ORB-SLAM2: Map Viewer** menampilkan peta *point cloud* 3D serta lintasan kamera.
* **Cek Linking OpenCV (Opsional):**
  ```bash
  ldd ~/ORB_SLAM2_CUDA/Examples/ROS/ORB_SLAM2_CUDA/Mono | grep opencv
  ```
  Pastikan hanya `libopencv_*.so.4.1` yang terdaftar (tidak ada `3.2`).

---

## 🐛 Ringkasan Troubleshooting

| Error | Penyebab | Solusi |
| :--- | :--- | :--- |
| `process has died [exit code -11]` | Bentrokan OpenCV 3.2 (`cv_bridge`) dan OpenCV 4.1.1 (`ORB_SLAM2_CUDA`). | Kompilasi ulang `cv_bridge` menggunakan OpenCV 4.1.1 (lihat **Bagian 4**). |
| `RLException: Unable to contact my own server` | `ROS_IP` masih mengarah ke IP Ethernet lama yang terputus. | Set `ROS_IP=127.0.0.1` dan `ROS_MASTER_URI=http://127.0.0.1:11311`. |
| `Could not find boost_python37` | Branch `noetic` mencari Boost Python 3.7 di Ubuntu 18.04 (Python 2.7). | Ubah `python37` menjadi `python` di `cv_bridge/CMakeLists.txt`. |
| `return-statement with no value (do_numpy_import)` | Perbedaan makro `import_array()` antara NumPy Python 2.7 dan Python 3. | Ubah return type `do_numpy_import()` di `module.hpp` menjadi `static void`. |

---

## 📝 Catatan Kalibrasi Kamera

Untuk akurasi pelacakan (*tracking*) dan skala pemetaan 3D yang presisi pada webcam Anda, lakukan kalibrasi kamera menggunakan `camera_calibration` ROS (`cameracalibrator.py`) dan perbarui parameter matriks intrinsik (`Camera.fx`, `Camera.fy`, `Camera.cx`, `Camera.cy`) serta koefisien distorsi (`Camera.k1`, `Camera.k2`, `Camera.p1`, `Camera.p2`) di dalam file `Examples/ROS/ORB_SLAM2_CUDA/conf/IDScam.yaml`.
