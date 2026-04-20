# Range Actions

Range Actions is a Home Assistant automation blueprint that watches one numeric entity and runs actions when the value enters a configured range.
It is designed for threshold-based automations where you need predictable first-match range behavior, reduced edge flapping, and clear context variables for notifications.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FmimoSK%2FRange-actions%2Frefs%2Fheads%2Fmain%2Frange-actions.yaml)

## What This Blueprint Does

- Monitors one numeric entity (`sensor`, `input_number`, or `number`).
- Evaluates ranges in order and uses first-match-wins behavior.
- Runs actions only when a new range is entered.
- Uses a required `input_number` helper to remember the last entered range.
- Keeps the last active range latched across allowed deadband gaps to reduce repeated triggers near thresholds.
- Supports optional extra conditions before actions run.
- Exposes useful template variables about previous/current values and selected range metadata.

## Required Entities

- One numeric entity:
	- `sensor`
	- `input_number`
	- `number`
- One `input_number` helper used to remember the last entered range index.

## Inputs

### Mandatory

- **Numeric Entity**: Entity whose state is monitored.
- **Monitored Ranges**: Ordered list of ranges with:
	- `name` (required)
	- `description` (optional)
	- `min` (required)
	- `max` (required)
- **Remembered Range Helper**: `input_number` helper that stores the last entered range index.
  Configure it with:
	- minimum `-1`
	- maximum at least `number_of_ranges - 1`
	- initial value `-1`

### Optional

- **Additional Conditions**: Extra conditions that must pass before actions execute.
- **On Range Entered**: Action sequence executed on range entry.

## Create The Helper

1. In Home Assistant, go to `Settings -> Devices & Services -> Helpers`.
2. Click `Create Helper`.
3. Choose `Number`.
4. Set a name such as `Helper remembered range`.
5. Set minimum to `-1`.
6. Set maximum to at least `number_of_ranges - 1`.
7. Set step to `1`.
8. Set the initial value to `-1`.
9. Select this helper in the blueprint as `Remembered Range Helper`.

## Available Templates In Actions

- `{{ monitored_entity }}`: Entity ID being monitored.
- `{{ current_value }}`: Current numeric value (or `none`).
- `{{ remembered_range_index }}`: Latched range index stored in the helper (or `-1`).
- `{{ current_range_index }}`: Current matching range index (or `-1`).
- `{{ entered_range_index }}`: Entered range index (or `-1`).
- `{{ entered_range_name }}`: Name of entered range.
- `{{ entered_range_description }}`: Description of entered range.
- `{{ entered_range_min }}`: Minimum bound of entered range.
- `{{ entered_range_max }}`: Maximum bound of entered range.
- `{{ range_was_entered }}`: Boolean indicating whether a new range was entered.

## Notes

- Ranges are inclusive on both ends.
- Overlapping ranges are allowed, but ordering matters because first match wins.
- Invalid ranges are ignored during evaluation (invalid bound values or `min > max`).
- The remembered range helper is updated before optional additional conditions are evaluated.
- Because of that, the current range remains remembered even when additional conditions block the action.
- Entry detection compares the current matching range to `remembered_range_index`, not to the trigger's previous state.
- This makes it possible to create deadbands between ranges without retriggering repeatedly while the value fluctuates around a threshold.
- The automation uses `mode: queued` so quick successive entries are processed in order.

## Example Range Configuration

```yaml
ranges:
	- name: Low
		description: Comfortable low level
		min: 0
		max: 25
	- name: Medium
		description: Mid range
		min: 26
		max: 60
	- name: High
		description: Elevated level
		min: 61
		max: 100
```

<details>
<summary><b>Example Automation: CO2 Guard Example</b></summary>

```yaml
alias: CO2 guard
description: >-
	Sends CO2 level notifications based on entered ranges while using deadband gaps
	and a remembered range helper to reduce repeated edge-trigger spam.
use_blueprint:
	path: mimoSK/range-actions.yaml
	input:
		numeric_entity: input_number.test_sensor_value
		ranges:
			- name: All clear
				description: All Clear. No action needed.
				min: 0
				max: 1350
			- min: 1500
				max: 1900
				name: Slightly Elevated
				description: >-
					CO2 level is above 1500ppm. Ventilation recommended. May cause
					drowsiness or reduced focus.
			- min: 2000
				max: 2900
				name: Moderately High
				description: >-
					CO2 level is above 2000ppm. Increase ventilation. May cause headaches
					or mild sleepiness.
			- min: 3000
				max: 3900
				name: High Levels
				description: >-
					CO2 level is above 3000ppm. Ventilate now! May cause increased
					sleepiness or poor concentration.
			- name: Very High Levels
				description: >-
					CO2 level is above 4000ppm. Increase ventilation urgently! May cause
					stronger headaches or cognitive impairment.
				min: 4000
				max: 4900
			- name: Critical Levels
				description: >-
					CO2 level is above 5000ppm. Increase ventilation immediately! May
					cause nausea, dizziness, or breathing issues.
				min: 5000
				max: 999999
		additional_conditions:
			- condition: template
				value_template: >-
					{% set co2_normal = current_range_index == 0 %}  {% set
					range_increased = current_range_index > remembered_range_index %} {% if
					range_increased or co2_normal %}
						true
					{% else %}
						false
					{% endif %}
		entered_range_actions:
			- action: notify.<add your device>
				metadata: {}
				data:
					title: "CO2: {{ entered_range_name }}"
					message: "{{ entered_range_description }}"
		remembered_range_helper: input_number.co2_guard_range_helper
```
</details>

## Importing The Blueprint

### Option 1: Use "Import blueprint" badge

1. Click on "Import blueprint" badge which is at the top of this document.
2. Follow the import instructions.

### Option 2: Local file import

1. Copy `range-actions.yaml` to your Home Assistant blueprints path, for example:
	 - `config/blueprints/automation/<your_namespace>/range-actions.yaml`
2. Reload automations/blueprints from Home Assistant.
3. Create an automation from this blueprint.
