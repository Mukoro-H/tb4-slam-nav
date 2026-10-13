# TurtleBot 4 – SLAM, Localization & Navigation (ROS 2 Humble + Gazebo)

Xây dựng bản đồ bằng **SLAM Toolbox**, định vị bằng **AMCL** và điều hướng tự động bằng **Nav2** cho robot TurtleBot 4 trong môi trường mô phỏng **Ignition Gazebo Fortress**.

| | |
|---|---|
| **Sinh viên** | _Họ tên – MSSV_ |
| **Video demo** | _[Google Drive link]_ |
| **Báo cáo PDF** | [`docs/report.pdf`](docs/report.pdf) |

---

## 1. Môi trường

| Thành phần | Phiên bản |
|---|---|
| Hệ điều hành | Ubuntu 22.04 LTS (native hoặc WSL2) |
| ROS 2 | Humble Hawksbill (Desktop) |
| Simulator | Ignition Gazebo Fortress |
| Robot | TurtleBot 4 (hoặc TurtleBot 4 Lite) |
| World | `warehouse` (có sẵn trong `turtlebot4_ignition_bringup`) |

## 2. Cài đặt

### 2.1. ROS 2 Humble

Cài theo hướng dẫn chính thức: <https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html>

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 2.2. Các package TurtleBot 4, SLAM Toolbox, Nav2

```bash
sudo apt update
sudo apt install -y \
  ros-humble-turtlebot4-simulator \
  ros-humble-turtlebot4-desktop \
  ros-humble-turtlebot4-navigation \
  ros-humble-slam-toolbox \
  ros-humble-navigation2 \
  ros-humble-nav2-bringup \
  ros-humble-nav2-rviz-plugins \
  ros-humble-teleop-twist-keyboard
```

### 2.3. Gazebo Fortress

Nếu lệnh `sudo apt install ignition-fortress` báo không tìm thấy gói thì thêm repo OSRF trước:

```bash
sudo curl https://packages.osrfoundation.org/gazebo.gpg -o /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] http://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/gazebo-stable.list
sudo apt update && sudo apt install -y ignition-fortress
```

### 2.4. Clone repository

```bash
git clone https://github.com/<your-username>/tb4-slam-nav.git
cd tb4-slam-nav
chmod +x scripts/cleanup.sh
mkdir -p ~/maps && cp maps/map.yaml maps/map.pgm ~/maps/
```

### 2.5. (Khuyến nghị) GPU NVIDIA

Gazebo chạy bằng CPU (`llvmpipe`) sẽ rất chậm. Khi đó robot gần như không di chuyển và controller dễ bị lỗi timeout. Kiểm tra như sau:

```bash
glxinfo -B | grep "OpenGL renderer"   # cần ra NVIDIA ..., không phải llvmpipe
```

Nếu kết quả chưa phải NVIDIA thì cài driver: `sudo ubuntu-drivers autoinstall && sudo reboot`.

## 3. Cấu trúc repository

```
tb4-slam-nav/
├── maps/
│   ├── map.yaml          # metadata bản đồ (resolution, origin, ngưỡng occupied/free)
│   └── map.pgm           # ảnh occupancy grid tạo bởi SLAM Toolbox
├── scripts/
│   └── cleanup.sh        # tắt sạch Gazebo/ROS còn chạy ngầm trước mỗi lần launch
├── docs/
│   ├── report.pdf
│   └── images/           # ảnh minh hoạ (rqt_graph, map, RViz, ...)
└── README.md
```

## 4. Chạy chương trình

> Mỗi dòng `T1`, `T2`, ... là **một terminal riêng**.
> Trước mỗi phần, chạy `./scripts/cleanup.sh` để tắt hết tiến trình cũ.
> Máy yếu thì thêm `model:=lite` vào lệnh Gazebo ở **tất cả** các phần.

### Phần 1 – TurtleBot 4 Simulation

