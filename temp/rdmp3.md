# Perception Engineering Roadmap (v3)

**What changed in v3**

Phase 1 is rebuilt around visible demos. Every theoretical concept now has a small Python (occasionally C++) exercise you finish in 1–3 hours that puts something on the screen. The math is still there — you earn it by needing it to make a demo work, the same way you learned Vulkan FP16 quirks during RTST. C++ shows up earlier than v2 so the eventual ByteTrack port isn't a cold plunge. Timeline is honestly extended.

**Two deliverables (unchanged)**

1. Benchmark training repo (~25–30% of effort) — PointPillars on KITTI, FCOS3D on nuScenes mini, with ablations
2. Android perception app (~70–75% of effort) — real-time pipeline on your phone

**Honest timeline:** 8–10 months on your current ~4.5h/day schedule. Don't pretend it's 6.

**Two rules for the whole roadmap:**

1. **Demo before derivation.** Get something on the screen, then go back and understand why it works.
2. **Python first, C++ second.** Once a Python demo works, port the core math to C++ with Eigen as a separate file in the same repo. Don't defer the C++ debt; it compounds.

---

## Phase 1: Foundations (Months 1–3)

Extended from 2 to 3 months. The 2-month target in v2 was tight even before adding demos. You'll move fast through some parts and slow through others — let that happen.

### 1.1 Camera Geometry — Month 1, Weeks 1–2

**Concepts:** pinhole model, intrinsic matrix K, focal length, principal point, projection equation, radial/tangential distortion, extrinsics [R|t].

**Demo 1 — Wireframe Cube Projector (Python, 2–3 hours)**
Open webcam with OpenCV. Define a 3D cube in world coordinates (8 vertices). Write `project(point_3d, K, R, t) -> point_2d` from scratch — no `cv2.projectPoints`, do the matrix multiplication yourself. Draw projected cube edges over the live frame. Keyboard controls to translate/rotate the cube in 3D.
*You should be able to delete every external function and rewrite project() from memory at the end.*

**Demo 2 — Phone Calibration (Python, 1 evening)**
Print an 8×6 checkerboard. Take 15–20 photos at different angles. Use `cv2.calibrateCamera` to extract K and distortion coefficients. Display raw vs undistorted side-by-side.
*Save your phone's K. You'll use it constantly later.*

**Demo 3 — C++ Port of Projection (4–6 hours)**
Rewrite Demo 1's `project()` in C++ with Eigen. Read one image from disk, project a cube, save output as PNG. No OpenGL, no real-time. Just prove you can do the matrix math in C++ with a build system that works.
*This is your first real C++ exercise. It will hurt. That's the point.*

**Checkpoint question:** Why does a wider focal length compress apparent depth in the image? Sketch the answer.

### 1.2 3D Transforms — Month 1, Weeks 3–4

**Concepts:** rotation matrices, Euler angles, axis-angle, quaternions, SE(3), frame composition, frame conventions.

**Demo 4 — Gimbal Lock Visualizer (Python, 2 hours)**
Implement rotation by Euler angles (XYZ order) and by quaternions. Animate both rotating a cube through a sequence that hits gimbal lock. Watch the Euler version flip; quaternion version stays smooth.
*Now you know why every robotics codebase uses quaternions.*

**Demo 5 — Three-frame Visualizer (Python + matplotlib 3D, 2 hours)**
Draw world, camera, and body coordinate frames. Define a transform chain `T_world_camera = T_world_body @ T_body_camera`. Move the body frame interactively; the camera frame follows.
*Frame conventions are the #1 silent bug source in perception code. Build the muscle now, not after a 4-hour debugging session.*

### 1.3 Two-view Geometry — Month 1 Week 5 → Month 2 Week 1

**Concepts:** homography, fundamental/essential matrices, epipolar geometry, triangulation, PnP.

**Demo 6 — Panorama Stitcher (Python, 3–4 hours)**
Two overlapping phone photos of a wall. SIFT/ORB features, RANSAC homography, warp and blend. Pretty output, and you also feel what happens with insufficient parallax.

**Demo 7 — Sparse 3D from Two Photos (Python, 1 weekend)**
Two photos of your room from different positions. Feature matching → essential matrix from K + matches → decompose to R, t → triangulate matches into 3D. Render the point cloud in Open3D and orbit it.
*First time you'll see "3D from images" actually work. Save the video — confidence anchor for the slumps.*

### 1.4 IMU Fundamentals — Month 2, Weeks 2–3

**Concepts:** accelerometer/gyro models, bias, noise, integration drift, complementary filter, sensor fusion intuition.

**Demo 8 — Phone Orientation Tracker (1 evening)**
Use "Sensor Logger" app to record accel + gyro to CSV. Plot raw integrated gyro orientation — watch it drift. Implement a complementary filter (accel low-pass + gyro high-pass). Plot fused orientation — watch drift disappear.
*You won't forget what sensor fusion buys you after watching that plot.*

**Demo 9 — IMU Dead Reckoning (failure case, 1 hour)**
Take Demo 8's data, double-integrate acceleration to estimate position. Watch the estimate fly off into space within seconds.
*This is why visual-inertial fusion exists. You needed to feel this fail.*

### 1.5 Kalman Filter — Month 2, Week 4 → Month 3, Week 1

**Concepts:** state-space models, prediction, update, EKF, observation models.

**Demo 10 — Ball Tracker in Noisy Video (Python, 1 day)**
Synthetic video of a ball with Gaussian noise on detections. First baseline: use raw detections directly — jitters. Then 4D constant-velocity KF (px, py, vx, vy) — implement F, H, Q, R from scratch, no FilterPy. Overlay raw detections (red), KF prediction (green), KF estimate (blue).
*When the blue line glides through the red jitter, that's when KF clicks.*

