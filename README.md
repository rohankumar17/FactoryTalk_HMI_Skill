# FactoryTalk View ME — XML Engineering & Automation Skill Guide

A comprehensive, battle-tested reference for programmatically generating, modifying, and troubleshooting **Rockwell Automation FactoryTalk View Machine Edition (ME)** graphic displays (`Gfx-ME` XML), reusable faceplates, parameter files (`.par`), and alarm databases (`Alarms.xml`).

---

## Table of Contents
1. [Core Architectural Rules](#1-core-architectural-rules)
2. [Graphic Display XML (`Gfx-ME.xsd`) Rules](#2-graphic-display-xml-gfx-mexsd-rules)
   - [Direct Referencing vs HMI Tag Database](#direct-referencing-vs-hmi-tag-database)
   - [Button Schema & the `keyAssignment` Crash](#button-schema--the-keyassignment-crash)
   - [Button Width & Caption Truncation Rules](#button-width--caption-truncation-rules)
   - [Multi-line Text Formatting (`&#xA;`)](#multi-line-text-formatting-x0a)
3. [Parameter Files (`.par`) & Reusable Faceplates](#3-parameter-files-par--reusable-faceplates)
   - [Parameter File Syntax & Quoting Traps](#parameter-file-syntax--quoting-traps)
   - [Parameter Import into Studio ME](#parameter-import-into-studio-me)
   - [Literal String Variable Syntax (`/*LS:0 {#4}*/`)](#literal-string-variable-syntax-ls0-4)
   - [Expression Parameter Syntax (`{#1.AUT}` vs `{{{#1}}.AUT}`)](#expression-parameter-syntax-1aut-vs-1aut)
4. [Alarm System XML (`Alarms.xml`) Deep Dive](#4-alarm-system-xml-alarmsxml-deep-dive)
   - [Exact Root Schema & Versioning](#exact-root-schema--versioning)
   - [Trigger Element Syntax & Expression Types](#trigger-element-syntax--expression-types)
   - [The Fatal `#` Prefix Trap in Message Triggers (`trigger="#T1"`)](#the-fatal--prefix-trap-in-message-triggers-triggert1)
   - [The "Trigger-Value = 0 is Invalid" Rule (E-Stop Inversion)](#the-trigger-value--0-is-invalid-rule-e-stop-inversion)
   - [ISA-18.2 Alarm Colour Standards](#isa-182-alarm-colour-standards)
5. [FactoryTalk Linx & Runtime Environment Gotchas](#5-factorytalk-linx--runtime-environment-gotchas)
   - [Design (Local) vs Runtime (Target) Copy](#design-local-vs-runtime-target-copy)
   - [Offline Tag File (.ACD) Configuration](#offline-tag-file-acd-configuration)
   - [Suppressing the Diagnostic List Pop-Up](#suppressing-the-diagnostic-list-pop-up)
6. [PowerShell Automation Scripts](#6-powershell-automation-scripts)

---

## 1. Core Architectural Rules

* **Direct Tag Referencing:** Modern Rockwell Logix controllers (ControlLogix 5580, CompactLogix 5380 / 5370) **never** use the internal HMI Tag Database (`HMI Tags -> Tags`). All animations, indicators, pushbuttons, and alarms directly reference the PLC shortcut: `{[SHORTCUT]TagName}`.
* **Touch-Only PanelViews:** PanelView Plus 7 Standard terminals (e.g. `2711P-T12W22D9P`) do not have physical keypad bezels. Never show `[F1]`, `[F2]`, `[F3]` in operator-facing text captions.
* **Australian English Standards:** Ensure industrial terms follow AS/NZS standards: `initialised`, `characterised`, `colour`, `submain`, `incomer`.

---

## 2. Graphic Display XML (`Gfx-ME.xsd`) Rules

### Direct Referencing vs HMI Tag Database
```xml
<!-- CORRECT: Direct PLC UDT Member Reference -->
<connection name="Value" expression="{[BNR_PLC]DO_002.MAN_SPEED_SP}"/>

<!-- FATAL ERROR: Underscore substitution when PLC tag is a UDT -->
<connection name="Value" expression="{[BNR_PLC]DO_002_MAN_SPEED_SP}"/>
```
* Always inspect the Controller Tags or UDT definitions (`.L5X`) to verify if a tag is a flat base tag or a structured UDT member (`.`).

---

### Button Schema & the `keyAssignment` Crash
When removing physical function keys from standard buttons (e.g. `acknowledgeAlarmButton`, `silenceAlarmsButton`, `clearAlarmBannerButton`), **never set `keyAssignment=""`**:

```xml
<!-- FATAL ERROR: Crashes XML import with "The attribute 'keyAssignment' with value '' failed to parse" -->
<acknowledgeAlarmButton name="Btn_Ack" ... keyAssignment="">
    <caption caption="ACKNOWLEDGE ALARM"/>
</acknowledgeAlarmButton>

<!-- CORRECT: Keep schema-compliant enum key in XML attribute, remove key from visible caption -->
<acknowledgeAlarmButton name="Btn_Ack" ... keyAssignment="F1">
    <caption caption="ACKNOWLEDGE ALARM"/>
</acknowledgeAlarmButton>
```
* **Explanation:** In the Rockwell `Gfx-ME` schema, `keyAssignment` is a strict `xs:enumeration` (`"F1"` through `"F16"`). An empty string violates schema validation. On a touch panel, `keyAssignment="F1"` is simply ignored by the hardware.

---

### Button Width & Caption Truncation Rules
FactoryTalk View ME does not dynamically auto-size buttons to text. If the bounding box is too narrow:
* Pushbuttons truncate with an asterisk: `ACKNOWLEDGE ALAR*`.
* Multistate indicators truncate with ellipsis: `EMERGENCY STOP T...`.

| Font & Size | Caption Text | Minimum Width | Recommended Width |
| :--- | :--- | :--- | :--- |
| Arial Unicode MS 11pt Bold | `ACKNOWLEDGE ALARM` | 260 px | **275 px** |
| Arial Unicode MS 11pt Bold | `SILENCE BUZZER` | 220 px | **250 px** |
| Arial Unicode MS 11pt Bold | `CLEAR INACTIVE` | 220 px | **250 px** |
| Arial Unicode MS 11pt Bold | `MASTER SYSTEM FAULT RESET` | 380 px | **405 px** |
| Arial Unicode MS 10pt Bold | `EMERGENCY STOP TRIPPED` | 310 px | Use `E-STOP TRIPPED` (285 px) |
| Arial Unicode MS 10pt Bold | `MCC-001 SUBMAIN OK (<230A)` | 305 px | Use `MCC-001 LOAD OK (<230A)` (285 px) |

---

### Multi-line Text Formatting (`&#xA;`)
To split text across multiple lines inside a button caption or header label in XML, use the XML line break entity `&#xA;`:
```xml
<!-- Two-line navigation button -->
<caption fontFamily="Arial Unicode MS" fontSize="10" bold="true" 
         caption="SYSTEM&#xA;OVERVIEW" alignment="middleCenter" wordWrap="true"/>
```

---

## 3. Parameter Files (`.par`) & Reusable Faceplates

Reusable popup faceplates (e.g. Aerator faceplates) use parameter substitution placeholders (`#1` through `#100`).

### Parameter File Syntax & Quoting Traps
Example parameter file (`AER06.par`):
```text
#1=[BNR_PLC]AER006_M01
#2=[BNR_PLC]AER006_M02
#3=[BNR_PLC]AER006_MIX
#4=AER-006
#5=MCC-002
#6=[BNR_PLC]AER006_Start_Permit
#7=[BNR_PLC]AER006_Twin_Permit
```
* **RULE 1 — Never put quotation marks around text:** `#4="AER-006"` will cause FactoryTalk View to literally print `"AER-006"` with quotes on screen.
* **RULE 2 — Spaces:** If your string contains spaces, newer FTView ME supports it, but using Non-Breaking Space (`Alt + 0160`) avoids token-splitting bugs.

---

### Parameter Import into Studio ME
* Parameter files (`.par`) **must be imported into the FTView Studio project** under the **Parameters** folder (`Right-click Parameters -> Add Component Into Application...`).
* If `.par` files are not added to the application, FactoryTalk cannot find them at runtime, causing all indicators to show `Error`, and parameters to remain un-substituted.

---

### Literal String Variable Syntax (`/*LS:0 {#4}*/`)
In FactoryTalk View ME, **static Text Objects do not evaluate plain text `#4` or `#5`**. If you type `AERATOR #4 - #5`, it literally prints `#4 - #5`.

| Syntax | What FactoryTalk Does | Result |
| :--- | :--- | :--- |
| `AERATOR #4 - #5` | Prints `#4 - #5` literally as static text | ❌ Fails |
| `AERATOR /*T:#4*/ - /*T:#5*/` | Treats `#4` as an OPC Tag path; lookups fail | ❌ Fails (`/*T:#4*/`) |
| `AERATOR /*LS:0 {#4}*/ - /*LS:0 {#5}*/` | Evaluates `#4` as a **Literal String** parameter |  **`AERATOR AER-006 - MCC-002`** |

* **`LS`** = Literal String (instructs parser not to poll a PLC tag).
* **`0`** = Primary language index.
* **`{#4}`** = Parameter placeholder.

```xml
<!-- CORRECT HEADER TITLE IN POPUP XML -->
<text name="Header_Title" height="28" width="430" left="15" top="16" ... 
      caption="AERATOR /*LS:0 {#4}*/ - /*LS:0 {#5}*/"/>
```

---

### Expression Parameter Syntax (`{#1.AUT}` vs `{{{#1}}.AUT}`)
When `#1=[BNR_PLC]AER006_M01`:
```xml
<!-- CORRECT: Exactly one outer pair of curly braces wraps the expanded tag -->
<connection name="Indicator" expression="{#1.AUT} * 1 + {#1.MAN} * 2"/>
<!-- Evaluates to: {[BNR_PLC]AER006_M01.AUT} * 1 + {[BNR_PLC]AER006_M01.MAN} * 2 -->

<!-- FATAL ERROR: Triple braces produce invalid double braces in Rockwell syntax -->
<connection name="Indicator" expression="{{{#1}}.AUT} * 1 + {{{#1}}.MAN} * 2"/>
<!-- Evaluates to: {{[BNR_PLC]AER006_M01}.AUT} * 1 + ... (SYNTAX ERROR) -->
```

---

### Numeric Input Cursor Point Objects (`<numericInputCursorPoint>`)
When operators need to enter or modify a setpoint on a touch PanelView terminal (e.g. Manual Speed Setpoint), use `<numericInputCursorPoint>` instead of a passive `<numericDisplay>`.

#### Key Attributes & Touch Keypad Activation:
* **Pop-up Keypad:** `numericPopup="keypad"` (automatically pops up the on-screen numeric keypad when touched).
* **Min/Max Limits:** `minValue="0"` `maxValue="50"` (enforces clamping limits directly in the HMI terminal).
* **Precision & Formatting:** `decimalPlaces="1"` `numberOfDigits="4"` `decimalPoint="implicit"`.
* **Bezel & Highlight:** `borderWidth="4"` `borderColor="#F0F0F0"` `highlightColor="#0078D7"` `touch="true"`.

#### Parameter Substitution Token in Connections:
When binding a reusable faceplate parameter (e.g. `#1`) to a member tag inside a `<numericInputCursorPoint>`, FactoryTalk View Studio serializes the expression as:
```xml
<!-- CORRECT: FactoryTalk View Studio XML Parameter Token for Numeric Inputs -->
<numericInputCursorPoint name="NumericInputCursorPoint1" height="25" width="95" left="157" top="189" 
                         visible="true" alignment="middleCenter" audio="true" backStyle="solid" 
                         backColor="#F0F0F0" foreColor="#003366" borderWidth="4" borderColor="#F0F0F0" 
                         highlightColor="#0078D7" touch="true" numericPopup="keypad" decimalPlaces="1" 
                         numberOfDigits="4" minValue="0" maxValue="50">
    <connections>
        <connection name="Value" expression="{/*LS:0 {{#1}}*/.MAN_SPEED}"/>
    </connections>
</numericInputCursorPoint>
```
> [!IMPORTANT]
> Notice the double curly braces around the parameter placeholder `{{#1}}` inside `{/*LS:0 {{#1}}*/.MAN_SPEED}`. FactoryTalk View ME uses double braces to escape the parameter token within connection expression strings. Single-bracket `{/*LS:0 {#1}*/.MAN_SPEED}` will fail to parse upon XML re-import.

---

## 4. Alarm System XML (`Alarms.xml`) Deep Dive

### Exact Root Schema & Versioning
When exporting or importing alarms in FactoryTalk View ME, the root element and GUID are strict:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<alarms version="1.0" product="{E44CB020-C21D-11D3-8A3F-0010A4EF3494}" id="Alarms">
    <alarm history-size="128" capacity-high-warning="90" capacity-high-high-warning="99" 
           display-name="[ALARM]" hold-time="250" max-update-rate="1.00" 
           embedded-server-update-rate="1.00" silence-tag="" remote-silence-exp="" 
           remote-ack-all-exp="" status-reset-tag="" remote-status-reset-exp="" 
           close-display-tag="" remote-close-display-exp="" use-alarm-identifier="false" 
           capacity-high-warning-tag="" capacity-high-high-warning-tag="" capacity-overrun-tag="" 
           remote-clear-history-exp="">
        <triggers>
            <!-- Trigger elements go here -->
        </triggers>
        <messages>
            <!-- Message elements go here -->
        </messages>
    </alarm>
</alarms>
```

---

### Trigger Element Syntax & Expression Types
For single discrete Boolean tags, use `type="value"`:
```xml
<trigger id="T1" type="value" ack-all-value="0" use-ack-all="false" ack-tag="" 
         exp="{[BNR_PLC]BNR_Total_Clamp_Active}" message-tag="" message-handshake-exp="" 
         message-notification-tag="" remote-ack-exp="" remote-ack-handshake-tag="" 
         label="T1" handshake-tag=""/>
```

---

### The Fatal `#` Prefix Trap in Message Triggers (`trigger="#T1"`)
Inside the `<messages>` section, each message **must reference the trigger ID with a `#` prefix**:

```xml
<!-- FATAL ERROR: Causes "trigger = < T1 > is invalid. Messages referencing this value will be associated with the <Unassigned> trigger." -->
<message id="M1" trigger-value="1" identifier="1" trigger="T1" text="ALARM TEXT"/>

<!-- CORRECT: Must use #T1 to reference trigger id="T1" -->
<message id="M1" trigger-value="1" identifier="1" trigger="#T1" text="ALARM TEXT"/>
```

---

### The "Trigger-Value = 0 is Invalid" Rule (E-Stop Inversion)
In FactoryTalk View ME, **`0` is strictly reserved as the healthy / cleared / inactive state**. Setting `trigger-value="0"` triggers an immediate parser failure:
```text
trigger-value = < 0 > is invalid. Messages referencing this value will be associated with the <Unassigned> trigger.
```

**The Solution for Normally-Closed / Fail-Safe Signals (e.g. E-Stop):**
Invert the trigger expression using `NOT` so it evaluates to `1` when tripped:
```xml
<!-- Trigger evaluates to 1 when Estop_Healthy drops to 0 -->
<trigger id="T6" type="value" ack-all-value="0" use-ack-all="false" ack-tag="" 
         exp="NOT {[BNR_PLC]Estop_Healthy}" label="T6" .../>

<!-- Message triggers on valid value 1 -->
<message id="M6" trigger-value="1" identifier="6" trigger="#T6" backcolor="#800000" forecolor="#FFFFFF" 
         text="[CRITICAL] EMERGENCY STOP ACTIVATED - BNR AERATION SYSTEM SHUTDOWN"/>
```

---

### ISA-18.2 Alarm Colour Standards
| Priority / Category | `backcolor` | `forecolor` | Use Case |
| :--- | :--- | :--- | :--- |
| **Critical / Trip / Safety** | `#800000` (Dark Red) | `#FFFFFF` | E-Stop, Comms Loss, Motor Overload Trips |
| **Warning / Clamping** | `#B8860B` (Amber) | `#FFFFFF` | Load Shedding, Incomer Current Clamp Active |
| **Advisory / Process High-Low** | `#FFD700` (Yellow) | `#000000` | Dissolved Oxygen High / Low Process Setpoints |

---

## 5. FactoryTalk Linx & Runtime Environment Gotchas

1. **Design vs Runtime Tabs:**
   * In **Communication Setup**, configuring the **Design (Local)** tab only sets up testing on your laptop.
   * You **must click "Copy from Design to Runtime"** at top right; otherwise, the downloaded `.mer` on the physical PanelView cannot find the PLC.
2. **Offline Tag File (`.ACD`):**
   * Link the shortcut's **Offline Tag File** to the Logix `.ACD` file. This permits offline browsing in FTView Studio tag browsers before hardware arrives.
3. **Diagnostic List Pop-Up Suppress:**
   * When testing screens offline without a physical PLC, FTView automatically pops up the dark blue Diagnostic List banner (`Expression contains a tag that has an error value`).
   * To prevent it from hijacking screens during development: Go to **System $\rightarrow$ Diagnostic List Setup $\rightarrow$ Display tab $\rightarrow$ Uncheck "Error"**.

---

## 6. PowerShell Automation Scripts

Use this reference script to programmatically convert an Alarm CSV register into an FTView ME XML import file:

```powershell
# Convert Alarms CSV to FactoryTalk View ME XML
$csv = Import-Csv 'FactoryTalk_Alarms_And_Events.csv'
$sbTriggers = New-Object System.Text.StringBuilder
$sbMessages = New-Object System.Text.StringBuilder
$i = 1

foreach ($row in $csv) {
    $tId = "T$i"
    $mId = "M$i"
    $cleanTag = "{[BNR_PLC]" + ($row.Trigger_Tag -replace '.*\[BNR_PLC\]|\}.*', '') + "}"
    $thresh = $row.Threshold.Trim()
    
    # Invert 0-active triggers
    if ($thresh -eq "0" -or $cleanTag -like "*Estop*") {
        $exp = "NOT $cleanTag"
        $thresh = "1"
    } else {
        $exp = $cleanTag
    }
    
    $msgText = [System.Security.SecurityElement]::Escape($row.Message_Text.Trim())
    $backColor = switch ($row.Display_Color.ToLower()) {
        "amber"  { "#B8860B" }
        "yellow" { "#FFD700" }
        default  { "#800000" }
    }
    $foreColor = if ($row.Display_Color.ToLower() -eq "yellow") { "#000000" } else { "#FFFFFF" }
    
    [void]$sbTriggers.AppendLine("            <trigger id=""$tId"" type=""value"" ack-all-value=""0"" use-ack-all=""false"" exp=""$exp"" label=""$tId""/>")
    [void]$sbMessages.AppendLine("            <message id=""$mId"" trigger-value=""$thresh"" identifier=""$i"" trigger=""#$tId"" backcolor=""$backColor"" forecolor=""$foreColor"" text=""$msgText""/>")
    $i++
}

$xml = @"
<?xml version="1.0" encoding="UTF-8"?>
<alarms version="1.0" product="{E44CB020-C21D-11D3-8A3F-0010A4EF3494}" id="Alarms">
    <alarm history-size="128" display-name="[ALARM]" hold-time="250" max-update-rate="1.00">
        <triggers>
$($sbTriggers.ToString())        </triggers>
        <messages>
$($sbMessages.ToString())        </messages>
    </alarm>
</alarms>
"@

[System.IO.File]::WriteAllText("Alarms_Import.xml", $xml, [System.Text.Encoding]::UTF8)
```
