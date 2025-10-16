# 08. Luồng Hoạt Động Chính (Main Flow)

## 📍 Tổng Quan

File này mô tả chi tiết luồng hoạt động của EA từ khi khởi động đến khi đóng position.

---

## 1️⃣ OnInit() - Khởi Tạo

```cpp
int OnInit() {
    // STEP 1: Print banner & config
    Print("SMC/ICT EA v1.2 - Initialization");
    Print("Symbol: ", _Symbol);
    Print("Timeframe: ", _Period);
    Print("Risk per trade: ", InpRiskPerTradePct, "%");
    
    // STEP 2: Initialize Detector
    g_detector = new CDetector();
    g_detector.Init(_Symbol, _Period, ...all detection params...);
    
    // STEP 3: Initialize Arbiter
    g_arbiter = new CArbiter();
    g_arbiter.Init(InpMinRR, InpOB_MaxTouches);
    
    // STEP 4: Initialize Executor
    g_executor = new CExecutor();
    g_executor.Init(_Symbol, _Period, ...execution params...);
    
    // STEP 5: Initialize Risk Manager
    g_riskMgr = new CRiskManager();
    g_riskMgr.Init(_Symbol, ...risk params...);
    g_riskMgr.SetLotSizingParams(...);
    g_riskMgr.SetDCALevels(...);
    
    // STEP 6: Initialize Stats & Dashboard
    g_stats = new CStatsManager();
    g_stats.Init(_Symbol, 500);
    
    if(InpShowDebugDraw || InpShowDashboard) {
        g_drawer = new CDrawDebug();
        g_drawer.Init("SMC");
    }
    
    // STEP 7: Initialize state
    g_lastBOS.valid = false;
    g_lastSweep.valid = false;
    g_lastOB.valid = false;
    g_lastFVG.valid = false;
    g_lastMomo.valid = false;
    g_lastCandidate.valid = false;
    
    g_lastBarTime = iTime(_Symbol, _Period, 0);
    g_lastOrderTime = 0;
    
    Print("Initialization completed successfully");
    return INIT_SUCCEEDED;
}
```

---

## 2️⃣ OnTick() - Mỗi Tick

### 📊 Overview Flow

```
OnTick()
  │
  ├─► Check New Bar
  │
  ├─► Pre-Checks
  │   ├─ Session Open?
  │   ├─ Spread OK?
  │   ├─ MDD Halted?
  │   └─ Rollover Time?
  │
  ├─► Update Price Series
  │
  ├─► Run Detectors
  │   ├─ DetectBOS()
  │   ├─ DetectSweep()
  │   ├─ FindOB()
  │   ├─ FindFVG()
  │   ├─ DetectMomentum()
  │   └─ GetMTFBias()
  │
  ├─► Build & Score Candidate
  │   ├─ BuildCandidate()
  │   └─ ScoreCandidate()
  │
  ├─► If Score >= 100
  │   ├─ GetTriggerCandle()
  │   ├─ CalculateEntry()
  │   ├─ Check Lot Limits
  │   ├─ Check Existing Positions
  │   └─ PlaceStopOrder()
  │
  ├─► Manage Existing Positions
  │   ├─ Breakeven
  │   ├─ Trailing
  │   └─ DCA
  │
  ├─► Manage Pending Orders (TTL)
  │
  └─► Update Dashboard
```

### 📝 Detailed Code Flow

