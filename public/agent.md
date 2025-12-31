# Travel Plan JSON Specification

This guide explains how to generate or create JSON files that are compatible with the Travel Planner application. These files can be imported via the "Import JSON" button in the sidebar.

## JSON Structure Overview

The root of the JSON file must contain the following fields:

| Field | Type | Description |
| :--- | :--- | :--- |
| `name` | String | The overall name of the trip. |
| `days` | Array | A list of daily itineraries. |
| `pocketList` | Array | (Optional) A list of inspirations/spots not yet assigned to a specific day. |

---

## Component Details

### 1. Day Object
Each item in the `days` array should have:
- `day` (Number): The sequence number of the day.
- `date` (String): The date in `YYYY-MM-DD` format.
- `title` (String): A brief theme for the day.
- `activities` (Array): A list of activity objects.

### 2. Activity Object
Used in both `days.activities` and `pocketList`.
- `time` (String): 24-hour format `HH:mm`.
- `title` (String): Name of the activity or spot.
- `location` (String): Human-readable address or landmark name.
- `description` (String): Details or notes.
- `mapLink` (String): Google Maps search URL (see below).
- `tags` (Array): Category labels (e.g., `["Food", "Sightseeing"]`).
- `visited` (Boolean): (Optional, used in Pocket List) Set to `true` if already visited.

---

## Guidelines for Quality Content

### 📍 Google Maps Links
For better reliability and readability, **ALWAYS** use the Google Maps Search API format instead of raw coordinates:
- **Format**: `https://www.google.com/maps/search/?api=1&query=Landmark+Name`
- **Example**: `https://www.google.com/maps/search/?api=1&query=Taipei+101+Observatory`

### 🏷️ Recommended Tags
The UI provides special styling for the following tags:
- `Transport`
- `Food`
- `Sightseeing`
- `Shopping`
- `Ski`
- `Accommodation`
- `Free`

---

## Example Template

```json
{
  "name": "Taiwan Trip Example",
  "days": [
    {
      "day": 1,
      "date": "2024-12-31",
      "title": "New Year Arrival",
      "activities": [
        {
          "time": "14:00",
          "title": "Arrive at TAIPEI 101",
          "location": "Taipei 101",
          "description": "Check-in and view city lights.",
          "mapLink": "https://www.google.com/maps/search/?api=1&query=Taipei+101",
          "tags": ["Sightseeing", "Shopping"]
        }
      ]
    }
  ],
  "pocketList": [
    {
      "title": "Din Tai Fung",
      "location": "Xinyi District",
      "description": "Famous xiaolongbao.",
      "mapLink": "https://www.google.com/maps/search/?api=1&query=Din+Tai+Fung+Xinyi",
      "tags": ["Food"],
      "visited": false
    }
  ]
}
```