```bash
# T1 – Gazebo + robot
ros2 launch turtlebot4_ignition_bringup turtlebot4_ignition.launch.py world:=warehouse

# T2 – robot spawn trên dock nên phải undock trước khi chạy
ros2 action send_goal /undock irobot_create_msgs/action/Undock "{}"

# T2 – kiểm tra topic
ros2 topic hz /scan                 # LiDAR
ros2 topic echo /odom --once        # odometry
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}}"   # Ctrl+C để dừng
ros2 run rqt_graph rqt_graph

# T3 – RViz (Fixed Frame = odom)
ros2 launch turtlebot4_viz view_robot.launch.py
```

### Phần 2 – Xây dựng bản đồ bằng SLAM Toolbox

```bash
# T1 – Gazebo (như Phần 1), T2 – undock (như Phần 1)

# T3 – SLAM Toolbox
ros2 launch turtlebot4_navigation slam.launch.py use_sim_time:=true

# T4 – RViz (Fixed Frame = map)
ros2 launch turtlebot4_viz view_robot.launch.py

# T5 – teleop (click vào terminal này rồi giữ phím i / j / l / ,)
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Một số lưu ý để bản đồ sạch:
- Đi chậm khoảng 0.2 m/s (bấm `z` để giảm tốc).
- Quay chậm.
- Quay lại điểm xuất phát để SLAM Toolbox thực hiện **loop closure**.

**Lưu bản đồ** khi SLAM vẫn đang chạy:

```bash
mkdir -p ~/maps && cd ~/maps
ros2 run nav2_map_server map_saver_cli -f map
ls ~/maps    # map.yaml  map.pgm
```

### Phần 3 – Localization (AMCL)

```bash
./scripts/cleanup.sh

# T1 – Gazebo, kiểm tra robot đang nằm trên dock
ros2 launch turtlebot4_ignition_bringup turtlebot4_ignition.launch.py world:=warehouse

# T2 – map_server + AMCL
ros2 launch turtlebot4_navigation localization.launch.py map:=$HOME/maps/map.yaml use_sim_time:=true

