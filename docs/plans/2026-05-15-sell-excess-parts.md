# Sell Excess Parts Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a "Sell Excess Parts" button to the Parts in Use report that sells anything above a configurable multiple of each part's minimum stock level.

**Architecture:** New `sellExcessThreshold` field on `Campaign` (persisted to XML). New `sellExcessPartsInUse()` method on `PartsInUseManager` mirrors `stockUpPartsInUse()`. New button + spinner in `PartsReportDialog` mirrors the top-up button. No new classes, no schema changes — slots into existing patterns throughout.

**Tech Stack:** Java 21, Swing GroupLayout, MHQXMLUtility, Quartermaster.sellPart()

---

## Files and line references (as of branch base 5db2a1584c)

| File (relative to worktree root) | Purpose |
|---|---|
| `MekHQ/src/mekhq/campaign/Campaign.java` | Data model |
| `MekHQ/src/mekhq/campaign/io/CampaignXmlParser.java` | XML read |
| `MekHQ/src/mekhq/campaign/market/PartsInUseManager.java` | Business logic |
| `MekHQ/src/mekhq/gui/dialog/PartsReportDialog.java` | UI |
| `MekHQ/resources/mekhq/resources/PartsReportDialog.properties` | Strings |

---

### Task 1: Campaign.java — add sellExcessThreshold field

**Files:**
- Modify: `MekHQ/src/mekhq/campaign/Campaign.java`

**Step 1: Add field declaration**

At line 482, the "parts in use" block looks like:
```java
    // options relating to parts in use and restock
    private boolean ignoreMothballed;
    private boolean topUpWeekly;
    private PartQuality ignoreSparesUnderQuality;
```
Add `sellExcessThreshold` on the line after `topUpWeekly`:
```java
    // options relating to parts in use and restock
    private boolean ignoreMothballed;
    private boolean topUpWeekly;
    private double sellExcessThreshold;
    private PartQuality ignoreSparesUnderQuality;
```

**Step 2: Initialize in constructor block**

At line 644, the init block looks like:
```java
        topUpWeekly = false;
        ignoreMothballed = true;
        ignoreSparesUnderQuality = QUALITY_A;
```
Add the default after `topUpWeekly`:
```java
        topUpWeekly = false;
        sellExcessThreshold = 200.0;
        ignoreMothballed = true;
        ignoreSparesUnderQuality = QUALITY_A;
```

**Step 3: Add getter and setter**

At line 9925–9927, after `setTopUpWeekly`:
```java
    public void setTopUpWeekly(boolean topUpWeekly) {
        this.topUpWeekly = topUpWeekly;
    }
```
Add after it:
```java
    public double getSellExcessThreshold() {
        return sellExcessThreshold;
    }

    public void setSellExcessThreshold(double sellExcessThreshold) {
        this.sellExcessThreshold = sellExcessThreshold;
    }
```

**Step 4: Add to writePartInUseToXML**

At line 9937–9944:
```java
    public void writePartInUseToXML(final PrintWriter pw, int indent) {
        MHQXMLUtility.writeSimpleXMLTag(pw, indent, "ignoreMothBalled", ignoreMothballed);
        MHQXMLUtility.writeSimpleXMLTag(pw, indent, "topUpWeekly", topUpWeekly);
        MHQXMLUtility.writeSimpleXMLTag(pw, indent, "ignoreSparesUnderQuality", ignoreSparesUnderQuality.name());
```
Add after `topUpWeekly`:
```java
    public void writePartInUseToXML(final PrintWriter pw, int indent) {
        MHQXMLUtility.writeSimpleXMLTag(pw, indent, "ignoreMothBalled", ignoreMothballed);
        MHQXMLUtility.writeSimpleXMLTag(pw, indent, "topUpWeekly", topUpWeekly);
        MHQXMLUtility.writeSimpleXMLTag(pw, indent, "sellExcessThreshold", sellExcessThreshold);
        MHQXMLUtility.writeSimpleXMLTag(pw, indent, "ignoreSparesUnderQuality", ignoreSparesUnderQuality.name());
```

**Step 5: Commit**
```
git add MekHQ/src/mekhq/campaign/Campaign.java
git commit -m "feat: add sellExcessThreshold field to Campaign"
```

---

### Task 2: CampaignXmlParser.java — parse sellExcessThreshold from XML

**Files:**
- Modify: `MekHQ/src/mekhq/campaign/io/CampaignXmlParser.java`

**Step 1: Add parse case**