```cpp
void OnTick() {
    // ═══════════════════════════════════════════════════════
    // STEP 1: Check for new bar
    // ═══════════════════════════════════════════════════════
    datetime currentBarTime = iTime(_Symbol, _Period, 0);
    bool newBar = (currentBarTime != g_lastBarTime);
    if(newBar) {
        g_lastBarTime = currentBarTime;
    }
    
    // ═══════════════════════════════════════════════════════
    // STEP 2: Pre-checks (critical filters)
    // ═══════════════════════════════════════════════════════
    
    // 2.1 Check session (supports both FULL DAY and MULTI-WINDOW modes)
    if(!g_executor.SessionOpen()) {
        // ⚠️ CRITICAL: Still manage existing positions outside session!
        // Reason: Position mở trong Window 1 có thể đóng trong Window 3
        g_riskMgr.ManageOpenPositions();
        g_executor.ManagePendingOrders();
        
        // Update dashboard to show session status
        if(InpShowDashboard && g_drawer != NULL) {
            string sessionInfo = g_executor.GetActiveWindow();
            g_drawer.UpdateDashboard("OUTSIDE SESSION - " + sessionInfo, ...);
        }
        return;
    }
    
    // 2.2 Check spread
    if(!g_executor.SpreadOK()) {
        if(InpShowDashboard && g_drawer != NULL) {
            g_drawer.UpdateDashboard("SPREAD TOO WIDE", ...);
        }
        g_riskMgr.ManageOpenPositions();
        g_executor.ManagePendingOrders();
        return;
    }
    
    // 2.3 Check MDD
    if(g_riskMgr.IsTradingHalted()) {
        if(InpShowDashboard && g_drawer != NULL) {
            g_drawer.UpdateDashboard("TRADING HALTED - MDD", ...);
        }
        return;
    }
    
    // 2.4 Check rollover
    if(g_executor.IsRolloverTime()) {
        return; // Don't trade during rollover
    }
    
    // ═══════════════════════════════════════════════════════
    // STEP 3: Update price series
    // ═══════════════════════════════════════════════════════
    g_detector.UpdateSeries();
    
    // ═══════════════════════════════════════════════════════
    // STEP 4: Run detectors (on new bar or if invalid)
    // ═══════════════════════════════════════════════════════
    
    if(newBar || !g_lastBOS.valid) {
        g_lastBOS = g_detector.DetectBOS();
        if(g_lastBOS.valid && InpShowDebugDraw && g_drawer != NULL) {
            g_drawer.MarkBOS(0, g_lastBOS.direction, 
                           g_lastBOS.breakLevel, TimeToString(TimeCurrent()));
        }
    }
    
    if(newBar || !g_lastSweep.valid) {
        g_lastSweep = g_detector.DetectSweep();
        if(g_lastSweep.detected && InpShowDebugDraw && g_drawer != NULL) {
            g_drawer.MarkSweep(g_lastSweep.level, g_lastSweep.side,
                             g_lastSweep.time, TimeToString(TimeCurrent()));
        }
    }
    
    // Get OB and FVG based on BOS direction
    if(g_lastBOS.valid) {
        g_lastOB = g_detector.FindOB(g_lastBOS.direction);
        if(g_lastOB.valid && InpShowDebugDraw && g_drawer != NULL && newBar) {
            g_drawer.DrawOB(g_lastOB.priceTop, g_lastOB.priceBottom,
                          g_lastOB.direction, g_lastOB.createdTime,
                          TimeToString(TimeCurrent()));
        }
        
        g_lastFVG = g_detector.FindFVG(g_lastBOS.direction);
        if(g_lastFVG.valid && InpShowDebugDraw && g_drawer != NULL && newBar) {
            g_drawer.DrawFVG(g_lastFVG.priceTop, g_lastFVG.priceBottom,
                           g_lastFVG.direction, g_lastFVG.state,
                           g_lastFVG.createdTime, TimeToString(TimeCurrent()));
        }
    }
    
    if(newBar) {
        g_lastMomo = g_detector.DetectMomentum();
    }
    
    // Get MTF bias
    int mtfBias = g_detector.GetMTFBias();
    
    // ═══════════════════════════════════════════════════════
    // STEP 5: Build candidate
    // ═══════════════════════════════════════════════════════
    g_lastCandidate = g_arbiter.BuildCandidate(
        g_lastBOS, g_lastSweep, g_lastOB, g_lastFVG, g_lastMomo,
        mtfBias, g_executor.SessionOpen(), g_executor.SpreadOK()
    );
    
    // ═══════════════════════════════════════════════════════
    // STEP 6: Score and validate
    // ═══════════════════════════════════════════════════════
    if(g_lastCandidate.valid) {
        double score = g_arbiter.ScoreCandidate(g_lastCandidate);
        g_lastCandidate.score = score;
        
        // Check if score meets threshold
        if(score >= 100.0) {
            // ═══════════════════════════════════════════════
            // STEP 7: Look for trigger candle
            // ═══════════════════════════════════════════════
            double triggerHigh, triggerLow;
            if(g_executor.GetTriggerCandle(g_lastCandidate.direction,
                                          triggerHigh, triggerLow)) {
                
                // ═══════════════════════════════════════════
                // STEP 8: Calculate entry, SL, TP
                // ═══════════════════════════════════════════
                double entry, sl, tp, rr;
                if(g_executor.CalculateEntry(g_lastCandidate,
                                            triggerHigh, triggerLow,
                                            entry, sl, tp, rr)) {
                    
                    // ═══════════════════════════════════════
                    // STEP 9: Calculate position size
                    // ═══════════════════════════════════════
                    double slDistance = MathAbs(entry - sl) / _Point;
                    double lots = g_riskMgr.CalcLotsByRisk(
                        InpRiskPerTradePct, slDistance
                    );
                    
                    // ═══════════════════════════════════════
                    // STEP 10: Check lot limits
                    // ═══════════════════════════════════════
                    double maxLotAllowed = g_riskMgr.GetMaxLotPerSide();
                    if(lots > maxLotAllowed) {
                        lots = maxLotAllowed;
                        Print("⚠ Lot capped to MaxLotPerSide: ", maxLotAllowed);
                    }
                    
                    // ═══════════════════════════════════════
                    // STEP 11: Check existing positions/orders
                    // ═══════════════════════════════════════
                    int existingPositions = 0;
                    int existingPendingOrders = 0;
                    
                    // Count positions in same direction
                    for(int i = 0; i < PositionsTotal(); i++) {
                        ulong ticket = PositionGetTicket(i);
                        if(ticket == 0) continue;
                        if(!PositionSelectByTicket(ticket)) continue;
                        
                        string sym = PositionGetString(POSITION_SYMBOL);
                        if(sym != _Symbol) continue;
                        
                        long posType = PositionGetInteger(POSITION_TYPE);
                        if((g_lastCandidate.direction == 1 && 
                            posType == POSITION_TYPE_BUY) ||
                           (g_lastCandidate.direction == -1 && 
                            posType == POSITION_TYPE_SELL)) {
                            existingPositions++;
                        }
                    }
                    
                    // Count pending orders in same direction
                    for(int i = 0; i < OrdersTotal(); i++) {
                        ulong ticket = OrderGetTicket(i);
                        if(ticket == 0) continue;
                        
                        string sym = OrderGetString(ORDER_SYMBOL);
                        if(sym != _Symbol) continue;
                        
                        long orderType = OrderGetInteger(ORDER_TYPE);
                        if((g_lastCandidate.direction == 1 && 
                            (orderType == ORDER_TYPE_BUY_STOP ||
                             orderType == ORDER_TYPE_BUY_LIMIT)) ||
                           (g_lastCandidate.direction == -1 &&
                            (orderType == ORDER_TYPE_SELL_STOP ||
                             orderType == ORDER_TYPE_SELL_LIMIT))) {
                            existingPendingOrders++;
                        }
                    }
                    
                    // ═══════════════════════════════════════
                    // STEP 12: One-trade-per-bar protection
                    // ═══════════════════════════════════════
                    bool alreadyTradedThisBar = 
                        (g_lastOrderTime == currentBarTime);
                    
                    // ═══════════════════════════════════════
                    // STEP 13: Place order if all checks pass
                    // ═══════════════════════════════════════
                    if(existingPositions == 0 && 
                       existingPendingOrders == 0 && 
                       !alreadyTradedThisBar) {
                        
                        string comment = StringFormat("SMC_%s_RR%.1f",
                            g_lastCandidate.direction == 1 ? "BUY" : "SELL",
                            rr);
                        
                        if(g_executor.PlaceStopOrder(
                            g_lastCandidate.direction,
                            entry, sl, tp, lots, comment)) {
                            
                            g_totalTrades++;
                            g_lastOrderTime = currentBarTime;
                            
                            // Determine pattern type
                            int patternType = GetPatternType(g_lastCandidate);
                            string patternName = 
                                g_stats.GetPatternName(patternType);
                            
                            Print("═══════════════════════════════════");
                            Print("TRADE #", g_totalTrades, " PLACED");
                            Print("Direction: ", 
                                  g_lastCandidate.direction == 1 ? "BUY" : "SELL");
                            Print("Pattern: ", patternName);
                            Print("Entry: ", entry);
                            Print("SL: ", sl);
                            Print("TP: ", tp);
                            Print("R:R: ", DoubleToString(rr, 2));
                            Print("Lots: ", lots);
                            Print("Score: ", score);
                            Print("═══════════════════════════════════");
                        }
                    } else {
                        // Log why order was skipped (once per bar)
                        if(g_lastSkipLogTime != currentBarTime) {
                            g_lastSkipLogTime = currentBarTime;
                            
                            if(existingPositions > 0) {
                                Print("⊘ Entry skipped: Already have ",
                                      existingPositions, " position(s)");
                            }
                            if(existingPendingOrders > 0) {
                                Print("⊘ Entry skipped: Already have ",
                                      existingPendingOrders, " pending order(s)");
                            }
                            if(alreadyTradedThisBar) {
                                Print("⊘ Entry skipped: One-trade-per-bar");
                            }
                        }
                    }
                }
            }
        }
    }
    
    // ═══════════════════════════════════════════════════════
    // STEP 14: Manage existing positions (BE, Trail, DCA)
    // ═══════════════════════════════════════════════════════
    g_riskMgr.ManageOpenPositions();
    
    // ═══════════════════════════════════════════════════════
    // STEP 15: Manage pending orders (TTL)
    // ═══════════════════════════════════════════════════════
    g_executor.ManagePendingOrders();
    
    // ═══════════════════════════════════════════════════════
    // STEP 16: Update dashboard
    // ═══════════════════════════════════════════════════════
    if(InpShowDashboard && g_drawer != NULL) {
        string status = "SCANNING";
        if(g_riskMgr.IsTradingHalted()) {
            status = "TRADING HALTED - Daily MDD";
        } else if(g_lastCandidate.valid && 
                  g_lastCandidate.score >= 100.0) {
            status = "SIGNAL DETECTED";
        } else if(g_lastBOS.valid) {
            status = "BOS DETECTED - Waiting";
        } else if(!g_executor.SessionOpen()) {
            status = "OUTSIDE SESSION";
        } else if(!g_executor.SpreadOK()) {
            status = "SPREAD TOO WIDE";
        }
        
        double score = g_lastCandidate.valid ? 
                      g_lastCandidate.score : 0;
        
        g_drawer.UpdateDashboard(status, g_riskMgr, g_executor,
                                g_detector, g_lastBOS, g_lastSweep,
                                g_lastOB, g_lastFVG, score, g_stats);
        
        // Cleanup old objects periodically
        if(newBar && g_totalTrades % 10 == 0) {
            g_drawer.CleanupOldObjects();
        }
    }
}
```

