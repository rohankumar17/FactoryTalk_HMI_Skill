---
name: factorytalk-me-hmi
description: Authoritative guide and technical reference for coding Rockwell Automation FactoryTalk View Machine Edition (ME) displays, faceplates, parameter files (.par), and alarm configurations in XML (Gfx-ME schema). Use whenever generating, editing, or troubleshooting FactoryTalk View ME XML graphics, buttons, expressions, parameters, or Alarms.xml.
---

# FactoryTalk View ME XML Engineering Skill

## 1. Quick Syntax Cheat Sheet

| Task | Correct Syntax | Common Fatal Mistake |
| :--- | :--- | :--- |
| **Direct PLC Tag Reference** | `{[SHORTCUT]TagName}` | Using internal HMI Tag database for Logix |
| **UDT Member Tag** | `{[SHORTCUT]TagName.Member}` | Using underscores `TagName_Member` |
| **Expression Tags** | `{[SHORTCUT]TagA} * 1 + {[SHORTCUT]TagB} * 2` | Omitting curly braces or using double braces |
| **Touch Button Function Key** | `keyAssignment="F1"` + clean visible caption | Setting `keyAssignment=""` (crashes XML import) |
| **Multi-line Caption** | `caption="LINE1&#xA;LINE2"` | Using literal Enter or `\n` in XML attributes |
| **Parameter Placeholder in Text** | `caption="PREFIX /*LS:0 {#4}*/"` | Using `#4` (prints literal "#4") or `/*T:#4*/` |
| **Parameter in Expression** | `expression="{#1.AUT} * 1 + {#1.MAN} * 2"` | Using triple braces `expression="{{{#1}}.AUT}"` |
| **Parameter in Numeric Input** | `expression="{/*LS:0 {{#1}}*/.MAN_SPEED}"` | Using single braces `{/*LS:0 {#1}*/.TAG}` |
| **Parameter File Text Value** | `#4=PUMP-101` (no quotes) | `#4="PUMP-101"` (prints literal quotation marks) |
| **Alarm Trigger Element** | `<trigger id="T1" type="value" exp="{[PLC]Tag}" .../>` | Leaving triggers empty |
| **Alarm Message Trigger Binding**| `<message id="M1" trigger="#T1" .../>` | `trigger="T1"` (missing `#` causes `<Unassigned>` error) |
| **Normally-Closed / E-Stop Alarm**| `exp="NOT {[PLC]Estop_Healthy}"` + `trigger-value="1"` | `trigger-value="0"` (causes `trigger-value = < 0 > is invalid`) |

---

## 2. Graphic Display XML (`Gfx-ME.xsd`) Rules

### Button Width Guidelines (Prevent Asterisk `*` Truncation)
Pushbuttons truncate with `*` if the pixel width is smaller than the rendered text length.
* `ACKNOWLEDGE ALARM` (11pt Bold): **275 px** minimum.
* `SILENCE BUZZER` (11pt Bold): **250 px** minimum.
* `CLEAR INACTIVE` (11pt Bold): **250 px** minimum.
* `MASTER SYSTEM FAULT RESET` (11pt Bold): **405 px** minimum.

### Function Key Attributes on Touch Terminals
On PanelView Plus 7 Standard (touch-only):
* Retain `keyAssignment="F1"`, `keyAssignment="F2"`, etc. in the XML attribute.
* Clean the caption: `caption="ACKNOWLEDGE ALARM"` (remove `[F1]` from visible caption text).
* Setting `keyAssignment=""` causes:
  `Cause: Value cannot be empty. The attribute 'keyAssignment' with value '' failed to parse.`

---

## 3. Reusable Faceplates & Parameter Files

1. **Parameter File Registration:**
   * `.par` files must be imported into the Project Explorer under **Parameters**.
2. **Text Object Parameter Replacement:**
   * Never put `#4` directly in a text caption. FactoryTalk View ME requires the Literal String token:
     `caption="EQUIPMENT /*LS:0 {#4}*/ - /*LS:0 {#5}*/"`
3. **Expression Tag Formatting:**
   * When `#1=[PLC]Pump01`:
   * Use `{#1.STATUS}` $\rightarrow$ expands to `{[PLC]Pump01.STATUS}`.

---

## 4. Alarm Setup XML (`Alarms.xml`)

* Root GUID: `{E44CB020-C21D-11D3-8A3F-0010A4EF3494}`
* **Message Binding:** `trigger="#T1"` (always prepend `#` to trigger ID).
* **Active State Value:** Must be `1` or greater. Never use `0`. Invert active-low tags with `NOT` in the trigger expression.
* **ISA-18.2 Colours:**
  * Critical / Trip: `backcolor="#800000"` `forecolor="#FFFFFF"`
  * Warning / Clamp: `backcolor="#B8860B"` `forecolor="#FFFFFF"`
  * Process High-Low: `backcolor="#FFD700"` `forecolor="#000000"`
