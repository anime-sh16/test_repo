# Perception Engineering — Weekly Study Plan (26 Weeks)

Each week: study the sources, build the verification project. The project is small — a notebook, a script, a visualization — not a production app. It exists to prove to yourself that you internalized the material, not to show anyone.

---

## Phase 1: Foundations (Weeks 1–9)

---

### Week 1 — Pinhole Camera Model & Intrinsics

**Study:**
- Hartley & Zisserman, "Multiple View Geometry," Chapter 6 (Sections 6.1–6.2). Free PDF floats around. This is the bible — read the derivation, not a blog summary
- OpenCV docs: Camera Calibration and 3D Reconstruction — https://docs.opencv.org/4.x/d9/d0c/group__calib3d.html
- First Principles of Computer Vision (YouTube, Shree Nayar, Columbia): "Camera Models" playlist — https://www.youtube.com/playlist?list=PL2zRqk16wsdoCCLpou-dGo7QQNks1Ppzo (watch the pinhole model and intrinsics videos)

**Verify:** Write a Python script that takes a 3D point in camera coordinates and projects it to a 2D pixel using a manually constructed K matrix. Then reverse it — take a pixel + depth and back-project to 3D. Print both results, verify the round-trip recovers the original point. No OpenCV projection functions allowed — raw matrix math only (numpy).

---

### Week 2 — Extrinsics, Coordinate Transforms & Calibration

**Study:**
- Hartley & Zisserman Chapter 6 (Sections 6.3–6.4) — extrinsic parameters, full projection equation p = K[R|t]P
- OpenCV tutorial: Camera Calibration with Checkerboard — https://docs.opencv.org/4.x/dc/dbb/tutorial_py_calibration.html
- 3Blue1Brown: "Essence of Linear Algebra" — watch the rotation/transformation videos if rotation matrices feel shaky

**Verify:** Print a checkerboard, calibrate your OnePlus camera using OpenCV's `calibrateCamera()`. Save the intrinsic matrix K, distortion coefficients, and reprojection error. Then take a photo, undistort it, and display before/after side by side. Save K somewhere — you'll use it for the rest of the project.

---

### Week 3 — Homogeneous Coordinates, Distortion & Epipolar Geometry

**Study:**
- Hartley & Zisserman Chapter 2 (projective geometry, homogeneous coords) and Chapter 9.1–9.2 (fundamental matrix, epipolar lines — conceptual only)
- First Principles of Computer Vision: "Stereo Vision" playlist (just the epipolar geometry videos, not the full stereo pipeline)
- Blog: Tomasi's "Geometric Camera Calibration" lecture notes (Duke CS) — good concise summary of distortion models

**Verify:** Take two photos of the same scene from slightly different positions with your phone. Use OpenCV to find feature matches (ORB or SIFT), compute the fundamental matrix, and draw epipolar lines on both images. You won't use stereo in your project — this is purely so you understand what these concepts mean when they come up in interviews.

---

### Week 4 — Monocular Depth Estimation

**Study:**
- Depth Anything V2 paper: https://arxiv.org/abs/2406.09414 — read Sections 1-3 (motivation, architecture, training)
- Original MiDaS paper: https://arxiv.org/abs/1907.01341 — for historical context on relative depth
- Metric3D paper: https://arxiv.org/abs/2307.10984 — skim for the metric depth approach
- Depth Anything V2 repo: https://github.com/DepthAnything/Depth-Anything-V2

**Verify:** Run Depth Anything V2 on 15 images you take around Tokyo (mix of indoor, outdoor, close objects, far objects, reflective surfaces). For 3 images, place an object at a known measured distance (e.g., 2m, 5m, 10m). Compare model output to ground truth. Compute a per-image scale factor. Write a notebook documenting failure cases with screenshots and your explanation of why each failed.

---

### Week 5 — Depth to 3D: Point Clouds from Monocular Depth

