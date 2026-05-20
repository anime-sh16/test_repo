# 6-Month Perception Engineering Roadmap (v2)

**Goal:** By month 6, be able to design, train, implement, debug, and explain a real-time multi-model perception pipeline from first principles — without AI assistance on the core logic. Demonstrate both the ability to train perception models on standard benchmarks AND build the full system around them.

**Project vehicle:** Monocular perception system on Android (detection + depth + tracking + 3D lifting + IMU fusion + geometric BEV).

**Two deliverables:**
1. **Benchmark training repo** (~25-30% of effort) — PointPillars on KITTI, FCOS3D on nuScenes mini, with ablations and clean documentation. Exists to prove you can train perception models and reason about why things work.
2. **Android perception app** (~70-75% of effort) — The full real-time pipeline on your phone. This is the main portfolio piece.

**Hardware reality:** RTX 3060 laptop, MacBook Pro, Kaggle/Colab free tier, small cloud GPU budget ($30-50 total). Everything in this roadmap is scoped to work within these constraints.

---

## Phase 1: Foundations (Month 1–2)

No code that ships. No Android. This is paper, whiteboard, Python notebooks, and building the mathematical intuition that everything else depends on. If you rush this, every phase after it will be painful.

### 1.1 Camera Geometry & Projective Math

**What to study:**
- Pinhole camera model — derive it from similar triangles, don't memorize the matrix
- Intrinsic matrix K: focal length (fx, fy), principal point (cx, cy). Know what happens to the image when you change each parameter
- Extrinsic matrix [R|t]: rotation and translation between coordinate frames. Camera-to-world and world-to-camera — know which direction you need and why
- Homogeneous coordinates: why 3D→2D projection can't be a matrix multiply without them
- The full projection equation: p = K[R|t]P (2D pixel ← 3D world point)
- The inverse problem: given a pixel + depth, back-project to 3D. This is what your pipeline actually does
- Distortion models: radial (k1, k2, k3) and tangential (p1, p2)
- Camera calibration: what checkerboard calibration computes, Zhang's method conceptually
- Epipolar geometry: fundamental matrix, essential matrix — conceptual only, you won't implement stereo but you must be able to discuss it in an interview

**Tips:**
- Draw everything. Literally draw the camera, the world point, the image plane, the rays. If you can't draw it, you don't understand it
- Every equation should map to a physical picture in your head. If K[R|t]P is just a formula to you, you've memorized, not learned
- The single most important skill from this section: given a 2D pixel and a depth, produce a 3D world coordinate. Practice this until it's automatic
- Don't skip distortion. Real phone cameras have significant lens distortion. If you skip this, your BEV output will be geometrically wrong and you won't know why

**Exercises:**
1. Given K and a 3D point (5, 2, 10) in camera frame, compute the pixel coordinate by hand
2. Given K, pixel (320, 240), and depth 8m, back-project to 3D by hand
3. Calibrate your OnePlus camera using OpenCV + a printed checkerboard pattern. Save the intrinsic matrix — you'll use it for the rest of the project
4. Take a photo with your phone, undistort it using your calibration. Compare before/after
5. Write the full round-trip: pick a 3D point → project to pixel → back-project with known depth → verify you recover the original point. If you don't, your math is wrong somewhere

### 1.2 Monocular Depth Estimation

**What to study:**
- Why monocular depth is fundamentally ill-posed (infinite 3D scenes produce the same 2D image)
- How modern depth models work: encoder (pretrained ViT) extracts features, decoder produces per-pixel depth
- Relative vs metric depth — Depth Anything V2 outputs relative depth. Metric3D and ZoeDepth output metric. Know the difference and when each matters
- How to convert relative depth to metric using known camera intrinsics + a scale factor
- Scale ambiguity: the core problem with monocular depth and how people work around it

**Tips:**
- Don't get lost in the model internals. You're not going to re-architect Depth Anything V2. You need to understand what goes in (RGB image), what comes out (dense depth map), and what the failure modes are (reflective surfaces, sky, thin structures)
- Spend time understanding the scale problem deeply. This is what makes monocular 3D perception hard, and interviewers will probe here
- Run Depth Anything V2 on some images from your phone early. Look at where it fails. That intuition matters more than reading the paper three times

