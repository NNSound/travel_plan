# Agent UI Operation Manual

This document is for AI Agents (like Gemini, GPT-4, etc.) who are "visiting" and "operating" the Travel Planner website via browser tools.

## 🤖 Semantic Interaction Layer
All interactive elements carry a `data-agent` attribute for precise targeting. Use these tokens with your `click` or `type` tools.

### Sidebar & Navigation
- `data-agent="sidebar-toggle"`: Open/Close the sidebar.
- `data-agent="trip-item-{filename}"`: Select a specific trip.
- `data-agent="upload-json"`: Trigger the file upload dialog.
- `data-agent="json-guide-toggle"`: Open the documentation modal.
- `data-agent="prev-day"` / `data-agent="next-day"`: Navigate between itinerary days.

### Main Actions
- `data-agent="export-trip"`: Export the current itinerary as JSON.
- `data-agent="delete-trip"`: Delete the currently selected trip.
- `data-agent="pocket-list-toggle"`: Open/Close the Pocket List modal.

### Day & Activity Management
- `data-agent="day-header-{dayNumber}"`: Information about the current day.
- `data-agent="edit-day-header"`: Enter edit mode for date and title of the day.
- `data-agent="day-date-input"` / `data-agent="day-title-input"`: Fields to edit day info.
- `data-agent="save-day-header"`: Exit day editing mode.
- `data-agent="add-activity"`: Add a new blank activity to the current day.

### Activity Card Operations
- `data-agent="activity-card"`: The card container.
- `data-agent="edit-activity-btn"`: Enter edit mode for an activity.
- `data-agent="activity-title-input"`: Edit the title.
- `data-agent="activity-time-input"`: Edit the time (HH:mm).
- `data-agent="activity-tags-input"`: Edit tags (comma separated).
- `data-agent="activity-location-input"`: Edit location name.
- `data-agent="activity-map-link-input"`: Edit Google Maps link.
- `data-agent="activity-description-input"`: Edit the description text.
- `data-agent="save-activity-btn"`: Save changes and exit edit mode.
- `data-agent="remove-activity-btn"`: Delete the activity.

## 🛠 Operational Workflows

### 1. Generating a New Activity
1. Locate the "Add Activity" button: `[data-agent="add-activity"]`.
2. Click it to create a card in edit mode.
3. Fill in the fields using the `data-agent` input selectors.
4. Click the "Done" button: `[data-agent="save-activity-btn"]`.

### 2. Organizing from Pocket List
1. Open the Pocket List: `[data-agent="pocket-list-toggle"]`.
2. Find an item and click its "Add to Trip" button.
3. The item will appear at the end of the current day.

---
**Status Monitor**: You can always check your status or find this manual via the **Agent Status Card** in the bottom-right corner of the viewport.