**Study:**
- Open3D documentation: Point Cloud basics — http://www.open3d.org/docs/release/tutorial/geometry/pointcloud.html
- Blog: "Monocular Depth Estimation to 3D Point Cloud" — there are several tutorials, pick one that uses numpy + camera intrinsics, not a magic library call
- Review your Week 1-2 back-projection math — this week is applying it at scale (every pixel, not one point)

**Verify:** Take 5 images with your phone. Run Depth Anything V2 on each. Using your calibrated K matrix from Week 2, back-project every pixel to 3D and produce a colored point cloud (use the RGB values from the image as point colors). Visualize in Open3D or matplotlib 3D scatter. Rotate the point cloud — does the 3D structure look correct? Document where it breaks (sky, reflections, thin objects).

---

### Week 6 — Object Detection: Architecture & Inference

**Study:**
- YOLOv8 paper/docs: https://docs.ultralytics.com/models/yolov8/ — focus on the anchor-free detection head
- RT-DETR paper: https://arxiv.org/abs/2304.08069 — Sections 1-3 (hybrid CNN-transformer, set prediction vs NMS)
- Ultralytics repo: https://github.com/ultralytics/ultralytics — run inference, read the prediction output format
- Blog: "Understanding mAP in Object Detection" by Jonathan Hui — clear explanation of precision/recall/IoU

**Verify:** Run YOLOv8-s and RT-DETR-l on the same 20 images (mix of street scenes, indoor, crowded, sparse). For each image, record: number of detections, inference time, any obvious misses or false positives. Run at 3 resolutions (320, 480, 640) and plot inference time vs resolution for both models. Write a one-paragraph conclusion: which model, at which resolution, would you choose for mobile deployment and why?

---

### Week 7 — Kalman Filter & Tracking Foundations

**Study:**
- ByteTrack paper: https://arxiv.org/abs/2110.06864 — full read, it's short and clear
- Roger Labbe's "Kalman and Bayesian Filters in Python" — free online book: https://github.com/rlabbe/Kalman-and-Bayesian-Filters-in-Python — Chapters 1-7 (skip the advanced stuff)
- scipy.optimize.linear_sum_assignment docs — understand the Hungarian algorithm as a black box

**Verify:** Implement a complete Kalman filter tracker from scratch (no filterpy, no library). State vector: [x, y, w, h, vx, vy, vw, vh]. Test on a video with 3+ moving objects (record one yourself or use a MOT benchmark clip). Produce a visualization: each tracked object gets a colored trail showing its trajectory over the last 30 frames. Count ID switches manually over 100 frames.

---

### Week 8 — ByteTrack & Advanced Association

**Study:**
- ByteTrack paper re-read: focus on the two-stage association (high-confidence first, then low-confidence)
- ByteTrack official repo: https://github.com/ifzhang/ByteTrack — read the core tracker code, it's surprisingly short
- BoT-SORT paper: https://arxiv.org/abs/2206.14651 — skim for the appearance-feature idea. You won't implement this, but know what it adds

**Verify:** Extend your Week 7 tracker to implement ByteTrack's two-pass association. Test on the same video. Compare ID switches: basic Kalman+Hungarian vs ByteTrack two-pass. The number should go down. If it doesn't, debug your second-pass matching. Produce a comparison GIF: left side = basic tracker with trails, right side = ByteTrack with trails.

---

### Week 9 — IMU, Rotations & LiDAR Point Cloud Basics

**Study:**
- IMU/Rotations:
  - 3Blue1Brown: "Quaternions and 3D rotation, explained interactively" — https://www.youtube.com/watch?v=d4EgbgTm0Bg (or the interactive page at eater.net/quaternions)
  - Android SensorManager docs: TYPE_ROTATION_VECTOR, TYPE_ACCELEROMETER, TYPE_GYROSCOPE
  - Blog: "Understanding Quaternions" by 3D Math Primer — any of the popular quaternion explainers work