---

## 3️⃣ OnTrade() - Khi Có Giao Dịch

```cpp
void OnTrade() {
    // ═══════════════════════════════════════════════════════
    // PART 1: Track new filled positions
    // ═══════════════════════════════════════════════════════
    for(int i = 0; i < PositionsTotal(); i++) {
        ulong ticket = PositionGetTicket(i);
        if(PositionSelectByTicket(ticket)) {
            if(PositionGetString(POSITION_SYMBOL) == _Symbol) {
                string comment = PositionGetString(POSITION_COMMENT);
                
                // Skip DCA positions (already tracked via original)
                if(StringFind(comment, "DCA Add-on") >= 0) {
                    continue;
                }
                
                double entry = PositionGetDouble(POSITION_PRICE_OPEN);
                double sl = PositionGetDouble(POSITION_SL);
                double tp = PositionGetDouble(POSITION_TP);
                double lots = PositionGetDouble(POSITION_VOLUME);
                
                // Track in risk manager (checks for duplicates)
                g_riskMgr.TrackPosition(ticket, entry, sl, tp, lots);
                
                // Record in stats
                int direction = (int)PositionGetInteger(POSITION_TYPE);
                direction = (direction == POSITION_TYPE_BUY) ? 1 : -1;
                int patternType = GetPatternType(g_lastCandidate);
                g_stats.RecordTrade(ticket, direction, entry, lots,
                                   patternType, sl, tp);
            }
        }
    }
    
    // ═══════════════════════════════════════════════════════
    // PART 2: Update stats for closed positions
    // ═══════════════════════════════════════════════════════
    if(HistorySelect(TimeCurrent() - 86400, TimeCurrent())) {
        for(int i = HistoryDealsTotal() - 1; i >= 0; i--) {
            ulong dealTicket = HistoryDealGetTicket(i);
            if(dealTicket > 0) {
                string symbol = HistoryDealGetString(dealTicket, DEAL_SYMBOL);
                if(symbol == _Symbol) {
                    long dealEntry = 
                        HistoryDealGetInteger(dealTicket, DEAL_ENTRY);
                    if(dealEntry == DEAL_ENTRY_OUT) {
                        ulong posTicket = 
                            HistoryDealGetInteger(dealTicket, DEAL_POSITION_ID);
                        double closePrice = 
                            HistoryDealGetDouble(dealTicket, DEAL_PRICE);
                        double profit = 
                            HistoryDealGetDouble(dealTicket, DEAL_PROFIT);
                        
                        // Update stats
                        g_stats.UpdateClosedTrade(posTicket, closePrice, profit);
                    }
                }
            }
        }
    }
}
```

