# TurtleBot 4 – SLAM, Localization & Navigation (ROS 2 Humble + Gazebo)

Repo này dùng robot **TurtleBot 4** trong môi trường mô phỏng **Ignition Gazebo Fortress** để làm 4 việc:

- Xây dựng bản đồ bằng **SLAM Toolbox**.
- Lưu bản đồ.
- Định vị robot trên bản đồ đã lưu bằng **AMCL**.
- Điều hướng tự động bằng **Nav2**.


- Sinh viên: Trần Lưu Hưng

- MSSV: 23020747 
- Video demo: https://drive.google.com/drive/folders/1F0_m1A2Kt4Kn88vxIO4cJDL93JtvkOUY?usp=sharing


## 1. Môi trường

| Thành phần | Phiên bản |
|---|---|
| Hệ điều hành | Ubuntu 22.04 LTS |
| ROS 2 | Humble Hawksbill (Desktop) |
| Simulator | Ignition Gazebo Fortress |
| Robot | TurtleBot 4 (RPLIDAR A1, Create 3 base) |
| World | `warehouse`, có sẵn trong `turtlebot4_ignition_bringup` |

## 2. Cấu trúc repository

```
tb4-slam-nav/
├── maps/
│   ├── map.yaml                        # metadata bản đồ: resolution, origin, ngưỡng occupied/free
│   └── map.pgm                         # ảnh occupancy grid tạo bởi SLAM Toolbox
├── launch/
│   ├── turtlebot4_ignition.launch.py   # khởi chạy Gazebo + robot + ros_gz_bridge
│   ├── slam.launch.py                  # khởi chạy SLAM Toolbox
│   ├── localization.launch.py          # khởi chạy map_server + AMCL
│   └── nav2.launch.py                  # khởi chạy Nav2 (planner, controller, BT navigator, costmaps)
├── config/
│   ├── slam.yaml                       # tham số SLAM Toolbox
│   ├── localization.yaml               # tham số map_server + AMCL
│   └── nav2.yaml                       # tham số Nav2
├── rviz/
│   └── nav2_default_view.rviz          # cấu hình RViz: map, scan, particle cloud, costmap, path
├── scripts/
│   └── cleanup.sh                      # tắt sạch Gazebo/ROS chạy ngầm trước mỗi lần launch
├── docs/
│   ├── report.pdf
│   └── images/
└── README.md
```

Các file trong `launch/` và `config/` được lấy từ hai package `turtlebot4_ignition_bringup` và `turtlebot4_navigation` (cài qua `apt` tại `/opt/ros/humble/share/`). Đây đúng là các launch file và tham số đã dùng khi chạy bài này. Khi chạy, các lệnh `ros2 launch` gọi bản đã cài trong hệ thống.

## 3. Cài đặt

### 3.1. ROS 2 Humble

Cài theo hướng dẫn chính thức: <https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html>

