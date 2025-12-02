---
title: Deepstream Main Application SRS
description: All system requirements of DS apps (which used to get the rtsp stream and processing frame,....)
published: true
date: 2025-12-01T10:37:14.942Z
tags: dsapp, security, v4.3, srs
editor: markdown
dateCreated: 2025-12-01T10:37:14.942Z
---

# Software Requirements Specification (SRS)

# Video Analytics System (VAS)

# DeepStream Main Application (DSAPP)

---

## 1. Overview

### SEC-VAS-DSAPP-001: Multi-Stream Processing Support
The main application shall process up to 80 concurrent video streams simultaneously.

### SEC-VAS-DSAPP-002: Input Source Type Support
The main application shall support the following input source types:
- RTSP streams
- USB camera devices
- Video file inputs

### SEC-VAS-DSAPP-003: GPU Acceleration Support
The main application shall:
- Operate using GPU-accelerated processing with NVIDIA CUDA support
- Support multi-GPU configurations
- Allow configurable GPU ID assignment per source

### SEC-VAS-DSAPP-004: Dynamic Source Management
The main application shall:
- Allow dynamic addition of video sources during runtime
- Allow dynamic removal of video sources during runtime
- Maintain separate processing pipelines for each video source

### SEC-VAS-DSAPP-005: Performance Monitoring
The main application shall provide the following capabilities:

**Performance Metrics:**
- Measure and report FPS (frames per second) for each video stream
- Measure and report average FPS across all streams
- Measure frame latency from input to output for each stream
- Export performance metrics in Prometheus-compatible format
- Update performance metrics at configurable intervals (default: 10 seconds)
- Report performance metrics through callback functions

---

## 2. Stream Decoder

### SEC-VAS-DSAPP-006: Video Format Decoding Support
The stream decoder shall decode video streams from the following sources:
- H.264 encoded video streams from RTSP sources
- H.265 encoded video streams from RTSP sources
- Video streams from USB cameras using V4L2 protocol
- Video files in MP4, AVI, and other supported container formats

### SEC-VAS-DSAPP-007: Decoder Selection and Fallback
The stream decoder shall:
- Automatically select the appropriate decoder based on stream format and GPU availability
- Use NVIDIA hardware decoders (nvh264dec, nvh265dec) when available
- Fall back to software decoders (avdec_h264, avdec_h265) when hardware decoders are unavailable

### SEC-VAS-DSAPP-008: Frame Rate Control
The stream decoder shall:
- Skip frames based on configurable skip interval to control processing frame rate
- Calculate frame skip interval automatically based on input FPS and target output FPS

### SEC-VAS-DSAPP-009: Stream Validation and Compatibility
The stream decoder shall:
- Validate stream compatibility before adding it to the pipeline
- Detect and report stream resolution (width and height)
- Report decoder errors and stream incompatibility issues through logging

### SEC-VAS-DSAPP-010: RTSP Connection Management
The stream decoder shall automatically reconnect to RTSP streams when connection is lost.

### SEC-VAS-DSAPP-011: Decoder Configuration Options
The stream decoder shall support:
- Configurable decoder type selection (NVGST, SWGST, NVV4L2)
- Frame rate adjustment based on source capabilities

### SEC-VAS-DSAPP-012: Frame Format Conversion
The stream decoder shall:
- Convert decoded frames to RGBA format with NVMM memory type
- Handle stream format conversion between different color spaces

### SEC-VAS-DSAPP-013: Stream Event Handling
The stream decoder shall:
- Detect and handle end-of-stream (EOS) events for file sources
- Support non-stop playback mode for video file sources when configured
- Maintain frame timing information for each decoded frame

---

## 3. JPEG Encoder

### SEC-VAS-DSAPP-014: JPEG Encoding with GPU Acceleration
The JPEG encoder shall:
- Encode video frames to JPEG format using GPU acceleration
- Process encoding asynchronously using CUDA streams
- Apply configurable JPEG quality settings (default: 50) during encoding

### SEC-VAS-DSAPP-015: Frame Resizing and Size Preservation
The JPEG encoder shall:
- Resize frames to configurable output dimensions (default: 640x360) before encoding
- Preserve original frame size when keep_original_size configuration is enabled
- Support configurable output dimensions per camera source

### SEC-VAS-DSAPP-016: Redis Cache Storage
The JPEG encoder shall store encoded JPEG images in Redis cache with the following specifications:
- **Storage Format:** Binary data
- **Frame-specific Keys:** Format `{namespace}/{camid}/frame_{frame_num}`
- **Camera-specific Keys:** Format `{namespace}/{camid}.jpeg`
- **Expiration:** Configurable expiration time with default TTL of 120 seconds for frame-specific entries
- **Updates:** Update camera-specific cache entries with the latest frame image

### SEC-VAS-DSAPP-017: Frame Tracking and Error Handling
The JPEG encoder shall:
- Track frame numbers for each encoded JPEG image
- Update camera-specific cache entries with the latest frame image
- Handle encoding errors and report them through logging

---

## 4. Pose Inference

### SEC-VAS-DSAPP-018: Human Pose Estimation
The pose inference shall:
- Perform human pose estimation on video frames using deep learning models
- Use NVIDIA TensorRT inference engine for pose estimation
- Output pose keypoint coordinates and confidence scores from inference