At line 2244–2245, the topUpWeekly parse block:
```java
            } else if (wn2.getNodeName().equalsIgnoreCase("topUpWeekly")) {
                retVal.setTopUpWeekly(Boolean.parseBoolean(wn2.getTextContent()));
            } else if (wn2.getNodeName().equalsIgnoreCase("ignoreSparesUnderQuality")) {
```
Add after topUpWeekly:
```java
            } else if (wn2.getNodeName().equalsIgnoreCase("topUpWeekly")) {
                retVal.setTopUpWeekly(Boolean.parseBoolean(wn2.getTextContent()));
            } else if (wn2.getNodeName().equalsIgnoreCase("sellExcessThreshold")) {
                retVal.setSellExcessThreshold(Double.parseDouble(wn2.getTextContent()));
            } else if (wn2.getNodeName().equalsIgnoreCase("ignoreSparesUnderQuality")) {
```

Older saves without `sellExcessThreshold` will keep the default of 200.0 (set in constructor), so no migration needed.

**Step 2: Commit**
```
git add MekHQ/src/mekhq/campaign/io/CampaignXmlParser.java
git commit -m "feat: persist sellExcessThreshold in campaign XML"
```

---

### Task 3: PartsInUseManager.java — add sellExcessPartsInUse

**Files:**
- Modify: `MekHQ/src/mekhq/campaign/market/PartsInUseManager.java`

**Step 1: Add the sell method and its helper**

Add after `stockUpPartsInUseGM` (after line 434), before `findStockUpAmount`:

```java
    /**
     * Sells parts from the warehouse that exceed the given threshold above each part's requested stock level.
     *
     * <p>For each part in the provided set, calculates how many units are above the sell ceiling
     * ({@code useCount * requestedStock% * threshold / 10000}) and sells the excess.</p>
     *
     * @param partsInUse the set of {@link PartInUse} instances to check
     * @param threshold  sell anything above this percentage of the requested stock level (e.g., 200.0 = 2×)
     *
     * @return the number of distinct part types sold
     */
    public int sellExcessPartsInUse(Set<PartInUse> partsInUse, double threshold) {
        int sold = 0;
        for (PartInUse partInUse : partsInUse) {
            int toSell = findSellExcessAmount(partInUse, threshold);
            if (toSell <= 0) {
                continue;
            }
            List<Part> spares = partInUse.getSpares();
            int remaining = toSell;
            for (Part spare : spares) {
                if (remaining <= 0) {
                    break;
                }
                int spareQty = spare.getSellableQuantity();
                if (spareQty <= 0) {
                    continue;
                }
                if (spareQty >= remaining) {
                    quartermaster.sellPart(spare, remaining);
                    remaining = 0;
                } else {
                    quartermaster.sellPart(spare, spareQty);
                    remaining -= spareQty;
                }
            }
            if (remaining < toSell) {
                sold++;
            }
        }
        return sold;
    }

    private int findSellExcessAmount(PartInUse partInUse, double threshold) {
        int ceiling = (int) Math.floor(partInUse.getRequestedStock() / 100.0 * threshold / 100.0 * partInUse.getUseCount());
        return Math.max(0, partInUse.getStoreCount() - ceiling);
    }
```

**Step 2: Add the missing import**

`List` is already imported via `java.util.Set` being present, but double-check that `java.util.List` is imported. The existing imports include `java.util.Set` — add `java.util.List` if it's missing.

**Step 3: Commit**
```
git add MekHQ/src/mekhq/campaign/market/PartsInUseManager.java
git commit -m "feat: add sellExcessPartsInUse to PartsInUseManager"
```

---

### Task 4: PartsReportDialog.java — add spinner and sell button

**Files:**
- Modify: `MekHQ/src/mekhq/gui/dialog/PartsReportDialog.java`

**Step 1: Add field declaration**

At line 77, the fields block:
```java
    private JCheckBox ignoreMothballedCheck, topUpWeeklyCheck;
    private RoundedJButton topUpGMButton;
```
Add the spinner field:
```java
    private JCheckBox ignoreMothballedCheck, topUpWeeklyCheck;
    private JSpinner sellExcessThresholdSpinner;
    private RoundedJButton topUpGMButton;
```

**Step 2: Add missing imports if needed**

Check that `javax.swing.JSpinner` and `javax.swing.SpinnerNumberModel` are present. The file already has `javax.swing.*` as a wildcard import, so no change needed.

**Step 3: Create spinner and sell button in initComponents**

After `resetRequestedStockButton` creation (after line 313), add:

```java
        RoundedJButton sellExcessButton = new RoundedJButton();
        sellExcessButton.setText(resourceMap.getString("sellExcessBtn.text"));
        sellExcessButton.setFocusPainted(false);
        sellExcessButton.setMargin(new Insets(10, 20, 10, 20));
        sellExcessButton.addActionListener(evt -> sellExcess());

        sellExcessThresholdSpinner = new JSpinner(new SpinnerNumberModel(
              (int) campaign.getSellExcessThreshold(), 0, 9999, 1));
        sellExcessThresholdSpinner.setMaximumSize(sellExcessThresholdSpinner.getPreferredSize());
        JLabel sellExcessLabel = new JLabel(resourceMap.getString("lblSellExcessThreshold.text"));
```