```bash
echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

> Nếu `apt update` báo `Conflicting values set for option Signed-By ... packages.ros.org`, máy đang có hai file nguồn ROS cùng lúc. Xoá file cũ đi:
> ```bash
> sudo rm /etc/apt/sources.list.d/ros2.list && sudo apt update
> ```

### 3.2. TurtleBot 4, SLAM Toolbox, Nav2

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

### 3.3. Gazebo Fortress

```bash
sudo curl https://packages.osrfoundation.org/gazebo.gpg -o /usr/share/keyrings/pkgs-osrf-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/pkgs-osrf-archive-keyring.gpg] http://packages.osrfoundation.org/gazebo/ubuntu-stable $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/gazebo-stable.list
sudo apt update && sudo apt install -y ignition-fortress
```

### 3.4. Driver card đồ hoạ (quan trọng)

Nếu Gazebo render bằng CPU (`llvmpipe`), mô phỏng chạy rất chậm:

- Nút ▶ nhấp nháy liên tục.
- Robot gần như không di chuyển khi teleop.
- Controller dễ bị lỗi timeout.

Kiểm tra card đang dùng:

```bash
glxinfo -B | grep "OpenGL renderer"   # cần ra NVIDIA ..., không phải llvmpipe
```

Nếu kết quả là `llvmpipe` thì cài driver NVIDIA:

```bash
sudo ubuntu-drivers autoinstall
sudo reboot
nvidia-smi                 # kiểm tra driver đã chạy
sudo prime-select nvidia   # dùng card NVIDIA cho toàn hệ thống
sudo reboot
```

> Nếu Secure Boot đang bật: khi cài driver, đặt mật khẩu MOK. Lúc reboot sẽ hiện màn hình xanh, chọn **Enroll MOK → Continue → Yes** rồi nhập mật khẩu đó.

### 3.5. Clone repository

```bash
git clone https://github.com/<username>/tb4-slam-nav.git ~/tb4-slam-nav
cd ~/tb4-slam-nav
chmod +x scripts/cleanup.sh
mkdir -p ~/maps && cp maps/map.yaml maps/map.pgm ~/maps/
```

## 4. Chạy chương trình

Ba quy ước dùng cho mọi phần:

- `T1`, `T2`, ... nghĩa là **mỗi lệnh chạy trong một terminal riêng**.
- **Trước mỗi phần**, chạy `~/tb4-slam-nav/scripts/cleanup.sh` để tắt hết các tiến trình cũ. Bấm `Ctrl+C` thường không tắt hẳn được Gazebo.
- Nếu máy yếu, thêm `model:=lite` vào lệnh Gazebo, và phải thêm ở **tất cả** các phần.

### Phần 1 – TurtleBot 4 Simulation

```bash
# T1 – Gazebo + robot
ros2 launch turtlebot4_ignition_bringup turtlebot4_ignition.launch.py world:=warehouse
```

Chờ log hiện `Configured and activated diffdrive_controller` và kiểm tra đồng hồ *Sim time* trong Gazebo có đang chạy. Nếu đang pause thì bấm ▶.

```bash
# T2 – robot spawn trên dock nên phải undock trước
ros2 action send_goal /undock irobot_create_msgs/action/Undock "{}"

# T2 – kiểm tra các topic cơ bản
ros2 topic hz /scan                 # LiDAR
ros2 topic echo /odom --once        # odometry
ros2 topic info /cmd_vel -v         # publisher / subscriber của lệnh vận tốc
ros2 topic pub -r 10 /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}}"   # Ctrl+C để dừng
ros2 run rqt_graph rqt_graph

# T3 – RViz (Fixed Frame = odom)
ros2 launch turtlebot4_viz view_robot.launch.py

# T4 – teleop
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Cách dùng teleop: click vào cửa sổ terminal teleop rồi **giữ** phím.

| Phím | Tác dụng |
|---|---|
| `i` | tiến |
| `,` | lùi |
| `j` / `l` | quay |
| `k` | dừng |
| `z` | giảm tốc |

### Phần 2 – Xây dựng bản đồ bằng SLAM Toolbox

```bash
~/tb4-slam-nav/scripts/cleanup.sh

# T1 – Gazebo
ros2 launch turtlebot4_ignition_bringup turtlebot4_ignition.launch.py world:=warehouse

# T2 – undock
ros2 action send_goal /undock irobot_create_msgs/action/Undock "{}"

# T3 – SLAM Toolbox
ros2 launch turtlebot4_navigation slam.launch.py use_sim_time:=true

# T4 – RViz (Fixed Frame = map)
ros2 launch turtlebot4_viz view_robot.launch.py

# T5 – teleop để khảo sát môi trường
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Để bản đồ không bị tường nhân đôi hay lệch:

- Đi chậm, khoảng 0.2 m/s.
- Quay chậm.
- Đi hết các dãy kệ.
- **Quay về điểm xuất phát** để SLAM Toolbox thực hiện loop closure.

**Lưu bản đồ** khi SLAM vẫn đang chạy. Phải `cd` vào `~/maps` trước, nếu không file sẽ được lưu vào thư mục hiện tại:

```bash
mkdir -p ~/maps && cd ~/maps
ros2 run nav2_map_server map_saver_cli -f map
ls ~/maps    # map.yaml  map.pgm
```

### Phần 3 – Localization (AMCL)

```bash
~/tb4-slam-nav/scripts/cleanup.sh

