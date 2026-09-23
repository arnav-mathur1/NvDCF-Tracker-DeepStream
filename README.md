# UAV Multi-Object Tracking with NVIDIA NvDCF

This was a central part of my work and research in Summer 2026. 

It is an exploration and tuning of NVIDIA's NvDCF multi-object tracker for tracking UAVs in a real-time DeepStream perception pipeline.

The project focused on understanding how detection, visual tracking, motion estimation, data association, and track management work together to maintain consistent object IDs across video frames.

I made a [presentation](https://drive.google.com/file/d/11qaaTEZbEJRMN1Cu_hqha5cqz3EFdIC9/view?usp=sharing) documenting my findings and I hope to share an overview here.

## Overview

NvDCF combines detector outputs with visual and motion-based tracking.

At a high level, the pipeline is:
1. Receive object detections from the detector
2. Predict where existing tracks should move
3. Visually localize tracked objects between detections
4. Associate new detections with existing tracks
5. Update, preserve, or terminate tracks
6. Output bounding boxes with persistent track IDs

The goal was to tune this process for small UAV targets, where detections can flicker and targets may move quickly or become temporarily occluded.

## What I Learned

### Discriminative Correlation Filter Tracking

NvDCF uses a Discriminative Correlation Filter (DCF) to learn the visual appearance of each tracked target.

For every frame, the tracker searches around the predicted target location, extracts visual features, and generates a correlation response map. The highest-response location becomes the estimated target position.

This allows the tracker to continue localizing an object even when the detector does not produce a bounding box on every frame.

### Motion Estimation

NvDCF also uses a Kalman filter to estimate target motion.

The filter predicts the next bounding box based on the previous state and velocity, then updates that prediction using measurements from:
- detector bounding boxes
- DCF visual localization

This helped me understand how state estimation can bridge noisy or intermittent perception measurements.

### Data Association

A major part of multi-object tracking is deciding whether a new detection belongs to an existing track.

NvDCF scores possible track-detection pairs using measurements such as:
- Intersection over Union (IoU)
- bounding-box size similarity
- visual similarity from the DCF response

The resulting scores are filtered using association thresholds before detections are assigned to tracks. Tuning these thresholds directly affects whether a UAV keeps its existing ID or incorrectly starts a new track.

### Track Lifecycle Management

I also explored how trackers manage objects that temporarily disappear.

Tracks move through states such as:
- tentative
- active
- shadow/inactive
- terminated

Parameters such as `probationAge`, `earlyTerminationAge`,
`minTrackerConfidence`, and `maxShadowTrackingAge` determine how quickly
tracks are created and how long they survive detector misses or occlusion.

This was particularly important for UAV tracking because small targets often produce intermittent detections.

## Tracker Tuning

I experimented with parameters controlling several parts of the tracker:

### Detection and Track Management
- minimum detector confidence
- minimum tracker confidence
- probation period
- shadow tracking duration
- early termination

### Data Association
- IoU threshold
- size-similarity threshold
- visual-similarity threshold
- relative weights of each association metric

### State Estimation
- position process noise
- size process noise
- velocity process noise
- detector measurement noise
- tracker measurement noise

### Visual Tracking
- ColorNames features
- HOG features
- feature-image resolution
- DCF filter learning rate
- channel-weight learning rate
- Gaussian response width

These experiments showed that tracking performance is not controlled by a
single parameter. Stable IDs depend on the interaction between detector
quality, motion prediction, visual localization, association thresholds,
and track lifetime.

## Files

### `config_tracker_NvDCF_perf.yml`

Performance-oriented NvDCF configuration used as the main baseline for the DeepStream UAV tracking pipeline. I tuned parameters related to track creation/termination, data association, Kalman-filter state estimation, and DCF visual tracking to improve ID consistency while maintaining real-time performance.

### `config_tracker_NvDCF_uav_reassoc.yml`

Experimental UAV-specific NvDCF configuration focused on recovering the same track ID after missed detections or short occlusions. This version adds motion-based track re-association, loosens association gates for small fast-moving targets, and adjusts Kalman-filter and visual-tracker parameters for low-texture UAVs. Appearance-based ReID is intentionally disabled because the targets are often too small for reliable appearance matching.