**Step 4: Update the horizontal layout group**

The existing button-row sequential group (lines 356–366):
```java
                    .addGroup(layout.createSequentialGroup()
                                    .addPreferredGap(LayoutStyle.ComponentPlacement.RELATED,
                                          GroupLayout.DEFAULT_SIZE,
                                          Short.MAX_VALUE)
                                    .addComponent(topUpButton)
                                    .addComponent(topUpGMButton)
                                    .addComponent(resetRequestedStockButton)
                                    .addComponent(btnClose)
                                    .addPreferredGap(LayoutStyle.ComponentPlacement.RELATED,
                                          GroupLayout.DEFAULT_SIZE,
                                          Short.MAX_VALUE))
```
Replace with:
```java
                    .addGroup(layout.createSequentialGroup()
                                    .addPreferredGap(LayoutStyle.ComponentPlacement.RELATED,
                                          GroupLayout.DEFAULT_SIZE,
                                          Short.MAX_VALUE)
                                    .addComponent(topUpButton)
                                    .addComponent(topUpGMButton)
                                    .addComponent(resetRequestedStockButton)
                                    .addComponent(sellExcessButton)
                                    .addComponent(sellExcessLabel)
                                    .addComponent(sellExcessThresholdSpinner)
                                    .addComponent(btnClose)
                                    .addPreferredGap(LayoutStyle.ComponentPlacement.RELATED,
                                          GroupLayout.DEFAULT_SIZE,
                                          Short.MAX_VALUE))
```

**Step 5: Update the vertical layout group**

The existing button-row parallel group (lines 377–381):
```java
                    .addGroup(layout.createParallelGroup(GroupLayout.Alignment.BASELINE)
                                    .addComponent(topUpButton)
                                    .addComponent(topUpGMButton)
                                    .addComponent(resetRequestedStockButton)
                                    .addComponent(btnClose))
```
Replace with:
```java
                    .addGroup(layout.createParallelGroup(GroupLayout.Alignment.BASELINE)
                                    .addComponent(topUpButton)
                                    .addComponent(topUpGMButton)
                                    .addComponent(resetRequestedStockButton)
                                    .addComponent(sellExcessButton)
                                    .addComponent(sellExcessLabel)
                                    .addComponent(sellExcessThresholdSpinner)
                                    .addComponent(btnClose))
```

**Step 6: Add sellExcess() method**

After `topUpGM()` (after line 467), add:
```java
    private void sellExcess() {
        commitTableEdits();
        storePartInUseRequestedStockMap();

        partsInUseManager.sellExcessPartsInUse(getPartsInUseFromTable(),
              campaign.getSellExcessThreshold());
        updateOverviewPartsInUse();
    }
```

**Step 7: Store spinner value in storePartInUseRequestedStockMap**

In `storePartInUseRequestedStockMap()` (line 469–497), after the `campaign.setTopUpWeekly(...)` line:
```java
        campaign.setIgnoreMothballed(ignoreMothballedCheck.isSelected());
        campaign.setTopUpWeekly(topUpWeeklyCheck.isSelected());
```
Add:
```java
        campaign.setIgnoreMothballed(ignoreMothballedCheck.isSelected());
        campaign.setTopUpWeekly(topUpWeeklyCheck.isSelected());
        campaign.setSellExcessThreshold(((Number) sellExcessThresholdSpinner.getValue()).doubleValue());
```

**Step 8: Commit**
```
git add MekHQ/src/mekhq/gui/dialog/PartsReportDialog.java
git commit -m "feat: add sell excess parts button and threshold spinner to PartsReportDialog"
```

---

### Task 5: PartsReportDialog.properties — add strings

**Files:**
- Modify: `MekHQ/resources/mekhq/resources/PartsReportDialog.properties`

**Step 1: Add the two new strings**

After the last line (`lblIgnoreSparesUnderQuality.text=Ignore Spare Parts Under Quality`), add:
```properties
sellExcessBtn.text=Sell Excess Parts
lblSellExcessThreshold.text=Sell above:
```

**Step 2: Commit**
```
git add MekHQ/resources/mekhq/resources/PartsReportDialog.properties
git commit -m "feat: add sell excess parts UI strings"
```

---

### Task 6: Build and verify

**Step 1: Run build from repo root**
```powershell
.\tools\build-mekhq.ps1
```
Expected: BUILD SUCCESSFUL

**Step 2: Manual test checklist**
- Open Parts in Use report — spinner shows 200, sell button visible
- Change spinner to 100, click "Sell Excess Parts" — sells down exactly to minimums
- Change spinner to 0 — sells all spares
- Close dialog, reopen — spinner remembers last value
- Save campaign, reload — spinner value survives the save/load cycle

**Step 3: Commit build result (submodule pin update, not source)**

Once verified, merge to `serelon/personal` and rebuild from there per the standard workflow in CLAUDE.md.