**Exercises:**
1. Run Depth Anything V2 on 10 images you take around Tokyo. Identify 3 failure cases and explain why they fail
2. Take a photo of an object at a known distance (measure with a tape). Compare the model's relative depth to your known ground truth. Compute the scale factor
3. Write a script that takes a depth map + camera intrinsics and produces a colored 3D point cloud (just scatter plot in matplotlib). This is the bridge between 2D depth and 3D understanding

### 1.3 Object Detection

**What to study:**
- Anchor-based (YOLOv5-era) vs anchor-free (YOLOv8+, RT-DETR) — understand why the field moved to anchor-free
- RT-DETR: transformer encoder-decoder for detection. Why the hybrid CNN+transformer design. What "set prediction" means vs NMS-based approaches
- Detection outputs: bounding box (x, y, w, h), class probability, confidence score. How NMS filters overlapping predictions
- mAP metric: what precision and recall mean per-class, how IoU threshold affects the number, why mAP@0.5 vs mAP@0.5:0.95 tell different stories
- Data augmentation for detection: mosaic, mixup, random crop, color jitter. Know what each does and why

**Tips:**
- You will use a pretrained detection model in the app. The studying here is so you understand what's happening inside and can make informed choices about which model, what resolution, what confidence threshold
- The most common interview question in this area is "walk me through the detection pipeline from image to final bounding boxes." Practice saying it out loud
- Don't study YOLO v1-v7 history in detail. Know v1 conceptually (single-shot, grid-based) and then jump to the current anchor-free paradigm. The evolution is interesting but not worth weeks

**Exercises:**
1. Run YOLOv8 and RT-DETR on the same 20 images. Compare speed and accuracy qualitatively. Which model misses more? Which has more false positives?
2. Take the same image, run detection at 320×320, 480×480, and 640×640. Document how resolution affects mAP and latency. This is directly relevant to your mobile deployment tradeoff
3. Explain NMS to an imaginary interviewer in under 2 minutes. Time yourself

### 1.4 Tracking Foundations

**What to study:**
- Kalman filter: predict/update cycle, state vector [x, y, a, h, vx, vy, va, vh], process noise vs measurement noise. Derive the intuition: it's just "predict where the box will be next frame, then correct with what you actually see"
- Hungarian algorithm: optimal bipartite matching between predicted positions and new detections. Know that scipy.optimize.linear_sum_assignment does this — you don't need to implement Hungarian from scratch
- ByteTrack paper (Zhang et al., 2022): the key insight is using low-confidence detections in a second association round to recover objects that were partially occluded
- Track lifecycle: how a new detection becomes a track (birth), how matched detections maintain it, how missed frames eventually kill it (death). The thresholds matter
- IoU-based vs appearance-based association: IoU is simpler and sufficient for many cases. Re-ID features (BoT-SORT) add robustness but add compute

**Tips:**
- Implement the Kalman filter from scratch in a Python notebook. Not because you'll use your implementation in production, but because the predict/update loop must be in your bones
- The most common failure mode in tracking is ID switches — where two crossing objects swap identities. Understand why this happens (IoU overlap during crossing) and how ByteTrack's two-stage association mitigates it
- Don't over-invest in appearance-based tracking (DeepSORT, BoT-SORT) right now. ByteTrack with IoU-only association is what you'll deploy on-device. Appearance features are expensive

**Exercises:**
1. Implement a 1D Kalman filter that tracks a ball moving in a straight line with noisy measurements. Plot predicted vs actual vs measured positions
2. Extend to 2D: track a bounding box across 30 frames of a video using your Kalman filter + Hungarian matching. No ByteTrack yet — just raw Kalman + assignment
3. Now implement ByteTrack's two-pass association on the same video. Compare ID switches before and after

### 1.5 IMU & Sensor Fusion Basics

