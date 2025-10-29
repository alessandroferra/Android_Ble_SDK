# Blood Glucose Calibration Algorithm Analysis

## Question
Does the blood glucose calibration system use historical calibration data (memory-based algorithm), or does it only use the current calibration values (stateless translation)?

In other words: If you send calibration arrays on days 1, 2, ... n with values A₁, A₂, ... Aₙ, does the corrected measurement on day n depend only on Aₙ, or also on previous values A₁, A₂, ... Aₙ₋₁?

## Answer: **Stateless (No Memory)**

The blood glucose calibration system appears to be **stateless** and does NOT use historical calibration data. Here's the evidence:

## Technical Evidence

### 1. **Calibration Data Structure**

The SDK uses two calibration modes:

#### Single Private Mode
- **Data sent**: One `float` value (`adjustingValue`)
- **Storage**: Stored on the device
- **No historical array**: Only current calibration value

```java
VPOperateManager.getInstance().setBloodGlucoseAdjustingData(
    fValue,     // Single calibration value
    isOpen,     // Enable/disable flag
    writeResponse,
    listener
);
```

#### Multiple Calibration Mode
- **Data sent**: Three `MealInfo` objects (breakfast, lunch, dinner)
- **Each MealInfo contains**:
  - `bgBeforeMeal`: Pre-meal calibration value
  - `bgAfterMeal`: Post-meal calibration value
  - `beforeMealTime`: Time window for pre-meal measurement
  - `afterMealTime`: Time window for post-meal measurement

```java
public class MealInfo {
    public int index;              // 1=breakfast, 2=lunch, 3=dinner
    private float bgBeforeMeal;    // Pre-meal calibration (mmol/L)
    private float bgAfterMeal;     // Post-meal calibration (mmol/L)
    public int beforeMealTime;     // Time in minutes (e.g., 8:30 = 510)
    public int afterMealTime;      // Time in minutes
    // No historical data fields
}
```

**Key observation**: `MealInfo` contains NO fields for historical data, previous calibrations, or timestamps of past calibrations.

### 2. **No Local Storage of Historical Data**

Analysis of the SDK bytecode shows:
- **No SharedPreferences usage** for storing historical calibration values
- **No database** for calibration history
- **No arrays or lists** that accumulate past calibration values
- Each `setBloodGlucoseAdjustingData()` call **overwrites** the previous calibration value

### 3. **Device-Side Processing**

The calibration flow is:
```
App → [Set calibration Aₙ] → Device stores Aₙ
Device → [Measures raw sensor value] → Device applies Aₙ → Calibrated value
Device → [Returns calibrated value] → App receives final result
```

The device:
1. Receives current calibration value(s)
2. Stores them (replacing any previous values)
3. Applies them to raw sensor readings based on current time
4. Returns the calibrated blood glucose value

### 4. **Time-Based Selection (Not History-Based)**

For multiple calibration mode, the device uses **current time** to select which calibration to apply:

- Before breakfast time → Apply `breakfast.bgBeforeMeal`
- After breakfast time → Apply `breakfast.bgAfterMeal`
- Before lunch time → Apply `lunch.bgBeforeMeal`
- And so on...

This is a **conditional selection based on time of day**, not a historical algorithm.

### 5. **Read Operation Returns Current State Only**

When reading calibration settings:

```java
void onBloodGlucoseAdjustingReadSuccess(boolean isOpen, float adjustingValue);
```

The callback returns:
- `isOpen`: Current on/off state
- `adjustingValue`: Current calibration value

**No historical values** are returned or accessible.

## Comparison with Blood Pressure

Interestingly, the SDK has a **different approach for blood pressure**:

```java
public class BpSetting {
    public boolean isAngioAdjuste;  // Dynamic blood pressure calibration flag
    // ...
}
```

Blood pressure has an `isAngioAdjuste` flag for "动态血压校准" (dynamic blood pressure calibration), suggesting some form of adaptive/historical calibration.

**However, blood glucose has NO equivalent dynamic calibration flag**, reinforcing that glucose calibration is stateless.

## Conclusion

Based on the code analysis:

**If you send calibration values A₁, A₂, ... Aₙ on days 1, 2, ... n:**
- The corrected measurement on day n depends **ONLY on Aₙ**
- Previous calibrations A₁, A₂, ... Aₙ₋₁ are **overwritten and not used**

The calibration system is a **pure translation/mapping function**:
```
correctedGlucose = f(rawSensorValue, currentCalibration, currentTime)
```

Where:
- `currentCalibration` = Aₙ (no history)
- `currentTime` = used only to select which meal calibration to apply (in multi-calibration mode)

## Implications

1. **Each calibration is independent** - sending a new calibration completely replaces the old one
2. **No learning over time** - the algorithm doesn't improve based on historical patterns
3. **Simple linear or non-linear transformation** - likely a formula like:
   ```
   correctedValue = rawValue × calibrationFactor
   ```
   or
   ```
   correctedValue = rawValue + calibrationOffset
   ```

4. **User must manually update** - if sensor drift occurs, the user must manually send a new calibration value

## Recommendation for Further Investigation

To fully understand the calibration algorithm, you would need to:

1. **Monitor raw vs. calibrated values** over time with different calibration settings
2. **Test with systematic calibration changes** to understand if it's:
   - Linear scaling: `corrected = raw × k`
   - Offset adjustment: `corrected = raw + offset`
   - Non-linear mapping: `corrected = f(raw, calibration)`

3. **Reverse engineer the device firmware** (if legally permitted) to see the exact calibration formula

---

**Analysis Date**: 2025-10-29
**SDK Version Analyzed**: vpprotocol-2.3.28.15.aar
**Confidence Level**: High (based on thorough code and documentation review)