---

## 4️⃣ Decision Tree

### 🔍 Entry Decision Flow

```
START
  │
  ├─► Session Open?
  │   NO → ManagePositions & Return
  │   YES ↓
  │
  ├─► Spread OK?
  │   NO → ManagePositions & Return
  │   YES ↓
  │
  ├─► Trading Halted (MDD)?
  │   YES → Show Dashboard & Return
  │   NO ↓
  │
  ├─► Rollover Time?
  │   YES → Return
  │   NO ↓
  │
  ├─► Detect Signals
  │   ├─ BOS detected?
  │   ├─ Sweep detected?
  │   ├─ OB found?
  │   ├─ FVG found?
  │   └─ Momentum detected?
  │
  ├─► Build Candidate
  │   Valid? NO → Return
  │   YES ↓
  │
  ├─► Score Candidate
  │   Score >= 100? NO → Return
  │   YES ↓
  │
  ├─► Get Trigger Candle
  │   Found? NO → Return
  │   YES ↓
  │
  ├─► Calculate Entry/SL/TP
  │   RR >= MinRR? NO → Return
  │   YES ↓
  │
  ├─► Calculate Lots
  │   ↓
  │
  ├─► Check Limits
  │   Lots > MaxLot? YES → Cap to MaxLot
  │   NO ↓
  │
  ├─► Check Existing
  │   Has position? YES → Skip
  │   Has pending? YES → Skip
  │   Traded this bar? YES → Skip
  │   NO ↓
  │
  └─► Place Order ✅
```

---

## 5️⃣ State Machine

### 📊 Position Lifecycle

```
┌─────────────────────────────────────────────┐
│                                             │
│  STATE 1: NO POSITION                       │
│  ├─ Scanning for signals                    │
│  └─ Waiting for setup                       │
│                                             │
└────────┬────────────────────────────────────┘
         │ Order Placed
         ▼
┌─────────────────────────────────────────────┐
│                                             │
│  STATE 2: PENDING ORDER                     │
│  ├─ Waiting for fill                        │
│  ├─ TTL countdown                           │
│  └─ Can be cancelled if TTL expires         │
│                                             │
└────────┬────────────────────────────────────┘
         │ Order Filled
         ▼
┌─────────────────────────────────────────────┐
│                                             │
│  STATE 3: OPEN POSITION (Initial)           │
│  ├─ Track in RiskManager                    │
│  ├─ Record in Stats                         │
│  ├─ originalSL saved                        │
│  └─ Monitor profit in R                     │
│                                             │
└────────┬────────────────────────────────────┘
         │ Profit >= 0.75R
         ▼
┌─────────────────────────────────────────────┐
│                                             │
│  STATE 4: DCA LEVEL 1                       │
│  ├─ Add DCA position #1 (0.5× original)     │
│  ├─ Same SL/TP                              │
│  └─ Continue monitoring                     │
│                                             │
└────────┬────────────────────────────────────┘
         │ Profit >= 1.0R
         ▼
┌─────────────────────────────────────────────┐
│                                             │
│  STATE 5: BREAKEVEN                         │
│  ├─ Move SL to entry (all positions)        │
│  ├─ Risk eliminated                         │
│  └─ Continue monitoring                     │
│                                             │
└────────┬────────────────────────────────────┘
         │ Profit >= 1.5R
         ▼
┌─────────────────────────────────────────────┐
│                                             │
│  STATE 6: DCA LEVEL 2 + TRAILING            │
│  ├─ Add DCA position #2 (0.33× original)    │
│  ├─ Start trailing SL                       │
│  ├─ Trail every +0.5R                       │
│  └─ Lock in profits                         │
│                                             │
└────────┬────────────────────────────────────┘
         │ TP Hit or SL Hit
         ▼
┌─────────────────────────────────────────────┐
│                                             │
│  STATE 7: CLOSED                            │
│  ├─ Close all positions in group            │
│  ├─ Calculate total profit                  │
│  ├─ Update stats (Win/Loss)                 │
│  └─ Remove from tracking                    │
│                                             │
└────────┬────────────────────────────────────┘
         │
         └──► Back to STATE 1 (No Position)
```