**What to study:**
- What an IMU measures: accelerometer (linear acceleration in m/s²), gyroscope (angular velocity in rad/s)
- Why raw accelerometer data is noisy and drifts, why gyroscope data drifts over time
- Rotation representations: rotation matrices (3×3), Euler angles (roll/pitch/yaw), quaternions. Why quaternions avoid gimbal lock. How to convert between them
- Complementary filter: the simplest way to fuse accelerometer + gyroscope for orientation
- Android SensorManager: TYPE_ROTATION_VECTOR gives you a fused quaternion directly. TYPE_ACCELEROMETER and TYPE_GYROSCOPE give raw data
- Ego-motion compensation: when your phone moves, the world frame changes. You need IMU data to undo your own movement before estimating where objects are in world coordinates

**Tips:**
- You will probably use Android's fused TYPE_ROTATION_VECTOR in the actual app rather than fusing raw sensors yourself. But understanding why the fusion exists and what it's doing is non-negotiable
- Don't go deep into extended Kalman filters or UKF for sensor fusion right now. That's a rabbit hole. Know what they are, know they exist for non-linear systems, move on
- The practical skill here is: given a rotation quaternion from the IMU at time t, rotate your 3D detection coordinates from camera frame to a gravity-aligned world frame. Practice this transform

**Exercises:**
1. Write an Android app (or Python script reading logged sensor data) that reads IMU data and visualizes your phone's orientation in real-time as a rotating 3D coordinate frame
2. Record a 10-second walk with your phone logging IMU + camera frames. Plot the orientation over time. Identify where you turned, stopped, and started
3. Take two camera frames from different orientations, back-project a point from each using depth + intrinsics, and use IMU rotation to verify both map to the same world coordinate

### 1.6 LiDAR Point Cloud Basics (Conceptual + KITTI Data)

**What to study:**
- What a LiDAR sensor produces: a set of (x, y, z, intensity) points in 3D space. No color, no texture — just geometry
- Point cloud data formats: .bin files in KITTI (flat arrays of float32), .pcd files (more structured)
- How point clouds relate to camera images: LiDAR and camera have different coordinate frames, connected by an extrinsic calibration matrix. KITTI provides these
- Voxelization: dividing 3D space into a grid and aggregating points per voxel. This is how PointPillars and VoxelNet convert unstructured points into structured tensors
- PointPillars specifically: vertical columns ("pillars") instead of full 3D voxels. Why this is clever — it reduces 3D to pseudo-2D, making it compatible with fast 2D CNN backbones
- Bird's eye view encoding: projecting the point cloud onto the ground plane. What information is preserved (x, y position, density) and what's lost (fine vertical structure)

**Tips:**
- You are NOT building a LiDAR-based system. The goal is to understand the data format and the core architectural ideas well enough to discuss them in interviews and to train a model in Phase 2
- Spend one afternoon just visualizing KITTI point clouds in 3D (matplotlib, open3d, or even just pandas scatter). Rotate them. Understand the sensor's field of view, density patterns (dense nearby, sparse far away), and what different object classes look like as point blobs
- Know the difference between LiDAR-only detection (PointPillars), camera-only detection (FCOS3D), and fusion approaches (BEVFusion). You won't implement fusion, but know where each sits