- LiDAR/PointPillars:
  - KITTI dataset page: https://www.cvlibs.net/datasets/kitti/eval_object.php?obj_benchmark=3d — download the velodyne point clouds, labels, calibration
  - PointPillars paper: https://arxiv.org/abs/1812.05784 — full read, it's short and the architecture is simple
  - OpenPCDet repo: https://github.com/open-mmlab/OpenPCDet — clone it, read the PointPillars config

**Verify — two mini-projects this week:**

**IMU:** Write a Python script (or simple Android app) that reads IMU rotation data and visualizes a 3D coordinate frame rotating in real-time as you tilt/rotate your phone. If Python: log sensor data from Android app, replay in matplotlib animation.

**LiDAR:** Load one KITTI frame's point cloud (.bin file). Visualize it in 3D (matplotlib or Open3D). Then project the LiDAR points onto the corresponding camera image using KITTI's calibration matrices (P2, Tr_velo_to_cam, R0_rect). Overlay the projected points colored by depth onto the image. Verify the points land on the correct objects.

---

### ✅ Phase 1 Gate — Week 9 End

Before moving on, do all of the following without looking anything up:
- Back-project a pixel + depth to 3D on paper
- Explain monocular depth scale ambiguity in 3 sentences
- Draw the Kalman filter predict/update loop
- Explain what a "pillar" is in PointPillars
- Project a LiDAR point onto a camera image using calibration matrices

If any of these are shaky, spend Week 10 revisiting. Don't proceed with gaps.

---

## Phase 2: Integration, Training & Systems (Weeks 10–18)

Two tracks run in parallel. Odd weeks focus on your Python perception pipeline. Even weeks focus on model training. Adjust as needed — the alternation prevents burnout on either track.

---

### Week 10 — Python Pipeline: Detection + Depth Integration

**Study:**
- Review your Week 5 (depth → 3D) and Week 6 (detection) work
- Study how to crop depth maps to bounding box regions and compute robust statistics (median vs mean depth sampling)

**Verify:** Build the first two stages of your pipeline. Input: a recorded phone video (30 seconds, Tokyo street). Output: per-frame visualization showing bounding boxes with distance labels (in meters) computed from detection bbox + median depth + camera intrinsics back-projection. Save as an MP4. Does the distance look physically reasonable?

---

### Week 11 — Training Track A: PointPillars on KITTI (Setup & Baseline)

**Study:**
- OpenPCDet documentation: installation, data preparation, training — https://github.com/open-mmlab/OpenPCDet/blob/master/docs/GETTING_STARTED.md
- KITTI evaluation protocol: what easy/moderate/hard mean (occlusion level, truncation, bounding box height)

**Verify:** Get OpenPCDet running. Prepare KITTI data in their expected format. Run PointPillars training with the default config on your RTX 3060 (reduce batch size if needed). Train to completion. Run evaluation. Record baseline mAP for Car (easy/moderate/hard). Commit the config and results to your repo.

---

### Week 12 — Python Pipeline: Tracking Integration

**Study:**
- Review your Week 8 ByteTrack implementation
- Study how to wire frame-by-frame detections into a persistent tracker with track ID management

**Verify:** Add ByteTrack tracking to your pipeline. Input: same 30-second video. Output: bounding boxes now have persistent track IDs and velocity arrows. Objects that leave and re-enter the frame should ideally get new IDs (not magically recover old ones — that requires re-ID). Save as MP4. Count ID switches over the video manually.

---

### Week 13 — Training Track A: PointPillars Ablations

**Study:**
- OpenPCDet config system: how to modify pillar size, backbone, augmentation settings
- KITTI data augmentation: ground-truth database sampling (what it does and why it helps)

**Verify:** Run 3 ablation experiments:
1. Pillar grid resolution: 0.16m → 0.2m → 0.32m
2. Toggle GT database sampling on/off
3. One more of your choice (backbone swap, point cloud range, number of pillar features)

Record mAP for each in a markdown table. Write one sentence per row explaining the result. Commit to repo.

---

### Week 14 — Python Pipeline: 3D Back-Projection + BEV