# T1 – Gazebo (robot phải nằm trên dock)
ros2 launch turtlebot4_ignition_bringup turtlebot4_ignition.launch.py world:=warehouse

# T2 – map_server + AMCL, load bản đồ đã lưu
ros2 launch turtlebot4_navigation localization.launch.py map:=$HOME/maps/map.yaml use_sim_time:=true

# T3 – RViz của Nav2 (có sẵn map, particle cloud, costmap, nút Nav2 Goal)
ros2 launch nav2_bringup rviz_launch.py
# hoặc dùng file cấu hình trong repo:
# rviz2 -d ~/tb4-slam-nav/rviz/nav2_default_view.rviz --ros-args -p use_sim_time:=true
```

Sau khi T2 chạy, log sẽ lặp lại dòng `AMCL cannot publish a pose ... Please set the initial pose`. Đây là bình thường, vì AMCL đang chờ vị trí ban đầu. Các bước tiếp theo:

1. **Đặt vị trí ban đầu.** Trên thanh công cụ RViz, bấm **2D Pose Estimate** (mũi tên xanh lá). Click vào vị trí dock trên map rồi kéo theo hướng mũi robot. Dock nằm tại gốc `(0, 0)` của bản đồ vì SLAM bắt đầu từ dock. Có thể đặt bằng lệnh thay cho thao tác trên RViz:
   ```bash
   ros2 topic pub --once /initialpose geometry_msgs/msg/PoseWithCovarianceStamped \
   "{header: {frame_id: map}, pose: {pose: {position: {x: 0.0, y: 0.0}, orientation: {w: 1.0}}}}"
   ```
2. **Cho AMCL hội tụ.** Ở T4, undock rồi chạy teleop để lái một đoạn ngắn. Particle cloud sẽ co lại và các điểm LiDAR trùng với tường trên map.
3. **Kiểm tra.** Đặt Gazebo và RViz cạnh nhau, vị trí robot ở hai bên phải khớp nhau. Kiểm tra xong thì tắt teleop.

### Phần 4 – Autonomous Navigation (Nav2)

Giữ nguyên các terminal của Phần 3:

```bash
# T4 – Nav2
ros2 launch turtlebot4_navigation nav2.launch.py use_sim_time:=true
```

Đợi log báo `Managed nodes are active`. Sau đó:

- Bấm **Nav2 Goal** (hoặc **2D Goal Pose**) trên RViz, click vào điểm đích rồi kéo để chọn hướng.
- Thực hiện **3 goal ở 3 khu vực khác nhau** của warehouse.
- Không điều khiển robot thủ công trong khi Nav2 đang chạy.

## 5. Node, topic và TF chính

| Node | Package | Vai trò |
|---|---|---|
| `ign gazebo`, `ros_gz_bridge` | `turtlebot4_ignition_bringup` | Mô phỏng robot và cảm biến; bridge Gazebo ⇄ ROS 2 |
| `motion_control`, `diffdrive_controller` | `irobot_create_*`, `ros2_control` | Nhận `/cmd_vel`, điều khiển bánh, publish `/odom` |
| `robot_state_publisher` | `robot_state_publisher` | Publish TF giữa các link robot từ URDF |
| `slam_toolbox` | `slam_toolbox` | Scan matching + pose graph → `/map`, TF `map → odom` |
| `map_server` | `nav2_map_server` | Load `map.yaml` → `/map` |
| `amcl` | `nav2_amcl` | Particle filter định vị robot trên map → TF `map → odom` |
| `bt_navigator` | `nav2_bt_navigator` | Behavior tree điều phối plan → follow → recovery |
| `planner_server` | `nav2_planner` | Lập đường đi toàn cục trên global costmap → `/plan` |
| `controller_server` | `nav2_controller` | Bám đường đi, tránh vật cản cục bộ → `/cmd_vel` |
| `teleop_twist_keyboard` | `teleop_twist_keyboard` | Điều khiển bằng bàn phím → `/cmd_vel` |

| Topic | Kiểu | Mô tả |
|---|---|---|
| `/scan` | `sensor_msgs/LaserScan` | Dữ liệu LiDAR |
| `/odom` | `nav_msgs/Odometry` | Odometry từ bánh xe |
| `/cmd_vel` | `geometry_msgs/Twist` | Lệnh vận tốc |
| `/map` | `nav_msgs/OccupancyGrid` | Bản đồ (từ SLAM hoặc map_server) |
| `/initialpose` | `geometry_msgs/PoseWithCovarianceStamped` | Vị trí ban đầu cho AMCL |
| `/particle_cloud` | `nav2_msgs/ParticleCloud` | Tập particle của AMCL |
| `/goal_pose` | `geometry_msgs/PoseStamped` | Goal đặt từ RViz |
| `/plan` | `nav_msgs/Path` | Đường đi toàn cục |

Cây TF:

```
map ──(SLAM / AMCL)──▶ odom ──(odometry)──▶ base_link ──(URDF)──▶ rplidar_link
```

- `odom → base_link`: tính từ bánh xe. Mượt nhưng bị trôi dần theo thời gian.
- `map → odom`: do SLAM Toolbox (Phần 2) hoặc AMCL (Phần 3, 4) tính bằng cách so khớp `/scan` với bản đồ, dùng để bù phần trôi đó.

## 6. Xử lý lỗi thường gặp

| Hiện tượng | Nguyên nhân / cách xử lý |
|---|---|
| Teleop không điều khiển được robot | Kiểm tra `ros2 topic info /cmd_vel`: teleop phải là một trong các publisher. Ngoài ra cần undock, click vào cửa sổ teleop và giữ phím |
| Nút ▶ của Gazebo nhấp nháy, robot gần như đứng yên | Gazebo đang render bằng `llvmpipe`. Cài driver NVIDIA (mục 3.4) |
| Log treo ở `waiting for service /controller_manager/list_controllers` | Các tiến trình khởi động lệch nhau. Chạy `scripts/cleanup.sh` rồi launch lại |
| Mở lại Gazebo thấy robot không nằm trên dock | Gazebo cũ chưa tắt hẳn. Chạy `scripts/cleanup.sh` |
| `Failed to load map yaml file ... bad file` | Sai đường dẫn map. Kiểm tra `ls ~/maps` |
| `AMCL cannot publish a pose ... set the initial pose` | Bình thường. Đặt **2D Pose Estimate** |
| Không thấy `/particle_cloud` trong RViz | Cài `ros-humble-nav2-rviz-plugins`, thêm display ParticleCloud và đặt QoS **Best Effort** |
| Không thấy nút 2D Pose Estimate | Bấm **[+]** trên thanh công cụ RViz → `SetInitialPose` |
| Nav2 goal bị từ chối | Chờ Nav2 active hết. Không đặt goal trong vùng unknown hoặc sát tường |

## 7. Kết quả

| Bản đồ sau SLAM | Localization | Navigation |
|---|---|---|
| ![map](docs/images/map.png) | ![amcl](docs/images/localization.png) | ![nav2](docs/images/nav2.png) |

Video demo gồm: quá trình xây dựng bản đồ, bản đồ hoàn chỉnh, load lại bản đồ, localization và 3 navigation goal.
Link: _[Google Drive link]_

## 8. Tài liệu tham khảo

- TurtleBot 4 Simulator: <https://github.com/turtlebot/turtlebot4_simulator>
- TurtleBot 4 User Manual: <https://turtlebot.github.io/turtlebot4-user-manual/>
- SLAM Toolbox: <https://github.com/SteveMacenski/slam_toolbox>
- Nav2 Documentation: <https://docs.nav2.org/>