**Exercises:**
1. Download KITTI 3D detection dataset. Load one frame's point cloud and camera image. Project the LiDAR points onto the image using the provided calibration matrices. Verify your projection by checking that points land on the correct objects
2. Visualize a point cloud from above (bird's eye view) using matplotlib. Color by height. You should see the road, cars, pedestrians as distinct clusters
3. Implement simple pillar encoding: divide the ground plane into a 0.2m × 0.2m grid, count points per cell, compute mean height per cell. Visualize this as a 2D heatmap. Congratulations — you just did the first step of PointPillars by hand

---

### Phase 1 Completion Checkpoint

You're done with Phase 1 when you can do ALL of the following without looking anything up:

- [ ] Given camera intrinsics and a pixel + depth, write the back-projection to 3D on a whiteboard
- [ ] Explain why monocular depth is ill-posed and what "scale ambiguity" means, in plain language
- [ ] Run detection + depth inference on a video and produce a visualization with distance labels per object
- [ ] Implement a Kalman filter tracker from scratch that maintains IDs across 30+ frames
- [ ] Explain what a pillar is in PointPillars and why it's computationally efficient vs full 3D voxelization
- [ ] Project KITTI LiDAR points onto a camera image using calibration matrices
- [ ] Given a rotation quaternion from an IMU, apply it to transform a 3D point from camera frame to world frame

If any of these feel shaky, stay in Phase 1. Seriously. Phase 2 will punish you for gaps here.

---

## Phase 2: Integration, Training & Systems (Month 3–4)

Two parallel tracks: building the perception pipeline in Python AND training perception models on benchmarks. These can interleave week by week.

### 2.1 The Core Perception Pipeline (Python, not Android yet)

Build the full pipeline on your laptop processing recorded video from your phone:

1. **Frame input** → run YOLO/RT-DETR detection → 2D bounding boxes
2. **Same frame** → run Depth Anything V2 → dense depth map
3. **Per detection** → sample depth within bounding box (median, not mean — outlier robust) → estimated distance
4. **Back-project** each detection center to 3D using depth + camera intrinsics
5. **Track** across frames using your ByteTrack implementation (Kalman predict → Hungarian match → update)
6. **IMU correction** → apply rotation from logged sensor data to transform 3D points into gravity-aligned world frame
7. **BEV projection** → render top-down view of all tracked objects with positions, velocities, track IDs

**Tips:**
- Steps 1 and 2 are API calls. Steps 4-7 are where your engineering lives. Spend your time accordingly
- Record test videos deliberately: walk down a busy Tokyo street, a quiet residential area, a parking lot with cars at known distances. You need diverse test data
- Log IMU data synchronously with camera frames from the start. Retrofitting synchronization later is miserable
- Build visualization early. A matplotlib animation showing BEV tracks updating frame-by-frame is both a debugging tool and a demo
- The median depth within a bounding box matters. Mean gets corrupted by background pixels. Median is robust. Even better: use the depth values in the center 50% of the box

**Exercises:**
1. Process a 30-second video end-to-end through your pipeline. Produce a BEV animation showing tracked objects moving. Does it look physically plausible? Are cars moving at reasonable speeds? Are stationary objects staying still?
2. Compute TTC (time-to-collision) for an approaching object. Walk toward a parked car while recording. Does TTC decrease as expected?
3. Deliberately break each component and observe how it cascades: feed the tracker wrong detections, corrupt the depth map, remove IMU correction. Understanding failure modes is as important as understanding the happy path

### 2.2 Training Track A: PointPillars on KITTI

**Goal:** Train a 3D object detection model on a standard benchmark. Report numbers. Run ablations. This is NOT about beating state of the art. It's about demonstrating you can do it and reason about the results.

**Setup:**
- Framework: OpenPCDet (from OpenMMLab). It has PointPillars configs ready to go
- Dataset: KITTI 3D object detection (~15GB). Download the velodyne point clouds, labels, and calibration files
- Hardware: RTX 3060 is sufficient. Batch size 2-4, train for ~80 epochs. Takes several hours, not days

**What to do:**
1. Get the baseline PointPillars config running on KITTI. Don't change anything yet. Just reproduce their numbers (or get close)
2. Record your baseline mAP for Car, Pedestrian, Cyclist at easy/moderate/hard difficulty
3. Run 3 ablations — change ONE thing per run:
   - Pillar grid resolution: 0.16m vs 0.2m vs 0.32m
   - Backbone: swap from the default to a lighter/heavier alternative
   - Augmentation: toggle ground-truth database sampling on/off
4. Report everything in a clean markdown table. For each ablation, write one sentence explaining what you expected and what actually happened

**Tips:**
- OpenPCDet's documentation is decent but not perfect. Budget a full day for environment setup and data preparation. This is normal, not a sign you're doing it wrong
- Your numbers will be slightly worse than published. That's expected — they trained on better hardware with tuned hyperparameters. What matters is that your numbers are in the right ballpark and your ablations show sensible trends
- If pillar resolution goes from 0.16m → 0.32m and mAP goes UP, something is wrong with your setup. Think about why before moving on
- Save your training configs and the exact commands you ran. Reproducibility is the point

**Deliverable:** A section in your GitHub repo with training configs, evaluation results table, and a paragraph per ablation explaining the result. Clean README with "how to reproduce."

### 2.3 Training Track B: FCOS3D on nuScenes Mini

**Goal:** Train a monocular 3D detection model — one that predicts 3D bounding boxes from a single camera image, no LiDAR. This is directly relevant to your final app (which is monocular) and to companies that do camera-only perception.

**Setup:**
- Framework: MMDetection3D (has FCOS3D configs)
- Dataset: nuScenes mini split (~4GB). The full nuScenes is 300GB+ — don't download that. Mini has 10 scenes, enough to train a model and evaluate
- Hardware: Kaggle T4 or your RTX 3060. FCOS3D is a 2D-backbone model — memory footprint is manageable

**What to do:**
1. Get the FCOS3D config running on nuScenes mini. This will take fighting with mmdet3d's config system — budget time for it
2. Report NDS (nuScenes Detection Score) and mAP. Your numbers on mini will be much lower than published full-dataset numbers. That's fine — document that you trained on mini and note the expected gap
3. Run 2 ablations:
   - Input resolution: does higher resolution help monocular 3D? (It should — more pixels = better depth cues)
   - Depth prediction head: compare direct regression vs discretized classification (if config allows)
4. Qualitative analysis: pick 5 predictions, visualize the predicted 3D boxes projected back onto the image. Where does the model get depth wrong? Why?

**Tips:**
- FCOS3D's key insight is predicting a 3D offset from the 2D center — it converts 2D detection into 3D by learning to regress depth, 3D dimensions, and orientation per object. Understand this mechanism, it comes up in interviews
- The nuScenes mini split is small enough that your model will overfit. That's expected. You're learning the training pipeline, not producing a production model
- mmdet3d's config inheritance is confusing at first. Trace through the config chain once manually — base config → model config → dataset config → schedule. Understanding this system transfers to any OpenMMLab project

**Deliverable:** Same as PointPillars — configs, results table, ablation analysis, clean README.

### 2.4 Systems Thinking

**What to study:**
- Pipeline latency breakdown: how to profile where time is spent (model inference vs preprocessing vs postprocessing vs tracking vs rendering)
- Parallel execution: can detection and depth run concurrently? What's the bottleneck — GPU compute or memory bandwidth?
- Memory budgeting: how much VRAM/RAM do N models consume simultaneously?
- Thermal throttling: why your phone slows down after 2 minutes of sustained inference. How to design for thermal limits (adaptive frame rate, model switching)
- Frame dropping: process every frame vs skip frames vs adaptive rate based on scene complexity

**Tips:**
- Build a simple profiling wrapper around your Python pipeline early. Even just `time.time()` around each stage. You need to know where the time goes before you can optimize
- The most common mistake: optimizing the model when the bottleneck is actually preprocessing or data copying. Profile first, optimize second. Always

---

### Phase 2 Completion Checkpoint

- [ ] Your Python pipeline processes a recorded video end-to-end and produces a BEV animation with tracked objects, distances, and velocities
- [ ] TTC computation works: approaching objects show decreasing TTC
- [ ] PointPillars trained on KITTI with baseline mAP reported and 3 ablations documented
- [ ] FCOS3D trained on nuScenes mini with NDS/mAP reported and 2 ablations documented
- [ ] Both training repos have clean READMEs with reproduction instructions
- [ ] You can explain the PointPillars pillar encoding AND the FCOS3D 3D regression head from memory
- [ ] You have a latency breakdown of your Python pipeline and know which stage is slowest

---

## Phase 3: Mobile Deployment & Optimization (Month 5–6)

Port the Python pipeline to Android. Your RTST experience is directly relevant here — CameraX, ExecuTorch/LiteRT, buffer management, you've done this before. The new challenge is multi-model orchestration and real-time geometric computation.

### 3.1 Model Export

- Export detection model (YOLOv8 or RT-DETR) to TFLite or compile via Qualcomm AI Hub
- Export depth model (Depth Anything V2) — check operator coverage. Depth models with ViT backbones can have unsupported ops. Have a fallback plan (smaller variant, MiDaS)
- Test both models on-device individually before integrating. Measure latency and memory per model in isolation

**Tips:**
- Check Qualcomm AI Hub first. If pre-compiled, quantized versions of your models exist, use them. Don't export from scratch unless you have to — you already proved you can do that with RTST
- If depth model export fails due to unsupported ops, try the small variant of Depth Anything V2 first, then MiDaS as a fallback. Don't spend weeks fighting export issues
- Measure thermal behavior: run each model in a loop for 5 minutes. Plot inference time over time. The curve will show when thermal throttling kicks in. Design your frame rate targets around the throttled speed, not the peak speed

### 3.2 Android Pipeline

- CameraX frame capture → preprocess → detection inference → depth inference → parse outputs
- Implement ByteTrack in C++ (preferred for interview signal and performance) or Kotlin (faster to iterate)
- Wire tracking: detections per frame → Kalman predict → Hungarian match → update → track IDs with velocity
- Distance estimation: median depth within bounding box, back-project using camera intrinsics from CameraCharacteristics API
- TTC computation: `TTC = distance / closing_velocity` per tracked object
- IMU integration: read TYPE_ROTATION_VECTOR from SensorManager, apply rotation to 3D coordinates
- BEV rendering: Canvas or OpenGL overlay showing top-down view. Pure geometry — no model

**Tips:**
- Build incrementally. Week 1: detection only running on device with bounding box overlay. Week 2: add depth and distance labels. Week 3: add tracking with IDs. Week 4: add IMU + BEV + TTC. Don't try to wire everything at once
- The multi-model scheduling problem is real. Two options: run sequentially (simpler, ~2× latency) or run in parallel on different hardware units (detection on NPU, depth on GPU). Start sequential, optimize later
- Pre-allocate all buffers. You know the input/output shapes. No dynamic allocation in the inference loop. You learned this from RTST — apply it
- Your RTST CameraX scaffold with STRATEGY_KEEP_ONLY_LATEST is directly reusable here

### 3.3 Optimization

- Profile the full pipeline on-device. Where is time spent?
- C++ critical path: if Kalman filter + Hungarian matching + back-projection + BEV math is in Kotlin, consider porting to C++ via JNI. These are pure math operations that benefit from native execution
- Threading: camera frame producer → inference consumer → tracking/rendering consumer. Don't do everything on the main thread
- Adaptive frame rate: if thermal throttling drops inference below your target, reduce frame rate gracefully rather than dropping frames randomly
- Battery profiling: how long can the app run continuously? This matters for a real-world demo

**Tips:**
- Don't optimize prematurely. Get it working end-to-end first, even if it's slow. Then profile. Then optimize the actual bottleneck, not what you assume is slow
- The C++ port of tracking is a strong interview talking point, but only if you can explain WHY C++ was necessary (measured latency improvement). "I used C++ because it's faster" is weak. "Kotlin tracking took 12ms per frame, C++ brought it to 3ms, which gave me enough headroom to run depth at full resolution" is strong

### 3.4 Stretch Features (if time permits)

- Tap-to-segment: SAM2 or EfficientSAM on device — this is genuinely hard and would be impressive
- Trajectory prediction: linear extrapolation of tracked velocity vectors, visualized as future position dots
- Style transfer on selected objects: you literally already have the model from RTST

**Tips:**
- Trajectory prediction via linear extrapolation is trivial to implement (30 minutes of code) and looks impressive in demos. Do this even if you skip the other stretch features
- SAM2 on-device is a real engineering challenge. Only attempt if the core pipeline is solid and you have 2+ weeks left. Don't let a stretch goal compromise your main deliverable

---

### Phase 3 Completion Checkpoint

- [ ] App runs on your OnePlus with real-time detection + depth + tracking + BEV overlay
- [ ] Demo video: 60-90 seconds of you walking down a Tokyo street with the full pipeline running. Real conditions — daylight, some pedestrians, some vehicles
- [ ] Latency breakdown documented: per-model inference time, tracking time, total frame time, sustained FPS after thermal throttling
- [ ] At least one optimization documented with before/after measurements
- [ ] README includes architecture diagram, component descriptions, performance table, known limitations
- [ ] Bonus: a second demo video in a different condition (evening, rain, crowded intersection) showing where the pipeline degrades and why

---

## Final Deliverable Checklist

### GitHub Repo Structure (suggested)
```
perception-pipeline/
├── training/
│   ├── pointpillars-kitti/
│   │   ├── configs/
│   │   ├── results/
│   │   └── README.md          ← baseline + ablation table + analysis
│   └── fcos3d-nuscenes/
│       ├── configs/
│       ├── results/
│       └── README.md          ← baseline + ablation table + analysis
├── pipeline/
│   ├── python/                ← offline Python pipeline (Phase 2)
│   └── android/               ← the app (Phase 3)
├── docs/
│   ├── architecture.png       ← system diagram
│   ├── performance.md         ← latency breakdown, FPS, thermal behavior
│   └── limitations.md         ← honest failure mode documentation
├── demos/
│   ├── bev-animation.mp4      ← Python pipeline output
│   ├── app-demo-daylight.mp4  ← Android app, good conditions
│   └── app-demo-evening.mp4   ← Android app, challenging conditions (stretch)
└── README.md                  ← project overview, motivation, results summary
```

### What Hiring Managers Will Actually Look At
1. The README — 90% of people stop here. Make it count. Architecture diagram, one-paragraph motivation, key results, demo video embedded
2. The demo video — they'll watch 30 seconds. Make the first 10 seconds show the BEV overlay working. That's the hook
3. The training results table — if they're technical, they'll check whether your numbers make sense and whether your ablations show you understand what you're doing
4. The limitations doc — this is where senior engineers separate "knows what they built" from "followed a tutorial." Honest documentation of failure modes is more impressive than polished demos

### What You Should Be Able To Do Without AI

- Camera geometry: projection, back-projection, calibration, coordinate transforms
- Kalman filter: implement from scratch, tune process/measurement noise
- Hungarian matching: set up the cost matrix, interpret the assignment
- BEV construction from monocular depth using pure geometry
- IMU data interpretation, rotation application, ego-motion compensation
- Latency profiling and identifying bottlenecks
- Explaining PointPillars architecture, FCOS3D regression heads, ByteTrack association logic
- Reasoning about training results — why an ablation moved numbers in a particular direction

### What AI Assistance Is Acceptable For
- Android boilerplate, UI layouts, XML configs
- Model export debugging (TFLite/ExecuTorch operator coverage issues)
- OpenPCDet/mmdet3d config syntax and data preparation scripts
- C++ JNI bridge boilerplate
- Visualization code (matplotlib animations, OpenGL overlays)

---

## Interview Talking Points This Roadmap Prepares You For

**"Walk me through your perception pipeline."**
→ Detection → depth → back-projection → tracking → IMU fusion → BEV. You built every component.

**"How did you handle the scale ambiguity in monocular depth?"**
→ Calibration with known distances, median depth sampling within bounding boxes, and you know the fundamental limitation.

**"Have you trained any 3D detection models?"**
→ PointPillars on KITTI, FCOS3D on nuScenes. Here are my numbers, here's what I learned from ablations.

**"What's the hardest engineering problem you solved?"**
→ Multi-model orchestration on a phone under thermal constraints, or tracking through occlusion, or IMU-camera synchronization. Pick whichever was actually hardest for you.

**"What would you do differently?"**
→ Your limitations doc. Companies that do perception at scale care deeply about whether you can identify your own system's weaknesses.

**Concepts you should know but won't implement (for interview discussion):**
- BEVFormer, PETR, StreamPETR — transformer-based multi-camera 3D detection. Know the idea (project image features into BEV space using attention), know why they need multi-GPU, know they're the current frontier
- LiDAR-camera fusion (BEVFusion) — know the concept of projecting both modalities into a shared BEV space
- Occupancy networks — know this is the latest trend beyond bounding boxes (predicting which 3D voxels are occupied)
- HD maps and map-conditioned planning — know that perception feeds into downstream planning modules