---

## 6️⃣ Timing Diagram

### 📅 Daily Cycle

#### Full Day Mode

```
00:00 ─────────────────────────────────────────
       │ Avoid trading (Rollover time)
       │
06:00 ─┤
       │ Daily Reset:
       │ - Reset startDayBalance
       │ - Update MaxLotPerSide
       │ - Resume trading if halted
       │
07:00 ─┤
       │ SESSION START (GMT+7)
       │ ├─ Start scanning
       │ ├─ Detect signals
       │ └─ Place orders
       │
12:00 ─┤
       │ Continue trading (no break)
       │
16:00 ─┤
       │ Continue trading
       │
20:00 ─┤
       │ Continue trading
       │
23:00 ─┤
       │ SESSION END (GMT+7)
       │ ├─ Stop new entries
       │ └─ Still manage existing positions
       │
00:00 ─┘ (Next day)
```

#### Multi-Window Mode

```
00:00 ─────────────────────────────────────────
       │ Rollover (avoid trading)
       │
06:00 ─┤
       │ Daily Reset:
       │ - Reset startDayBalance
       │ - Update MaxLotPerSide
       │ - Resume trading if halted
       │
07:00 ─┤
       │ ┌─────────────────────────────────────
       │ │ WINDOW 1: ASIA SESSION
       │ │ ├─ Start scanning
       │ │ ├─ Detect signals
       │ │ └─ Place orders
       │ │
11:00 ─┤ └───────────────────────────────────── Window 1 END
       │ 
       │ ⊘ BREAK PERIOD (11:00-12:00)
       │ ├─ Stop scanning signals
       │ ├─ No new orders
       │ └─ Still manage existing positions ✅
       │
12:00 ─┤
       │ ┌─────────────────────────────────────
       │ │ WINDOW 2: LONDON SESSION
       │ │ ├─ Resume scanning
       │ │ ├─ Detect signals
       │ │ └─ Place orders
       │ │
16:00 ─┤ └───────────────────────────────────── Window 2 END
       │
       │ ⊘ BREAK PERIOD (16:00-18:00)
       │ ├─ Stop scanning signals
       │ ├─ No new orders
       │ └─ Still manage existing positions ✅
       │
18:00 ─┤
       │ ┌─────────────────────────────────────
       │ │ WINDOW 3: NY SESSION
       │ │ ├─ Resume scanning
       │ │ ├─ Detect signals
       │ │ └─ Place orders
       │ │
23:00 ─┤ └───────────────────────────────────── Window 3 END
       │
       │ CLOSED (23:00-07:00)
       │ ├─ Stop new entries
       │ └─ Still manage existing positions ✅
       │
00:00 ─┘ (Next day)
```

**Key Difference**:
- **Full Day**: Scan liên tục 16 hours
- **Multi-Window**: Scan chỉ trong windows (13h total), có 2 break periods

**Position Management**: Luôn chạy 24/7 trong cả 2 modes! ✅

### ⏱️ M15 Bar Cycle

```
BAR N-1 Close
  │
  ├─► OnTick() called
  │   ├─ newBar = false
  │   ├─ Manage existing positions
  │   └─ Update dashboard
  │
  ├─► OnTick() called
  │   └─ (repeated every tick)
  │
BAR N Open (New Bar!)
  │
  ├─► OnTick() called
  │   ├─ newBar = true ✅
  │   ├─ Run detectors
  │   ├─ Build candidate
  │   ├─ Score
  │   └─ Place order (if valid)
  │
  ├─► OnTick() called
  │   ├─ newBar = false
  │   └─ Continue managing
  │
BAR N Close
  │
  └─► (Cycle repeats)
```

---

## 🎓 Key Points

### ✅ Best Practices
1. **New Bar Only**: Detectors run on new bar to avoid spam
2. **Pre-checks First**: Session, Spread, MDD checked before scanning
3. **One Trade Per Bar**: Prevents over-trading
4. **Position Management**: Always runs, even outside session
5. **Dashboard Always Updated**: Real-time monitoring

### ⚠️ Common Pitfalls
1. **Don't forget ManagePositions()** even when skipping entry
2. **Track DCA separately** - don't track them as new positions
3. **Use originalSL for R calc** - not current SL after BE/Trail
4. **Check for duplicates** before tracking position
5. **Update dashboard** in all return paths

### 📈 Performance Tips
1. Run heavy calculations (detectors) only on new bar
2. Cache signals between ticks
3. Cleanup old objects periodically
4. Use early returns to skip unnecessary processing
5. Log important events but avoid spam

---

---

## 🆕 v2.0 Updates: Enhanced Main Flow

### Updated OnTick() Flow

