# Multimodal Sensors Solution
## Camera
- Passive sensors
- 2D images

### Limitations
- Sensitive to illumination and weather conditions

## LiDAR
LiDARs are largely insensitive to illumination conditions because they are active sensors: they emit near-infrared laser pulses and measure the reflected returns. However, their performance still degrades in adverse weather (fog, rain, snow) due to scattering and absorption.

> *Infrared (IR) is electromagnetic radiation with wavelengths just longer than visible red light. Humans can't see it. "Infrared" means "below red". Infrared light has a lower frequency, and therefore lower energy, than red light.*

### How LiDAR works, step by step

#### 1. Core principle: time of flight (ToF)

```
   LiDAR                                       Object
  ┌──────┐   ① laser pulse (905/1550 nm)       ┌───┐
  │ TX ──┼───────────────────────────────────▶ │   │
  │      │                                     │   │ ② reflects
  │ RX ◀─┼───────────────────────────────────  │   │
  └──────┘   ③ return pulse (weaker)           └───┘
      t₀ ─────────────── Δt ──────────────── t₁

          distance  d = c · Δt / 2
```

The light travels to the object and back, so the formula divides by 2. A 1 ns timing error equals about 15 cm of range error, which is why LiDAR needs very fast detectors and timing electronics.

#### 2. One point from each pulse

Each return gives:
- **Range** *d*, from time of flight
- **Azimuth** θ and **elevation** φ, from the beam direction at that moment
- **Intensity**, which depends on how strongly the surface reflects

Azimuth (θ) tells you which way the beam is pointing horizontally, as the sensor sweeps around:
```
        Top view (looking down on the car)

                 0°  (forward, x-axis)
                  ↑
                  │   ╱ beam
                  │ θ╱
                  │ ╱
   90° (left) ────◉──── 270° (right)
                 car
                  │
                 180° (behind)
```
Elevation (φ) tells you how steeply a beam points up or down. Each laser channel is mounted at a fixed elevation angle:

```
        Side view

                  ╱  ch 0   φ = +15°  (up)
                ╱
   sensor  ◉ ───────────── φ = 0°   (horizontal)
                ╲
                  ╲  ch 63  φ = −25°  (down)
   ─────────────────────── ground
```

Converting from spherical to Cartesian coordinates gives one 3D point:

```
x = d·cosφ·cosθ    y = d·cosφ·sinθ    z = d·sinφ
```

#### 3. Scanning builds the point cloud

The sensor fires hundreds of thousands to millions of pulses per second in different directions:

```
 Side view (vertical channels)       Top view (rotation)
                                          ↻ 10–20 Hz
      ╱  ch 0 (top)                     · · · · ·
  ◉ ─── ch 1                          ·     ↑     ·
      ╲  ...                         ·    [car]    ·
       ╲ ch 63 (bottom)               ·           ·
  ─────────────── ground                ·  · · ·
```

- **Channels** are stacked lasers, for example 32, 64 or 128 beams. They set the **vertical resolution**.
- **Rotation** or a beam-steering mechanism sweeps the beams horizontally.
- Each full sweep produces one **frame**, a point cloud of roughly 100k or more points with values (x, y, z, intensity) per point.

#### 4. How the beam is steered

| Type | Mechanism | Field of view | Notes |
|---|---|---|---|
| **Mechanical spinning** | Whole laser/detector stack rotates | 360° | Mature and high quality, but bulky and expensive (e.g., classic roof-mounted units) |
| **MEMS** | A tiny oscillating mirror deflects one beam | ~120° | Compact, partly solid-state; common in production cars |
| **Flash** | One wide burst lights the whole scene, read by a 2D detector array (like a camera) | Narrow | No moving parts; range is limited |
| **OPA (optical phased array)** | Steers the beam electronically through phase shifts | Narrow | Truly solid-state; still maturing |

**Product examples**

| Steering type | Product examples | Notes / deployments |
|---|---|---|
| **Mechanical spinning** | Velodyne HDL-64E, VLP-16/32; Ouster OS0/OS1/OS2; Hesai Pandar64, XT32, OT128; RoboSense RS-Ruby, RS-Helios | The HDL-64E is the sensor behind the **KITTI** dataset. Hesai Pandar64 and Velodyne HDL-32E were used in **nuScenes**. Common on robotaxis and research cars. Velodyne merged with Ouster in 2023. |
| **Rotating mirror / polygon / prism** (often called "hybrid solid-state") | Valeo SCALA 1/2; Luminar Iris; Seyond (formerly Innovusion) Falcon; Hesai AT128; Livox HAP | The laser stays still and only a mirror or prism moves. **SCALA**: Audi A8, Mercedes S-Class Drive Pilot (L3). **Iris**: Volvo EX90. **Falcon**: NIO ET7. **AT128**: Li Auto L-series. **HAP**: Xpeng P5. Currently the most common type in mass-produced cars. |
| **MEMS** | RoboSense M1 / M1 Plus; InnovizOne / InnovizTwo; MicroVision MAVIN; Blickfeld Cube | **InnovizOne**: BMW 7 Series (L3 Personal Pilot). **RoboSense M1**: Xpeng G9 and several other Chinese EVs. |
| **Flash** | Continental HFL110; Hesai FT120; Ouster DF / Sense Photonics; MicroVision (ex-Ibeo) ibeoNEXT | Mostly **short range**: blind-spot coverage, near-field, low-speed maneuvering. Not used as the main long-range sensor. |
| **OPA (optical phased array)** | Quanergy S3 (early attempt); Analog Photonics; Voyant Photonics | **Not in mass-production cars.** Quanergy went bankrupt in 2022. Still mostly research and early-stage development. |

> FMCW is a ranging method and is separate from beam steering. Examples: **Aeva Aeries II / Atlas** (partnership with Daimler Truck), **SiLC Eyeonic**, **Aurora FirstLight** (in-house, used on Aurora's self-driving trucks).

- Research and robotaxis (Waymo, Zoox, datasets): mechanical spinning, giving 360° coverage. Waymo designs its own LiDARs.
- Production passenger cars: forward-facing rotating mirror or MEMS units in the roofline or grille, with about 120° field of view and long range.
- Supplementary short range: flash.
- Future: OPA and FMCW chip-scale LiDAR.


#### 5. Processing pipeline in autonomous driving

```
Raw returns → point cloud → motion compensation (ego-motion de-skew)
 → ground removal → detection/segmentation (e.g., PointPillars, CenterPoint)
 → fusion with camera/radar → tracking → planning
```

*Reference: [Lidar for Autonomous Driving: The principles, challenges, and trends for automotive lidar and perception systems](https://arxiv.org/pdf/2004.08467)*

## Radar
Radars are another type of active sensor used in autonomous vehicles. Unlike LiDARs, radars emit radio waves rather than infrared light waves. While they can detect objects at greater distances, their accuracy is generally lower compared to LiDARs.

## GPS
## IMU

## Sensor fusion
### Camera and LiDAR fusion


# Camera-only Solution
## Camera
## GPS
## IMU
