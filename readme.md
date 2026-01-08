# Open Illuminations Standard (OIS)

**Version:** 1.0 (Draft)  
**Status:** Active Development  
**JSON Schema:** [schema.json](https://raw.githubusercontent.com/neshani/open-illuminations-standard/main/schema.json)

## Overview

The **Open Illuminations Standard (OIS)** is a specification for creating, distributing, and rendering immersive visual accompaniments for audiobooks. 

Unlike standard ebook formats (EPUB) or proprietary audiobook formats (M4B/AAX), OIS is designed as a **synchronization layer**. It allows a "Visual Pack" (Illuminations) to be distributed independently of the audio files, enabling any audio player to render timed artwork, Ken Burns-style animations, and quotes synchronized to the user's existing audiobook library.

## Core Principles

1.  **Audio Agnostic:** The standard works regardless of the audio source (MP3 folders, single M4B, AAC).
2.  **Screen Agnostic:** The coordinate system is normalized to function on any aspect ratio, from smartwatches to widescreen TVs.
3.  **Flat Architecture:** Illumination packs are simple, flat Zip archives containing images and JSON.
4.  **Global Timeline:** Timestamps are absolute relative to the start of the book's narrative content.

---

## The Container: `.illuminations.zip`

An Illumination Pack is a standard ZIP archive. By convention, files may use the `.illuminations.zip` extension to distinguish them from generic archives, though `.zip` is acceptable.

### File Structure
The internal structure must be **flat**. No subdirectories are allowed.

```text
my-book.illuminations.zip
├── manifest.json         <-- The Source of Truth (Default/Mobile)
├── manifest.desktop.json <-- Optional Variant
├── cover.jpg
├── 001_intro.webp
├── 002_scene_a.webp
└── ...
```

---

## The Manifest

The `manifest.json` file defines the metadata, the available variants, and the default keyframe timeline.

### Coordinate System (The "View")
To support dynamic aspect ratios, OIS uses a normalized coordinate system.

*   **`pan_x`, `pan_y`**: A float between `0.0` and `1.0`.
    *   `(0.0, 0.0)` is the top-left of the **image**.
    *   `(1.0, 1.0)` is the bottom-right of the **image**.
    *   These coordinates represent the **center point** of the camera's focus.
*   **`scale`**: A float representing zoom relative to the **container (screen)**.
    *   `1.0`: The image is scaled to fit fully within the screen (letterboxed or pillarboxed) with no cropping.
    *   `>1.0`: The image is zoomed in relative to the "fit" size.

### Timeline Logic
OIS uses a **Start-Only Keyframe** system. There are no "Stop" timestamps.

1.  **Interpolation (Animation):**
    If `Keyframe A` and `Keyframe B` are sequential and reference the **same image file**:
    *   The player animates (pans/zooms) from View A to View B.
    *   The duration is: `start[B] - start[A]`.
    *   *Note:* The standard assumes linear interpolation, though players may apply smoothing/easing.

2.  **The "Hold" Rule:**
    If `Keyframe B` (Image X) ends at `00:10`, and `Keyframe C` (Image Y) starts at `00:20`:
    *   The animation for Image X completes at `00:10`.
    *   Image X remains static (holding the View defined in Keyframe B) until `00:20`.
    *   At `00:20`, the player cuts to Image Y.

### Example `manifest.json`

```json
{
  "manifest_version": "1.3",
  "book_title": "The Call of Cthulhu",
  "variants": [
    { "slug": "default", "name": "Mobile", "description": "Optimized for Portrait" },
    { "slug": "Desktop", "name": "Desktop", "description": "Optimized for Landscape" }
  ],
  "keyframes": [
    {
      "image": "scene_01.webp",
      "start": "00:00:00.00",
      "quote": "We live on a placid island of ignorance...",
      "view": { "scale": 1.2, "pan_x": 0.5, "pan_y": 0.5 }
    },
    {
      "image": "scene_01.webp",
      "start": "00:00:15.50",
      "view": { "scale": 1.8, "pan_x": 0.2, "pan_y": 0.3 }
    },
    {
      "image": "scene_02.webp",
      "start": "00:00:15.50",
      "view": { "scale": 1.0, "pan_x": 0.5, "pan_y": 0.5 }
    }
  ]
}
```

---

## Integration Guidelines

These guidelines differ from strict standards; they are recommendations for player developers implementing OIS.

### 1. The Time Base
The `start` timestamp represents the elapsed time from the beginning of the book. 
*   **Multi-file Books:** For audiobooks split into multiple files (e.g., `Chapter1.mp3`, `Chapter2.mp3`), the player is responsible for calculating the aggregate duration to map the current playback position to the global OIS timeline.
*   **Duration Mismatches:** If the audio file duration differs from the `authored_for_duration_seconds` in the manifest (e.g., missing intros/outros), the player must decide how to reconcile this (e.g., stretching timestamps, anchoring to start, or anchoring to end).

### 2. File Association
The method for associating an `.illuminations.zip` file with an audio file is implementation-specific.
*   **Local:** Players may look for zip files in the same directory as the audio.
*   **Repository:** Players may search a central repository using Author/Title metadata.
*   **Manual:** Players may allow users to manually import a zip file.

### 3. Rendering Variants
The `variants` array in `manifest.json` allows for different choreography based on device state.
*   The `default` variant is represented by the `keyframes` array in the root `manifest.json`.
*   Named variants (e.g., `Desktop`) correspond to a file named `manifest.{slug}.json` (e.g., `manifest.desktop.json`).
*   Players are encouraged to dynamically switch variants if the user changes device orientation or window size.

---

## Manifest Reference

The `manifest.json` file is the entry point for the illumination pack. You can validate files using the [official JSON Schema](https://raw.githubusercontent.com/neshani/open-illuminations-standard/main/schema.json).

### Root Object

| Field | Type | Status | Description |
| :--- | :--- | :--- | :--- |
| `manifest_version` | String | **Required** | The version of the OIS spec this pack adheres to (e.g., "1.3"). |
| `book_title` | String | **Required** | Title of the audiobook this pack is designed for. |
| `book_author` | String | **Required** | Author of the original book. |
| `pack_title` | String | **Required** | Title of this specific illumination pack (e.g., "Cthulhu Illustrated"). |
| `pack_version` | String | **Required** | Version of the pack itself (e.g., "1.0.0"). |
| `keyframes` | Array | **Required** | An array of [Keyframe Objects](#keyframe-object). |
| `pack_author` | String | Optional | Name of the creator/curator of the pack. |
| `author_website` | String | Optional | URL for the pack creator. |
| `pack_description` | String | Optional | Short description of the visual style or content. |
| `art_type` | String | Optional | Categorization of art (e.g., "ai-generated", "original", "public-domain"). |
| `curation_type` | String | Optional | e.g., "full-curation", "automated". |
| `content_rating` | String | Optional | e.g., "everyone", "teen", "mature". |
| `authored_for_duration_seconds`| Float | **Required** | The total duration (in seconds) of the audio file used during creation. Used to help players detect duration mismatches. |
| `variants` | Array | Optional | List of available [Variant Objects](#variant-object). |

### Keyframe Object

| Field | Type | Status | Description |
| :--- | :--- | :--- | :--- |
| `start` | String | **Required** | Timestamp in `HH:MM:SS.ss` format relative to the start of the book. |
| `image` | String | **Required** | Filename of the image located in the zip root (e.g., "001.webp"). |
| `view` | Object | **Required** | The [View Object](#view-object) defining camera position. |
| `quote` | String | Optional | Text text being spoken at this timestamp. |
| `title` | String | Optional | A chapter or section title associated with this moment. |
| `notes` | String | Optional | Curator notes or context. |

### Text Propagation (The "Dot" Rule)

To prevent file bloat and avoid re-triggering UI overlays during smooth pans/zooms, OIS uses a specific character to signify text "Carry Over".

*   **`null` or omitted**: The text field is cleared. The player should hide the text overlay.
*   **`"String"`**: The player displays the new text. If text was already visible, it may trigger a transition animation.
*   **`"."` (Period)**: The player maintains the **previous keyframe's text value**. The UI should **not** re-trigger an "appear" animation. This allows text to remain on screen while the image underneath animates through multiple keyframes.

**Example:**
```json
[
  {
    "start": "00:10",
    "quote": "The door creaked open...",
    "view": { "pan_x": 0.0, ... } 
  },
  {
    "start": "00:15",
    "quote": ".", 
    "view": { "pan_x": 0.5, ... }
    // Result: Image pans, but quote "The door creaked open..." remains visible without flickering.
  },
  {
    "start": "00:20",
    "quote": null,
    "view": { "pan_x": 1.0, ... }
    // Result: Image finishes panning, text fades out.
  }
]
```

### View Object

| Field | Type | Status | Description |
| :--- | :--- | :--- | :--- |
| `scale` | Float | **Required** | Zoom level relative to the screen container (`1.0` = fit). |
| `pan_x` | Float | **Required** | X focal point (`0.0` to `1.0`). |
| `pan_y` | Float | **Required** | Y focal point (`0.0` to `1.0`). |

### Variant Object

Used to define alternative manifests for specific scenarios (e.g., Desktop/Landscape).

| Field | Type | Status | Description |
| :--- | :--- | :--- | :--- |
| `slug` | String | **Required** | Unique identifier (e.g., "desktop"). Used to find the file `manifest.{slug}.json`. |
| `name` | String | **Required** | Human-readable name (e.g., "Desktop Mode"). |
| `description` | String | Optional | Description of what makes this variant different. |

---

## License

The Open Illuminations Standard definitions are licensed under [MIT](LICENSE).