### SEC-VAS-DSAPP-019: Batch Processing and Synchronization
The pose inference shall:
- Process multiple video streams in batch mode for efficient inference
- Synchronize frames from multiple sources before batch inference
- Configure batch size dynamically based on number of active sources

### SEC-VAS-DSAPP-020: Model Engine Management
The pose inference shall:
- Load model engine files from configurable directory paths
- Select model engine files based on camera mode configuration

### SEC-VAS-DSAPP-021: Preprocessing Support
The pose inference shall support optional preprocessing with the following capabilities:
- ROI-based preprocessing before inference
- Tensor metadata generation when preprocessing is enabled
- 4K resolution preprocessing when configured

### SEC-VAS-DSAPP-022: Inference Configuration and Performance
The pose inference shall:
- Support configurable GPU ID for inference operations
- Maintain inference performance metrics including processing time
- Support disabling inference when not required
- Pass inference results to downstream components in the pipeline
- Handle inference errors and report them through logging

---

## 5. Pose Decoder

### SEC-VAS-DSAPP-023: Pose Data Extraction
The pose decoder shall:
- Decode TensorRT inference output into structured pose keypoint data
- Extract pose coordinates (x, y) for each detected keypoint
- Extract confidence scores for each detected keypoint
- Support 17-18 keypoints per detected person

### SEC-VAS-DSAPP-024: Person Identification and Metadata
The pose decoder shall:
- Assign person identification (PID) to each detected pose
- Associate frame metadata (frame ID, timestamp) with pose data

### SEC-VAS-DSAPP-025: Pose Format Support and Export
The pose decoder shall:
- Support multiple pose format types (bottom-up, one-stage)
- Export pose data to JSON format when evaluation mode is enabled

### SEC-VAS-DSAPP-026: Pose Decoder Configuration and Filtering
The pose decoder shall:
- Track processing FPS for pose decoding operations
- Pass decoded pose data to the tracker component
- Maintain camera-specific configuration for pose decoding
- Support configurable processing frame rate per camera
- Filter invalid or low-confidence pose detections
- Handle decoding errors and report them through logging

---

## 6. Tracker

### SEC-VAS-DSAPP-027: Pose-Based Tracking Algorithm
The tracker shall:
- Track individuals across video frames using pose-based tracking
- Use OC-SORT (Observation-Centric Single Object Real-time Tracking) algorithm
- Predict person positions using Kalman Filter algorithms
- Associate detected poses with existing tracks using multi-stage matching (IoU, distance)

### SEC-VAS-DSAPP-028: Person ID Management
The tracker shall manage person IDs (PID) with the following capabilities:
- Assign unique person IDs (PID) to each tracked individual
- Rotate person IDs when maximum PID limit is reached
- Limit maximum person IDs when max_pid configuration is set
- Support reloading person IDs from persistent storage when configured

### SEC-VAS-DSAPP-029: Pose Sequence Management
The tracker shall:
- Maintain pose sequences of configurable length (default: 20 frames) for each tracked person
- Generate force completion flags for pose sequences when required

### SEC-VAS-DSAPP-030: Track Association and Threshold Management
The tracker shall adjust association thresholds dynamically based on track history.

### SEC-VAS-DSAPP-031: Missing Frame and Border Handling
The tracker shall:
- Handle missing frames by updating tracks with predicted positions
- Detect when tracked persons are at image borders

### SEC-VAS-DSAPP-032: Track History and Advanced Features
The tracker shall support the following advanced features:
- Maintain track history with configurable short age and long age thresholds
- Support pose tracking hybrid mode when configured
- Support pose control versions (v1, v2) with configurable options
- Detect and track "keeping pose" states for individuals

### SEC-VAS-DSAPP-033: Tracker Output and Error Handling
The tracker shall:
- Output tracked pose sequences with associated person IDs
- Provide Kalman filter predictions for each tracked person
- Handle tracking errors and report them through logging

---

## 7. Stream Muxer/Demuxer

### SEC-VAS-DSAPP-034: Stream Batching and Synchronization
The stream muxer shall:
- Batch multiple video streams into single processing batches
- Synchronize frames from multiple sources before batching
- Maintain aspect ratio of input streams when padding is enabled
- Configure batch size dynamically based on number of active sources
- Handle batch timeout when frames are not available from all sources

### SEC-VAS-DSAPP-035: Stream Demultiplexing
The stream demuxer shall split batched results back to individual streams after processing.

---

## 8. Stream Management

### SEC-VAS-DSAPP-036: Runtime Source Operations
The stream management shall:
- Add new video sources to the pipeline during runtime
- Remove video sources from the pipeline during runtime

### SEC-VAS-DSAPP-037: Stream Compatibility and Validation
The stream management shall check stream compatibility before adding new sources.

### SEC-VAS-DSAPP-038: RTSP Reconnection and Status Reporting
The stream management shall:
- Handle stream reconnection automatically for RTSP sources
- Report stream connection status (running, reconnecting, disconnected)

### SEC-VAS-DSAPP-039: Stream Error Handling and External Integration
The stream management shall:
- Handle stream errors without affecting other active streams
- Update camera status in external systems when stream state changes
- Support fake/test streams for pipeline testing and validation

---