**Study:**
- Review your Week 5 back-projection code
- Study BEV (bird's eye view) rendering: projecting 3D tracked positions onto a 2D ground plane
- Look at how AV companies visualize BEV: simple 2D plot, ego-vehicle at center, objects as colored rectangles with heading arrows

**Verify:** Add back-projection and BEV rendering to your pipeline. Each tracked detection gets back-projected to 3D using median depth + intrinsics. BEV rendered as a top-down matplotlib plot updating per frame, ego-position at bottom center. Save the BEV animation as MP4. Are cars positioned at reasonable distances? Are stationary objects staying still?

---

### Week 15 — Training Track B: FCOS3D on nuScenes Mini (Setup & Baseline)

**Study:**
- FCOS3D paper: https://arxiv.org/abs/2104.10956 — focus on how 2D centers are lifted to 3D via depth regression
- MMDetection3D documentation: installation, nuScenes data prep — https://github.com/open-mmlab/mmdetection3d
- nuScenes devkit: https://github.com/nutonomy/nuscenes-devkit — for data loading and evaluation

**Verify:** Get MMDetection3D running. Download nuScenes mini split. Run FCOS3D training with default config on Kaggle T4 or RTX 3060. Train to completion. Record NDS and mAP. Commit config and results.

---

### Week 16 — Python Pipeline: IMU Fusion + TTC

**Study:**
- Review your Week 9 IMU work
- Study ego-motion compensation: if you (the camera) are also moving, raw 3D positions include your own motion. IMU rotation data lets you factor that out
- Time-to-collision formula: TTC = distance / closing_velocity. Closing velocity = rate of change of distance over frames

**Verify:** Record a new video while walking, with IMU data logged synchronously. Add IMU rotation correction to your pipeline — 3D positions should now be in a gravity-aligned world frame, not the camera frame that rotates as you walk. Add TTC computation for each tracked object. Walk toward a parked car — TTC should decrease. Walk parallel to traffic — TTC for same-direction cars should be large/infinite. Save the full pipeline output (detection + tracking + BEV + TTC) as MP4.

---

### Week 17 — Training Track B: FCOS3D Ablations + Qualitative Analysis

**Study:**
- FCOS3D depth prediction: direct regression vs depth bins. How input resolution affects depth accuracy
- nuScenes evaluation: what NDS (nuScenes Detection Score) actually combines (mAP + TP metrics for translation, scale, orientation, velocity, attribute)

**Verify:** Run 2 ablations:
1. Input resolution comparison (e.g., 800×450 vs 1280×720 — or whatever your GPU allows)
2. One more of your choice (backbone, depth loss weighting, train schedule length)

Record results in markdown table. Then do qualitative analysis: pick 5 predictions, project the predicted 3D boxes back onto the camera image. Screenshot each one. Where does depth estimation fail? Where does orientation estimation fail? Write a paragraph explaining the patterns. Commit everything.

---

### Week 18 — Pipeline Polish + Profiling

**Study:**
- Python profiling: `cProfile`, `time.perf_counter()` per stage, or `line_profiler` for hot functions
- Review your full pipeline end-to-end — identify any hacks or shortcuts from earlier weeks that need cleanup

**Verify:** Run your full pipeline on 3 different test videos (different times of day, different scenes). Profile each stage: detection inference, depth inference, back-projection, tracking, BEV rendering. Produce a latency breakdown table. Identify the top 2 bottlenecks. Write a brief "optimization plan" for Phase 3 based on what you find. This document directly feeds into your deployment decisions.

---

### ✅ Phase 2 Gate — Week 18 End

- [ ] Full Python pipeline processes recorded video → BEV animation with tracked objects, distances, velocities, TTC
- [ ] PointPillars trained on KITTI: baseline + 3 ablations documented with analysis
- [ ] FCOS3D trained on nuScenes mini: baseline + 2 ablations + qualitative analysis documented
- [ ] Latency breakdown of Python pipeline with identified bottlenecks
- [ ] All training work committed with reproducible configs and READMEs

---

## Phase 3: Mobile Deployment & Optimization (Weeks 19–26)

---

### Week 19 — Model Export & On-Device Benchmarking

**Study:**
- Qualcomm AI Hub model zoo: check if YOLOv8/RT-DETR and Depth Anything V2 (small) have pre-compiled variants — https://aihub.qualcomm.com/
- TFLite export path: Ultralytics export docs for TFLite — https://docs.ultralytics.com/modes/export/
- ExecuTorch export if going that route (you know this from RTST)
- Review your RTST export pipeline and Android app scaffold

**Verify:** Export both models (detection + depth) to your chosen mobile runtime. Deploy each one individually on your OnePlus. Run a benchmark: 100 frames, record per-frame inference time. Run for 5 minutes continuously, plot inference time over time to see thermal throttling curve. Document: peak FPS, sustained FPS, memory usage per model.

---

### Week 20 — Android App: Detection on Device

**Study:**
- Review your RTST CameraX pipeline — reuse the camera scaffold, frame buffer strategy, STRATEGY_KEEP_ONLY_LATEST
- LiteRT / ExecuTorch Android inference API — whichever runtime you chose

**Verify:** Working Android app that runs detection model on live camera feed. Bounding boxes drawn as overlay. FPS counter visible on screen. Record a 15-second screen capture showing it working in a real scene.

---

### Week 21 — Android App: Add Depth + Distance Labels

**Study:**
- Multi-model inference on Android: sequential execution (run detection, then depth on same frame)
- Android CameraCharacteristics API: extract focal length, sensor size for runtime intrinsics

**Verify:** Add depth model running on each frame after detection. For each detected object, compute median depth within bbox, back-project to 3D distance using intrinsics from CameraCharacteristics. Display distance labels on bounding boxes. Record a 15-second demo showing distance labels updating as you walk toward/away from objects.

---

### Week 22 — Android App: Add Tracking

**Study:**
- JNI basics if implementing tracker in C++: https://developer.android.com/ndk/guides
- Or Kotlin implementation of ByteTrack — your Week 8 Python code as reference

**Verify:** Add ByteTrack tracking. Bounding boxes now have persistent IDs and colored trails. Walk past a row of parked cars — each car should maintain its ID. Walk toward a group of pedestrians — IDs should be stable as they move. Record a 30-second demo showing tracking working with ID labels visible. Document any ID switch cases.

---

### Week 23 — Android App: IMU Fusion + BEV Overlay

**Study:**
- Android SensorManager: registering for TYPE_ROTATION_VECTOR, sensor event timestamps, synchronization with camera frames
- Canvas or SurfaceView rendering for BEV overlay — a simple 2D top-down view rendered on a portion of the screen

**Verify:** Add IMU rotation reading and BEV rendering. The screen now shows the camera feed with detections AND a mini-map BEV overlay (corner of screen) showing tracked objects from above. Record a 30-second demo while walking down a street — the BEV should show objects at plausible positions relative to ego.

---

### Week 24 — TTC + Alerts + Pipeline Polish

**Study:**
- TTC computation from tracked distances over time
- Alert UX: audio cues, screen flash, or haptic feedback for low-TTC objects
- Edge cases: what happens when depth estimation is wrong? When tracking loses an object? Graceful degradation

**Verify:** Add TTC computation and a simple alert (screen border goes red when any object has TTC < 3 seconds). Test by walking toward a stationary object. Polish the UI: clean overlays, readable labels, stable rendering. The app should feel like a demo you'd show in an interview, not a debug tool.

---

### Week 25 — Optimization + Profiling

**Study:**
- Android Profiler: CPU, memory, GPU usage — https://developer.android.com/studio/profile
- Threading strategies: producer-consumer for camera→inference→rendering
- C++ JNI port for tracking/math if not already done

**Verify:** Profile the full on-device pipeline. Produce a latency breakdown: camera capture → preprocess → detection inference → depth inference → tracking → BEV rendering → display. Identify the bottleneck. A