```cpp
void OnTick() {
    // ═══════════════════════════════════════════════════════
    // STEP 1: Check for new bar
    // ═══════════════════════════════════════════════════════
    datetime currentBarTime = iTime(_Symbol, _Period, 0);
    bool newBar = (currentBarTime != g_lastBarTime);
    if(newBar) g_lastBarTime = currentBarTime;
    
    // ═══════════════════════════════════════════════════════
    // STEP 2: Pre-checks (UPDATED with News & Regime)
    // ═══════════════════════════════════════════════════════
    
    if(!g_executor.SessionOpen()) {
        g_riskMgr.ManageOpenPositions();
        g_executor.ManagePendingOrders();
        return;
    }
    
    if(!g_executor.SpreadOK()) {
        g_riskMgr.ManageOpenPositions();
        g_executor.ManagePendingOrders();
        return;
    }
    
    if(g_riskMgr.IsTradingHalted()) return;
    
    if(g_executor.IsRolloverTime()) return;
    
    // ═══════════════════════════════════════════════════════
    // NEW: News Embargo Check
    // ═══════════════════════════════════════════════════════
    if(g_newsFilter != NULL && 
       g_newsFilter.IsWithinNewsWindow(TimeCurrent())) {
        g_riskMgr.ManageOpenPositions();  // Still manage existing
        g_executor.ManagePendingOrders();
        
        if(InpShowDashboard && g_drawer != NULL) {
            g_drawer.UpdateDashboard("NEWS EMBARGO", ...);
        }
        return;
    }
    
    // ═══════════════════════════════════════════════════════
    // NEW: Detect Volatility Regime (once per bar)
    // ═══════════════════════════════════════════════════════
    static ENUM_REGIME g_currentRegime = REGIME_MID;
    if(newBar && InpRegimeEnable && g_regime != NULL) {
        g_currentRegime = g_regime.DetectRegime();
        Print("📊 Regime: ", GetRegimeName(g_currentRegime),
              " | ATR: ", DoubleToString(g_regime.GetCurrentATR(), 2));
    }
    
    // ═══════════════════════════════════════════════════════
    // NEW: Risk Overlay Checks
    // ═══════════════════════════════════════════════════════
    if(!g_riskMgr.CanOpenNewTrade()) {
        g_riskMgr.ManageOpenPositions(g_currentRegime);
        g_executor.ManagePendingOrders();
        return;
    }
    
    // ═══════════════════════════════════════════════════════
    // STEP 3: Update price series
    // ═══════════════════════════════════════════════════════
    g_detector.UpdateSeries();
    
    // ═══════════════════════════════════════════════════════
    // STEP 4: Run detectors
    // ═══════════════════════════════════════════════════════
    if(newBar || !g_lastBOS.valid) {
        g_lastBOS = g_detector.DetectBOS();
        // ... visualization ...
    }
    
    if(newBar || !g_lastSweep.valid) {
        g_lastSweep = g_detector.DetectSweep();
        
        // NEW: Calculate sweep proximity in ATR
        if(g_lastSweep.detected && g_currentRegime != NULL) {
            double atr = g_regime.GetCurrentATR();
            double distance = MathAbs(SymbolInfoDouble(_Symbol, SYMBOL_BID) -
                                     g_lastSweep.level);
            g_lastSweep.proximityATR = distance / atr;
        }
    }
    
    // ... other detectors ...
    
    // NEW: Get MTF bias & check HTF confluence
    int mtfBias = g_detector.GetMTFBias();
    
    // ═══════════════════════════════════════════════════════
    // STEP 5: Build & Score candidate (UPDATED)
    // ═══════════════════════════════════════════════════════
    g_lastCandidate = g_arbiter.BuildCandidate(g_lastBOS, g_lastSweep,
                                               g_lastOB, g_lastFVG, 
                                               g_lastMomo, mtfBias,
                                               g_executor.SessionOpen(),
                                               g_executor.SpreadOK());
    
    if(g_lastCandidate.valid) {
        // NEW: Check HTF confluence
        g_arbiter.CheckHTFConfluence(g_lastCandidate);
        
        // NEW: Extended scoring with regime & time
        double score = g_arbiter.ScoreCandidateExtended(
            g_lastCandidate,
            g_currentRegime,
            GetLocalHour(),
            GetLocalMin()
        );
        
        g_lastCandidate.score = score;
        
        // Check threshold
        if(score >= InpScoreEnter) {
            // ═══════════════════════════════════════════════
            // STEP 6: Look for trigger (regime-adaptive)
            // ═══════════════════════════════════════════════
            double triggerHigh, triggerLow;
            if(g_executor.GetTriggerCandle(g_lastCandidate.direction,
                                          g_currentRegime,  // NEW
                                          triggerHigh, triggerLow)) {
                
                // ═══════════════════════════════════════════
                // STEP 7: Calculate entry (regime-adaptive)
                // ═══════════════════════════════════════════
                double entry, sl, tp, rr;
                if(g_executor.CalculateEntry(g_lastCandidate,
                                            g_currentRegime,  // NEW
                                            triggerHigh, triggerLow,
                                            entry, sl, tp, rr)) {
                    
                    // ... lot sizing ...
                    // ... existing position checks ...
                    
                    if(g_executor.PlaceStopOrder(...)) {
                        g_totalTrades++;
                        g_riskMgr.OnTradeOpened();  // NEW: Update counter
                        
                        // ... logging ...
                    }
                }
            }
        }
    }
    
    // ═══════════════════════════════════════════════════════
    // STEP 8: Manage existing positions (regime-adaptive)
    // ═══════════════════════════════════════════════════════
    g_riskMgr.ManageOpenPositions(g_currentRegime);  // NEW: Pass regime
    
    // STEP 9: Manage pending orders
    g_executor.ManagePendingOrders();
    
    // STEP 10: Update dashboard
    if(InpShowDashboard && g_drawer != NULL) {
        string status = DetermineStatus();  // ... logic ...
        g_drawer.UpdateDashboard(status, g_riskMgr, g_executor,
                                g_detector, g_lastBOS, g_lastSweep,
                                g_lastOB, g_lastFVG, 
                                g_lastCandidate.score,
                                g_stats,
                                g_currentRegime);  // NEW: Pass regime
    }
}
```