# T3 – RViz của Nav2 (có sẵn map, particle cloud, costmap, nút Nav2 Goal)
ros2 launch nav2_bringup rviz_launch.py
```

1. Bấm **2D Pose Estimate**, click vào vị trí dock trên map rồi kéo theo hướng mũi robot. Dock nằm ở gốc `(0, 0)` của bản đồ vì SLAM bắt đầu từ dock. Có thể đặt bằng lệnh thay cho thao tác trên RViz:
   ```bash
   ros2 topic pub --once /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
   "{header: {frame_id: map}, pose: {pose: {position: {x: 0.0, y: 0.0}, orientation: {w: 1.0}}}}"
   ```
2. Ở T4, undock rồi chạy teleop để lái một đoạn ngắn. Particle cloud sẽ hội tụ và các điểm LiDAR khớp với tường trên map.
3. So sánh vị trí robot trên RViz với Gazebo, sau đó tắt teleop.

### Phần 4 – Autonomous Navigation (Nav2)

Giữ nguyên các terminal của Phần 3:

```bash
# T4 – Nav2 (planner, controller, behavior tree, costmaps)
ros2 launch turtlebot4_navigation nav2.launch.py use_sim_time:=true
```

Đợi log báo `Managed nodes are active`. Sau đó dùng **Nav2 Goal** trên RViz để đặt goal: click vào điểm đích rồi kéo để chọn hướng. Đề yêu cầu thực hiện **ít nhất 3 goal ở các khu vực khác nhau** và không điều khiển robot thủ công trong khi Nav2 đang chạy.

## 5. Node, topic và TF chính

| Node | Package | Vai trò |
|---|---|---|
| `ign gazebo` + `ros_gz_bridge` | `turtlebot4_ignition_bringup` | Mô phỏng robot, cảm biến; bridge Gazebo ↔ ROS 2 |
| `motion_control`, `diffdrive_controller` | `irobot_create_*`, `ros2_control` | Nhận `/cmd_vel`, điều khiển bánh, publish `/odom` |
| `robot_state_publisher` | `robot_state_publisher` | Publish TF giữa các link của robot từ URDF |
| `slam_toolbox` | `slam_toolbox` | Scan matching + pose graph → `/map`, TF `map → odom` |
| `map_server` | `nav2_map_server` | Load `map.yaml` → `/map` |
| `amcl` | `nav2_amcl` | Particle filter định vị robot trên map → TF `map → odom` |
| `planner_server` | `nav2_planner` | Lập đường đi toàn cục trên global costmap |
| `controller_server` | `nav2_controller` | Bám đường đi, tránh vật cản cục bộ → `/cmd_vel` |
| `bt_navigator` | `nav2_bt_navigator` | Behavior tree điều phối plan / follow / recovery |
| `teleop_twist_keyboard` | `teleop_twist_keyboard` | Điều khiển bằng bàn phím → `/cmd_vel` |

| Topic | Kiểu | Mô tả |
|---|---|---|
| `/scan` | `sensor_msgs/LaserScan` | Dữ liệu LiDAR RPLIDAR A1 |
| `/odom` | `nav_msgs/Odometry` | Odometry từ bánh xe |
| `/cmd_vel` | `geometry_msgs/Twist` | Lệnh vận tốc |
| `/map` | `nav_msgs/OccupancyGrid` | Bản đồ (từ SLAM hoặc map_server) |
| `/initialpose` | `geometry_msgs/PoseWithCovarianceStamped` | Vị trí ban đầu cho AMCL |
| `/particle_cloud` | `nav2_msgs/ParticleCloud` | Tập particle của AMCL |
| `/plan` | `nav_msgs/Path` | Đường đi toàn cục |
| `/goal_pose` | `geometry_msgs/PoseStamped` | Goal từ RViz |

Cây TF: `map → odom → base_link → rplidar_link`

- `map → odom`: do SLAM Toolbox publish ở Phần 2, do AMCL publish ở Phần 3 và Phần 4.
- `odom → base_link`: tính từ odometry.
- `base_link → rplidar_link`: lấy từ URDF.

## 6. Xử lý lỗi thường gặp

| Hiện tượng | Nguyên nhân / cách xử lý |
|---|---|
| Teleop không điều khiển được robot | Robot chưa undock; chưa click vào cửa sổ teleop; Gazebo đang pause; sim chạy bằng CPU (RTF thấp) |
| Nút ▶ của Gazebo nhấp nháy, robot gần như không di chuyển | Gazebo đang render bằng `llvmpipe`, cần cài driver NVIDIA (xem mục 2.5) |
| `waiting for service /controller_manager/list_controllers` | Các tiến trình khởi động lệch nhau. Chạy `./scripts/cleanup.sh` rồi launch lại |
| Robot không nằm trên dock khi mở lại Gazebo | Gazebo cũ chưa tắt hẳn. Chạy `./scripts/cleanup.sh` |
| `Failed to load map yaml file ... bad file` | Sai đường dẫn: kiểm tra `ls ~/maps` |
| `AMCL cannot publish a pose ... set the initial pose` | Bình thường. Đặt **2D Pose Estimate** |
| Không thấy `/particle_cloud` trong RViz | Cài `ros-humble-nav2-rviz-plugins`; display ParticleCloud để QoS **Best Effort** |
| Nav2 goal bị từ chối | Chờ Nav2 active hết; goal nằm trong vùng unknown hoặc sát tường quá |
| `Conflicting values set for option Signed-By` khi `apt update` | Trùng nguồn ROS: `sudo rm /etc/apt/sources.list.d/ros2.list` |

## 7. Kết quả

| Bản đồ sau SLAM | Localization | Navigation |
|---|---|---|
| ![map](docs/images/map.png) | ![amcl](docs/images/localization.png) | ![nav2](docs/images/nav2.png) |

Video demo: _[Google Drive link]_

## 8. Tài liệu tham khảo

- TurtleBot 4 Simulator: <https://github.com/turtlebot/turtlebot4_simulator>
- TurtleBot 4 User Manual: <https://turtlebot.github.io/turtlebot4-user-manual/>
- SLAM Toolbox: <https://github.com/SteveMacenski/slam_toolbox>
- Nav2 Documentation: <https://docs.nav2.org/>