**Demo 11 — KF in C++ (3–4 hours)**
Port Demo 10's KF to C++ with Eigen. Read detections from CSV; write filtered estimates to a new CSV; plot in Python for verification.
*Second C++ exercise. Less painful than the first.*

**Demo 12 — EKF for Non-linear Motion (1 day)**
Switch dynamics to constant turn rate (non-linear). Linearize via Jacobian, implement EKF update. Track a target moving in a circle. Compare KF vs EKF errors.
*You now understand why production trackers are EKF/UKF, not vanilla KF.*

### 1.6 Phase 1 Capstone — Month 3, Weeks 2–3

**AR Ruler App (1 week of evenings, Python desktop or simple Android)**

- Phone calibrated (Demo 2)
- ArUco marker on the floor, detected each frame
- Solve PnP for camera pose w.r.t. marker
- Draw a 3D coordinate axis sticking up from the marker — stays locked to the floor as the phone moves
- Tap two points on screen → app reports real-world distance between them (assuming floor plane)

This exercises calibration, projection, pose estimation, and homography in one demo. **This is Phase 1 graduation.** If it works without AI writing your projection math, you're done. If it doesn't, you have a concrete debugging target — much better than re-reading a textbook.

### 1.7 Phase 1 Buffer — Month 3, Week 4

Catch-up week. Use it. Nobody finishes on schedule.

---

## Phase 2: Pipeline & Training (Months 4–6)

This is where perception models join the geometric scaffolding, and where the benchmark training repo gets built.

### 2.1 2D Detection — Month 4

- Train YOLOv8n or similar on a small subset (BDD100K or COCO subset) on Kaggle/Colab
- Don't aim for SOTA. Aim for a clean training script you understand line by line.
- Deliverable: a notebook with training curves, mAP numbers, qualitative predictions on held-out frames

**C++ touchpoint:** Implement NMS in C++. Compare against torchvision's NMS on a few thousand boxes; numbers should match.

### 2.2 Monocular Depth — Month 4, Week 4 → Month 5, Week 2

- Use a pretrained MiDaS or Depth Anything model — don't train from scratch
- Run on KITTI and on your own phone footage
- Side-by-side visualization: RGB frame | colorized depth map
- Study how depth degrades for transparent objects, sky, distant objects. Document the failure modes — this is interview gold.

### 2.3 PointPillars on KITTI (Benchmark Repo, Part 1) — Month 5

- Use OpenPCDet or mmdetection3d as starting point — don't write from scratch
- Train PointPillars on KITTI cars
- Reproduce a number close to published (within a few mAP is fine)
- Ablation 1: voxel size sweep
- Ablation 2: anchor sizing
- Clean README with method, numbers, qualitative results

This is the part of the portfolio that proves you can train perception models, not just use them.

### 2.4 Tracking — Month 6

- Implement ByteTrack in Python on top of your YOLO detector
- Validate on a MOT17 sequence
- Plug your Phase 1 KF code in as the motion model
- Side-by-side video: detection-only (boxes flickering) vs tracking (stable IDs)

**C++ touchpoint:** Start porting the ByteTrack matching + KF logic to C++. Aim for feature parity with the Python version on the same MOT17 sequence. This is the bigger C++ project — give it 2–3 weeks of evenings.

### 2.5 3D Lifting & BEV — Month 6, Week 4

- Use your calibrated phone K, ground plane assumption, and 2D box bottom-midpoints to lift detections to 3D
- Plot detected objects in a top-down BEV view
- Compute crude distance + velocity from tracked IDs

This is the moment all of Phase 1 pays off. The geometry, the KF, the calibration, the frame conventions — they all show up in one visualization.

---

## Phase 3: Deployment & Polish (Months 7–10)

### 3.1 Android Skeleton — Month 7

- CameraX pipeline, frame capture, preview overlay
- Port the YOLO detector to ExecuTorch (you've done this for RTST — much faster the second time)
- Get real-time detection on phone with bounding box overlay
- Measure latency, profile, identify bottlenecks

### 3.2 Multi-model Scheduling — Month 8

- Add monocular depth running in parallel (different backend if needed — XNNPACK for detector, Vulkan for depth, or vice versa)
- Frame skipping, async execution, model pipelining
- Document the scheduling decisions and tradeoffs — this is the systems engineering story that distinguishes you

### 3.3 Tracking + BEV on Device — Month 9

- Port your C++ ByteTrack to the Android NDK
- IMU integration via Android sensors API
- Live BEV overlay on the phone

### 3.4 Fine-tuning + FCOS3D + Polish — Month 10

- Fine-tune the detector on ~500–1000 labeled frames of Tokyo street footage you record yourself. This is the ML depth proof that the app isn't just an inference wrapper.
- Train FCOS3D on nuScenes mini as second benchmark entry
- Final README, demo video, blog post

---

## Working principles

**Boredom is a signal, not a failure mode.** If a sub-section starts feeling like college, you've drifted from demo-first. Stop, build something visual that *needs* whatever math you're stuck on, then come back with a reason to care.

**The C++ track is non-negotiable but small.** Skipping C++ now means you walk into a perception interview without an answer to "implement a Kalman filter in C++ on the whiteboard." Don't do that.

**Re-deriving > re-reading.** When concepts re-appear later (and they will — KFs and projections show up everywhere), re-derive from scratch on paper. Faster than re-reading and the retention is qualitatively different.

**AI usage policy:** Phase 1, AI for setup/syntax only — write the math yourself. Phase 2, AI for boilerplate (data loaders, training loops) is fine — keep the core algorithm code yours. Phase 3, AI freely — by then you'll know what's correct.

**Honesty check every month:** Did I actually build the demos, or did I skim and convince myself I'd do them later? If the latter, redo the month. No portfolio survives self-deception.