---

### Updated OnTrade() Flow

```cpp
void OnTrade() {
    // ═══════════════════════════════════════════════════════
    // PART 1: Track new filled positions
    // ═══════════════════════════════════════════════════════
    for(int i = 0; i < PositionsTotal(); i++) {
        ulong ticket = PositionGetTicket(i);
        if(PositionSelectByTicket(ticket)) {
            if(PositionGetString(POSITION_SYMBOL) == _Symbol) {
                string comment = PositionGetString(POSITION_COMMENT);
                
                if(StringFind(comment, "DCA Add-on") >= 0) {
                    continue;  // Skip DCA tracking
                }
                
                // ... get entry, sl, tp, lots ...
                
                g_riskMgr.TrackPosition(ticket, entry, sl, tp, lots);
                
                int direction = (int)PositionGetInteger(POSITION_TYPE);
                direction = (direction == POSITION_TYPE_BUY) ? 1 : -1;
                int patternType = GetPatternType(g_lastCandidate);
                
                // NEW: Record with regime context
                g_stats.RecordTrade(ticket, direction, entry, lots,
                                   patternType, sl, tp, g_currentRegime);
            }
        }
    }
    
    // ═══════════════════════════════════════════════════════
    // PART 2: Update stats for closed positions
    // ═══════════════════════════════════════════════════════
    if(HistorySelect(TimeCurrent() - 86400, TimeCurrent())) {
        for(int i = HistoryDealsTotal() - 1; i >= 0; i--) {
            ulong dealTicket = HistoryDealGetTicket(i);
            if(dealTicket > 0) {
                string symbol = HistoryDealGetString(dealTicket, DEAL_SYMBOL);
                if(symbol == _Symbol) {
                    long dealEntry = HistoryDealGetInteger(dealTicket, DEAL_ENTRY);
                    if(dealEntry == DEAL_ENTRY_OUT) {
                        ulong posTicket = HistoryDealGetInteger(dealTicket, 
                                                               DEAL_POSITION_ID);
                        double closePrice = HistoryDealGetDouble(dealTicket, 
                                                                 DEAL_PRICE);
                        double profit = HistoryDealGetDouble(dealTicket, 
                                                            DEAL_PROFIT);
                        
                        g_stats.UpdateClosedTrade(posTicket, closePrice, profit);
                        
                        // NEW: Update risk overlays (streak, cooldown)
                        bool isWin = (profit > 0);
                        g_riskMgr.OnTradeClose(isWin, profit);
                    }
                }
            }
        }
    }
}
```

---

### Updated Decision Tree (v2.0)

```
START
  │
  ├─► Session Open?
  │   NO → ManagePositions & Return
  │   YES ↓
  │
  ├─► Spread OK?
  │   NO → ManagePositions & Return
  │   YES ↓
  │
  ├─► Trading Halted (MDD)?
  │   YES → Dashboard & Return
  │   NO ↓
  │
  ├─► Rollover Time?
  │   YES → Return
  │   NO ↓
  │
  ├─► 🆕 News Window?
  │   YES → ManagePositions & Return (skip new entries)
  │   NO ↓
  │
  ├─► 🆕 Detect Regime (new bar)
  │   → LOW/MID/HIGH
  │   → Apply tuning
  │
  ├─► 🆕 Risk Overlays Check
  │   ├─ MaxTradesPerDay reached?
  │   ├─ In Cooldown?
  │   └─ MaxConsecLoss reached?
  │   ANY YES → ManagePositions & Return
  │   ALL NO ↓
  │
  ├─► Detect Signals
  │   ├─ BOS detected?
  │   ├─ Sweep detected? → Calculate proximityATR
  │   ├─ OB found?
  │   ├─ FVG found?
  │   └─ Momentum detected?
  │
  ├─► Build Candidate
  │   Valid? NO → Return
  │   YES ↓
  │   → 🆕 Check HTF Confluence
  │
  ├─► 🆕 Score Candidate (Extended)
  │   → Pass regime, localHour, localMin
  │   Score >= 100? NO → Return
  │   YES ↓
  │
  ├─► 🆕 Get Trigger Candle (regime-adaptive threshold)
  │   Found? NO → Return
  │   YES ↓
  │
  ├─► 🆕 Calculate Entry/SL/TP (ATR-scaled)
  │   → Buffer, MinStop scaled by ATR & regime
  │   RR >= MinRR? NO → Return
  │   YES ↓
  │
  ├─► Calculate Lots
  │   ↓
  │
  ├─► Check Limits
  │   ↓
  │
  ├─► Check Existing
  │   ↓
  │
  ├─► Place Order
  │   → 🆕 Set TTL by regime
  │   → 🆕 Increment todayTrades
  │   ✅
  │
  └─► 🆕 Manage Positions (regime-adaptive DCA/Trail)
```

---

### New State Machine (v2.0)

```
┌─────────────────────────────────────────────┐
│  STATE 0: COOLDOWN                          │
│  ├─ After 3 consecutive losses              │
│  ├─ Wait X minutes                          │
│  └─ Need a win to reset                     │
└────────┬────────────────────────────────────┘
         │ Cooldown expired + Win
         ▼
┌─────────────────────────────────────────────┐
│  STATE 1: NO POSITION (Enhanced)            │
│  ├─ Check News Window                       │
│  ├─ Detect Regime                           │
│  ├─ Check MaxTrades/Day                     │
│  └─ Scanning for signals                    │
└────────┬────────────────────────────────────┘
         │ Order Placed (regime-tuned)
         ▼
┌─────────────────────────────────────────────┐
│  STATE 2: PENDING ORDER                     │
│  ├─ TTL by regime (10/16/24 bars)           │
│  ├─ Cancel if expired                       │
│  └─ Waiting for fill                        │
└────────┬────────────────────────────────────┘
         │ Order Filled
         ▼
┌─────────────────────────────────────────────┐
│  STATE 3: OPEN POSITION                     │
│  ├─ Track with regime context               │
│  ├─ Monitor profit in R                     │
│  └─ Regime: MID                             │
└────────┬────────────────────────────────────┘
         │ Profit >= DcaLevel1 (by regime)
         ▼
┌─────────────────────────────────────────────┐
│  STATE 4: DCA LEVEL 1                       │
│  ├─ Add DCA position (size by regime)       │
│  ├─ LOW: +0.75R (0.50×)                     │
│  ├─ MID: +0.90R (0.45×)                     │
│  └─ HIGH: +1.00R (0.33×)                    │
└────────┬────────────────────────────────────┘
         │ Profit >= BeLevel (+1.0R)
         ▼
┌─────────────────────────────────────────────┐
│  STATE 5: BREAKEVEN                         │
│  ├─ Move all SLs to entry                   │
│  └─ Risk eliminated                         │
└────────┬────────────────────────────────────┘
         │ Profit >= TrailStart (by regime)
         ▼
┌─────────────────────────────────────────────┐
│  STATE 6: TRAILING (Regime-Adaptive)        │
│  ├─ Start: LOW +1.0R / MID +1.2R / HIGH +1.5R│
│  ├─ Step: LOW 0.6R / MID 0.5R / HIGH 0.3R   │
│  ├─ Distance: LOW 2×ATR / MID 2.5× / HIGH 3×│
│  └─ Lock profits progressively              │
└────────┬────────────────────────────────────┘
         │ TP Hit or SL Hit
         ▼
┌─────────────────────────────────────────────┐
│  STATE 7: CLOSED                            │
│  ├─ Calculate total profit                  │
│  ├─ Update stats (pattern + regime)         │
│  ├─ Update streak counter                   │
│  └─ Check cooldown trigger                  │
└────────┬────────────────────────────────────┘
         │ If LOSS streak >=3 → STATE 0 (Cooldown)
         │ Else → STATE 1 (No Position)
         └───────────────────────────────────────►
```

---

### Updated Daily Cycle (v2.0)

```
00:00 ─────────────────────────────────────────
       │ Rollover (avoid trading)
       │
06:00 ─┤
       │ Daily Reset:
       │ - Reset startDayBalance
       │ - Update MaxLotPerSide
       │ - 🆕 Reset todayTrades = 0
       │ - 🆕 Reset consecLoss = 0
       │ - Resume trading if halted
       │
07:00 ─┤
       │ SESSION START (GMT+7)
       │ ├─ 🆕 Check News Calendar
       │ ├─ 🆕 Detect Regime
       │ ├─ 🆕 Check Risk Overlays
       │ └─ Start scanning
       │
13:00 ─┤
       │ 🆕 LONDON WINDOW (+10 score bonus)
       │ ├─ High quality setups
       │ └─ Increased TTL (+4 bars)
       │
17:00 ─┤
       │ London window ends
       │
19:30 ─┤
       │ 🆕 NY OVERLAP (+8 score bonus)
       │ ├─ Maximum volatility
       │ └─ Best trending setups
       │
22:30 ─┤
       │ NY window ends
       │
23:00 ─┤
       │ SESSION END (GMT+7)
       │ ├─ Stop new entries
       │ └─ Still manage existing
       │
00:00 ─┘ (Next day)
```

---

### Comparison: v1.2 vs v2.0 Flow

| Step | v1.2 | v2.0 |
|------|------|------|
| **Pre-checks** | Session, Spread, MDD, Rollover | + News Window |
| **Regime** | None | Detect LOW/MID/HIGH (new bar) |
| **Risk Checks** | Only MDD | + MaxTrades/Day, Cooldown, ConsecLoss |
| **Sweep** | Basic detection | + ProximityATR calculation |
| **Scoring** | Basic (100-200) | Extended (with regime, time, HTF) |
| **Trigger** | Fixed 0.30 ATR | Regime-based (0.25/0.30/0.35) |
| **Entry Calc** | Fixed buffers/stops | ATR-scaled by regime |
| **TTL** | Fixed 16 bars | Adaptive (10/16/24 + micro-window boost) |
| **DCA** | Fixed levels | Regime-adaptive levels & sizes |
| **Trailing** | Fixed params | Regime-adaptive start/step/distance |
| **Stats** | Pattern only | Pattern + Regime + Session |

---

## 🎓 Đọc Tiếp

- [09_EXAMPLES.md](09_EXAMPLES.md) - Real trade flow examples
- [05_RISK_MANAGER.md](05_RISK_MANAGER.md) - ManagePositions() details
- [MULTI_SESSION_TRADING.md](MULTI_SESSION_TRADING.md) - Multi-session mode guide
- [TIMEZONE_CONVERSION.md](TIMEZONE_CONVERSION.md) - Timezone conversion details

