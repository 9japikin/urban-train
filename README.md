#property copyright "Lawrence O."
#property link      "https://www.mql5.com"
#property version   "3.00"
#property strict

//==================================================================
// CHANGE LOG (HFM build - converted from HUB RECEIVER to MASTER)
//  This EA now behaves like REAL_315E_Master_V1:
//   - It trades ONLY on its own ZigZag / structure conditions.
//   - After a successful OrderSend() it broadcasts the side ("BUY"/"SELL")
//     and the M1 bar time to REAL_315E_SIGNAL.txt (FILE_COMMON) through
//     REAL315E_SendSignal(), exactly as REAL_315E_Master_V1 does.
//  REMOVED (all receiver / hub code):
//   - sender status-file + signal-file readers, vote counting, consensus gate
//     (IsSafeToOpenNewTrade), one-shot consumption of sender trades,
//     CLOSE_ALL command files written to senders, all sender/hub inputs,
//     ActiveTradeStatus struct, hub sanity checks in OnInit.
//==================================================================

   
//====================================================================
// GLOBAL SNAPSHOT STORAGE MATRIX FOR _GetLowestPointAfterDailyHighM1
//====================================================================
#define MAX_SNAPSHOT_RECORDS_GetLowestPointAfterDailyHigh 500

// Explicitly unique ledger arrays for the standard lower low engine
datetime CapturedSnapshot_Times_GetLowestPointAfterDailyHigh[MAX_SNAPSHOT_RECORDS_GetLowestPointAfterDailyHigh];
double   CapturedSnapshot_SLPrices_GetLowestPointAfterDailyHigh[MAX_SNAPSHOT_RECORDS_GetLowestPointAfterDailyHigh];
int      TotalCapturedSnapshots_GetLowestPointAfterDailyHigh = 0;


//====================================================================
// GLOBAL SNAPSHOT STORAGE MATRIX FOR _GetHighestPointAfterDailyLow
//====================================================================
#define MAX_SNAPSHOT_RECORDS_GetHighestPointAfterDailyLow 500

datetime CapturedSnapshot_Times_GetHighestPointAfterDailyLow[MAX_SNAPSHOT_RECORDS_GetHighestPointAfterDailyLow];
double   CapturedSnapshot_SLPrices_GetHighestPointAfterDailyLow[MAX_SNAPSHOT_RECORDS_GetHighestPointAfterDailyLow];
int      TotalCapturedSnapshots_GetHighestPointAfterDailyLow = 0;


//====================================================================
// FROZEN SNAPSHOT ACCESSORS
// First call for a given M1 bar time locks in the live value into the
// ledger arrays above; every subsequent call for that SAME bar time
// returns the originally-frozen value instead of a fresh (possibly
// drifted) live read. This is what lets SL/TP be computed once per
// bar even as GetLowestPointAfterDailyHigh()/GetHighestPointAfterDailyLow()
// keep changing tick-to-tick.
//====================================================================
double GetLowestPointAfterDailyHigh();
double GetHighestPointAfterDailyLow();

double GetFrozenStopLossSnapshot_GetLowestPointAfterDailyHigh(datetime barTime)
{
   for(int i = 0; i < TotalCapturedSnapshots_GetLowestPointAfterDailyHigh; i++)
   {
      if(CapturedSnapshot_Times_GetLowestPointAfterDailyHigh[i] == barTime)
         return CapturedSnapshot_SLPrices_GetLowestPointAfterDailyHigh[i];
   }

   double liveValue = GetLowestPointAfterDailyHigh();

   if(TotalCapturedSnapshots_GetLowestPointAfterDailyHigh < MAX_SNAPSHOT_RECORDS_GetLowestPointAfterDailyHigh)
   {
      CapturedSnapshot_Times_GetLowestPointAfterDailyHigh[TotalCapturedSnapshots_GetLowestPointAfterDailyHigh]    = barTime;
      CapturedSnapshot_SLPrices_GetLowestPointAfterDailyHigh[TotalCapturedSnapshots_GetLowestPointAfterDailyHigh] = liveValue;
      TotalCapturedSnapshots_GetLowestPointAfterDailyHigh++;
   }

   return liveValue;
}

double GetFrozenStopLossSnapshot_GetHighestPointAfterDailyLow(datetime barTime)
{
   for(int i = 0; i < TotalCapturedSnapshots_GetHighestPointAfterDailyLow; i++)
   {
      if(CapturedSnapshot_Times_GetHighestPointAfterDailyLow[i] == barTime)
         return CapturedSnapshot_SLPrices_GetHighestPointAfterDailyLow[i];
   }

   double liveValue = GetHighestPointAfterDailyLow();

   if(TotalCapturedSnapshots_GetHighestPointAfterDailyLow < MAX_SNAPSHOT_RECORDS_GetHighestPointAfterDailyLow)
   {
      CapturedSnapshot_Times_GetHighestPointAfterDailyLow[TotalCapturedSnapshots_GetHighestPointAfterDailyLow]    = barTime;
      CapturedSnapshot_SLPrices_GetHighestPointAfterDailyLow[TotalCapturedSnapshots_GetHighestPointAfterDailyLow] = liveValue;
      TotalCapturedSnapshots_GetHighestPointAfterDailyLow++;
   }

   return liveValue;
}


//--- Global variables

double g_LastDailyHigh = 0;
double g_LastDailyLow  = 0;
//--- Global variables
datetime g_LastDailyUpdate = 0;



//--- Global variables

input ENUM_TIMEFRAMES TrendTF       = PERIOD_H4;
input int    TrendSMA               = 50;





 // Global scope
bool buyCondMet307 = false;
bool sellCondMet307 = false;
datetime buyCondTime307 = 0;
datetime sellCondTime307 = 0;
double buyCondHigh307 = 0.0;
double buyCondLow307 = 0.0;
double buyCondOpen307 = 0.0;

double sellCondLow307 = 0.0;
double sellCondHigh307 = 0.0;
double sellCondOpen307 = 0.0;


 // Global scope
bool buyCondMet9 = false;
bool sellCondMet9 = false;
datetime buyCondTime9 = 0;
datetime sellCondTime9 = 0;
double buyCondHigh9 = 0.0;
double buyCondLow9 = 0.0;
double buyCondOpen9 = 0.0;

double sellCondLow9 = 0.0;
double sellCondHigh9 = 0.0;
double sellCondOpen9 = 0.0;




 // Global scope
bool buyCondMet308 = false;
bool sellCondMet308 = false;
datetime buyCondTime308 = 0;
datetime sellCondTime308 = 0;
double buyCondHigh308 = 0.0;
double buyCondLow308 = 0.0;
double buyCondOpen308 = 0.0;

double sellCondLow308 = 0.0;
double sellCondHigh308 = 0.0;
double sellCondOpen308 = 0.0;





extern string Remark6="== Zig Zag Settings ==";
extern int ExtDepth=2;
extern int ExtDeviation=1;
extern int ExtBackstep=1;
extern string Remark9="== NZigg Zagg Settings ==";
extern int ExtDept=12;
extern int ExtDeviatn=5;
extern int ExtBakstep=3;


extern int PipBuffer = 0;

//==============================================================
// MASTER COMMUNICATION (OUTPUT)
//==============================================================
string REAL315E_SignalFile = "REAL_315E_SIGNAL.txt";

void REAL315E_SendSignal(string side)
{
   int h = FileOpen(REAL315E_SignalFile, FILE_WRITE|FILE_TXT|FILE_COMMON);

   if(h == INVALID_HANDLE)
   {
      Print("MASTER: Failed to open signal file. Error ", GetLastError());
      return;
   }

   FileWrite(h, side);
   FileWrite(h, TimeToString(iTime(Symbol(), PERIOD_M1, 0), TIME_DATE|TIME_MINUTES));
   FileClose(h);

   Print("MASTER SENT: ", side);
}

// --- Strategy Inputs
extern double LotSize                 = 2.11;
extern double StopLossPadding         = 1.0;
extern double TakeProfitPoints        = 30.0;
extern int    MagicNumber             = 9129951; // HFM master magic - must stay unique on the HFM account






 
//+------------------------------------------------------------------+
//| Modified Helper Function for Centered Labels                     |
//+------------------------------------------------------------------+
void CreateCenteredLabel(string name, string text, color clr, int xDist, int yDist)
  {
   if(ObjectFind(0, name) < 0)
      ObjectCreate(0, name, OBJ_LABEL, 0, 0, 0);

   ObjectSetString(0, name, OBJPROP_TEXT, text);
   ObjectSetString(0, name, OBJPROP_FONT, "Arial Bold");
   ObjectSetInteger(0, name, OBJPROP_FONTSIZE, 14); // Slightly larger for visibility
   ObjectSetInteger(0, name, OBJPROP_COLOR, clr);

// Set corner to Top-Left but use Center Anchor to align the middle of the text
   ObjectSetInteger(0, name, OBJPROP_CORNER, CORNER_LEFT_UPPER);
   ObjectSetInteger(0, name, OBJPROP_ANCHOR, ANCHOR_CENTER);

   ObjectSetInteger(0, name, OBJPROP_XDISTANCE, xDist);
   ObjectSetInteger(0, name, OBJPROP_YDISTANCE, yDist);
   ObjectSetInteger(0, name, OBJPROP_SELECTABLE, false);
   ObjectSetInteger(0, name, OBJPROP_HIDDEN, true);
  }

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//+------------------------------------------------------------------+
//| Trend filter (closed candle)                                     |
//+------------------------------------------------------------------+
bool TrendBullish()
  {
   double ma = iMA(NULL, TrendTF, TrendSMA, 0, MODE_SMA, PRICE_CLOSE, 1);
   double cl = iClose(NULL, TrendTF, 1);
   return (cl > ma);
  }

//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
bool TrendBearish()
  {
   double ma = iMA(NULL, TrendTF, TrendSMA, 0, MODE_SMA, PRICE_CLOSE, 1);
   double cl = iClose(NULL, TrendTF, 1);
   return (cl < ma);
  }



//+------------------------------------------------------------------+
//| Trend filter (closed candle)                                     |

//+------------------------------------------------------------------+

//+------------------------------------------------------------------+


//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
bool TrendBearishGBPUSD()
  {
   double ma = iMA("GBPUSD", TrendTF, TrendSMA, 0, MODE_SMA, PRICE_CLOSE, 1);
   double cl = iClose("GBPUSD", TrendTF, 1);
   return (cl < ma);
  }

//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
bool TrendBullishGBPUSD()
  {
   double ma = iMA("GBPUSD", TrendTF, TrendSMA, 0, MODE_SMA, PRICE_CLOSE, 1);
   double cl = iClose("GBPUSD", TrendTF, 1);
   return (cl > ma);
  }

//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
bool TrendBullishEURUSD()
  {
   double ma = iMA("EURUSD", TrendTF, TrendSMA, 0, MODE_SMA, PRICE_CLOSE, 1);
   double cl = iClose("EURUSD", TrendTF, 1);
   return (cl > ma);
  }

//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
bool TrendBearishEURUSD()
  {
   double ma = iMA("EURUSD", TrendTF, TrendSMA, 0, MODE_SMA, PRICE_CLOSE, 1);
   double cl = iClose("EURUSD", TrendTF, 1);
   return (cl > ma);
  }







double GetPipMultiplier(string sym)
{
   int digits = (int)SymbolInfoInteger(sym, SYMBOL_DIGITS);
   return (digits == 3 || digits == 5) ? 10.0 : 1.0;
}

bool TradeTimeAllowed()
{
   int currentHour = Hour();
   int currentMinute = Minute();
   int currentTimeInMinutes = (currentHour * 60) + currentMinute;
   
   int startAllowedTime = (1 * 60) + 00;
   int endAllowedTime   = (20 * 60) + 00;

   return (currentTimeInMinutes >= startAllowedTime && currentTimeInMinutes <= endAllowedTime);
}

bool AlreadyTraded_60Minutes()
{
   datetime currentTime = TimeCurrent();
   datetime thirtyMinutesAgo = currentTime - (60 * 30);

   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(OrderSelect(i, SELECT_BY_POS, MODE_TRADES))
      {
         if(OrderSymbol() == Symbol() && OrderMagicNumber() == MagicNumber)
         {
            if(OrderOpenTime() >= thirtyMinutesAgo) return true;
         }
      }
   }

   for(int j = OrdersHistoryTotal() - 1; j >= 0; j--)
   {
      if(OrderSelect(j, SELECT_BY_POS, MODE_HISTORY))
      {
         if(OrderSymbol() == Symbol() && OrderMagicNumber() == MagicNumber)
         {
            if(OrderOpenTime() >= thirtyMinutesAgo) return true;
         }
      }
   }
   return false;
}

bool HasOpenTrade()
{
   for(int i = OrdersTotal() - 1; i >= 0; i--)
   {
      if(OrderSelect(i, SELECT_BY_POS, MODE_TRADES))
      {
         if(OrderSymbol() == Symbol() && OrderMagicNumber() == MagicNumber) return true;
      }
   }
   return false;
}

void DrawTradeSignal(int type, double openPrice, double sl, double tp)
{
   string name = "Signal_" + IntegerToString(TimeCurrent());
   ObjectCreate(0, name, OBJ_ARROW, 0, TimeCurrent(), openPrice);
   ObjectSetInteger(0, name, OBJPROP_ARROWCODE, (type == OP_BUY) ? 233 : 234);
   ObjectSetInteger(0, name, OBJPROP_COLOR, (type == OP_BUY) ? clrOrchid : clrMediumOrchid);
}



/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

  
 datetime GBPUSD_GetDailyHighTimeOnM1()
{
   string symbol = "GBPUSD";
   
   double dailyHigh = iHigh(symbol, PERIOD_D1, 0);
   if (dailyHigh <= 0.0) return 0;

   datetime dayStart = iTime(symbol, PERIOD_D1, 0);
   datetime dayEnd   = dayStart + PeriodSeconds(PERIOD_D1);

   int startM1 = iBarShift(symbol, PERIOD_M1, dayEnd, false);
   int endM1   = iBarShift(symbol, PERIOD_M1, dayStart, false);

   for (int j = endM1; j >= startM1; j--)
   {
      // Normalizing to 5 digits for GBPUSD precision stability
      if (NormalizeDouble(iHigh(symbol, PERIOD_M1, j), 5) == NormalizeDouble(dailyHigh, 5))
      {
         return iTime(symbol, PERIOD_M1, j);
      }
   }
   return 0;
}

datetime EURUSD_GetDailyHighTimeOnM1()
{
   string symbol = "EURUSD";
   
   double dailyHigh = iHigh(symbol, PERIOD_D1, 0);
   if (dailyHigh <= 0.0) return 0;

   datetime dayStart = iTime(symbol, PERIOD_D1, 0);
   datetime dayEnd   = dayStart + PeriodSeconds(PERIOD_D1);

   int startM1 = iBarShift(symbol, PERIOD_M1, dayEnd, false);
   int endM1   = iBarShift(symbol, PERIOD_M1, dayStart, false);

   for (int j = endM1; j >= startM1; j--)
   {
      // Normalizing to 5 digits for EURUSD precision stability
      if (NormalizeDouble(iHigh(symbol, PERIOD_M1, j), 5) == NormalizeDouble(dailyHigh, 5))
      {
         return iTime(symbol, PERIOD_M1, j);
      }
   }
   return 0;
}




datetime GBPUSD_GetDailyLowTimeOnM1()
{
   string symbol = "GBPUSD";
   
   // Get the current day's Low price instead of High
   double dailyLow = iLow(symbol, PERIOD_D1, 0);
   if (dailyLow <= 0.0) return 0;

   datetime dayStart = iTime(symbol, PERIOD_D1, 0);
   datetime dayEnd   = dayStart + PeriodSeconds(PERIOD_D1);

   int startM1 = iBarShift(symbol, PERIOD_M1, dayEnd, false);
   int endM1   = iBarShift(symbol, PERIOD_M1, dayStart, false);

   for (int j = endM1; j >= startM1; j--)
   {
      // Scanning for the exact low match on the 1-minute chart
      if (NormalizeDouble(iLow(symbol, PERIOD_M1, j), 5) == NormalizeDouble(dailyLow, 5))
      {
         return iTime(symbol, PERIOD_M1, j);
      }
   }
   return 0;
}


datetime EURUSD_GetDailyLowTimeOnM1()
{
   string symbol = "EURUSD";
   
   // Get the current day's Low price instead of High
   double dailyLow = iLow(symbol, PERIOD_D1, 0);
   if (dailyLow <= 0.0) return 0;

   datetime dayStart = iTime(symbol, PERIOD_D1, 0);
   datetime dayEnd   = dayStart + PeriodSeconds(PERIOD_D1);

   int startM1 = iBarShift(symbol, PERIOD_M1, dayEnd, false);
   int endM1   = iBarShift(symbol, PERIOD_M1, dayStart, false);

   for (int j = endM1; j >= startM1; j--)
   {
      // Scanning for the exact low match on the 1-minute chart
      if (NormalizeDouble(iLow(symbol, PERIOD_M1, j), 5) == NormalizeDouble(dailyLow, 5))
      {
         return iTime(symbol, PERIOD_M1, j);
      }
   }
   return 0;
}




//+------------------------------------------------------------------+
//| Get the HIGH of the M1 bar where the Highest Point After Daily Low occurred |
//+------------------------------------------------------------------+
double GetHighAtHighestPointAfterDailyLowM1()
{
   // Step 1: Get the M1 bar time of the highest point after daily low
   datetime hpTimeM1 = GetHighestPointAfterDailyLowTimeOnM1();
   if (hpTimeM1 == 0) return 0;

   // Step 2: Convert that time to an M1 bar index
   int idx = iBarShift(NULL, PERIOD_M1, hpTimeM1, true);
   if (idx < 0) return 0;

   // Step 3: Get the HIGH of that bar
   double highPrice = iHigh(NULL, PERIOD_M1, idx);

   

   return highPrice;
}

//+------------------------------------------------------------------+
//| Get the CLOSE of the M1 bar where the Highest Point After Daily Low occurred |
//+------------------------------------------------------------------+
double GetCloseAtHighestPointAfterDailyLowM1()
{
   // Step 1: Find the exact M1 time of the highest point after daily low
   datetime hpTimeM1 = GetHighestPointAfterDailyLowTimeOnM1();
   if (hpTimeM1 == 0) return 0;

   // Step 2: Convert time to bar index
   int idx = iBarShift(NULL, PERIOD_M1, hpTimeM1, true);
   if (idx < 0) return 0;

   // Step 3: Get the CLOSE of that bar
   double closePrice = iClose(NULL, PERIOD_M1, idx);

   

   return closePrice;
}

//+------------------------------------------------------------------+
//| Pivot at the M1 bar where the Highest Point After Daily Low occurred |
//+------------------------------------------------------------------+
double PivotAtHighestPointAfterDailyLowM1()
{
   // Get High, Low, Close of that specific M1 bar
   double hi = GetHighAtHighestPointAfterDailyLowM1();
   double lo = GetLowAtHighestPointAfterDailyLowM1();
   double cl = GetCloseAtHighestPointAfterDailyLowM1();

   // If any failed, return 0
   if (hi == 0 || lo == 0 || cl == 0)
      return 0;

   // Pivot calculation
   double pivot = (hi + lo + cl) / 3.0;

   

   return pivot;
}


//+------------------------------------------------------------------+
//| Get the LOW of the M1 bar where the Highest Point After Daily Low occurred |
//+------------------------------------------------------------------+
double GetLowAtHighestPointAfterDailyLowM1()
{
   // Step 1: Find the time of the M1 bar where the Highest Point After Daily Low occurred
   datetime hpTimeM1 = GetHighestPointAfterDailyLowTimeOnM1();
   if (hpTimeM1 == 0) return 0;

   // Step 2: Find the M1 bar index corresponding to that time
   int idx = iBarShift(NULL, PERIOD_M1, hpTimeM1, true);
   if (idx < 0) return 0;

   // Step 3: Get the LOW of that exact M1 bar
   double lowPrice = iLow(NULL, PERIOD_M1, idx);

   

   // Step 5: Return the low value
   return lowPrice;
}

//+------------------------------------------------------------------+
//| Get the LOW of the M1 bar where the Lowest Point After Daily High occurred |
//+------------------------------------------------------------------+
double GetLowAtLowestPointAfterDailyHighM1()
{
   // 1. Get the exact M1 time of the lowest point after the daily high
   datetime lpTimeM1 = GetLowestPointAfterDailyHighTimeOnM1();
   if (lpTimeM1 == 0) return 0;

   // 2. Convert that time to an M1 bar index
   int idx = iBarShift(NULL, PERIOD_M1, lpTimeM1, true);
   if (idx < 0) return 0;

   // 3. Get the LOW price of that M1 bar
   double lowPrice = iLow(NULL, PERIOD_M1, idx);

  

   return lowPrice;
}

double GetCloseAtLowestPointAfterDailyHighM1()
{
   // 1. Get the M1 bar time where the lowest point after daily high occurred
   datetime lpTimeM1 = GetLowestPointAfterDailyHighTimeOnM1();
   if (lpTimeM1 == 0) return 0;

   // 2. Convert that time to the M1 bar index
   int idx = iBarShift(NULL, PERIOD_M1, lpTimeM1, true);
   if (idx < 0) return 0;

   // 3. Get the close price of that bar
   double closePrice = iClose(NULL, PERIOD_M1, idx);

   

   return closePrice;
}

 
//+------------------------------------------------------------------+
//| Pivot at the M1 bar where the Lowest Point After Daily High occurred |
//+------------------------------------------------------------------+
double PivotAtLowestPointAfterDailyHighM1()
{
   // Get High, Low, Close of that exact M1 bar
   double hi = GetHighAtLowestPointAfterDailyHighM1();
   double lo = GetLowAtLowestPointAfterDailyHighM1();
   double cl = GetCloseAtLowestPointAfterDailyHighM1();

   // If any of them failed, return 0
   if (hi == 0 || lo == 0 || cl == 0)
      return 0;

   // Pivot formula
   double pivot = (hi + lo + cl) / 3.0;

   

   return pivot;
}

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////



 /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  
  /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

    
//+------------------------------------------------------------------+
//| Function to check if buyCondLow9 has remained unbreached        |
//+------------------------------------------------------------------+
bool IsBuyCondLow9Unbreached(double buyCondLow9, datetime buyCondTime9)
  {
   int startBar = iBarShift(NULL, PERIOD_M1, buyCondTime9, false);
   if(startBar < 0) return true; 
   int lowestBarIndex = iLowest(NULL, PERIOD_M1, MODE_LOW, startBar + 1, 0);
   if(lowestBarIndex != -1)
     {
      if(iLow(NULL, PERIOD_M1, lowestBarIndex) < buyCondLow9)
         return false;
     }
   return true;
  }
  
  
  /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//+------------------------------------------------------------------+
//| Function to check if sellCondHigh9 has remained unbreached     |
//+------------------------------------------------------------------+
bool IsSellCondHigh9Unbreached(double sellCondHigh9, datetime sellCondTime9)
  {
   int startBar = iBarShift(NULL, PERIOD_M1, sellCondTime9, false);
   if(startBar < 0) return true; 
   int highestBarIndex = iHighest(NULL, PERIOD_M1, MODE_HIGH, startBar + 1, 0);
   if(highestBarIndex != -1)
     {
      if(iHigh(NULL, PERIOD_M1, highestBarIndex) > sellCondHigh9)
         return false;
     }
   return true;
  }

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

  
  
  
//+------------------------------------------------------------------+
//| Function to check if buyCondLow307 has remained unbreached        |
//+------------------------------------------------------------------+
bool IsBuyCondLow307Unbreached(double buyCondLow307, datetime buyCondTime307)
  {
   int startBar = iBarShift(NULL, PERIOD_M1, buyCondTime307, false);
   if(startBar < 0) return true; 
   int lowestBarIndex = iLowest(NULL, PERIOD_M1, MODE_LOW, startBar + 1, 0);
   if(lowestBarIndex != -1)
     {
      if(iLow(NULL, PERIOD_M1, lowestBarIndex) < buyCondLow307)
         return false;
     }
   return true;
  }
  
  
  /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//+------------------------------------------------------------------+
//| Function to check if sellCondHigh307 has remained unbreached     |
//+------------------------------------------------------------------+
bool IsSellCondHigh307Unbreached(double sellCondHigh307, datetime sellCondTime307)
  {
   int startBar = iBarShift(NULL, PERIOD_M1, sellCondTime307, false);
   if(startBar < 0) return true; 
   int highestBarIndex = iHighest(NULL, PERIOD_M1, MODE_HIGH, startBar + 1, 0);
   if(highestBarIndex != -1)
     {
      if(iHigh(NULL, PERIOD_M1, highestBarIndex) > sellCondHigh307)
         return false;
     }
   return true;
  }

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

    
//+------------------------------------------------------------------+
//| Function to check if buyCondLow308 has remained unbreached        |
//+------------------------------------------------------------------+
bool IsBuyCondLow308Unbreached(double buyCondLow308, datetime buyCondTime308)
  {
   int startBar = iBarShift(NULL, PERIOD_M1, buyCondTime308, false);
   if(startBar < 0) return true; 
   int lowestBarIndex = iLowest(NULL, PERIOD_M1, MODE_LOW, startBar + 1, 0);
   if(lowestBarIndex != -1)
     {
      if(iLow(NULL, PERIOD_M1, lowestBarIndex) < buyCondLow308)
         return false;
     }
   return true;
  }
  
  
  /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//+------------------------------------------------------------------+
//| Function to check if sellCondHigh308 has remained unbreached     |
//+------------------------------------------------------------------+
bool IsSellCondHigh308Unbreached(double sellCondHigh308, datetime sellCondTime308)
  {
   int startBar = iBarShift(NULL, PERIOD_M1, sellCondTime308, false);
   if(startBar < 0) return true; 
   int highestBarIndex = iHighest(NULL, PERIOD_M1, MODE_HIGH, startBar + 1, 0);
   if(highestBarIndex != -1)
     {
      if(iHigh(NULL, PERIOD_M1, highestBarIndex) > sellCondHigh308)
         return false;
     }
   return true;
  }

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

  
  
//+------------------------------------------------------------------+
//| Get the HIGH of the M1 bar where the Lowest Point After Daily High occurred |
//+------------------------------------------------------------------+
double GetHighAtLowestPointAfterDailyHighM1()
{
   datetime lpTimeM1 = GetLowestPointAfterDailyHighTimeOnM1();
   if (lpTimeM1 == 0) return 0;

   int idx = iBarShift(NULL, PERIOD_M1, lpTimeM1, true);
   if (idx < 0) return 0;

   double highPrice = iHigh(NULL, PERIOD_M1, idx);

  

   return highPrice;
}

//+------------------------------------------------------------------+
//| Finds the time of the Lowest Point After the Daily High on M1    |
//+------------------------------------------------------------------+
datetime GetLowestPointAfterDailyHighTimeOnM1()
{
   datetime dayStart = iTime(Symbol(), PERIOD_D1, 0);
   datetime prevDayStart = iTime(Symbol(), PERIOD_D1, 1);

   double dailyHigh = iHigh(Symbol(), PERIOD_D1, 0);

   int startIdx = iBarShift(Symbol(), PERIOD_M1, dayStart, false);
   if (startIdx < 0) return 0;

   double tolerance = Point * 3;
   int highBar = -1;

   for (int i = startIdx; i >= 0; i--)
   {
      double hi = iHigh(Symbol(), PERIOD_M1, i);
      if (MathAbs(hi - dailyHigh) <= tolerance)
      {
         highBar = i;
         break;
      }
   }

   if (highBar == -1)
   {
      Print("Daily high not found on M1 chart");
      return 0;
   }

   double lowestLow = iLow(Symbol(), PERIOD_M1, highBar);
   int lowestBar = -1;

   for (int ib = highBar - 1; ib >= 0; ib--)
   {
      datetime t = iTime(Symbol(), PERIOD_M1, ib);
      if (t < prevDayStart) break;

      double lo = iLow(Symbol(), PERIOD_M1, ib);
      if (lo < lowestLow)
      {
         lowestLow = lo;
         lowestBar = ib;  // ✅ fixed variable
      }
   }

   if (lowestBar == -1)
      return 0;

   return iTime(Symbol(), PERIOD_M1, lowestBar);
}


//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////





//+------------------------------------------------------------------+
//| Get the OPEN of the M1 bar where the Lowest Point After Daily High occurred |
//+------------------------------------------------------------------+
double GetOpenAtLowestPointAfterDailyHighM1()
{
   // Get the M1 time of the lowest point
   datetime lpTimeM1 = GetLowestPointAfterDailyHighTimeOnM1();
   if (lpTimeM1 == 0) return 0;

   // Convert the exact time to an M1 bar index
   int idx = iBarShift(NULL, PERIOD_M1, lpTimeM1, true);
   if (idx < 0) return 0;

   // Get the OPEN price of that M1 bar
   double openPrice = iOpen(NULL, PERIOD_M1, idx);

  

   return openPrice;
}

//+------------------------------------------------------------------+
//| Finds the time of the Highest Point After the Daily Low on M1    |
//+------------------------------------------------------------------+
datetime GetHighestPointAfterDailyLowTimeOnM1()
  {
// 1. Get today's start and yesterday's start
   datetime dayStart     = iTime(Symbol(), PERIOD_D1, 0);
   datetime prevDayStart = iTime(Symbol(), PERIOD_D1, 1);

// 2. Get today's low value
   double dailyLow = iLow(Symbol(), PERIOD_D1, 0);

// 3. Find the first M1 bar where the daily low occurred (within tolerance)
   int startIdx = iBarShift(Symbol(), PERIOD_M1, dayStart, false);
   if(startIdx < 0)
      return 0;

   double tolerance = Point * 3;
   int lowBar = -1;

   for(int i = startIdx; i >= 0; i--)
     {
      double lo = iLow(Symbol(), PERIOD_M1, i);
      if(MathAbs(lo - dailyLow) <= tolerance)
        {
         lowBar = i;
         break;
        }
     }

   if(lowBar == -1)
     {
      Print("Daily low bar not found on M1 chart.");
      return 0;
     }

// 4. Scan forward (after the daily low) for the highest high
   double highestHigh = iHigh(Symbol(), PERIOD_M1, lowBar);
   int highestBar = -1;
   int totalBars = iBars(Symbol(), PERIOD_M1);

   for(int ia = lowBar - 1; ia >= 0; ia--)
     {
      datetime t = iTime(Symbol(), PERIOD_M1, ia);
      if(t < prevDayStart)
         break;  // stop if we reached previous day

      double hi = iHigh(Symbol(), PERIOD_M1, ia);
      if(hi > highestHigh)
        {
         highestHigh = hi;
         highestBar = ia;
        }
     }

   if(highestBar == -1)
      return 0;

   return iTime(Symbol(), PERIOD_M1, highestBar);
  }



  



/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////



//+------------------------------------------------------------------+
//| Helper Utilities                                                 |
//+------------------------------------------------------------------+
string GetBrokerSymbol(string baseSymbol)
{
   if(SymbolInfoDouble(baseSymbol, SYMBOL_BID) > 0) return baseSymbol;
   string currentChart = Symbol();
   if(StringFind(currentChart, baseSymbol) >= 0) return currentChart;
   return baseSymbol;
}





void UpdateDailyHighLowMarkers_Continuous()
{
   double dailyHigh = iHigh(NULL, PERIOD_D1, 0);
   double dailyLow  = iLow(NULL, PERIOD_D1, 0);

   if (dailyHigh != g_LastDailyHigh)
   {
      g_LastDailyHigh = dailyHigh;
      GetHighAtDailyHighM1();  // redraw marker
      GetLowAtDailyHighM1();
   }

   if (dailyLow != g_LastDailyLow)
   {
      g_LastDailyLow = dailyLow;
      GetLowAtDailyLowM1();   // redraw marker
      GetHighAtDailyLowM1();
   }
}



//+------------------------------------------------------------------+
//| Finds the lowest point after the daily high                     |
//+------------------------------------------------------------------+
double GetLowestPointAfterDailyHigh()
  {
   double dailyHigh = iHigh(Symbol(), PERIOD_D1, 0);
   int dailyHighBar = -1;

// Fxnd the bar of the daxly hxgh
   for(int x = 0; x < Bars; x++)
     {
      if(High[x] == dailyHigh)
        {
         dailyHighBar = x;
         break;
        }
     }

   if(dailyHighBar == -1)
     {
      // Daily high not found
      return EMPTY_VALUE;
     }

// Fxnd the lowest low after the daily high
   double lowestLow = High[dailyHighBar]; // Inxtxalxze wxth the daxly hxgh
   for(int z = dailyHighBar - 1; z >= 0; z--)
     {
      if(Low[z] < lowestLow)
        {
         lowestLow = Low[z];
        }
     }

   return lowestLow;
  }


//+------------------------------------------------------------------+
//| Finds the highest point after the daily low                      |
//+------------------------------------------------------------------+
double GetHighestPointAfterDailyLow()
  {
   double dailyLow = iLow(Symbol(), PERIOD_D1, 0);
   int dailyLowBar = -1;

// Fynd the bar of the dayly low
   for(int y = 0; y < Bars; y++)
     {
      if(Low[y] == dailyLow)
        {
         dailyLowBar = y;
         break;
        }
     }

   if(dailyLowBar == -1)
     {
      // Dayly low not found
      return EMPTY_VALUE;
     }

// Fynd the hyghest hygh after the dayly low
   double highestHigh = Low[dailyLowBar]; // Inytyalyze wyth the dayly low
   for(int m = dailyLowBar - 1; m >= 0; m--)
     {
      if(High[m] > highestHigh)
        {
         highestHigh = High[m];
        }
     }

   return highestHigh;
  }

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////



//+------------------------------------------------------------------+
//| === DAILY LOW FUNCTIONS ======================================== |
//+------------------------------------------------------------------+
double GetLowAtDailyLowM1()
{
   datetime dlTimeM1 = GetDailyLowTimeOnM1();
   if (dlTimeM1 == 0) return 0;

   int idx = iBarShift(NULL, PERIOD_M1, dlTimeM1, true);
   if (idx < 0) return 0;

   double lowPrice = iLow(NULL, PERIOD_M1, idx);

  
   return lowPrice;
}

datetime GetDailyLowTimeOnM1()
{
   double dailyLow = iLow(NULL, PERIOD_D1, 0);
   if (dailyLow <= 0.0) return 0;

   datetime dayStart = iTime(NULL, PERIOD_D1, 0);
   datetime dayEnd   = dayStart + PeriodSeconds(PERIOD_D1);

   int startM1 = iBarShift(NULL, PERIOD_M1, dayEnd, false);
   int endM1   = iBarShift(NULL, PERIOD_M1, dayStart, false);

   for (int j = endM1; j >= startM1; j--)
   {
      if (NormalizeDouble(iLow(NULL, PERIOD_M1, j), Digits) == NormalizeDouble(dailyLow, Digits))
      {
         datetime dlTimeM1 = iTime(NULL, PERIOD_M1, j);
        
      }
   }
   return 0;
}

double GetHighAtDailyLowM1()
{
   datetime dlTimeM1 = GetDailyLowTimeOnM1();
   if (dlTimeM1 == 0) return 0;

   int idx = iBarShift(NULL, PERIOD_M1, dlTimeM1, true);
   if (idx < 0) return 0;

   double highPrice = iHigh(NULL, PERIOD_M1, idx);

 
  
   return highPrice;
}

//+------------------------------------------------------------------+
//| === DAILY HIGH FUNCTIONS ======================================= |
//+------------------------------------------------------------------+
double GetHighAtDailyHighM1()
{
   datetime dhTimeM1 = GetDailyHighTimeOnM1();
   if (dhTimeM1 == 0) return 0;

   int idx = iBarShift(NULL, PERIOD_M1, dhTimeM1, true);
   if (idx < 0) return 0;

   double highPrice = iHigh(NULL, PERIOD_M1, idx);

  
   
   return highPrice;
}

datetime GetDailyHighTimeOnM1()
{
   double dailyHigh = iHigh(NULL, PERIOD_D1, 0);
   if (dailyHigh <= 0.0) return 0;

   datetime dayStart = iTime(NULL, PERIOD_D1, 0);
   datetime dayEnd   = dayStart + PeriodSeconds(PERIOD_D1);

   int startM1 = iBarShift(NULL, PERIOD_M1, dayEnd, false);
   int endM1   = iBarShift(NULL, PERIOD_M1, dayStart, false);

   for (int j = endM1; j >= startM1; j--)
   {
      if (NormalizeDouble(iHigh(NULL, PERIOD_M1, j), Digits) == NormalizeDouble(dailyHigh, Digits))
      {
         datetime dhTimeM1 = iTime(NULL, PERIOD_M1, j);
         
        
         return dhTimeM1;
     
      }
   }
   return 0;
}

double GetLowAtDailyHighM1()
{
   datetime dhTimeM1 = GetDailyHighTimeOnM1();
   if (dhTimeM1 == 0) return 0;

   int idx = iBarShift(NULL, PERIOD_M1, dhTimeM1, true);
   if (idx < 0) return 0;

   double lowPrice = iLow(NULL, PERIOD_M1, idx);

   
   
   return lowPrice;
}


/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  
  
//+------------------------------------------------------------------+
//| FIND TODAY'S HIGH TIME                                            |
//| Replicates the indicator exactly                                 |
//+------------------------------------------------------------------+
datetime GetBullishVolume_HighTime()
{
   double highToday = iHigh(Symbol(), PERIOD_D1, 0);
   double lowToday  = iLow(Symbol(), PERIOD_D1, 0);

   datetime highTime = 0;
   datetime lowTime  = 0;

   for(int i = 0; i < 1440; i++)
   {
      if(iHigh(Symbol(), PERIOD_M1, i) == highToday && highTime == 0)
         highTime = iTime(Symbol(), PERIOD_M1, i);

      if(iLow(Symbol(), PERIOD_M1, i) == lowToday && lowTime == 0)
         lowTime = iTime(Symbol(), PERIOD_M1, i);

      if(highTime > 0 && lowTime > 0)
         break;
   }

   return highTime;
}


//+------------------------------------------------------------------+
//| FIND TODAY'S LOW TIME                                             |
//+------------------------------------------------------------------+
datetime GetBearishVolume_LowTime()
{
   double highToday = iHigh(Symbol(), PERIOD_D1, 0);
   double lowToday  = iLow(Symbol(), PERIOD_D1, 0);

   datetime highTime = 0;
   datetime lowTime  = 0;

   for(int i = 0; i < 1440; i++)
   {
      if(iHigh(Symbol(), PERIOD_M1, i) == highToday && highTime == 0)
         highTime = iTime(Symbol(), PERIOD_M1, i);

      if(iLow(Symbol(), PERIOD_M1, i) == lowToday && lowTime == 0)
         lowTime = iTime(Symbol(), PERIOD_M1, i);

      if(highTime > 0 && lowTime > 0)
         break;
   }

   return lowTime;
}


//+------------------------------------------------------------------+
//| FIND BULLISH MAXIMUM VOLUME BAR SHIFT                            |
//+------------------------------------------------------------------+
int GetBullishVolumeShift()
{
   double highToday = iHigh(Symbol(), PERIOD_D1, 0);
   double lowToday  = iLow(Symbol(), PERIOD_D1, 0);

   datetime highTime = 0;
   datetime lowTime  = 0;

   for(int i = 0; i < 1440; i++)
   {
      if(iHigh(Symbol(), PERIOD_M1, i) == highToday && highTime == 0)
         highTime = iTime(Symbol(), PERIOD_M1, i);

      if(iLow(Symbol(), PERIOD_M1, i) == lowToday && lowTime == 0)
         lowTime = iTime(Symbol(), PERIOD_M1, i);

      if(highTime > 0 && lowTime > 0)
         break;
   }

   if(highTime == 0 || lowTime == 0)
      return -1;

   bool isUpTrend = highTime > lowTime;

   if(!isUpTrend)
      return -1;

   int maxVolume = -1;
   int maxIndex  = -1;

   for(int i = 0; i < 1440; i++)
   {
      datetime t = iTime(Symbol(), PERIOD_M1, i);
      double o   = iOpen(Symbol(), PERIOD_M1, i);
      double c   = iClose(Symbol(), PERIOD_M1, i);
      int vol    = iVolume(Symbol(), PERIOD_M1, i);

      if(t > highTime && c > o && vol > maxVolume)
      {
         maxVolume = vol;
         maxIndex  = i;
      }
   }

   return maxIndex;
}


//+------------------------------------------------------------------+
//| FIND BEARISH MAXIMUM VOLUME BAR SHIFT                            |
//+------------------------------------------------------------------+
int GetBearishVolumeShift()
{
   double highToday = iHigh(Symbol(), PERIOD_D1, 0);
   double lowToday  = iLow(Symbol(), PERIOD_D1, 0);

   datetime highTime = 0;
   datetime lowTime  = 0;

   for(int i = 0; i < 1440; i++)
   {
      if(iHigh(Symbol(), PERIOD_M1, i) == highToday && highTime == 0)
         highTime = iTime(Symbol(), PERIOD_M1, i);

      if(iLow(Symbol(), PERIOD_M1, i) == lowToday && lowTime == 0)
         lowTime = iTime(Symbol(), PERIOD_M1, i);

      if(highTime > 0 && lowTime > 0)
         break;
   }

   if(highTime == 0 || lowTime == 0)
      return -1;

   bool isDownTrend = lowTime > highTime;

   if(!isDownTrend)
      return -1;

   int maxVolume = -1;
   int maxIndex  = -1;

   for(int i = 0; i < 1440; i++)
   {
      datetime t = iTime(Symbol(), PERIOD_M1, i);
      double o   = iOpen(Symbol(), PERIOD_M1, i);
      double c   = iClose(Symbol(), PERIOD_M1, i);
      int vol    = iVolume(Symbol(), PERIOD_M1, i);

      if(t > lowTime && c < o && vol > maxVolume)
      {
         maxVolume = vol;
         maxIndex  = i;
      }
   }

   return maxIndex;
}


//+------------------------------------------------------------------+
//| BULLISH VOLUME BAR TIME                                          |
//+------------------------------------------------------------------+
datetime GetBullishVolumeTime()
{
   int shift = GetBullishVolumeShift();

   if(shift < 0)
      return 0;

   return iTime(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BEARISH VOLUME BAR TIME                                          |
//+------------------------------------------------------------------+
datetime GetBearishVolumeTime()
{
   int shift = GetBearishVolumeShift();

   if(shift < 0)
      return 0;

   return iTime(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BULLISH VOLUME BAR OPEN                                          |
//+------------------------------------------------------------------+
double GetBullishVolumeOpen()
{
   int shift = GetBullishVolumeShift();

   if(shift < 0)
      return 0.0;

   return iOpen(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BULLISH VOLUME BAR HIGH                                          |
//+------------------------------------------------------------------+
double GetBullishVolumeHigh()
{
   int shift = GetBullishVolumeShift();

   if(shift < 0)
      return 0.0;

   return iHigh(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BULLISH VOLUME BAR LOW                                           |
//+------------------------------------------------------------------+
double GetBullishVolumeLow()
{
   int shift = GetBullishVolumeShift();

   if(shift < 0)
      return 0.0;

   return iLow(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BULLISH VOLUME BAR CLOSE                                         |
//+------------------------------------------------------------------+
double GetBullishVolumeClose()
{
   int shift = GetBullishVolumeShift();

   if(shift < 0)
      return 0.0;

   return iClose(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BULLISH VOLUME BAR TICK VOLUME                                   |
//+------------------------------------------------------------------+
long GetBullishVolume()
{
   int shift = GetBullishVolumeShift();

   if(shift < 0)
      return 0;

   return iVolume(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BEARISH VOLUME BAR OPEN                                          |
//+------------------------------------------------------------------+
double GetBearishVolumeOpen()
{
   int shift = GetBearishVolumeShift();

   if(shift < 0)
      return 0.0;

   return iOpen(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BEARISH VOLUME BAR HIGH                                          |
//+------------------------------------------------------------------+
double GetBearishVolumeHigh()
{
   int shift = GetBearishVolumeShift();

   if(shift < 0)
      return 0.0;

   return iHigh(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BEARISH VOLUME BAR LOW                                           |
//+------------------------------------------------------------------+
double GetBearishVolumeLow()
{
   int shift = GetBearishVolumeShift();

   if(shift < 0)
      return 0.0;

   return iLow(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BEARISH VOLUME BAR CLOSE                                         |
//+------------------------------------------------------------------+
double GetBearishVolumeClose()
{
   int shift = GetBearishVolumeShift();

   if(shift < 0)
      return 0.0;

   return iClose(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| BEARISH VOLUME BAR TICK VOLUME                                   |
//+------------------------------------------------------------------+
long GetBearishVolume()
{
   int shift = GetBearishVolumeShift();

   if(shift < 0)
      return 0;

   return iVolume(Symbol(), PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| Deletes every object whose name starts with 'prefix', except the |
//| one called 'keepName' (pass "" to delete them all).              |
//+------------------------------------------------------------------+
void DeleteOldLines(string prefix, string keepName)
{
   for(int i = ObjectsTotal() - 1; i >= 0; i--)   // loop backwards when deleting
   {
      string objName = ObjectName(0, i);
      if(StringFind(objName, prefix) == 0 && objName != keepName)
         ObjectDelete(0, objName);
   }
}

//+------------------------------------------------------------------+
//| DRAW GREEN VERTICAL LINE ON BULLISH VOLUME BAR                   |
//+------------------------------------------------------------------+
void DrawBullishVolumeLine()
{
   string prefix  = "BullishVolume_";
   datetime barTime = GetBullishVolumeTime();

   // No valid bar any more -> remove all old bullish lines
   if(barTime == 0)
   {
      DeleteOldLines(prefix, "");
      return;
   }

   string name = prefix + IntegerToString((int)barTime);

   // Remove lines belonging to previous (now irrelevant) bars
   DeleteOldLines(prefix, name);

   if(ObjectFind(0, name) < 0)
   {
      ObjectCreate(0, name, OBJ_VLINE, 0, barTime, 0);
      ObjectSetInteger(0, name, OBJPROP_COLOR, clrLime);
      ObjectSetInteger(0, name, OBJPROP_WIDTH, 2);
   }
}

//+------------------------------------------------------------------+
//| DRAW RED VERTICAL LINE ON BEARISH VOLUME BAR                     |
//+------------------------------------------------------------------+
void DrawBearishVolumeLine()
{
   string prefix  = "BearishVolume_";
   datetime barTime = GetBearishVolumeTime();

   // No valid bar any more -> remove all old bearish lines
   if(barTime == 0)
   {
      DeleteOldLines(prefix, "");
      return;
   }

   string name = prefix + IntegerToString((int)barTime);

   // Remove lines belonging to previous (now irrelevant) bars
   DeleteOldLines(prefix, name);

   if(ObjectFind(0, name) < 0)
   {
      ObjectCreate(0, name, OBJ_VLINE, 0, barTime, 0);
      ObjectSetInteger(0, name, OBJPROP_COLOR, clrRed);
      ObjectSetInteger(0, name, OBJPROP_WIDTH, 2);
   }
}


/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////






 
 //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 
double EURUSD_GetZZZ3ZigZagStopLossLevel()
{
   
   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ || iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   int PointShift3=PointShift2;
   string ConfirmedPoint3="Not Found";

   while(ConfirmedPoint3!="Found")
     {
      double ZZ3=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift3);
      if(iHigh("EURUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint3="Found";
         break;
        }
      PointShift3++;
     }

   double ZZZ1 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift);
   double ZZZ2 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift2);
   double ZZZ3 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift3);

   double zigzagLongEntry=0;
   double zigzagShortEntry=0;

   if(ZZZ3<ZZZ2 && ZZZ2>ZZZ1 && ZZZ1>=ZZZ3)
     {
      zigzagLongEntry=ZZZ2+(PipBuffer*Point);
      zigzagShortEntry=0;
     }

   if(ZZZ3>ZZZ2 && ZZZ2<ZZZ1 && ZZZ1<=ZZZ3)
     {
      zigzagLongEntry=0;
      zigzagShortEntry=ZZZ2 -(PipBuffer*Point);
     }


   
   
   // your full logic...
   return ZZZ3;
}



//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


 
 //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 
double EURUSD_GetZZZ1ZigZagStopLossLevel()
{
   
   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ || iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   int PointShift3=PointShift2;
   string ConfirmedPoint3="Not Found";

   while(ConfirmedPoint3!="Found")
     {
      double ZZ3=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift3);
      if(iHigh("EURUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint3="Found";
         break;
        }
      PointShift3++;
     }

   double ZZZ1 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift);
   double ZZZ2 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift2);
   double ZZZ3 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift3);

   double zigzagLongEntry=0;
   double zigzagShortEntry=0;

   if(ZZZ3<ZZZ2 && ZZZ2>ZZZ1 && ZZZ1>=ZZZ3)
     {
      zigzagLongEntry=ZZZ2+(PipBuffer*Point);
      zigzagShortEntry=0;
     }

   if(ZZZ3>ZZZ2 && ZZZ2<ZZZ1 && ZZZ1<=ZZZ3)
     {
      zigzagLongEntry=0;
      zigzagShortEntry=ZZZ2 -(PipBuffer*Point);
     }


   
   
   // your full logic...
   return ZZZ1;
}



//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 
 //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 
double EURUSD_GetZZZ2ZigZagStopLossLevel()
{
   
   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ || iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   int PointShift3=PointShift2;
   string ConfirmedPoint3="Not Found";

   while(ConfirmedPoint3!="Found")
     {
      double ZZ3=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift3);
      if(iHigh("EURUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint3="Found";
         break;
        }
      PointShift3++;
     }

   double ZZZ1 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift);
   double ZZZ2 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift2);
   double ZZZ3 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift3);

   double zigzagLongEntry=0;
   double zigzagShortEntry=0;

   if(ZZZ3<ZZZ2 && ZZZ2>ZZZ1 && ZZZ1>=ZZZ3)
     {
      zigzagLongEntry=ZZZ2+(PipBuffer*Point);
      zigzagShortEntry=0;
     }

   if(ZZZ3>ZZZ2 && ZZZ2<ZZZ1 && ZZZ1<=ZZZ3)
     {
      zigzagLongEntry=0;
      zigzagShortEntry=ZZZ2 -(PipBuffer*Point);
     }


   
   
   // your full logic...
   return ZZZ2;
}



//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////



//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN1ZZZ2ZigZagStopLossLevel()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   int N1PointShift3=N1PointShift2;
   string N1ConfirmedPoint3="Not Found";

   while(N1ConfirmedPoint3!="Found")
     {
      double N1ZZ3=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift3);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      N1PointShift3++;
     }

   double N1ZZZ1 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift);
   double N1ZZZ2 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift2);
   double N1ZZZ3 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift3);

   double N1zigzagLongEntry=0;
   double N1zigzagShortEntry=0;

   if(N1ZZZ3<N1ZZZ2 && N1ZZZ2>N1ZZZ1 && N1ZZZ1>N1ZZZ3)
     {
      N1zigzagLongEntry=N1ZZZ2+(PipBuffer*Point);
      N1zigzagShortEntry=0;
     }

   if(N1ZZZ3>N1ZZZ2 && N1ZZZ2<N1ZZZ1 && N1ZZZ1<N1ZZZ3)
     {
      N1zigzagLongEntry=0;
      N1zigzagShortEntry=N1ZZZ2 -(PipBuffer*Point);
     }




// your full logic...
   return N1ZZZ2;
  }








//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
 





//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN5ZZZ2ZigZagStopLossLevel()
  {



   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   int N5PointShift3=N5PointShift2;
   string N5ConfirmedPoint3="Not Found";

   while(N5ConfirmedPoint3!="Found")
     {
      double N5ZZ3=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift3);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      N5PointShift3++;
     }

   double N5ZZZ1 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift);
   double N5ZZZ2 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift2);
   double N5ZZZ3 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift3);

   double N5zigzagLongEntry=0;
   double N5zigzagShortEntry=0;

   if(N5ZZZ3<N5ZZZ2 && N5ZZZ2>N5ZZZ1 && N5ZZZ1>N5ZZZ3)
     {
      N5zigzagLongEntry=N5ZZZ2+(PipBuffer*Point);
      N5zigzagShortEntry=0;
     }

   if(N5ZZZ3>N5ZZZ2 && N5ZZZ2<N5ZZZ1 && N5ZZZ1<N5ZZZ3)
     {
      N5zigzagLongEntry=0;
      N5zigzagShortEntry=N5ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return N5ZZZ2;
  }





//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
 
 
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN15ZZZ2ZigZagStopLossLevel()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   int N15PointShift3=N15PointShift2;
   string N15ConfirmedPoint3="Not Found";

   while(N15ConfirmedPoint3!="Found")
     {
      double N15ZZ3=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift3);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      N15PointShift3++;
     }

   double N15ZZZ1 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift);
   double N15ZZZ2 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift2);
   double N15ZZZ3 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift3);

   double N15zigzagLongEntry=0;
   double N15zigzagShortEntry=0;

   if(N15ZZZ3<N15ZZZ2 && N15ZZZ2>N15ZZZ1 && N15ZZZ1>N15ZZZ3)
     {
      N15zigzagLongEntry=N15ZZZ2+(PipBuffer*Point);
      N15zigzagShortEntry=0;
     }

   if(N15ZZZ3>N15ZZZ2 && N15ZZZ2<N15ZZZ1 && N15ZZZ1<N15ZZZ3)
     {
      N15zigzagLongEntry=0;
      N15zigzagShortEntry=N15ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N15ZZZ2;
  }



 
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN30ZZZ2ZigZagStopLossLevel()
  {
   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   int N30PointShift3=N30PointShift2;
   string N30ConfirmedPoint3="Not Found";

   while(N30ConfirmedPoint3!="Found")
     {
      double N30ZZ3=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift3);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      N30PointShift3++;
     }

   double N30ZZZ1 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift);
   double N30ZZZ2 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift2);
   double N30ZZZ3 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift3);

   double N30zigzagLongEntry=0;
   double N30zigzagShortEntry=0;

   if(N30ZZZ3<N30ZZZ2 && N30ZZZ2>N30ZZZ1 && N30ZZZ1>N30ZZZ3)
     {
      N30zigzagLongEntry=N30ZZZ2+(PipBuffer*Point);
      N30zigzagShortEntry=0;
     }

   if(N30ZZZ3>N30ZZZ2 && N30ZZZ2<N30ZZZ1 && N30ZZZ1<N30ZZZ3)
     {
      N30zigzagLongEntry=0;
      N30zigzagShortEntry=N30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N30ZZZ2;
  }




//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetG1ZZZ2ZigZagStopLossLevel()
  {
   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   int G1PointShift3=G1PointShift2;
   string G1ConfirmedPoint3="Not Found";

   while(G1ConfirmedPoint3!="Found")
     {
      double G1ZZ3=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift3);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      G1PointShift3++;
     }

   double G1ZZZ1 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift);
   double G1ZZZ2 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift2);
   double G1ZZZ3 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift3);

   double G1zigzagLongEntry=0;
   double G1zigzagShortEntry=0;

   if(G1ZZZ3<G1ZZZ2 && G1ZZZ2>G1ZZZ1 && G1ZZZ1>G1ZZZ3)
     {
      G1zigzagLongEntry=G1ZZZ2+(PipBuffer*Point);
      G1zigzagShortEntry=0;
     }

   if(G1ZZZ3>G1ZZZ2 && G1ZZZ2<G1ZZZ1 && G1ZZZ1<G1ZZZ3)
     {
      G1zigzagLongEntry=0;
      G1zigzagShortEntry=G1ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G1ZZZ2;
  }




//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetM5ZZZ2ZigZagStopLossLevel()
  {


   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   int M5PointShift3=M5PointShift2;
   string M5ConfirmedPoint3="Not Found";

   while(M5ConfirmedPoint3!="Found")
     {
      double M5ZZ3=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift3);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      M5PointShift3++;
     }

   double M5ZZZ1 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift);
   double M5ZZZ2 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift2);
   double M5ZZZ3 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift3);

   double M5zigzagLongEntry=0;
   double M5zigzagShortEntry=0;

   if(M5ZZZ3<M5ZZZ2 && M5ZZZ2>M5ZZZ1 && M5ZZZ1>=M5ZZZ3)
     {
      M5zigzagLongEntry=M5ZZZ2+(PipBuffer*Point);
      M5zigzagShortEntry=0;
     }

   if(M5ZZZ3>M5ZZZ2 && M5ZZZ2<M5ZZZ1 && M5ZZZ1<=M5ZZZ3)
     {
      M5zigzagLongEntry=0;
      M5zigzagShortEntry=M5ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return M5ZZZ2;
  }
  
  
  
  
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetM15ZZZ2ZigZagStopLossLevel()
  {
   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   int M15PointShift3=M15PointShift2;
   string M15ConfirmedPoint3="Not Found";

   while(M15ConfirmedPoint3!="Found")
     {
      double M15ZZ3=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift3);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      M15PointShift3++;
     }

   double M15ZZZ1 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift);
   double M15ZZZ2 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift2);
   double M15ZZZ3 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift3);

   double M15zigzagLongEntry=0;
   double M15zigzagShortEntry=0;

   if(M15ZZZ3<M15ZZZ2 && M15ZZZ2>M15ZZZ1 && M15ZZZ1>=M15ZZZ3)
     {
      M15zigzagLongEntry=M15ZZZ2+(PipBuffer*Point);
      M15zigzagShortEntry=0;
     }

   if(M15ZZZ3>M15ZZZ2 && M15ZZZ2<M15ZZZ1 && M15ZZZ1<=M15ZZZ3)
     {
      M15zigzagLongEntry=0;
      M15zigzagShortEntry=M15ZZZ2 -(PipBuffer*Point);
     }

// your full logic...
   return M15ZZZ2;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetM30ZZZ2ZigZagStopLossLevel()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   int M30PointShift3=M30PointShift2;
   string M30ConfirmedPoint3="Not Found";

   while(M30ConfirmedPoint3!="Found")
     {
      double M30ZZ3=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift3);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      M30PointShift3++;
     }

   double M30ZZZ1 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift);
   double M30ZZZ2 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift2);
   double M30ZZZ3 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift3);

   double M30zigzagLongEntry=0;
   double M30zigzagShortEntry=0;

   if(M30ZZZ3<M30ZZZ2 && M30ZZZ2>M30ZZZ1 && M30ZZZ1>=M30ZZZ3)
     {
      M30zigzagLongEntry=M30ZZZ2+(PipBuffer*Point);
      M30zigzagShortEntry=0;
     }

   if(M30ZZZ3>M30ZZZ2 && M30ZZZ2<M30ZZZ1 && M30ZZZ1<=M30ZZZ3)
     {
      M30zigzagLongEntry=0;
      M30zigzagShortEntry=M30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return M30ZZZ2;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetH1ZZZ2ZigZagStopLossLevel()
  {
   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   int H1PointShift3=H1PointShift2;
   string H1ConfirmedPoint3="Not Found";

   while(H1ConfirmedPoint3!="Found")
     {
      double H1ZZ3=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift3);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      H1PointShift3++;
     }

   double H1ZZZ1 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift);
   double H1ZZZ2 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift2);
   double H1ZZZ3 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift3);

   double H1zigzagLongEntry=0;
   double H1zigzagShortEntry=0;

   if(H1ZZZ3<H1ZZZ2 && H1ZZZ2>H1ZZZ1 && H1ZZZ1>=H1ZZZ3)
     {
      H1zigzagLongEntry=H1ZZZ2+(PipBuffer*Point);
      H1zigzagShortEntry=0;
     }

   if(H1ZZZ3>H1ZZZ2 && H1ZZZ2<H1ZZZ1 && H1ZZZ1<=H1ZZZ3)
     {
      H1zigzagLongEntry=0;
      H1zigzagShortEntry=H1ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return H1ZZZ2;
  }




/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////




 //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetH4ZZZ2ZigZagStopLossLevel()
  {
   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   int H4PointShift3=H4PointShift2;
   string H4ConfirmedPoint3="Not Found";

   while(H4ConfirmedPoint3!="Found")
     {
      double H4ZZ3=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift3);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      H4PointShift3++;
     }

   double H4ZZZ1 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift);
   double H4ZZZ2 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift2);
   double H4ZZZ3 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift3);

   double H4zigzagLongEntry=0;
   double H4zigzagShortEntry=0;

   if(H4ZZZ3<H4ZZZ2 && H4ZZZ2>H4ZZZ1 && H4ZZZ1>H4ZZZ3)
     {
      H4zigzagLongEntry=H4ZZZ2+(PipBuffer*Point);
      H4zigzagShortEntry=0;
     }

   if(H4ZZZ3>H4ZZZ2 && H4ZZZ2<H4ZZZ1 && H4ZZZ1<H4ZZZ3)
     {
      H4zigzagLongEntry=0;
      H4zigzagShortEntry=H4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return H4ZZZ2;
  }



  




 //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetG4ZZZ2ZigZagStopLossLevel()
  {
   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   int G4PointShift3=G4PointShift2;
   string G4ConfirmedPoint3="Not Found";

   while(G4ConfirmedPoint3!="Found")
     {
      double G4ZZ3=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift3);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      G4PointShift3++;
     }

   double G4ZZZ1 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift);
   double G4ZZZ2 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift2);
   double G4ZZZ3 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift3);

   double G4zigzagLongEntry=0;
   double G4zigzagShortEntry=0;

   if(G4ZZZ3<G4ZZZ2 && G4ZZZ2>G4ZZZ1 && G4ZZZ1>G4ZZZ3)
     {
      G4zigzagLongEntry=G4ZZZ2+(PipBuffer*Point);
      G4zigzagShortEntry=0;
     }

   if(G4ZZZ3>G4ZZZ2 && G4ZZZ2<G4ZZZ1 && G4ZZZ1<G4ZZZ3)
     {
      G4zigzagLongEntry=0;
      G4zigzagShortEntry=G4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G4ZZZ2;
  }







































//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////




//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN1ZZZ3ZigZagStopLossLevel()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   int N1PointShift3=N1PointShift2;
   string N1ConfirmedPoint3="Not Found";

   while(N1ConfirmedPoint3!="Found")
     {
      double N1ZZ3=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift3);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      N1PointShift3++;
     }

   double N1ZZZ1 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift);
   double N1ZZZ2 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift2);
   double N1ZZZ3 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift3);

   double N1zigzagLongEntry=0;
   double N1zigzagShortEntry=0;

   if(N1ZZZ3<N1ZZZ2 && N1ZZZ2>N1ZZZ1 && N1ZZZ1>N1ZZZ3)
     {
      N1zigzagLongEntry=N1ZZZ2+(PipBuffer*Point);
      N1zigzagShortEntry=0;
     }

   if(N1ZZZ3>N1ZZZ2 && N1ZZZ2<N1ZZZ1 && N1ZZZ1<N1ZZZ3)
     {
      N1zigzagLongEntry=0;
      N1zigzagShortEntry=N1ZZZ2 -(PipBuffer*Point);
     }




// your full logic...
   return N1ZZZ3;
  }




//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN1ZZZ1ZigZagStopLossLevel()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   int N1PointShift3=N1PointShift2;
   string N1ConfirmedPoint3="Not Found";

   while(N1ConfirmedPoint3!="Found")
     {
      double N1ZZ3=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift3);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      N1PointShift3++;
     }

   double N1ZZZ1 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift);
   double N1ZZZ2 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift2);
   double N1ZZZ3 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift3);

   double N1zigzagLongEntry=0;
   double N1zigzagShortEntry=0;

   if(N1ZZZ3<N1ZZZ2 && N1ZZZ2>N1ZZZ1 && N1ZZZ1>N1ZZZ3)
     {
      N1zigzagLongEntry=N1ZZZ2+(PipBuffer*Point);
      N1zigzagShortEntry=0;
     }

   if(N1ZZZ3>N1ZZZ2 && N1ZZZ2<N1ZZZ1 && N1ZZZ1<N1ZZZ3)
     {
      N1zigzagLongEntry=0;
      N1zigzagShortEntry=N1ZZZ2 -(PipBuffer*Point);
     }




// your full logic...
   return N1ZZZ1;
  }






//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN5ZZZ3ZigZagStopLossLevel()
  {



   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   int N5PointShift3=N5PointShift2;
   string N5ConfirmedPoint3="Not Found";

   while(N5ConfirmedPoint3!="Found")
     {
      double N5ZZ3=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift3);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      N5PointShift3++;
     }

   double N5ZZZ1 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift);
   double N5ZZZ2 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift2);
   double N5ZZZ3 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift3);

   double N5zigzagLongEntry=0;
   double N5zigzagShortEntry=0;

   if(N5ZZZ3<N5ZZZ2 && N5ZZZ2>N5ZZZ1 && N5ZZZ1>N5ZZZ3)
     {
      N5zigzagLongEntry=N5ZZZ2+(PipBuffer*Point);
      N5zigzagShortEntry=0;
     }

   if(N5ZZZ3>N5ZZZ2 && N5ZZZ2<N5ZZZ1 && N5ZZZ1<N5ZZZ3)
     {
      N5zigzagLongEntry=0;
      N5zigzagShortEntry=N5ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return N5ZZZ3;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN5ZZZ1ZigZagStopLossLevel()
  {



   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   int N5PointShift3=N5PointShift2;
   string N5ConfirmedPoint3="Not Found";

   while(N5ConfirmedPoint3!="Found")
     {
      double N5ZZ3=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift3);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      N5PointShift3++;
     }

   double N5ZZZ1 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift);
   double N5ZZZ2 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift2);
   double N5ZZZ3 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift3);

   double N5zigzagLongEntry=0;
   double N5zigzagShortEntry=0;

   if(N5ZZZ3<N5ZZZ2 && N5ZZZ2>N5ZZZ1 && N5ZZZ1>N5ZZZ3)
     {
      N5zigzagLongEntry=N5ZZZ2+(PipBuffer*Point);
      N5zigzagShortEntry=0;
     }

   if(N5ZZZ3>N5ZZZ2 && N5ZZZ2<N5ZZZ1 && N5ZZZ1<N5ZZZ3)
     {
      N5zigzagLongEntry=0;
      N5zigzagShortEntry=N5ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return N5ZZZ1;
  }






//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN15ZZZ3ZigZagStopLossLevel()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   int N15PointShift3=N15PointShift2;
   string N15ConfirmedPoint3="Not Found";

   while(N15ConfirmedPoint3!="Found")
     {
      double N15ZZ3=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift3);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      N15PointShift3++;
     }

   double N15ZZZ1 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift);
   double N15ZZZ2 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift2);
   double N15ZZZ3 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift3);

   double N15zigzagLongEntry=0;
   double N15zigzagShortEntry=0;

   if(N15ZZZ3<N15ZZZ2 && N15ZZZ2>N15ZZZ1 && N15ZZZ1>N15ZZZ3)
     {
      N15zigzagLongEntry=N15ZZZ2+(PipBuffer*Point);
      N15zigzagShortEntry=0;
     }

   if(N15ZZZ3>N15ZZZ2 && N15ZZZ2<N15ZZZ1 && N15ZZZ1<N15ZZZ3)
     {
      N15zigzagLongEntry=0;
      N15zigzagShortEntry=N15ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N15ZZZ3;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN15ZZZ1ZigZagStopLossLevel()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   int N15PointShift3=N15PointShift2;
   string N15ConfirmedPoint3="Not Found";

   while(N15ConfirmedPoint3!="Found")
     {
      double N15ZZ3=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift3);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      N15PointShift3++;
     }

   double N15ZZZ1 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift);
   double N15ZZZ2 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift2);
   double N15ZZZ3 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift3);

   double N15zigzagLongEntry=0;
   double N15zigzagShortEntry=0;

   if(N15ZZZ3<N15ZZZ2 && N15ZZZ2>N15ZZZ1 && N15ZZZ1>N15ZZZ3)
     {
      N15zigzagLongEntry=N15ZZZ2+(PipBuffer*Point);
      N15zigzagShortEntry=0;
     }

   if(N15ZZZ3>N15ZZZ2 && N15ZZZ2<N15ZZZ1 && N15ZZZ1<N15ZZZ3)
     {
      N15zigzagLongEntry=0;
      N15zigzagShortEntry=N15ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N15ZZZ1;
  }





//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN30ZZZ3ZigZagStopLossLevel()
  {
   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   int N30PointShift3=N30PointShift2;
   string N30ConfirmedPoint3="Not Found";

   while(N30ConfirmedPoint3!="Found")
     {
      double N30ZZ3=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift3);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      N30PointShift3++;
     }

   double N30ZZZ1 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift);
   double N30ZZZ2 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift2);
   double N30ZZZ3 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift3);

   double N30zigzagLongEntry=0;
   double N30zigzagShortEntry=0;

   if(N30ZZZ3<N30ZZZ2 && N30ZZZ2>N30ZZZ1 && N30ZZZ1>N30ZZZ3)
     {
      N30zigzagLongEntry=N30ZZZ2+(PipBuffer*Point);
      N30zigzagShortEntry=0;
     }

   if(N30ZZZ3>N30ZZZ2 && N30ZZZ2<N30ZZZ1 && N30ZZZ1<N30ZZZ3)
     {
      N30zigzagLongEntry=0;
      N30zigzagShortEntry=N30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N30ZZZ3;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN30ZZZ1ZigZagStopLossLevel()
  {
   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   int N30PointShift3=N30PointShift2;
   string N30ConfirmedPoint3="Not Found";

   while(N30ConfirmedPoint3!="Found")
     {
      double N30ZZ3=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift3);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      N30PointShift3++;
     }

   double N30ZZZ1 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift);
   double N30ZZZ2 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift2);
   double N30ZZZ3 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift3);

   double N30zigzagLongEntry=0;
   double N30zigzagShortEntry=0;

   if(N30ZZZ3<N30ZZZ2 && N30ZZZ2>N30ZZZ1 && N30ZZZ1>N30ZZZ3)
     {
      N30zigzagLongEntry=N30ZZZ2+(PipBuffer*Point);
      N30zigzagShortEntry=0;
     }

   if(N30ZZZ3>N30ZZZ2 && N30ZZZ2<N30ZZZ1 && N30ZZZ1<N30ZZZ3)
     {
      N30zigzagLongEntry=0;
      N30zigzagShortEntry=N30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N30ZZZ1;
  }





//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetG1ZZZ3ZigZagStopLossLevel()
  {
   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   int G1PointShift3=G1PointShift2;
   string G1ConfirmedPoint3="Not Found";

   while(G1ConfirmedPoint3!="Found")
     {
      double G1ZZ3=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift3);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      G1PointShift3++;
     }

   double G1ZZZ1 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift);
   double G1ZZZ2 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift2);
   double G1ZZZ3 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift3);

   double G1zigzagLongEntry=0;
   double G1zigzagShortEntry=0;

   if(G1ZZZ3<G1ZZZ2 && G1ZZZ2>G1ZZZ1 && G1ZZZ1>G1ZZZ3)
     {
      G1zigzagLongEntry=G1ZZZ2+(PipBuffer*Point);
      G1zigzagShortEntry=0;
     }

   if(G1ZZZ3>G1ZZZ2 && G1ZZZ2<G1ZZZ1 && G1ZZZ1<G1ZZZ3)
     {
      G1zigzagLongEntry=0;
      G1zigzagShortEntry=G1ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G1ZZZ3;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetG1ZZZ1ZigZagStopLossLevel()
  {
   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   int G1PointShift3=G1PointShift2;
   string G1ConfirmedPoint3="Not Found";

   while(G1ConfirmedPoint3!="Found")
     {
      double G1ZZ3=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift3);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      G1PointShift3++;
     }

   double G1ZZZ1 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift);
   double G1ZZZ2 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift2);
   double G1ZZZ3 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift3);

   double G1zigzagLongEntry=0;
   double G1zigzagShortEntry=0;

   if(G1ZZZ3<G1ZZZ2 && G1ZZZ2>G1ZZZ1 && G1ZZZ1>G1ZZZ3)
     {
      G1zigzagLongEntry=G1ZZZ2+(PipBuffer*Point);
      G1zigzagShortEntry=0;
     }

   if(G1ZZZ3>G1ZZZ2 && G1ZZZ2<G1ZZZ1 && G1ZZZ1<G1ZZZ3)
     {
      G1zigzagLongEntry=0;
      G1zigzagShortEntry=G1ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G1ZZZ1;
  }





//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetM5ZZZ3ZigZagStopLossLevel()
  {


   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   int M5PointShift3=M5PointShift2;
   string M5ConfirmedPoint3="Not Found";

   while(M5ConfirmedPoint3!="Found")
     {
      double M5ZZ3=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift3);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      M5PointShift3++;
     }

   double M5ZZZ1 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift);
   double M5ZZZ2 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift2);
   double M5ZZZ3 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift3);

   double M5zigzagLongEntry=0;
   double M5zigzagShortEntry=0;

   if(M5ZZZ3<M5ZZZ2 && M5ZZZ2>M5ZZZ1 && M5ZZZ1>=M5ZZZ3)
     {
      M5zigzagLongEntry=M5ZZZ2+(PipBuffer*Point);
      M5zigzagShortEntry=0;
     }

   if(M5ZZZ3>M5ZZZ2 && M5ZZZ2<M5ZZZ1 && M5ZZZ1<=M5ZZZ3)
     {
      M5zigzagLongEntry=0;
      M5zigzagShortEntry=M5ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return M5ZZZ3;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetM5ZZZ1ZigZagStopLossLevel()
  {


   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   int M5PointShift3=M5PointShift2;
   string M5ConfirmedPoint3="Not Found";

   while(M5ConfirmedPoint3!="Found")
     {
      double M5ZZ3=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift3);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      M5PointShift3++;
     }

   double M5ZZZ1 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift);
   double M5ZZZ2 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift2);
   double M5ZZZ3 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift3);

   double M5zigzagLongEntry=0;
   double M5zigzagShortEntry=0;

   if(M5ZZZ3<M5ZZZ2 && M5ZZZ2>M5ZZZ1 && M5ZZZ1>=M5ZZZ3)
     {
      M5zigzagLongEntry=M5ZZZ2+(PipBuffer*Point);
      M5zigzagShortEntry=0;
     }

   if(M5ZZZ3>M5ZZZ2 && M5ZZZ2<M5ZZZ1 && M5ZZZ1<=M5ZZZ3)
     {
      M5zigzagLongEntry=0;
      M5zigzagShortEntry=M5ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return M5ZZZ1;
  }











//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetM15ZZZ3ZigZagStopLossLevel()
  {
   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   int M15PointShift3=M15PointShift2;
   string M15ConfirmedPoint3="Not Found";

   while(M15ConfirmedPoint3!="Found")
     {
      double M15ZZ3=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift3);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      M15PointShift3++;
     }

   double M15ZZZ1 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift);
   double M15ZZZ2 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift2);
   double M15ZZZ3 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift3);

   double M15zigzagLongEntry=0;
   double M15zigzagShortEntry=0;

   if(M15ZZZ3<M15ZZZ2 && M15ZZZ2>M15ZZZ1 && M15ZZZ1>=M15ZZZ3)
     {
      M15zigzagLongEntry=M15ZZZ2+(PipBuffer*Point);
      M15zigzagShortEntry=0;
     }

   if(M15ZZZ3>M15ZZZ2 && M15ZZZ2<M15ZZZ1 && M15ZZZ1<=M15ZZZ3)
     {
      M15zigzagLongEntry=0;
      M15zigzagShortEntry=M15ZZZ2 -(PipBuffer*Point);
     }

// your full logic...
   return M15ZZZ3;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetM15ZZZ1ZigZagStopLossLevel()
  {
   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   int M15PointShift3=M15PointShift2;
   string M15ConfirmedPoint3="Not Found";

   while(M15ConfirmedPoint3!="Found")
     {
      double M15ZZ3=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift3);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      M15PointShift3++;
     }

   double M15ZZZ1 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift);
   double M15ZZZ2 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift2);
   double M15ZZZ3 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift3);

   double M15zigzagLongEntry=0;
   double M15zigzagShortEntry=0;

   if(M15ZZZ3<M15ZZZ2 && M15ZZZ2>M15ZZZ1 && M15ZZZ1>=M15ZZZ3)
     {
      M15zigzagLongEntry=M15ZZZ2+(PipBuffer*Point);
      M15zigzagShortEntry=0;
     }

   if(M15ZZZ3>M15ZZZ2 && M15ZZZ2<M15ZZZ1 && M15ZZZ1<=M15ZZZ3)
     {
      M15zigzagLongEntry=0;
      M15zigzagShortEntry=M15ZZZ2 -(PipBuffer*Point);
     }

// your full logic...
   return M15ZZZ1;
  }











//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetM30ZZZ3ZigZagStopLossLevel()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   int M30PointShift3=M30PointShift2;
   string M30ConfirmedPoint3="Not Found";

   while(M30ConfirmedPoint3!="Found")
     {
      double M30ZZ3=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift3);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      M30PointShift3++;
     }

   double M30ZZZ1 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift);
   double M30ZZZ2 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift2);
   double M30ZZZ3 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift3);

   double M30zigzagLongEntry=0;
   double M30zigzagShortEntry=0;

   if(M30ZZZ3<M30ZZZ2 && M30ZZZ2>M30ZZZ1 && M30ZZZ1>=M30ZZZ3)
     {
      M30zigzagLongEntry=M30ZZZ2+(PipBuffer*Point);
      M30zigzagShortEntry=0;
     }

   if(M30ZZZ3>M30ZZZ2 && M30ZZZ2<M30ZZZ1 && M30ZZZ1<=M30ZZZ3)
     {
      M30zigzagLongEntry=0;
      M30zigzagShortEntry=M30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return M30ZZZ3;
  }






//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetM30ZZZ1ZigZagStopLossLevel()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   int M30PointShift3=M30PointShift2;
   string M30ConfirmedPoint3="Not Found";

   while(M30ConfirmedPoint3!="Found")
     {
      double M30ZZ3=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift3);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      M30PointShift3++;
     }

   double M30ZZZ1 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift);
   double M30ZZZ2 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift2);
   double M30ZZZ3 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift3);

   double M30zigzagLongEntry=0;
   double M30zigzagShortEntry=0;

   if(M30ZZZ3<M30ZZZ2 && M30ZZZ2>M30ZZZ1 && M30ZZZ1>=M30ZZZ3)
     {
      M30zigzagLongEntry=M30ZZZ2+(PipBuffer*Point);
      M30zigzagShortEntry=0;
     }

   if(M30ZZZ3>M30ZZZ2 && M30ZZZ2<M30ZZZ1 && M30ZZZ1<=M30ZZZ3)
     {
      M30zigzagLongEntry=0;
      M30zigzagShortEntry=M30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return M30ZZZ1;
  }





//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetH1ZZZ3ZigZagStopLossLevel()
  {
   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   int H1PointShift3=H1PointShift2;
   string H1ConfirmedPoint3="Not Found";

   while(H1ConfirmedPoint3!="Found")
     {
      double H1ZZ3=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift3);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      H1PointShift3++;
     }

   double H1ZZZ1 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift);
   double H1ZZZ2 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift2);
   double H1ZZZ3 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift3);

   double H1zigzagLongEntry=0;
   double H1zigzagShortEntry=0;

   if(H1ZZZ3<H1ZZZ2 && H1ZZZ2>H1ZZZ1 && H1ZZZ1>=H1ZZZ3)
     {
      H1zigzagLongEntry=H1ZZZ2+(PipBuffer*Point);
      H1zigzagShortEntry=0;
     }

   if(H1ZZZ3>H1ZZZ2 && H1ZZZ2<H1ZZZ1 && H1ZZZ1<=H1ZZZ3)
     {
      H1zigzagLongEntry=0;
      H1zigzagShortEntry=H1ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return H1ZZZ3;
  }





//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetH1ZZZ1ZigZagStopLossLevel()
  {
   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   int H1PointShift3=H1PointShift2;
   string H1ConfirmedPoint3="Not Found";

   while(H1ConfirmedPoint3!="Found")
     {
      double H1ZZ3=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift3);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      H1PointShift3++;
     }

   double H1ZZZ1 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift);
   double H1ZZZ2 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift2);
   double H1ZZZ3 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift3);

   double H1zigzagLongEntry=0;
   double H1zigzagShortEntry=0;

   if(H1ZZZ3<H1ZZZ2 && H1ZZZ2>H1ZZZ1 && H1ZZZ1>=H1ZZZ3)
     {
      H1zigzagLongEntry=H1ZZZ2+(PipBuffer*Point);
      H1zigzagShortEntry=0;
     }

   if(H1ZZZ3>H1ZZZ2 && H1ZZZ2<H1ZZZ1 && H1ZZZ1<=H1ZZZ3)
     {
      H1zigzagLongEntry=0;
      H1zigzagShortEntry=H1ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return H1ZZZ1;
  }
  
  
 
 
 
 
 
 //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 
double GBPUSD_GetZZZ3ZigZagStopLossLevel()
{
   
   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ || iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   int PointShift3=PointShift2;
   string ConfirmedPoint3="Not Found";

   while(ConfirmedPoint3!="Found")
     {
      double ZZ3=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift3);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint3="Found";
         break;
        }
      PointShift3++;
     }

   double ZZZ1 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift);
   double ZZZ2 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift2);
   double ZZZ3 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift3);

   double zigzagLongEntry=0;
   double zigzagShortEntry=0;

   if(ZZZ3<ZZZ2 && ZZZ2>ZZZ1 && ZZZ1>=ZZZ3)
     {
      zigzagLongEntry=ZZZ2+(PipBuffer*Point);
      zigzagShortEntry=0;
     }

   if(ZZZ3>ZZZ2 && ZZZ2<ZZZ1 && ZZZ1<=ZZZ3)
     {
      zigzagLongEntry=0;
      zigzagShortEntry=ZZZ2 -(PipBuffer*Point);
     }


   
   
   // your full logic...
   return ZZZ3;
}



//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


 
 //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 
double GBPUSD_GetZZZ1ZigZagStopLossLevel()
{
   
   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ || iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   int PointShift3=PointShift2;
   string ConfirmedPoint3="Not Found";

   while(ConfirmedPoint3!="Found")
     {
      double ZZ3=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift3);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint3="Found";
         break;
        }
      PointShift3++;
     }

   double ZZZ1 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift);
   double ZZZ2 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift2);
   double ZZZ3 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift3);

   double zigzagLongEntry=0;
   double zigzagShortEntry=0;

   if(ZZZ3<ZZZ2 && ZZZ2>ZZZ1 && ZZZ1>=ZZZ3)
     {
      zigzagLongEntry=ZZZ2+(PipBuffer*Point);
      zigzagShortEntry=0;
     }

   if(ZZZ3>ZZZ2 && ZZZ2<ZZZ1 && ZZZ1<=ZZZ3)
     {
      zigzagLongEntry=0;
      zigzagShortEntry=ZZZ2 -(PipBuffer*Point);
     }


   
   
   // your full logic...
   return ZZZ1;
}



//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 
 //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 
double GBPUSD_GetZZZ2ZigZagStopLossLevel()
{
   
   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ || iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   int PointShift3=PointShift2;
   string ConfirmedPoint3="Not Found";

   while(ConfirmedPoint3!="Found")
     {
      double ZZ3=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift3);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint3="Found";
         break;
        }
      PointShift3++;
     }

   double ZZZ1 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift);
   double ZZZ2 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift2);
   double ZZZ3 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift3);

   double zigzagLongEntry=0;
   double zigzagShortEntry=0;

   if(ZZZ3<ZZZ2 && ZZZ2>ZZZ1 && ZZZ1>=ZZZ3)
     {
      zigzagLongEntry=ZZZ2+(PipBuffer*Point);
      zigzagShortEntry=0;
     }

   if(ZZZ3>ZZZ2 && ZZZ2<ZZZ1 && ZZZ1<=ZZZ3)
     {
      zigzagLongEntry=0;
      zigzagShortEntry=ZZZ2 -(PipBuffer*Point);
     }


   
   
   // your full logic...
   return ZZZ2;
}



//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


  //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetG4ZZZ3ZigZagStopLossLevel()
  {
   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   int G4PointShift3=G4PointShift2;
   string G4ConfirmedPoint3="Not Found";

   while(G4ConfirmedPoint3!="Found")
     {
      double G4ZZ3=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift3);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      G4PointShift3++;
     }

   double G4ZZZ1 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift);
   double G4ZZZ2 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift2);
   double G4ZZZ3 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift3);

   double G4zigzagLongEntry=0;
   double G4zigzagShortEntry=0;

   if(G4ZZZ3<G4ZZZ2 && G4ZZZ2>G4ZZZ1 && G4ZZZ1>G4ZZZ3)
     {
      G4zigzagLongEntry=G4ZZZ2+(PipBuffer*Point);
      G4zigzagShortEntry=0;
     }

   if(G4ZZZ3>G4ZZZ2 && G4ZZZ2<G4ZZZ1 && G4ZZZ1<G4ZZZ3)
     {
      G4zigzagLongEntry=0;
      G4zigzagShortEntry=G4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G4ZZZ3;
  }




//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


  //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetG4ZZZ1ZigZagStopLossLevel()
  {
   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   int G4PointShift3=G4PointShift2;
   string G4ConfirmedPoint3="Not Found";

   while(G4ConfirmedPoint3!="Found")
     {
      double G4ZZ3=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift3);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      G4PointShift3++;
     }

   double G4ZZZ1 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift);
   double G4ZZZ2 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift2);
   double G4ZZZ3 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift3);

   double G4zigzagLongEntry=0;
   double G4zigzagShortEntry=0;

   if(G4ZZZ3<G4ZZZ2 && G4ZZZ2>G4ZZZ1 && G4ZZZ1>G4ZZZ3)
     {
      G4zigzagLongEntry=G4ZZZ2+(PipBuffer*Point);
      G4zigzagShortEntry=0;
     }

   if(G4ZZZ3>G4ZZZ2 && G4ZZZ2<G4ZZZ1 && G4ZZZ1<G4ZZZ3)
     {
      G4zigzagLongEntry=0;
      G4zigzagShortEntry=G4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G4ZZZ1;
  }





  //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetG4ZZZ2ZigZagStopLossLevel()
  {
   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   int G4PointShift3=G4PointShift2;
   string G4ConfirmedPoint3="Not Found";

   while(G4ConfirmedPoint3!="Found")
     {
      double G4ZZ3=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift3);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      G4PointShift3++;
     }

   double G4ZZZ1 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift);
   double G4ZZZ2 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift2);
   double G4ZZZ3 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift3);

   double G4zigzagLongEntry=0;
   double G4zigzagShortEntry=0;

   if(G4ZZZ3<G4ZZZ2 && G4ZZZ2>G4ZZZ1 && G4ZZZ1>G4ZZZ3)
     {
      G4zigzagLongEntry=G4ZZZ2+(PipBuffer*Point);
      G4zigzagShortEntry=0;
     }

   if(G4ZZZ3>G4ZZZ2 && G4ZZZ2<G4ZZZ1 && G4ZZZ1<G4ZZZ3)
     {
      G4zigzagLongEntry=0;
      G4zigzagShortEntry=G4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G4ZZZ2;
  }




//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


 //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetH4ZZZ3ZigZagStopLossLevel()
  {
   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   int H4PointShift3=H4PointShift2;
   string H4ConfirmedPoint3="Not Found";

   while(H4ConfirmedPoint3!="Found")
     {
      double H4ZZ3=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift3);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      H4PointShift3++;
     }

   double H4ZZZ1 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift);
   double H4ZZZ2 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift2);
   double H4ZZZ3 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift3);

   double H4zigzagLongEntry=0;
   double H4zigzagShortEntry=0;

   if(H4ZZZ3<H4ZZZ2 && H4ZZZ2>H4ZZZ1 && H4ZZZ1>H4ZZZ3)
     {
      H4zigzagLongEntry=H4ZZZ2+(PipBuffer*Point);
      H4zigzagShortEntry=0;
     }

   if(H4ZZZ3>H4ZZZ2 && H4ZZZ2<H4ZZZ1 && H4ZZZ1<H4ZZZ3)
     {
      H4zigzagLongEntry=0;
      H4zigzagShortEntry=H4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return H4ZZZ3;
  }


//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetH4ZZZ1ZigZagStopLossLevel()
  {
   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   int H4PointShift3=H4PointShift2;
   string H4ConfirmedPoint3="Not Found";

   while(H4ConfirmedPoint3!="Found")
     {
      double H4ZZ3=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift3);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      H4PointShift3++;
     }

   double H4ZZZ1 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift);
   double H4ZZZ2 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift2);
   double H4ZZZ3 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift3);

   double H4zigzagLongEntry=0;
   double H4zigzagShortEntry=0;

   if(H4ZZZ3<H4ZZZ2 && H4ZZZ2>H4ZZZ1 && H4ZZZ1>H4ZZZ3)
     {
      H4zigzagLongEntry=H4ZZZ2+(PipBuffer*Point);
      H4zigzagShortEntry=0;
     }

   if(H4ZZZ3>H4ZZZ2 && H4ZZZ2<H4ZZZ1 && H4ZZZ1<H4ZZZ3)
     {
      H4zigzagLongEntry=0;
      H4zigzagShortEntry=H4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return H4ZZZ1;
  }






 //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetH4ZZZ2ZigZagStopLossLevel()
  {
   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   int H4PointShift3=H4PointShift2;
   string H4ConfirmedPoint3="Not Found";

   while(H4ConfirmedPoint3!="Found")
     {
      double H4ZZ3=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift3);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      H4PointShift3++;
     }

   double H4ZZZ1 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift);
   double H4ZZZ2 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift2);
   double H4ZZZ3 = iCustom("GBPUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift3);

   double H4zigzagLongEntry=0;
   double H4zigzagShortEntry=0;

   if(H4ZZZ3<H4ZZZ2 && H4ZZZ2>H4ZZZ1 && H4ZZZ1>H4ZZZ3)
     {
      H4zigzagLongEntry=H4ZZZ2+(PipBuffer*Point);
      H4zigzagShortEntry=0;
     }

   if(H4ZZZ3>H4ZZZ2 && H4ZZZ2<H4ZZZ1 && H4ZZZ1<H4ZZZ3)
     {
      H4zigzagLongEntry=0;
      H4zigzagShortEntry=H4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return H4ZZZ2;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetM5ZZZ2ZigZagStopLossLevel()
  {


   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   int M5PointShift3=M5PointShift2;
   string M5ConfirmedPoint3="Not Found";

   while(M5ConfirmedPoint3!="Found")
     {
      double M5ZZ3=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift3);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      M5PointShift3++;
     }

   double M5ZZZ1 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift);
   double M5ZZZ2 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift2);
   double M5ZZZ3 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift3);

   double M5zigzagLongEntry=0;
   double M5zigzagShortEntry=0;

   if(M5ZZZ3<M5ZZZ2 && M5ZZZ2>M5ZZZ1 && M5ZZZ1>=M5ZZZ3)
     {
      M5zigzagLongEntry=M5ZZZ2+(PipBuffer*Point);
      M5zigzagShortEntry=0;
     }

   if(M5ZZZ3>M5ZZZ2 && M5ZZZ2<M5ZZZ1 && M5ZZZ1<=M5ZZZ3)
     {
      M5zigzagLongEntry=0;
      M5zigzagShortEntry=M5ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return M5ZZZ2;
  }
  
  


   
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetM15ZZZ2ZigZagStopLossLevel()
  {
   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   int M15PointShift3=M15PointShift2;
   string M15ConfirmedPoint3="Not Found";

   while(M15ConfirmedPoint3!="Found")
     {
      double M15ZZ3=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift3);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      M15PointShift3++;
     }

   double M15ZZZ1 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift);
   double M15ZZZ2 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift2);
   double M15ZZZ3 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift3);

   double M15zigzagLongEntry=0;
   double M15zigzagShortEntry=0;

   if(M15ZZZ3<M15ZZZ2 && M15ZZZ2>M15ZZZ1 && M15ZZZ1>=M15ZZZ3)
     {
      M15zigzagLongEntry=M15ZZZ2+(PipBuffer*Point);
      M15zigzagShortEntry=0;
     }

   if(M15ZZZ3>M15ZZZ2 && M15ZZZ2<M15ZZZ1 && M15ZZZ1<=M15ZZZ3)
     {
      M15zigzagLongEntry=0;
      M15zigzagShortEntry=M15ZZZ2 -(PipBuffer*Point);
     }

// your full logic...
   return M15ZZZ2;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetM30ZZZ2ZigZagStopLossLevel()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   int M30PointShift3=M30PointShift2;
   string M30ConfirmedPoint3="Not Found";

   while(M30ConfirmedPoint3!="Found")
     {
      double M30ZZ3=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift3);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      M30PointShift3++;
     }

   double M30ZZZ1 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift);
   double M30ZZZ2 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift2);
   double M30ZZZ3 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift3);

   double M30zigzagLongEntry=0;
   double M30zigzagShortEntry=0;

   if(M30ZZZ3<M30ZZZ2 && M30ZZZ2>M30ZZZ1 && M30ZZZ1>=M30ZZZ3)
     {
      M30zigzagLongEntry=M30ZZZ2+(PipBuffer*Point);
      M30zigzagShortEntry=0;
     }

   if(M30ZZZ3>M30ZZZ2 && M30ZZZ2<M30ZZZ1 && M30ZZZ1<=M30ZZZ3)
     {
      M30zigzagLongEntry=0;
      M30zigzagShortEntry=M30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return M30ZZZ2;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetH1ZZZ2ZigZagStopLossLevel()
  {
   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   int H1PointShift3=H1PointShift2;
   string H1ConfirmedPoint3="Not Found";

   while(H1ConfirmedPoint3!="Found")
     {
      double H1ZZ3=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift3);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      H1PointShift3++;
     }

   double H1ZZZ1 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift);
   double H1ZZZ2 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift2);
   double H1ZZZ3 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift3);

   double H1zigzagLongEntry=0;
   double H1zigzagShortEntry=0;

   if(H1ZZZ3<H1ZZZ2 && H1ZZZ2>H1ZZZ1 && H1ZZZ1>=H1ZZZ3)
     {
      H1zigzagLongEntry=H1ZZZ2+(PipBuffer*Point);
      H1zigzagShortEntry=0;
     }

   if(H1ZZZ3>H1ZZZ2 && H1ZZZ2<H1ZZZ1 && H1ZZZ1<=H1ZZZ3)
     {
      H1zigzagLongEntry=0;
      H1zigzagShortEntry=H1ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return H1ZZZ2;
  }



/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////



//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN1ZZZ2ZigZagStopLossLevel()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   int N1PointShift3=N1PointShift2;
   string N1ConfirmedPoint3="Not Found";

   while(N1ConfirmedPoint3!="Found")
     {
      double N1ZZ3=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift3);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      N1PointShift3++;
     }

   double N1ZZZ1 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift);
   double N1ZZZ2 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift2);
   double N1ZZZ3 = iCustom("GBPUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift3);

   double N1zigzagLongEntry=0;
   double N1zigzagShortEntry=0;

   if(N1ZZZ3<N1ZZZ2 && N1ZZZ2>N1ZZZ1 && N1ZZZ1>N1ZZZ3)
     {
      N1zigzagLongEntry=N1ZZZ2+(PipBuffer*Point);
      N1zigzagShortEntry=0;
     }

   if(N1ZZZ3>N1ZZZ2 && N1ZZZ2<N1ZZZ1 && N1ZZZ1<N1ZZZ3)
     {
      N1zigzagLongEntry=0;
      N1zigzagShortEntry=N1ZZZ2 -(PipBuffer*Point);
     }




// your full logic...
   return N1ZZZ2;
  }








//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
 





//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN5ZZZ2ZigZagStopLossLevel()
  {



   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   int N5PointShift3=N5PointShift2;
   string N5ConfirmedPoint3="Not Found";

   while(N5ConfirmedPoint3!="Found")
     {
      double N5ZZ3=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift3);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      N5PointShift3++;
     }

   double N5ZZZ1 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift);
   double N5ZZZ2 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift2);
   double N5ZZZ3 = iCustom("GBPUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift3);

   double N5zigzagLongEntry=0;
   double N5zigzagShortEntry=0;

   if(N5ZZZ3<N5ZZZ2 && N5ZZZ2>N5ZZZ1 && N5ZZZ1>N5ZZZ3)
     {
      N5zigzagLongEntry=N5ZZZ2+(PipBuffer*Point);
      N5zigzagShortEntry=0;
     }

   if(N5ZZZ3>N5ZZZ2 && N5ZZZ2<N5ZZZ1 && N5ZZZ1<N5ZZZ3)
     {
      N5zigzagLongEntry=0;
      N5zigzagShortEntry=N5ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return N5ZZZ2;
  }





//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
 
 
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN15ZZZ2ZigZagStopLossLevel()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   int N15PointShift3=N15PointShift2;
   string N15ConfirmedPoint3="Not Found";

   while(N15ConfirmedPoint3!="Found")
     {
      double N15ZZ3=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift3);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      N15PointShift3++;
     }

   double N15ZZZ1 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift);
   double N15ZZZ2 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift2);
   double N15ZZZ3 = iCustom("GBPUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift3);

   double N15zigzagLongEntry=0;
   double N15zigzagShortEntry=0;

   if(N15ZZZ3<N15ZZZ2 && N15ZZZ2>N15ZZZ1 && N15ZZZ1>N15ZZZ3)
     {
      N15zigzagLongEntry=N15ZZZ2+(PipBuffer*Point);
      N15zigzagShortEntry=0;
     }

   if(N15ZZZ3>N15ZZZ2 && N15ZZZ2<N15ZZZ1 && N15ZZZ1<N15ZZZ3)
     {
      N15zigzagLongEntry=0;
      N15zigzagShortEntry=N15ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N15ZZZ2;
  }



 
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetN30ZZZ2ZigZagStopLossLevel()
  {
   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   int N30PointShift3=N30PointShift2;
   string N30ConfirmedPoint3="Not Found";

   while(N30ConfirmedPoint3!="Found")
     {
      double N30ZZ3=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift3);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      N30PointShift3++;
     }

   double N30ZZZ1 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift);
   double N30ZZZ2 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift2);
   double N30ZZZ3 = iCustom("GBPUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift3);

   double N30zigzagLongEntry=0;
   double N30zigzagShortEntry=0;

   if(N30ZZZ3<N30ZZZ2 && N30ZZZ2>N30ZZZ1 && N30ZZZ1>N30ZZZ3)
     {
      N30zigzagLongEntry=N30ZZZ2+(PipBuffer*Point);
      N30zigzagShortEntry=0;
     }

   if(N30ZZZ3>N30ZZZ2 && N30ZZZ2<N30ZZZ1 && N30ZZZ1<N30ZZZ3)
     {
      N30zigzagLongEntry=0;
      N30zigzagShortEntry=N30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N30ZZZ2;
  }




//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double GBPUSD_GetG1ZZZ2ZigZagStopLossLevel()
  {
   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   int G1PointShift3=G1PointShift2;
   string G1ConfirmedPoint3="Not Found";

   while(G1ConfirmedPoint3!="Found")
     {
      double G1ZZ3=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift3);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      G1PointShift3++;
     }

   double G1ZZZ1 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift);
   double G1ZZZ2 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift2);
   double G1ZZZ3 = iCustom("GBPUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift3);

   double G1zigzagLongEntry=0;
   double G1zigzagShortEntry=0;

   if(G1ZZZ3<G1ZZZ2 && G1ZZZ2>G1ZZZ1 && G1ZZZ1>G1ZZZ3)
     {
      G1zigzagLongEntry=G1ZZZ2+(PipBuffer*Point);
      G1zigzagShortEntry=0;
     }

   if(G1ZZZ3>G1ZZZ2 && G1ZZZ2<G1ZZZ1 && G1ZZZ1<G1ZZZ3)
     {
      G1zigzagLongEntry=0;
      G1zigzagShortEntry=G1ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G1ZZZ2;
  }




  
  
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetM5ZZZ1ZigZagStopLossLevel()
  {


   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   int M5PointShift3=M5PointShift2;
   string M5ConfirmedPoint3="Not Found";

   while(M5ConfirmedPoint3!="Found")
     {
      double M5ZZ3=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift3);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      M5PointShift3++;
     }

   double M5ZZZ1 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift);
   double M5ZZZ2 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift2);
   double M5ZZZ3 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift3);

   double M5zigzagLongEntry=0;
   double M5zigzagShortEntry=0;

   if(M5ZZZ3<M5ZZZ2 && M5ZZZ2>M5ZZZ1 && M5ZZZ1>=M5ZZZ3)
     {
      M5zigzagLongEntry=M5ZZZ2+(PipBuffer*Point);
      M5zigzagShortEntry=0;
     }

   if(M5ZZZ3>M5ZZZ2 && M5ZZZ2<M5ZZZ1 && M5ZZZ1<=M5ZZZ3)
     {
      M5zigzagLongEntry=0;
      M5zigzagShortEntry=M5ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return M5ZZZ1;
  }



 



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetM5ZZZ3ZigZagStopLossLevel()
  {


   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   int M5PointShift3=M5PointShift2;
   string M5ConfirmedPoint3="Not Found";

   while(M5ConfirmedPoint3!="Found")
     {
      double M5ZZ3=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift3);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      M5PointShift3++;
     }

   double M5ZZZ1 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift);
   double M5ZZZ2 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift2);
   double M5ZZZ3 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift3);

   double M5zigzagLongEntry=0;
   double M5zigzagShortEntry=0;

   if(M5ZZZ3<M5ZZZ2 && M5ZZZ2>M5ZZZ1 && M5ZZZ1>=M5ZZZ3)
     {
      M5zigzagLongEntry=M5ZZZ2+(PipBuffer*Point);
      M5zigzagShortEntry=0;
     }

   if(M5ZZZ3>M5ZZZ2 && M5ZZZ2<M5ZZZ1 && M5ZZZ1<=M5ZZZ3)
     {
      M5zigzagLongEntry=0;
      M5zigzagShortEntry=M5ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return M5ZZZ3;
  }




//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetM15ZZZ1ZigZagStopLossLevel()
  {
   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   int M15PointShift3=M15PointShift2;
   string M15ConfirmedPoint3="Not Found";

   while(M15ConfirmedPoint3!="Found")
     {
      double M15ZZ3=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift3);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      M15PointShift3++;
     }

   double M15ZZZ1 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift);
   double M15ZZZ2 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift2);
   double M15ZZZ3 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift3);

   double M15zigzagLongEntry=0;
   double M15zigzagShortEntry=0;

   if(M15ZZZ3<M15ZZZ2 && M15ZZZ2>M15ZZZ1 && M15ZZZ1>=M15ZZZ3)
     {
      M15zigzagLongEntry=M15ZZZ2+(PipBuffer*Point);
      M15zigzagShortEntry=0;
     }

   if(M15ZZZ3>M15ZZZ2 && M15ZZZ2<M15ZZZ1 && M15ZZZ1<=M15ZZZ3)
     {
      M15zigzagLongEntry=0;
      M15zigzagShortEntry=M15ZZZ2 -(PipBuffer*Point);
     }

// your full logic...
   return M15ZZZ1;
  }




//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetM15ZZZ3ZigZagStopLossLevel()
  {
   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   int M15PointShift3=M15PointShift2;
   string M15ConfirmedPoint3="Not Found";

   while(M15ConfirmedPoint3!="Found")
     {
      double M15ZZ3=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift3);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      M15PointShift3++;
     }

   double M15ZZZ1 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift);
   double M15ZZZ2 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift2);
   double M15ZZZ3 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift3);

   double M15zigzagLongEntry=0;
   double M15zigzagShortEntry=0;

   if(M15ZZZ3<M15ZZZ2 && M15ZZZ2>M15ZZZ1 && M15ZZZ1>=M15ZZZ3)
     {
      M15zigzagLongEntry=M15ZZZ2+(PipBuffer*Point);
      M15zigzagShortEntry=0;
     }

   if(M15ZZZ3>M15ZZZ2 && M15ZZZ2<M15ZZZ1 && M15ZZZ1<=M15ZZZ3)
     {
      M15zigzagLongEntry=0;
      M15zigzagShortEntry=M15ZZZ2 -(PipBuffer*Point);
     }

// your full logic...
   return M15ZZZ3;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetM30ZZZ1ZigZagStopLossLevel()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   int M30PointShift3=M30PointShift2;
   string M30ConfirmedPoint3="Not Found";

   while(M30ConfirmedPoint3!="Found")
     {
      double M30ZZ3=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift3);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      M30PointShift3++;
     }

   double M30ZZZ1 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift);
   double M30ZZZ2 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift2);
   double M30ZZZ3 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift3);

   double M30zigzagLongEntry=0;
   double M30zigzagShortEntry=0;

   if(M30ZZZ3<M30ZZZ2 && M30ZZZ2>M30ZZZ1 && M30ZZZ1>=M30ZZZ3)
     {
      M30zigzagLongEntry=M30ZZZ2+(PipBuffer*Point);
      M30zigzagShortEntry=0;
     }

   if(M30ZZZ3>M30ZZZ2 && M30ZZZ2<M30ZZZ1 && M30ZZZ1<=M30ZZZ3)
     {
      M30zigzagLongEntry=0;
      M30zigzagShortEntry=M30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return M30ZZZ1;
  }




//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetM30ZZZ3ZigZagStopLossLevel()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   int M30PointShift3=M30PointShift2;
   string M30ConfirmedPoint3="Not Found";

   while(M30ConfirmedPoint3!="Found")
     {
      double M30ZZ3=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift3);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      M30PointShift3++;
     }

   double M30ZZZ1 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift);
   double M30ZZZ2 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift2);
   double M30ZZZ3 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift3);

   double M30zigzagLongEntry=0;
   double M30zigzagShortEntry=0;

   if(M30ZZZ3<M30ZZZ2 && M30ZZZ2>M30ZZZ1 && M30ZZZ1>=M30ZZZ3)
     {
      M30zigzagLongEntry=M30ZZZ2+(PipBuffer*Point);
      M30zigzagShortEntry=0;
     }

   if(M30ZZZ3>M30ZZZ2 && M30ZZZ2<M30ZZZ1 && M30ZZZ1<=M30ZZZ3)
     {
      M30zigzagLongEntry=0;
      M30zigzagShortEntry=M30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return M30ZZZ3;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetH1ZZZ1ZigZagStopLossLevel()
  {
   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   int H1PointShift3=H1PointShift2;
   string H1ConfirmedPoint3="Not Found";

   while(H1ConfirmedPoint3!="Found")
     {
      double H1ZZ3=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift3);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      H1PointShift3++;
     }

   double H1ZZZ1 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift);
   double H1ZZZ2 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift2);
   double H1ZZZ3 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift3);

   double H1zigzagLongEntry=0;
   double H1zigzagShortEntry=0;

   if(H1ZZZ3<H1ZZZ2 && H1ZZZ2>H1ZZZ1 && H1ZZZ1>=H1ZZZ3)
     {
      H1zigzagLongEntry=H1ZZZ2+(PipBuffer*Point);
      H1zigzagShortEntry=0;
     }

   if(H1ZZZ3>H1ZZZ2 && H1ZZZ2<H1ZZZ1 && H1ZZZ1<=H1ZZZ3)
     {
      H1zigzagLongEntry=0;
      H1zigzagShortEntry=H1ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return H1ZZZ1;
  }
  
  
 
 
 
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetH1ZZZ3ZigZagStopLossLevel()
  {
   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   int H1PointShift3=H1PointShift2;
   string H1ConfirmedPoint3="Not Found";

   while(H1ConfirmedPoint3!="Found")
     {
      double H1ZZ3=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift3);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      H1PointShift3++;
     }

   double H1ZZZ1 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift);
   double H1ZZZ2 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift2);
   double H1ZZZ3 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift3);

   double H1zigzagLongEntry=0;
   double H1zigzagShortEntry=0;

   if(H1ZZZ3<H1ZZZ2 && H1ZZZ2>H1ZZZ1 && H1ZZZ1>=H1ZZZ3)
     {
      H1zigzagLongEntry=H1ZZZ2+(PipBuffer*Point);
      H1zigzagShortEntry=0;
     }

   if(H1ZZZ3>H1ZZZ2 && H1ZZZ2<H1ZZZ1 && H1ZZZ1<=H1ZZZ3)
     {
      H1zigzagLongEntry=0;
      H1zigzagShortEntry=H1ZZZ2 -(PipBuffer*Point);
     }



// your full logic...
   return H1ZZZ3;
  }




 
 //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetH4ZZZ3ZigZagStopLossLevel()
  {
   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   int H4PointShift3=H4PointShift2;
   string H4ConfirmedPoint3="Not Found";

   while(H4ConfirmedPoint3!="Found")
     {
      double H4ZZ3=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift3);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      H4PointShift3++;
     }

   double H4ZZZ1 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift);
   double H4ZZZ2 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift2);
   double H4ZZZ3 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift3);

   double H4zigzagLongEntry=0;
   double H4zigzagShortEntry=0;

   if(H4ZZZ3<H4ZZZ2 && H4ZZZ2>H4ZZZ1 && H4ZZZ1>H4ZZZ3)
     {
      H4zigzagLongEntry=H4ZZZ2+(PipBuffer*Point);
      H4zigzagShortEntry=0;
     }

   if(H4ZZZ3>H4ZZZ2 && H4ZZZ2<H4ZZZ1 && H4ZZZ1<H4ZZZ3)
     {
      H4zigzagLongEntry=0;
      H4zigzagShortEntry=H4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return H4ZZZ3;
  }


 //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN5ZZZ1ZigZagStopLossLevel()
  {



   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   int N5PointShift3=N5PointShift2;
   string N5ConfirmedPoint3="Not Found";

   while(N5ConfirmedPoint3!="Found")
     {
      double N5ZZ3=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift3);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      N5PointShift3++;
     }

   double N5ZZZ1 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift);
   double N5ZZZ2 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift2);
   double N5ZZZ3 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift3);

   double N5zigzagLongEntry=0;
   double N5zigzagShortEntry=0;

   if(N5ZZZ3<N5ZZZ2 && N5ZZZ2>N5ZZZ1 && N5ZZZ1>N5ZZZ3)
     {
      N5zigzagLongEntry=N5ZZZ2+(PipBuffer*Point);
      N5zigzagShortEntry=0;
     }

   if(N5ZZZ3>N5ZZZ2 && N5ZZZ2<N5ZZZ1 && N5ZZZ1<N5ZZZ3)
     {
      N5zigzagLongEntry=0;
      N5zigzagShortEntry=N5ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return N5ZZZ1;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN1ZZZ1ZigZagStopLossLevel()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   int N1PointShift3=N1PointShift2;
   string N1ConfirmedPoint3="Not Found";

   while(N1ConfirmedPoint3!="Found")
     {
      double N1ZZ3=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift3);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      N1PointShift3++;
     }

   double N1ZZZ1 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift);
   double N1ZZZ2 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift2);
   double N1ZZZ3 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift3);

   double N1zigzagLongEntry=0;
   double N1zigzagShortEntry=0;

   if(N1ZZZ3<N1ZZZ2 && N1ZZZ2>N1ZZZ1 && N1ZZZ1>N1ZZZ3)
     {
      N1zigzagLongEntry=N1ZZZ2+(PipBuffer*Point);
      N1zigzagShortEntry=0;
     }

   if(N1ZZZ3>N1ZZZ2 && N1ZZZ2<N1ZZZ1 && N1ZZZ1<N1ZZZ3)
     {
      N1zigzagLongEntry=0;
      N1zigzagShortEntry=N1ZZZ2 -(PipBuffer*Point);
     }




// your full logic...
   return N1ZZZ1;
  }


  
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN1ZZZ3ZigZagStopLossLevel()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   int N1PointShift3=N1PointShift2;
   string N1ConfirmedPoint3="Not Found";

   while(N1ConfirmedPoint3!="Found")
     {
      double N1ZZ3=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift3);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      N1PointShift3++;
     }

   double N1ZZZ1 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift);
   double N1ZZZ2 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift2);
   double N1ZZZ3 = iCustom("EURUSD", PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift3);

   double N1zigzagLongEntry=0;
   double N1zigzagShortEntry=0;

   if(N1ZZZ3<N1ZZZ2 && N1ZZZ2>N1ZZZ1 && N1ZZZ1>N1ZZZ3)
     {
      N1zigzagLongEntry=N1ZZZ2+(PipBuffer*Point);
      N1zigzagShortEntry=0;
     }

   if(N1ZZZ3>N1ZZZ2 && N1ZZZ2<N1ZZZ1 && N1ZZZ1<N1ZZZ3)
     {
      N1zigzagLongEntry=0;
      N1zigzagShortEntry=N1ZZZ2 -(PipBuffer*Point);
     }




// your full logic...
   return N1ZZZ3;
  }



 
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN5ZZZ3ZigZagStopLossLevel()
  {



   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   int N5PointShift3=N5PointShift2;
   string N5ConfirmedPoint3="Not Found";

   while(N5ConfirmedPoint3!="Found")
     {
      double N5ZZ3=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift3);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      N5PointShift3++;
     }

   double N5ZZZ1 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift);
   double N5ZZZ2 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift2);
   double N5ZZZ3 = iCustom("EURUSD",  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift3);

   double N5zigzagLongEntry=0;
   double N5zigzagShortEntry=0;

   if(N5ZZZ3<N5ZZZ2 && N5ZZZ2>N5ZZZ1 && N5ZZZ1>N5ZZZ3)
     {
      N5zigzagLongEntry=N5ZZZ2+(PipBuffer*Point);
      N5zigzagShortEntry=0;
     }

   if(N5ZZZ3>N5ZZZ2 && N5ZZZ2<N5ZZZ1 && N5ZZZ1<N5ZZZ3)
     {
      N5zigzagLongEntry=0;
      N5zigzagShortEntry=N5ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return N5ZZZ3;
  }


 
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN15ZZZ1ZigZagStopLossLevel()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   int N15PointShift3=N15PointShift2;
   string N15ConfirmedPoint3="Not Found";

   while(N15ConfirmedPoint3!="Found")
     {
      double N15ZZ3=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift3);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      N15PointShift3++;
     }

   double N15ZZZ1 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift);
   double N15ZZZ2 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift2);
   double N15ZZZ3 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift3);

   double N15zigzagLongEntry=0;
   double N15zigzagShortEntry=0;

   if(N15ZZZ3<N15ZZZ2 && N15ZZZ2>N15ZZZ1 && N15ZZZ1>N15ZZZ3)
     {
      N15zigzagLongEntry=N15ZZZ2+(PipBuffer*Point);
      N15zigzagShortEntry=0;
     }

   if(N15ZZZ3>N15ZZZ2 && N15ZZZ2<N15ZZZ1 && N15ZZZ1<N15ZZZ3)
     {
      N15zigzagLongEntry=0;
      N15zigzagShortEntry=N15ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N15ZZZ1;
  }




//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN15ZZZ3ZigZagStopLossLevel()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   int N15PointShift3=N15PointShift2;
   string N15ConfirmedPoint3="Not Found";

   while(N15ConfirmedPoint3!="Found")
     {
      double N15ZZ3=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift3);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      N15PointShift3++;
     }

   double N15ZZZ1 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift);
   double N15ZZZ2 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift2);
   double N15ZZZ3 = iCustom("EURUSD", PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift3);

   double N15zigzagLongEntry=0;
   double N15zigzagShortEntry=0;

   if(N15ZZZ3<N15ZZZ2 && N15ZZZ2>N15ZZZ1 && N15ZZZ1>N15ZZZ3)
     {
      N15zigzagLongEntry=N15ZZZ2+(PipBuffer*Point);
      N15zigzagShortEntry=0;
     }

   if(N15ZZZ3>N15ZZZ2 && N15ZZZ2<N15ZZZ1 && N15ZZZ1<N15ZZZ3)
     {
      N15zigzagLongEntry=0;
      N15zigzagShortEntry=N15ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N15ZZZ3;
  }



  
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN30ZZZ1ZigZagStopLossLevel()
  {
   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   int N30PointShift3=N30PointShift2;
   string N30ConfirmedPoint3="Not Found";

   while(N30ConfirmedPoint3!="Found")
     {
      double N30ZZ3=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift3);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      N30PointShift3++;
     }

   double N30ZZZ1 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift);
   double N30ZZZ2 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift2);
   double N30ZZZ3 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift3);

   double N30zigzagLongEntry=0;
   double N30zigzagShortEntry=0;

   if(N30ZZZ3<N30ZZZ2 && N30ZZZ2>N30ZZZ1 && N30ZZZ1>N30ZZZ3)
     {
      N30zigzagLongEntry=N30ZZZ2+(PipBuffer*Point);
      N30zigzagShortEntry=0;
     }

   if(N30ZZZ3>N30ZZZ2 && N30ZZZ2<N30ZZZ1 && N30ZZZ1<N30ZZZ3)
     {
      N30zigzagLongEntry=0;
      N30zigzagShortEntry=N30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N30ZZZ1;
  }




 
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetN30ZZZ3ZigZagStopLossLevel()
  {
   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   int N30PointShift3=N30PointShift2;
   string N30ConfirmedPoint3="Not Found";

   while(N30ConfirmedPoint3!="Found")
     {
      double N30ZZ3=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift3);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      N30PointShift3++;
     }

   double N30ZZZ1 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift);
   double N30ZZZ2 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift2);
   double N30ZZZ3 = iCustom("EURUSD", PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift3);

   double N30zigzagLongEntry=0;
   double N30zigzagShortEntry=0;

   if(N30ZZZ3<N30ZZZ2 && N30ZZZ2>N30ZZZ1 && N30ZZZ1>N30ZZZ3)
     {
      N30zigzagLongEntry=N30ZZZ2+(PipBuffer*Point);
      N30zigzagShortEntry=0;
     }

   if(N30ZZZ3>N30ZZZ2 && N30ZZZ2<N30ZZZ1 && N30ZZZ1<N30ZZZ3)
     {
      N30zigzagLongEntry=0;
      N30zigzagShortEntry=N30ZZZ2 -(PipBuffer*Point);
     }


// your full logic...
   return N30ZZZ3;
  }




  
//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetG1ZZZ1ZigZagStopLossLevel()
  {
   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   int G1PointShift3=G1PointShift2;
   string G1ConfirmedPoint3="Not Found";

   while(G1ConfirmedPoint3!="Found")
     {
      double G1ZZ3=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift3);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      G1PointShift3++;
     }

   double G1ZZZ1 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift);
   double G1ZZZ2 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift2);
   double G1ZZZ3 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift3);

   double G1zigzagLongEntry=0;
   double G1zigzagShortEntry=0;

   if(G1ZZZ3<G1ZZZ2 && G1ZZZ2>G1ZZZ1 && G1ZZZ1>G1ZZZ3)
     {
      G1zigzagLongEntry=G1ZZZ2+(PipBuffer*Point);
      G1zigzagShortEntry=0;
     }

   if(G1ZZZ3>G1ZZZ2 && G1ZZZ2<G1ZZZ1 && G1ZZZ1<G1ZZZ3)
     {
      G1zigzagLongEntry=0;
      G1zigzagShortEntry=G1ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G1ZZZ1;
  }



//+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetG1ZZZ3ZigZagStopLossLevel()
  {
   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   int G1PointShift3=G1PointShift2;
   string G1ConfirmedPoint3="Not Found";

   while(G1ConfirmedPoint3!="Found")
     {
      double G1ZZ3=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift3);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      G1PointShift3++;
     }

   double G1ZZZ1 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift);
   double G1ZZZ2 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift2);
   double G1ZZZ3 = iCustom("EURUSD", PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift3);

   double G1zigzagLongEntry=0;
   double G1zigzagShortEntry=0;

   if(G1ZZZ3<G1ZZZ2 && G1ZZZ2>G1ZZZ1 && G1ZZZ1>G1ZZZ3)
     {
      G1zigzagLongEntry=G1ZZZ2+(PipBuffer*Point);
      G1zigzagShortEntry=0;
     }

   if(G1ZZZ3>G1ZZZ2 && G1ZZZ2<G1ZZZ1 && G1ZZZ1<G1ZZZ3)
     {
      G1zigzagLongEntry=0;
      G1zigzagShortEntry=G1ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G1ZZZ3;
  }





  //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetG4ZZZ1ZigZagStopLossLevel()
  {
   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   int G4PointShift3=G4PointShift2;
   string G4ConfirmedPoint3="Not Found";

   while(G4ConfirmedPoint3!="Found")
     {
      double G4ZZ3=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift3);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      G4PointShift3++;
     }

   double G4ZZZ1 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift);
   double G4ZZZ2 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift2);
   double G4ZZZ3 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift3);

   double G4zigzagLongEntry=0;
   double G4zigzagShortEntry=0;

   if(G4ZZZ3<G4ZZZ2 && G4ZZZ2>G4ZZZ1 && G4ZZZ1>G4ZZZ3)
     {
      G4zigzagLongEntry=G4ZZZ2+(PipBuffer*Point);
      G4zigzagShortEntry=0;
     }

   if(G4ZZZ3>G4ZZZ2 && G4ZZZ2<G4ZZZ1 && G4ZZZ1<G4ZZZ3)
     {
      G4zigzagLongEntry=0;
      G4zigzagShortEntry=G4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G4ZZZ1;
  }



  //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetG4ZZZ3ZigZagStopLossLevel()
  {
   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   int G4PointShift3=G4PointShift2;
   string G4ConfirmedPoint3="Not Found";

   while(G4ConfirmedPoint3!="Found")
     {
      double G4ZZ3=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift3);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      G4PointShift3++;
     }

   double G4ZZZ1 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift);
   double G4ZZZ2 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift2);
   double G4ZZZ3 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift3);

   double G4zigzagLongEntry=0;
   double G4zigzagShortEntry=0;

   if(G4ZZZ3<G4ZZZ2 && G4ZZZ2>G4ZZZ1 && G4ZZZ1>G4ZZZ3)
     {
      G4zigzagLongEntry=G4ZZZ2+(PipBuffer*Point);
      G4zigzagShortEntry=0;
     }

   if(G4ZZZ3>G4ZZZ2 && G4ZZZ2<G4ZZZ1 && G4ZZZ1<G4ZZZ3)
     {
      G4zigzagLongEntry=0;
      G4zigzagShortEntry=G4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return G4ZZZ3;
  }



   //+------------------------------------------------------------------+
//|                                                                  |
//+------------------------------------------------------------------+
double EURUSD_GetH4ZZZ1ZigZagStopLossLevel()
  {
   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   int H4PointShift3=H4PointShift2;
   string H4ConfirmedPoint3="Not Found";

   while(H4ConfirmedPoint3!="Found")
     {
      double H4ZZ3=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift3);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      H4PointShift3++;
     }

   double H4ZZZ1 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift);
   double H4ZZZ2 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift2);
   double H4ZZZ3 = iCustom("EURUSD", PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, H4PointShift3);

   double H4zigzagLongEntry=0;
   double H4zigzagShortEntry=0;

   if(H4ZZZ3<H4ZZZ2 && H4ZZZ2>H4ZZZ1 && H4ZZZ1>H4ZZZ3)
     {
      H4zigzagLongEntry=H4ZZZ2+(PipBuffer*Point);
      H4zigzagShortEntry=0;
     }

   if(H4ZZZ3>H4ZZZ2 && H4ZZZ2<H4ZZZ1 && H4ZZZ1<H4ZZZ3)
     {
      H4zigzagLongEntry=0;
      H4zigzagShortEntry=H4ZZZ2 -(PipBuffer*Point);
     }





// your full logic...
   return H4ZZZ1;
  }


// ZZZ3 ZigZag pivot TIME functions - GBPUSD and EURUSD (full-length style)
// Each returns the bar open time of the third most recent confirmed ZigZag pivot.

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

//+------------------------------------------------------------------+
//+------------------------------------------------------------------+
//| ZZZ2 ZigZag TIME functions - GBPUSD and EURUSD                   |
//| ZZZ2 = second most recent confirmed ZigZag pivot                 |
//| Generated directly from the existing GetXZZZ3ZigZagTime()        |
//| functions: the third loop is removed and the time is taken at    |
//| PointShift2. Variants: M1, M5, M15, M30, H1, H4, N1, N5, N15,    |
//| N30, G1, G4 (each keeps its own period and Ext* parameters).     |
//+------------------------------------------------------------------+

// ================= GBPUSD =================

datetime GBPUSD_GetZZZ2ZigZagTime()
  {

   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ || iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime ZZZ2Time = iTime("GBPUSD",PERIOD_M1,PointShift2);

   return ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetM5ZZZ2ZigZagTime()
  {

   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime M5ZZZ2Time = iTime("GBPUSD",PERIOD_M5,M5PointShift2);

   return M5ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetM15ZZZ2ZigZagTime()
  {

   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime M15ZZZ2Time = iTime("GBPUSD",PERIOD_M15,M15PointShift2);

   return M15ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetM30ZZZ2ZigZagTime()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime M30ZZZ2Time = iTime("GBPUSD",PERIOD_M30,M30PointShift2);

   return M30ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetH1ZZZ2ZigZagTime()
  {

   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime H1ZZZ2Time = iTime("GBPUSD",PERIOD_H1,H1PointShift2);

   return H1ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetH4ZZZ2ZigZagTime()
  {

   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime H4ZZZ2Time = iTime("GBPUSD",PERIOD_H4,H4PointShift2);

   return H4ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetN1ZZZ2ZigZagTime()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime N1ZZZ2Time = iTime("GBPUSD",PERIOD_M1,N1PointShift2);

   return N1ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetN5ZZZ2ZigZagTime()
  {

   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime N5ZZZ2Time = iTime("GBPUSD",PERIOD_M5,N5PointShift2);

   return N5ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetN15ZZZ2ZigZagTime()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime N15ZZZ2Time = iTime("GBPUSD",PERIOD_M15,N15PointShift2);

   return N15ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetN30ZZZ2ZigZagTime()
  {

   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime N30ZZZ2Time = iTime("GBPUSD",PERIOD_M30,N30PointShift2);

   return N30ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetG1ZZZ2ZigZagTime()
  {

   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime G1ZZZ2Time = iTime("GBPUSD",PERIOD_H1,G1PointShift2);

   return G1ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetG4ZZZ2ZigZagTime()
  {

   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime G4ZZZ2Time = iTime("GBPUSD",PERIOD_H4,G4PointShift2);

   return G4ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


// ================= EURUSD =================

datetime EURUSD_GetZZZ2ZigZagTime()
  {

   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ || iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime ZZZ2Time = iTime("EURUSD",PERIOD_M1,PointShift2);

   return ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetM5ZZZ2ZigZagTime()
  {

   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime M5ZZZ2Time = iTime("EURUSD",PERIOD_M5,M5PointShift2);

   return M5ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetM15ZZZ2ZigZagTime()
  {

   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime M15ZZZ2Time = iTime("EURUSD",PERIOD_M15,M15PointShift2);

   return M15ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetM30ZZZ2ZigZagTime()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime M30ZZZ2Time = iTime("EURUSD",PERIOD_M30,M30PointShift2);

   return M30ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetH1ZZZ2ZigZagTime()
  {

   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime H1ZZZ2Time = iTime("EURUSD",PERIOD_H1,H1PointShift2);

   return H1ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetH4ZZZ2ZigZagTime()
  {

   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime H4ZZZ2Time = iTime("EURUSD",PERIOD_H4,H4PointShift2);

   return H4ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetN1ZZZ2ZigZagTime()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime N1ZZZ2Time = iTime("EURUSD",PERIOD_M1,N1PointShift2);

   return N1ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetN5ZZZ2ZigZagTime()
  {

   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime N5ZZZ2Time = iTime("EURUSD",PERIOD_M5,N5PointShift2);

   return N5ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetN15ZZZ2ZigZagTime()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime N15ZZZ2Time = iTime("EURUSD",PERIOD_M15,N15PointShift2);

   return N15ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetN30ZZZ2ZigZagTime()
  {

   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime N30ZZZ2Time = iTime("EURUSD",PERIOD_M30,N30PointShift2);

   return N30ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetG1ZZZ2ZigZagTime()
  {

   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime G1ZZZ2Time = iTime("EURUSD",PERIOD_H1,G1PointShift2);

   return G1ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetG4ZZZ2ZigZagTime()
  {

   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   // time of the second most recent confirmed ZigZag pivot
   datetime G4ZZZ2Time = iTime("EURUSD",PERIOD_H4,G4PointShift2);

   return G4ZZZ2Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////





datetime GBPUSD_GetZZZ3ZigZagTime()
  {

   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ || iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   int PointShift3=PointShift2;
   string ConfirmedPoint3="Not Found";

   while(ConfirmedPoint3!="Found")
     {
      double ZZ3=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift3);
      if(iHigh("GBPUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint3="Found";
         break;
        }
      PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime ZZZ3Time = iTime("GBPUSD",PERIOD_M1,PointShift3);

   return ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetM5ZZZ3ZigZagTime()
  {

   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   int M5PointShift3=M5PointShift2;
   string M5ConfirmedPoint3="Not Found";

   while(M5ConfirmedPoint3!="Found")
     {
      double M5ZZ3=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift3);
      if(iHigh("GBPUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      M5PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime M5ZZZ3Time = iTime("GBPUSD",PERIOD_M5,M5PointShift3);

   return M5ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetM15ZZZ3ZigZagTime()
  {

   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   int M15PointShift3=M15PointShift2;
   string M15ConfirmedPoint3="Not Found";

   while(M15ConfirmedPoint3!="Found")
     {
      double M15ZZ3=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift3);
      if(iHigh("GBPUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      M15PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime M15ZZZ3Time = iTime("GBPUSD",PERIOD_M15,M15PointShift3);

   return M15ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetM30ZZZ3ZigZagTime()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   int M30PointShift3=M30PointShift2;
   string M30ConfirmedPoint3="Not Found";

   while(M30ConfirmedPoint3!="Found")
     {
      double M30ZZ3=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift3);
      if(iHigh("GBPUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      M30PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime M30ZZZ3Time = iTime("GBPUSD",PERIOD_M30,M30PointShift3);

   return M30ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetH1ZZZ3ZigZagTime()
  {

   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   int H1PointShift3=H1PointShift2;
   string H1ConfirmedPoint3="Not Found";

   while(H1ConfirmedPoint3!="Found")
     {
      double H1ZZ3=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift3);
      if(iHigh("GBPUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      H1PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime H1ZZZ3Time = iTime("GBPUSD",PERIOD_H1,H1PointShift3);

   return H1ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetH4ZZZ3ZigZagTime()
  {

   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   int H4PointShift3=H4PointShift2;
   string H4ConfirmedPoint3="Not Found";

   while(H4ConfirmedPoint3!="Found")
     {
      double H4ZZ3=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift3);
      if(iHigh("GBPUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      H4PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime H4ZZZ3Time = iTime("GBPUSD",PERIOD_H4,H4PointShift3);

   return H4ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetN1ZZZ3ZigZagTime()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   int N1PointShift3=N1PointShift2;
   string N1ConfirmedPoint3="Not Found";

   while(N1ConfirmedPoint3!="Found")
     {
      double N1ZZ3=iCustom("GBPUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift3);
      if(iHigh("GBPUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      N1PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime N1ZZZ3Time = iTime("GBPUSD",PERIOD_M1,N1PointShift3);

   return N1ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetN5ZZZ3ZigZagTime()
  {

   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   int N5PointShift3=N5PointShift2;
   string N5ConfirmedPoint3="Not Found";

   while(N5ConfirmedPoint3!="Found")
     {
      double N5ZZ3=iCustom("GBPUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift3);
      if(iHigh("GBPUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      N5PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime N5ZZZ3Time = iTime("GBPUSD",PERIOD_M5,N5PointShift3);

   return N5ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetN15ZZZ3ZigZagTime()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   int N15PointShift3=N15PointShift2;
   string N15ConfirmedPoint3="Not Found";

   while(N15ConfirmedPoint3!="Found")
     {
      double N15ZZ3=iCustom("GBPUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift3);
      if(iHigh("GBPUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      N15PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime N15ZZZ3Time = iTime("GBPUSD",PERIOD_M15,N15PointShift3);

   return N15ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetN30ZZZ3ZigZagTime()
  {

   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   int N30PointShift3=N30PointShift2;
   string N30ConfirmedPoint3="Not Found";

   while(N30ConfirmedPoint3!="Found")
     {
      double N30ZZ3=iCustom("GBPUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift3);
      if(iHigh("GBPUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      N30PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime N30ZZZ3Time = iTime("GBPUSD",PERIOD_M30,N30PointShift3);

   return N30ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetG1ZZZ3ZigZagTime()
  {

   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   int G1PointShift3=G1PointShift2;
   string G1ConfirmedPoint3="Not Found";

   while(G1ConfirmedPoint3!="Found")
     {
      double G1ZZ3=iCustom("GBPUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift3);
      if(iHigh("GBPUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      G1PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime G1ZZZ3Time = iTime("GBPUSD",PERIOD_H1,G1PointShift3);

   return G1ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime GBPUSD_GetG4ZZZ3ZigZagTime()
  {

   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("GBPUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   int G4PointShift3=G4PointShift2;
   string G4ConfirmedPoint3="Not Found";

   while(G4ConfirmedPoint3!="Found")
     {
      double G4ZZ3=iCustom("GBPUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift3);
      if(iHigh("GBPUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("GBPUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      G4PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime G4ZZZ3Time = iTime("GBPUSD",PERIOD_H4,G4PointShift3);

   return G4ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetZZZ3ZigZagTime()
  {

   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ || iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   int PointShift3=PointShift2;
   string ConfirmedPoint3="Not Found";

   while(ConfirmedPoint3!="Found")
     {
      double ZZ3=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift3);
      if(iHigh("EURUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint3="Found";
         break;
        }
      PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime ZZZ3Time = iTime("EURUSD",PERIOD_M1,PointShift3);

   return ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetM5ZZZ3ZigZagTime()
  {

   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ || iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   int M5PointShift3=M5PointShift2;
   string M5ConfirmedPoint3="Not Found";

   while(M5ConfirmedPoint3!="Found")
     {
      double M5ZZ3=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift3);
      if(iHigh("EURUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      M5PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime M5ZZZ3Time = iTime("EURUSD",PERIOD_M5,M5PointShift3);

   return M5ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetM15ZZZ3ZigZagTime()
  {

   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ || iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   int M15PointShift3=M15PointShift2;
   string M15ConfirmedPoint3="Not Found";

   while(M15ConfirmedPoint3!="Found")
     {
      double M15ZZ3=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift3);
      if(iHigh("EURUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      M15PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime M15ZZZ3Time = iTime("EURUSD",PERIOD_M15,M15PointShift3);

   return M15ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetM30ZZZ3ZigZagTime()
  {

   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ || iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   int M30PointShift3=M30PointShift2;
   string M30ConfirmedPoint3="Not Found";

   while(M30ConfirmedPoint3!="Found")
     {
      double M30ZZ3=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift3);
      if(iHigh("EURUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      M30PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime M30ZZZ3Time = iTime("EURUSD",PERIOD_M30,M30PointShift3);

   return M30ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetH1ZZZ3ZigZagTime()
  {

   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ || iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   int H1PointShift3=H1PointShift2;
   string H1ConfirmedPoint3="Not Found";

   while(H1ConfirmedPoint3!="Found")
     {
      double H1ZZ3=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift3);
      if(iHigh("EURUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      H1PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime H1ZZZ3Time = iTime("EURUSD",PERIOD_H1,H1PointShift3);

   return H1ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetH4ZZZ3ZigZagTime()
  {

   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ || iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   int H4PointShift3=H4PointShift2;
   string H4ConfirmedPoint3="Not Found";

   while(H4ConfirmedPoint3!="Found")
     {
      double H4ZZ3=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,H4PointShift3);
      if(iHigh("EURUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      H4PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime H4ZZZ3Time = iTime("EURUSD",PERIOD_H4,H4PointShift3);

   return H4ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetN1ZZZ3ZigZagTime()
  {

   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ || iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   int N1PointShift3=N1PointShift2;
   string N1ConfirmedPoint3="Not Found";

   while(N1ConfirmedPoint3!="Found")
     {
      double N1ZZ3=iCustom("EURUSD",PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift3);
      if(iHigh("EURUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      N1PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime N1ZZZ3Time = iTime("EURUSD",PERIOD_M1,N1PointShift3);

   return N1ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetN5ZZZ3ZigZagTime()
  {

   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ || iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   int N5PointShift3=N5PointShift2;
   string N5ConfirmedPoint3="Not Found";

   while(N5ConfirmedPoint3!="Found")
     {
      double N5ZZ3=iCustom("EURUSD",PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift3);
      if(iHigh("EURUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      N5PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime N5ZZZ3Time = iTime("EURUSD",PERIOD_M5,N5PointShift3);

   return N5ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetN15ZZZ3ZigZagTime()
  {

   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ || iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   int N15PointShift3=N15PointShift2;
   string N15ConfirmedPoint3="Not Found";

   while(N15ConfirmedPoint3!="Found")
     {
      double N15ZZ3=iCustom("EURUSD",PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift3);
      if(iHigh("EURUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      N15PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime N15ZZZ3Time = iTime("EURUSD",PERIOD_M15,N15PointShift3);

   return N15ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetN30ZZZ3ZigZagTime()
  {

   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ || iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   int N30PointShift3=N30PointShift2;
   string N30ConfirmedPoint3="Not Found";

   while(N30ConfirmedPoint3!="Found")
     {
      double N30ZZ3=iCustom("EURUSD",PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift3);
      if(iHigh("EURUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      N30PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime N30ZZZ3Time = iTime("EURUSD",PERIOD_M30,N30PointShift3);

   return N30ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetG1ZZZ3ZigZagTime()
  {

   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ || iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   int G1PointShift3=G1PointShift2;
   string G1ConfirmedPoint3="Not Found";

   while(G1ConfirmedPoint3!="Found")
     {
      double G1ZZ3=iCustom("EURUSD",PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift3);
      if(iHigh("EURUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      G1PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime G1ZZZ3Time = iTime("EURUSD",PERIOD_H1,G1PointShift3);

   return G1ZZZ3Time;
  }

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

datetime EURUSD_GetG4ZZZ3ZigZagTime()
  {

   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ || iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow("EURUSD",PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   int G4PointShift3=G4PointShift2;
   string G4ConfirmedPoint3="Not Found";

   while(G4ConfirmedPoint3!="Found")
     {
      double G4ZZ3=iCustom("EURUSD",PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift3);
      if(iHigh("EURUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      if(iLow("EURUSD",PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      G4PointShift3++;
     }

   // time of the third most recent confirmed ZigZag pivot
   datetime G4ZZZ3Time = iTime("EURUSD",PERIOD_H4,G4PointShift3);

   return G4ZZZ3Time;
  }




/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

  

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////




//+------------------------------------------------------------------+
//| GBPUSD_LargestM1Candles.mq4                                      |
//| Largest bullish / bearish M1 candle (by body) on GBPUSD          |
//+------------------------------------------------------------------+


//+------------------------------------------------------------------+
//| BULLISH                                                          |
//+------------------------------------------------------------------+

// Shift of largest bullish GBPUSD M1 candle in last N completed bars
int GBPUSD_GetLargestBullishM1_Shift(int bars = 20)
{
   double largestBody = -1.0;
   int    largestShift = -1;

   for(int i = 1; i <= bars; i++)
   {
      double o = iOpen("GBPUSD", PERIOD_M1, i);
      double c = iClose("GBPUSD", PERIOD_M1, i);

      if(c > o)
      {
         double body = c - o;
         if(body > largestBody)
         {
            largestBody  = body;
            largestShift = i;
         }
      }
   }
   return largestShift;
}

double GBPUSD_GetLargestBullishM1_Open(int bars = 20)
{
   int shift = GBPUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iOpen("GBPUSD", PERIOD_M1, shift);
}

double GBPUSD_GetLargestBullishM1_High(int bars = 20)
{
   int shift = GBPUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iHigh("GBPUSD", PERIOD_M1, shift);
}

double GBPUSD_GetLargestBullishM1_Low(int bars = 20)
{
   int shift = GBPUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iLow("GBPUSD", PERIOD_M1, shift);
}

double GBPUSD_GetLargestBullishM1_Close(int bars = 20)
{
   int shift = GBPUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iClose("GBPUSD", PERIOD_M1, shift);
}

datetime GBPUSD_GetLargestBullishM1_Time(int bars = 20)
{
   int shift = GBPUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iTime("GBPUSD", PERIOD_M1, shift);
}

//+------------------------------------------------------------------+
//| BEARISH                                                          |
//+------------------------------------------------------------------+

// Shift of largest bearish GBPUSD M1 candle in last N completed bars
int GBPUSD_GetLargestBearishM1_Shift(int bars = 20)
{
   double largestBody = -1.0;
   int    largestShift = -1;

   for(int i = 1; i <= bars; i++)
   {
      double o = iOpen("GBPUSD", PERIOD_M1, i);
      double c = iClose("GBPUSD", PERIOD_M1, i);

      if(c < o)
      {
         double body = o - c;
         if(body > largestBody)
         {
            largestBody  = body;
            largestShift = i;
         }
      }
   }
   return largestShift;
}

double GBPUSD_GetLargestBearishM1_Open(int bars = 20)
{
   int shift = GBPUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iOpen("GBPUSD", PERIOD_M1, shift);
}

double GBPUSD_GetLargestBearishM1_High(int bars = 20)
{
   int shift = GBPUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iHigh("GBPUSD", PERIOD_M1, shift);
}

double GBPUSD_GetLargestBearishM1_Low(int bars = 20)
{
   int shift = GBPUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iLow("GBPUSD", PERIOD_M1, shift);
}

double GBPUSD_GetLargestBearishM1_Close(int bars = 20)
{
   int shift = GBPUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iClose("GBPUSD", PERIOD_M1, shift);
}

datetime GBPUSD_GetLargestBearishM1_Time(int bars = 20)
{
   int shift = GBPUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iTime("GBPUSD", PERIOD_M1, shift);
}

//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


 //+------------------------------------------------------------------+
//| Average Candle Body (M1)                                        |
//+------------------------------------------------------------------+
double GetAverageBody(int bars = 20)
{
   double sum = 0;

   for(int i=1; i<=bars; i++)
   {
      double o = iOpen(NULL, PERIOD_M1, i);
      double c = iClose(NULL, PERIOD_M1, i);
      sum += MathAbs(c - o);
   }

   return sum / bars;
}


//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 //+------------------------------------------------------------------+
//| Find shift of largest bullish M1 candle in last 20 completed bars|
//+------------------------------------------------------------------+
int GetLargestBullishM1_Shift(int bars = 20)
{
   double largestBody = -1.0;
   int largestShift = -1;

   for(int i = 1; i <= bars; i++)
   {
      double o = iOpen(NULL, PERIOD_M1, i);
      double c = iClose(NULL, PERIOD_M1, i);

      // Bullish candle
      if(c > o)
      {
         double body = c - o;

         if(body > largestBody)
         {
            largestBody = body;
            largestShift = i;
         }
      }
   }

   return largestShift;
}


//+------------------------------------------------------------------+
//| Largest Bullish M1 Candle - Open                                |
//+------------------------------------------------------------------+
double GetLargestBullishM1_Open(int bars = 20)
{
   int shift = GetLargestBullishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iOpen(NULL, PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| Largest Bullish M1 Candle - High                                |
//+------------------------------------------------------------------+
double GetLargestBullishM1_High(int bars = 20)
{
   int shift = GetLargestBullishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iHigh(NULL, PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| Largest Bullish M1 Candle - Low                                 |
//+------------------------------------------------------------------+
double GetLargestBullishM1_Low(int bars = 20)
{
   int shift = GetLargestBullishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iLow(NULL, PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| Largest Bullish M1 Candle - Close                               |
//+------------------------------------------------------------------+
double GetLargestBullishM1_Close(int bars = 20)
{
   int shift = GetLargestBullishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iClose(NULL, PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| Largest Bullish M1 Candle - Time                                |
//+------------------------------------------------------------------+
datetime GetLargestBullishM1_Time(int bars = 20)
{
   int shift = GetLargestBullishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iTime(NULL, PERIOD_M1, shift);
}



//+------------------------------------------------------------------+
//| Find shift of largest bearish M1 candle in last 20 completed bars|
//+------------------------------------------------------------------+
int GetLargestBearishM1_Shift(int bars = 20)
{
   double largestBody = -1.0;
   int largestShift = -1;

   for(int i = 1; i <= bars; i++)
   {
      double o = iOpen(NULL, PERIOD_M1, i);
      double c = iClose(NULL, PERIOD_M1, i);

      // Bearish candle
      if(c < o)
      {
         double body = o - c;

         if(body > largestBody)
         {
            largestBody = body;
            largestShift = i;
         }
      }
   }

   return largestShift;
}


//+------------------------------------------------------------------+
//| Largest Bearish M1 Candle - Open                                |
//+------------------------------------------------------------------+
double GetLargestBearishM1_Open(int bars = 20)
{
   int shift = GetLargestBearishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iOpen(NULL, PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| Largest Bearish M1 Candle - High                                |
//+------------------------------------------------------------------+
double GetLargestBearishM1_High(int bars = 20)
{
   int shift = GetLargestBearishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iHigh(NULL, PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| Largest Bearish M1 Candle - Low                                 |
//+------------------------------------------------------------------+
double GetLargestBearishM1_Low(int bars = 20)
{
   int shift = GetLargestBearishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iLow(NULL, PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| Largest Bearish M1 Candle - Close                               |
//+------------------------------------------------------------------+
double GetLargestBearishM1_Close(int bars = 20)
{
   int shift = GetLargestBearishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iClose(NULL, PERIOD_M1, shift);
}


//+------------------------------------------------------------------+
//| Largest Bearish M1 Candle - Time                                |
//+------------------------------------------------------------------+
datetime GetLargestBearishM1_Time(int bars = 20)
{
   int shift = GetLargestBearishM1_Shift(bars);

   if(shift < 0)
      return 0;

   return iTime(NULL, PERIOD_M1, shift);
}



 /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  
  
  //+------------------------------------------------------------------+
//| EURUSD_LargestM1Candles.mq4                                      |
//| Largest bullish / bearish M1 candle (by body) on EURUSD          |
//+------------------------------------------------------------------+


//+------------------------------------------------------------------+
//| BULLISH                                                          |
//+------------------------------------------------------------------+

// Shift of largest bullish EURUSD M1 candle in last N completed bars
int EURUSD_GetLargestBullishM1_Shift(int bars = 20)
{
   double largestBody = -1.0;
   int    largestShift = -1;

   for(int i = 1; i <= bars; i++)
   {
      double o = iOpen("EURUSD", PERIOD_M1, i);
      double c = iClose("EURUSD", PERIOD_M1, i);

      if(c > o)
      {
         double body = c - o;
         if(body > largestBody)
         {
            largestBody  = body;
            largestShift = i;
         }
      }
   }
   return largestShift;
}

double EURUSD_GetLargestBullishM1_Open(int bars = 20)
{
   int shift = EURUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iOpen("EURUSD", PERIOD_M1, shift);
}

double EURUSD_GetLargestBullishM1_High(int bars = 20)
{
   int shift = EURUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iHigh("EURUSD", PERIOD_M1, shift);
}

double EURUSD_GetLargestBullishM1_Low(int bars = 20)
{
   int shift = EURUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iLow("EURUSD", PERIOD_M1, shift);
}

double EURUSD_GetLargestBullishM1_Close(int bars = 20)
{
   int shift = EURUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iClose("EURUSD", PERIOD_M1, shift);
}

datetime EURUSD_GetLargestBullishM1_Time(int bars = 20)
{
   int shift = EURUSD_GetLargestBullishM1_Shift(bars);
   if(shift < 0) return 0;
   return iTime("EURUSD", PERIOD_M1, shift);
}

//+------------------------------------------------------------------+
//| BEARISH                                                          |
//+------------------------------------------------------------------+

// Shift of largest bearish EURUSD M1 candle in last N completed bars
int EURUSD_GetLargestBearishM1_Shift(int bars = 20)
{
   double largestBody = -1.0;
   int    largestShift = -1;

   for(int i = 1; i <= bars; i++)
   {
      double o = iOpen("EURUSD", PERIOD_M1, i);
      double c = iClose("EURUSD", PERIOD_M1, i);

      if(c < o)
      {
         double body = o - c;
         if(body > largestBody)
         {
            largestBody  = body;
            largestShift = i;
         }
      }
   }
   return largestShift;
}

double EURUSD_GetLargestBearishM1_Open(int bars = 20)
{
   int shift = EURUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iOpen("EURUSD", PERIOD_M1, shift);
}

double EURUSD_GetLargestBearishM1_High(int bars = 20)
{
   int shift = EURUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iHigh("EURUSD", PERIOD_M1, shift);
}

double EURUSD_GetLargestBearishM1_Low(int bars = 20)
{
   int shift = EURUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iLow("EURUSD", PERIOD_M1, shift);
}

double EURUSD_GetLargestBearishM1_Close(int bars = 20)
{
   int shift = EURUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iClose("EURUSD", PERIOD_M1, shift);
}

datetime EURUSD_GetLargestBearishM1_Time(int bars = 20)
{
   int shift = EURUSD_GetLargestBearishM1_Shift(bars);
   if(shift < 0) return 0;
   return iTime("EURUSD", PERIOD_M1, shift);
}


/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////




//+------------------------------------------------------------------+
//| GBPUSD / EURUSD versions                                         |
//| NOTE: If your broker uses a suffix (e.g. "GBPUSD.m" or "GBPUSDm")|
//| change the symbol strings below to match Market Watch exactly.   |
//+------------------------------------------------------------------+

//==================================================================//
//                              GBPUSD                              //
//==================================================================//

//+------------------------------------------------------------------+
//| Get the LOW of the M1 bar where the Lowest Point After Daily High|
//| occurred (GBPUSD)                                                |
//+------------------------------------------------------------------+
double GBPUSD_GetLowAtLowestPointAfterDailyHighM1()
{
   string sym = "GBPUSD";

   // 1. Get the M1 bar open time of the lowest point after the daily high
   datetime lpTimeM1 = GBPUSD_GetLowestPointAfterDailyHighTimeOnM1();
   if (lpTimeM1 == 0) return 0;

   // 2. Convert that time to an M1 bar index
   int idx = iBarShift(sym, PERIOD_M1, lpTimeM1, true);
   if (idx < 0) return 0;

   // 3. Get the LOW price of that M1 bar
   return iLow(sym, PERIOD_M1, idx);
}

//+------------------------------------------------------------------+
//| Finds the time of the Lowest Point After the Daily High on M1    |
//| (GBPUSD)                                                         |
//+------------------------------------------------------------------+
datetime GBPUSD_GetLowestPointAfterDailyHighTimeOnM1()
{
   string sym = "GBPUSD";

   datetime dayStart     = iTime(sym, PERIOD_D1, 0);
   datetime prevDayStart = iTime(sym, PERIOD_D1, 1);

   double dailyHigh = iHigh(sym, PERIOD_D1, 0);

   int startIdx = iBarShift(sym, PERIOD_M1, dayStart, false);
   if (startIdx < 0) return 0;

   // Use the Point of GBPUSD, not the chart's symbol
   double tolerance = MarketInfo(sym, MODE_POINT) * 3;
   int highBar = -1;

   for (int i = startIdx; i >= 0; i--)
   {
      double hi = iHigh(sym, PERIOD_M1, i);
      if (MathAbs(hi - dailyHigh) <= tolerance)
      {
         highBar = i;
         break;
      }
   }

   if (highBar == -1)
   {
      Print("GBPUSD: Daily high not found on M1 chart");
      return 0;
   }

   double lowestLow = iLow(sym, PERIOD_M1, highBar);
   int lowestBar = -1;

   for (int ib = highBar - 1; ib >= 0; ib--)
   {
      datetime t = iTime(sym, PERIOD_M1, ib);
      if (t < prevDayStart) break;

      double lo = iLow(sym, PERIOD_M1, ib);
      if (lo < lowestLow)
      {
         lowestLow = lo;
         lowestBar = ib;
      }
   }

   if (lowestBar == -1)
      return 0;

   return iTime(sym, PERIOD_M1, lowestBar);
}


//==================================================================//
//                              EURUSD                              //
//==================================================================//

//+------------------------------------------------------------------+
//| Get the LOW of the M1 bar where the Lowest Point After Daily High|
//| occurred (EURUSD)                                                |
//+------------------------------------------------------------------+
double EURUSD_GetLowAtLowestPointAfterDailyHighM1()
{
   string sym = "EURUSD";

   // 1. Get the M1 bar open time of the lowest point after the daily high
   datetime lpTimeM1 = EURUSD_GetLowestPointAfterDailyHighTimeOnM1();
   if (lpTimeM1 == 0) return 0;

   // 2. Convert that time to an M1 bar index
   int idx = iBarShift(sym, PERIOD_M1, lpTimeM1, true);
   if (idx < 0) return 0;

   // 3. Get the LOW price of that M1 bar
   return iLow(sym, PERIOD_M1, idx);
}

//+------------------------------------------------------------------+
//| Finds the time of the Lowest Point After the Daily High on M1    |
//| (EURUSD)                                                         |
//+------------------------------------------------------------------+
datetime EURUSD_GetLowestPointAfterDailyHighTimeOnM1()
{
   string sym = "EURUSD";

   datetime dayStart     = iTime(sym, PERIOD_D1, 0);
   datetime prevDayStart = iTime(sym, PERIOD_D1, 1);

   double dailyHigh = iHigh(sym, PERIOD_D1, 0);

   int startIdx = iBarShift(sym, PERIOD_M1, dayStart, false);
   if (startIdx < 0) return 0;

   // Use the Point of EURUSD, not the chart's symbol
   double tolerance = MarketInfo(sym, MODE_POINT) * 3;
   int highBar = -1;

   for (int i = startIdx; i >= 0; i--)
   {
      double hi = iHigh(sym, PERIOD_M1, i);
      if (MathAbs(hi - dailyHigh) <= tolerance)
      {
         highBar = i;
         break;
      }
   }

   if (highBar == -1)
   {
      Print("EURUSD: Daily high not found on M1 chart");
      return 0;
   }

   double lowestLow = iLow(sym, PERIOD_M1, highBar);
   int lowestBar = -1;

   for (int ib = highBar - 1; ib >= 0; ib--)
   {
      datetime t = iTime(sym, PERIOD_M1, ib);
      if (t < prevDayStart) break;

      double lo = iLow(sym, PERIOD_M1, ib);
      if (lo < lowestLow)
      {
         lowestLow = lo;
         lowestBar = ib;
      }
   }

   if (lowestBar == -1)
      return 0;

   return iTime(sym, PERIOD_M1, lowestBar);
}


//+------------------------------------------------------------------+
//| GBPUSD / EURUSD versions: Highest Point After Daily Low (M1)     |
//| NOTE: If your broker uses a suffix (e.g. "GBPUSD.m" or "GBPUSDm")|
//| change the symbol strings below to match Market Watch exactly.   |
//+------------------------------------------------------------------+

//==================================================================//
//                              GBPUSD                              //
//==================================================================//

//+------------------------------------------------------------------+
//| Get the HIGH of the M1 bar where the Highest Point After Daily   |
//| Low occurred (GBPUSD)                                            |
//+------------------------------------------------------------------+
double GBPUSD_GetHighAtHighestPointAfterDailyLowM1()
{
   string sym = "GBPUSD";

   // Step 1: Get the M1 bar time of the highest point after daily low
   datetime hpTimeM1 = GBPUSD_GetHighestPointAfterDailyLowTimeOnM1();
   if (hpTimeM1 == 0) return 0;

   // Step 2: Convert that time to an M1 bar index
   int idx = iBarShift(sym, PERIOD_M1, hpTimeM1, true);
   if (idx < 0) return 0;

   // Step 3: Get the HIGH of that bar
   return iHigh(sym, PERIOD_M1, idx);
}

//+------------------------------------------------------------------+
//| Finds the time of the Highest Point After the Daily Low on M1    |
//| (GBPUSD)                                                         |
//+------------------------------------------------------------------+
datetime GBPUSD_GetHighestPointAfterDailyLowTimeOnM1()
{
   string sym = "GBPUSD";

   // 1. Get today's start and yesterday's start
   datetime dayStart     = iTime(sym, PERIOD_D1, 0);
   datetime prevDayStart = iTime(sym, PERIOD_D1, 1);

   // 2. Get today's low value
   double dailyLow = iLow(sym, PERIOD_D1, 0);

   // 3. Find the first M1 bar where the daily low occurred (within tolerance)
   int startIdx = iBarShift(sym, PERIOD_M1, dayStart, false);
   if (startIdx < 0)
      return 0;

   // Use the Point of GBPUSD, not the chart's symbol
   double tolerance = MarketInfo(sym, MODE_POINT) * 3;
   int lowBar = -1;

   for (int i = startIdx; i >= 0; i--)
   {
      double lo = iLow(sym, PERIOD_M1, i);
      if (MathAbs(lo - dailyLow) <= tolerance)
      {
         lowBar = i;
         break;
      }
   }

   if (lowBar == -1)
   {
      Print("GBPUSD: Daily low bar not found on M1 chart.");
      return 0;
   }

   // 4. Scan forward (after the daily low) for the highest high
   double highestHigh = iHigh(sym, PERIOD_M1, lowBar);
   int highestBar = -1;

   for (int ia = lowBar - 1; ia >= 0; ia--)
   {
      datetime t = iTime(sym, PERIOD_M1, ia);
      if (t < prevDayStart)
         break;  // stop if we reached previous day

      double hi = iHigh(sym, PERIOD_M1, ia);
      if (hi > highestHigh)
      {
         highestHigh = hi;
         highestBar = ia;
      }
   }

   if (highestBar == -1)
      return 0;

   return iTime(sym, PERIOD_M1, highestBar);
}


//==================================================================//
//                              EURUSD                              //
//==================================================================//

//+------------------------------------------------------------------+
//| Get the HIGH of the M1 bar where the Highest Point After Daily   |
//| Low occurred (EURUSD)                                            |
//+------------------------------------------------------------------+
double EURUSD_GetHighAtHighestPointAfterDailyLowM1()
{
   string sym = "EURUSD";

   // Step 1: Get the M1 bar time of the highest point after daily low
   datetime hpTimeM1 = EURUSD_GetHighestPointAfterDailyLowTimeOnM1();
   if (hpTimeM1 == 0) return 0;

   // Step 2: Convert that time to an M1 bar index
   int idx = iBarShift(sym, PERIOD_M1, hpTimeM1, true);
   if (idx < 0) return 0;

   // Step 3: Get the HIGH of that bar
   return iHigh(sym, PERIOD_M1, idx);
}

//+------------------------------------------------------------------+
//| Finds the time of the Highest Point After the Daily Low on M1    |
//| (EURUSD)                                                         |
//+------------------------------------------------------------------+
datetime EURUSD_GetHighestPointAfterDailyLowTimeOnM1()
{
   string sym = "EURUSD";

   // 1. Get today's start and yesterday's start
   datetime dayStart     = iTime(sym, PERIOD_D1, 0);
   datetime prevDayStart = iTime(sym, PERIOD_D1, 1);

   // 2. Get today's low value
   double dailyLow = iLow(sym, PERIOD_D1, 0);

   // 3. Find the first M1 bar where the daily low occurred (within tolerance)
   int startIdx = iBarShift(sym, PERIOD_M1, dayStart, false);
   if (startIdx < 0)
      return 0;

   // Use the Point of EURUSD, not the chart's symbol
   double tolerance = MarketInfo(sym, MODE_POINT) * 3;
   int lowBar = -1;

   for (int i = startIdx; i >= 0; i--)
   {
      double lo = iLow(sym, PERIOD_M1, i);
      if (MathAbs(lo - dailyLow) <= tolerance)
      {
         lowBar = i;
         break;
      }
   }

   if (lowBar == -1)
   {
      Print("EURUSD: Daily low bar not found on M1 chart.");
      return 0;
   }

   // 4. Scan forward (after the daily low) for the highest high
   double highestHigh = iHigh(sym, PERIOD_M1, lowBar);
   int highestBar = -1;

   for (int ia = lowBar - 1; ia >= 0; ia--)
   {
      datetime t = iTime(sym, PERIOD_M1, ia);
      if (t < prevDayStart)
         break;  // stop if we reached previous day

      double hi = iHigh(sym, PERIOD_M1, ia);
      if (hi > highestHigh)
      {
         highestHigh = hi;
         highestBar = ia;
      }
   }

   if (highestBar == -1)
      return 0;

   return iTime(sym, PERIOD_M1, highestBar);
}

//+------------------------------------------------------------------+
//| M5 Lower Low / Higher High on M1 - paste-ready                   |
//| Copy everything below into your EA (anywhere outside OnTick etc.)|
//| No #define, no #ifndef, no inputs, no globals.                   |
//| Each function returns 0 when nothing qualifies.                  |
//| M5 scan window = bars 0..299.                                    |
//+------------------------------------------------------------------+

//--- helper: draws/refreshes a horizontal line -----------------------
void DrawStructureLine2(string name, double price, color clr)
{
   if(ObjectFind(0, name) >= 0)
      ObjectDelete(0, name);

   ObjectCreate(0, name, OBJ_HLINE, 0, 0, price);
   ObjectSetInteger(0, name, OBJPROP_COLOR, clr);
   ObjectSetInteger(0, name, OBJPROP_WIDTH, 2);
}

//--- price of the relevant M5 lower low ------------------------------
double M5LowerLow_on_M1timefame()
{
   double dailyLow = iLow(NULL, PERIOD_D1, 0);
   double bestRelevantLow = 0;

   for(int i = 0; i < 300; i++)
   {
      double low_i  = iLow(NULL, PERIOD_M5, i);
      double low_p1 = iLow(NULL, PERIOD_M5, i + 1);
      double low_m1 = (i > 0) ? iLow(NULL, PERIOD_M5, i - 1) : DBL_MAX;   // bar 0 has no newer neighbour

      if(low_i < low_p1 && low_i < low_m1 && low_i > dailyLow)
      {
         bool breached = false;

         for(int j = i - 1; j >= 0; j--)
         {
            if(iLow(NULL, PERIOD_M5, j) < low_i)
            {
               breached = true;
               break;
            }
         }

         if(!breached)
            bestRelevantLow = low_i;
      }
   }

   if(bestRelevantLow > 0)
      DrawStructureLine2("M5_LowerLow", bestRelevantLow, clrAqua);

   return bestRelevantLow;
}

//--- price of the relevant M5 higher high ----------------------------
double M5HigherHigh_on_M1timeframe()
{
   double dailyHigh = iHigh(NULL, PERIOD_D1, 0);
   double bestRelevantHigh = 0;

   for(int i = 0; i < 300; i++)
   {
      double high_i  = iHigh(NULL, PERIOD_M5, i);
      double high_p1 = iHigh(NULL, PERIOD_M5, i + 1);
      double high_m1 = (i > 0) ? iHigh(NULL, PERIOD_M5, i - 1) : 0;   // bar 0 has no newer neighbour

      if(high_i > high_p1 && high_i > high_m1 && high_i < dailyHigh)
      {
         bool breached = false;

         for(int j = i - 1; j >= 0; j--)
         {
            if(iHigh(NULL, PERIOD_M5, j) > high_i)
            {
               breached = true;
               break;
            }
         }

         if(!breached)
            bestRelevantHigh = high_i;
      }
   }

   if(bestRelevantHigh > 0)
      DrawStructureLine2("M5_HigherHigh", bestRelevantHigh, clrRed);

   return bestRelevantHigh;
}

//--- open time of the M1 bar that made the lower low -----------------
datetime TIMEof_M5LowerLow_on_M1timefame()
{
   double m5LowRef = M5LowerLow_on_M1timefame();
   if(m5LowRef <= 0) return 0;

   int totalBarsM5 = iBars(NULL, PERIOD_M5);

   for(int i = 0; i < 300 && i < totalBarsM5; i++)
   {
      if(NormalizeDouble(iLow(NULL, PERIOD_M5, i), Digits) !=
         NormalizeDouble(m5LowRef, Digits))
         continue;

      datetime m5Time  = iTime(NULL, PERIOD_M5, i);
      datetime m5Close = m5Time + PeriodSeconds(PERIOD_M5);

      int startM1 = iBarShift(NULL, PERIOD_M1, m5Close, false);
      int endM1   = iBarShift(NULL, PERIOD_M1, m5Time,  false);
      if(startM1 < 0 || endM1 < 0) continue;

      // oldest M1 bar inside that M5 candle whose low equals the level
      for(int j = endM1; j >= startM1; j--)
      {
         if(NormalizeDouble(iLow(NULL, PERIOD_M1, j), Digits) ==
            NormalizeDouble(m5LowRef, Digits))
            return iTime(NULL, PERIOD_M1, j);
      }
   }

   return 0;
}

//--- open time of the M1 bar that made the higher high ---------------
datetime TIMEof_M5HigherHigh_on_M1timeframe()
{
   double m5HighRef = M5HigherHigh_on_M1timeframe();
   if(m5HighRef <= 0) return 0;

   int totalBarsM5 = iBars(NULL, PERIOD_M5);

   for(int i = 0; i < 300 && i < totalBarsM5; i++)
   {
      if(NormalizeDouble(iHigh(NULL, PERIOD_M5, i), Digits) !=
         NormalizeDouble(m5HighRef, Digits))
         continue;

      datetime m5Time  = iTime(NULL, PERIOD_M5, i);
      datetime m5Close = m5Time + PeriodSeconds(PERIOD_M5);

      int startM1 = iBarShift(NULL, PERIOD_M1, m5Close, false);
      int endM1   = iBarShift(NULL, PERIOD_M1, m5Time,  false);
      if(startM1 < 0 || endM1 < 0) continue;

      for(int j = endM1; j >= startM1; j--)
      {
         if(NormalizeDouble(iHigh(NULL, PERIOD_M1, j), Digits) ==
            NormalizeDouble(m5HighRef, Digits))
            return iTime(NULL, PERIOD_M1, j);
      }
   }

   return 0;
}

//+------------------------------------------------------------------+
//| M5 LOWER LOW / M5 HIGHER HIGH : M1 bar data + pivot              |
//| Pattern copied from PivotAtLowestPointAfterDailyHighM1 /          |
//| PivotAtHighestPointAfterDailyLowM1. Chart symbol only (NULL).     |
//| Every function returns 0 when no level / bar is found.           |
//+------------------------------------------------------------------+

//==================================================================//
//                        M5 LOWER LOW                              //
//==================================================================//
double GetHighAtM5LowerLow_on_M1timefame()
{
   // 1. Exact M1 open time of the M5 lower low
   datetime t = TIMEof_M5LowerLow_on_M1timefame();
   if (t == 0) return 0;

   // 2. Convert the time to an M1 bar index
   int idx = iBarShift(NULL, PERIOD_M1, t, true);
   if (idx < 0) return 0;

   // 3. HIGH of that M1 bar
   return iHigh(NULL, PERIOD_M1, idx);
}

double GetLowAtM5LowerLow_on_M1timefame()
{
   datetime t = TIMEof_M5LowerLow_on_M1timefame();
   if (t == 0) return 0;

   int idx = iBarShift(NULL, PERIOD_M1, t, true);
   if (idx < 0) return 0;

   return iLow(NULL, PERIOD_M1, idx);
}

double GetCloseAtM5LowerLow_on_M1timefame()
{
   datetime t = TIMEof_M5LowerLow_on_M1timefame();
   if (t == 0) return 0;

   int idx = iBarShift(NULL, PERIOD_M1, t, true);
   if (idx < 0) return 0;

   return iClose(NULL, PERIOD_M1, idx);
}

double PivotatM5LowerLow_on_M1timefame()
{
   // Get High, Low, Close of that exact M1 bar
   double hi = GetHighAtM5LowerLow_on_M1timefame();
   double lo = GetLowAtM5LowerLow_on_M1timefame();
   double cl = GetCloseAtM5LowerLow_on_M1timefame();

   // If any of them failed, return 0
   if (hi == 0 || lo == 0 || cl == 0)
      return 0;

   // Pivot formula
   return (hi + lo + cl) / 3.0;
}

//==================================================================//
//                        M5 HIGHER HIGH                            //
//==================================================================//
double GetHighAtM5HigherHigh_on_M1timeframe()
{
   // 1. Exact M1 open time of the M5 higher high
   datetime t = TIMEof_M5HigherHigh_on_M1timeframe();
   if (t == 0) return 0;

   // 2. Convert the time to an M1 bar index
   int idx = iBarShift(NULL, PERIOD_M1, t, true);
   if (idx < 0) return 0;

   // 3. HIGH of that M1 bar
   return iHigh(NULL, PERIOD_M1, idx);
}

double GetLowAtM5HigherHigh_on_M1timeframe()
{
   datetime t = TIMEof_M5HigherHigh_on_M1timeframe();
   if (t == 0) return 0;

   int idx = iBarShift(NULL, PERIOD_M1, t, true);
   if (idx < 0) return 0;

   return iLow(NULL, PERIOD_M1, idx);
}

double GetCloseAtM5HigherHigh_on_M1timeframe()
{
   datetime t = TIMEof_M5HigherHigh_on_M1timeframe();
   if (t == 0) return 0;

   int idx = iBarShift(NULL, PERIOD_M1, t, true);
   if (idx < 0) return 0;

   return iClose(NULL, PERIOD_M1, idx);
}

double PivotatM5HigherHigh_on_M1timeframe()
{
   // Get High, Low, Close of that exact M1 bar
   double hi = GetHighAtM5HigherHigh_on_M1timeframe();
   double lo = GetLowAtM5HigherHigh_on_M1timeframe();
   double cl = GetCloseAtM5HigherHigh_on_M1timeframe();

   // If any of them failed, return 0
   if (hi == 0 || lo == 0 || cl == 0)
      return 0;

   // Pivot formula
   return (hi + lo + cl) / 3.0;
}

//+------------------------------------------------------------------+
//| OPTIONAL TEST HELPER - not called anywhere. Call it once from    |
//| OnTick temporarily, compare the Experts-log values with the chart|
//+------------------------------------------------------------------+
void PrintM5PivotDebug()
{
   Print("M5 LOWER LOW  | level=", DoubleToString(M5LowerLow_on_M1timefame(), Digits),
         " | M1 time=", TimeToString(TIMEof_M5LowerLow_on_M1timefame(), TIME_DATE|TIME_MINUTES),
         " | H=", DoubleToString(GetHighAtM5LowerLow_on_M1timefame(), Digits),
         " L=", DoubleToString(GetLowAtM5LowerLow_on_M1timefame(), Digits),
         " C=", DoubleToString(GetCloseAtM5LowerLow_on_M1timefame(), Digits),
         " | pivot=", DoubleToString(PivotatM5LowerLow_on_M1timefame(), Digits));
   Print("M5 HIGHER HIGH| level=", DoubleToString(M5HigherHigh_on_M1timeframe(), Digits),
         " | M1 time=", TimeToString(TIMEof_M5HigherHigh_on_M1timeframe(), TIME_DATE|TIME_MINUTES),
         " | H=", DoubleToString(GetHighAtM5HigherHigh_on_M1timeframe(), Digits),
         " L=", DoubleToString(GetLowAtM5HigherHigh_on_M1timeframe(), Digits),
         " C=", DoubleToString(GetCloseAtM5HigherHigh_on_M1timeframe(), Digits),
         " | pivot=", DoubleToString(PivotatM5HigherHigh_on_M1timeframe(), Digits));
}





int OnInit()
{
   return(INIT_SUCCEEDED);
}



void OnDeinit(const int reason)
{
   // Master: nothing to tear down.
}

//+------------------------------------------------------------------+
//| Event Processing                                                 |
//+------------------------------------------------------------------+
void OnTick()
{
  
  ///////////////GBPUSD_DAILYLONG_AND_SHORTENTRY//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  ///////////////GBPUSD_DAILYLONG_AND_SHORTENTRY///////////////////////////////////////////////////////

// Find the first bar of the current day ON GBPUSD's OWN bars (same symbol/timeframe that
// iHighest/iLowest use below), so the shift is valid whichever chart the EA is attached to.
int GBPUSD_DayStartShift = -1;
int GBPUSD_DayTestShift  = 0;
int GBPUSD_BarsTotal     = iBars("GBPUSD", 0);

while(GBPUSD_DayStartShift == -1 && GBPUSD_DayTestShift + 1 < GBPUSD_BarsTotal)
  {
   datetime GBPUSD_TimeNewer = iTime("GBPUSD", 0, GBPUSD_DayTestShift);
   datetime GBPUSD_TimeOlder = iTime("GBPUSD", 0, GBPUSD_DayTestShift + 1);

   if(GBPUSD_TimeNewer == 0 || GBPUSD_TimeOlder == 0)
      break;   // GBPUSD history not loaded - stop instead of reading bad data

   if(TimeDayOfYear(GBPUSD_TimeNewer) != TimeDayOfYear(GBPUSD_TimeOlder))
     {
      GBPUSD_DayStartShift = GBPUSD_DayTestShift;
      break;
     }
   GBPUSD_DayTestShift++;
  }

double GBPUSD_DailyLongEntry     = 0;
double GBPUSD_DailyShortEntry    = 0;
bool   GBPUSD_DailyLevelsValid   = false;   // include this in your entry conditions

if(GBPUSD_DayStartShift != -1)
  {
   // MODE_HIGH / MODE_LOW (not PRICE_HIGH / PRICE_LOW) are the correct constants for iHighest/iLowest
   int GBPUSD_HighIdx = iHighest("GBPUSD", 0, MODE_HIGH, GBPUSD_DayStartShift - 0 + 1, 0 + 1);
   int GBPUSD_LowIdx  = iLowest ("GBPUSD", 0, MODE_LOW,  GBPUSD_DayStartShift - 0 + 1, 0 + 1);

   if(GBPUSD_HighIdx >= 0 && GBPUSD_LowIdx >= 0)
     {
      GBPUSD_DailyLongEntry   = iHigh("GBPUSD", 0, GBPUSD_HighIdx);
      GBPUSD_DailyShortEntry  = iLow ("GBPUSD", 0, GBPUSD_LowIdx);
      GBPUSD_DailyLevelsValid = (GBPUSD_DailyLongEntry > 0 && GBPUSD_DailyShortEntry > 0);
     }
  }

///////////////////////////////////////////////////////////////////////////////////////////////////


///////////////EURUSD_DAILYLONG_AND_SHORTENTRY///////////////////////////////////////////////////////

// Same logic, on EURUSD's own bars.
int EURUSD_DayStartShift = -1;
int EURUSD_DayTestShift  = 0;
int EURUSD_BarsTotal     = iBars("EURUSD", 0);

while(EURUSD_DayStartShift == -1 && EURUSD_DayTestShift + 1 < EURUSD_BarsTotal)
  {
   datetime EURUSD_TimeNewer = iTime("EURUSD", 0, EURUSD_DayTestShift);
   datetime EURUSD_TimeOlder = iTime("EURUSD", 0, EURUSD_DayTestShift + 1);

   if(EURUSD_TimeNewer == 0 || EURUSD_TimeOlder == 0)
      break;   // EURUSD history not loaded - stop instead of reading bad data

   if(TimeDayOfYear(EURUSD_TimeNewer) != TimeDayOfYear(EURUSD_TimeOlder))
     {
      EURUSD_DayStartShift = EURUSD_DayTestShift;
      break;
     }
   EURUSD_DayTestShift++;
  }

double EURUSD_DailyLongEntry     = 0;
double EURUSD_DailyShortEntry    = 0;
bool   EURUSD_DailyLevelsValid   = false;   // include this in your entry conditions

if(EURUSD_DayStartShift != -1)
  {
   int EURUSD_HighIdx = iHighest("EURUSD", 0, MODE_HIGH, EURUSD_DayStartShift - 0 + 1, 0 + 1);
   int EURUSD_LowIdx  = iLowest ("EURUSD", 0, MODE_LOW,  EURUSD_DayStartShift - 0 + 1, 0 + 1);

   if(EURUSD_HighIdx >= 0 && EURUSD_LowIdx >= 0)
     {
      EURUSD_DailyLongEntry   = iHigh("EURUSD", 0, EURUSD_HighIdx);
      EURUSD_DailyShortEntry  = iLow ("EURUSD", 0, EURUSD_LowIdx);
      EURUSD_DailyLevelsValid = (EURUSD_DailyLongEntry > 0 && EURUSD_DailyShortEntry > 0);
     }
  }

///////////////////////////////////////////////////////////////////////////////////////////////////


///////////////CURRENT CHART: DAILYLONG_AND_SHORTENTRY///////////////////////////////////////////////

// Find the first bar of the current day on the chart's own bars.
int DayStartShift = -1;
int DayTestShift  = 0;
int DayBarsTotal  = iBars(NULL, 0);

while(DayStartShift == -1 && DayTestShift + 1 < DayBarsTotal)
  {
   datetime Day_TimeNewer = iTime(NULL, 0, DayTestShift);
   datetime Day_TimeOlder = iTime(NULL, 0, DayTestShift + 1);

   if(Day_TimeNewer == 0 || Day_TimeOlder == 0)
      break;   // history not loaded - stop instead of reading bad data

   if(TimeDayOfYear(Day_TimeNewer) != TimeDayOfYear(Day_TimeOlder))
     {
      DayStartShift = DayTestShift;
      break;
     }
   DayTestShift++;
  }

double DailyLongEntry   = 0;
double DailyShortEntry  = 0;
bool   DailyLevelsValid = false;   // include this in your entry conditions

if(DayStartShift != -1)
  {
   // MODE_HIGH / MODE_LOW (not PRICE_HIGH / PRICE_LOW) are the correct constants for iHighest/iLowest
   int Day_HighIdx = iHighest(NULL, 0, MODE_HIGH, DayStartShift - 0 + 1, 0 + 1);
   int Day_LowIdx  = iLowest (NULL, 0, MODE_LOW,  DayStartShift - 0 + 1, 0 + 1);

   if(Day_HighIdx >= 0 && Day_LowIdx >= 0)
     {
      DailyLongEntry   = iHigh(NULL, 0, Day_HighIdx);
      DailyShortEntry  = iLow (NULL, 0, Day_LowIdx);
      DailyLevelsValid = (DailyLongEntry > 0 && DailyShortEntry > 0);
     }
  }


//
   ObjectDelete(" DayStartShift ");
   ObjectCreate(" DayStartShift ",OBJ_VLINE,0,Time[DayStartShift],0);
   ObjectSet(" DayStartShift ",OBJPROP_COLOR,Yellow );
   ObjectSet(" DayStartShift ",OBJPROP_STYLE,STYLE_DASH);

 
 
 //
///////////////////////////////////////////////////////////////////////////////////////////////////

  
   
  DrawBearishVolumeLine();
  DrawBullishVolumeLine() ;
  
  
    M5LowerLow_on_M1timefame() ;
   
   M5HigherHigh_on_M1timeframe();
  
   /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////




   int N1PointShift=1;
   string N1ConfirmedPoint="Not Found";
   string N1PointShiftDirection="None";

   while(N1ConfirmedPoint!="Found")
     {
      double N1ZZ=iCustom(NULL,PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift);
      if(iHigh(NULL,PERIOD_M1,N1PointShift)==N1ZZ || iLow(NULL,PERIOD_M1,N1PointShift)==N1ZZ)
        {
         N1ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_M1,N1PointShift)==N1ZZ)
           {
            N1PointShiftDirection="N1zigzagLow";
            break;
           }
        }
      N1PointShift++;
     }

   int N1PointShift2=N1PointShift;
   string N1ConfirmedPoint2="Not Found";

   while(N1ConfirmedPoint2!="Found")
     {
      double N1ZZ2=iCustom(NULL,PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift2);
      if(iHigh(NULL,PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M1,N1PointShift2)==N1ZZ2 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint2="Found";
         break;
        }
      N1PointShift2++;
     }

   int N1PointShift3=N1PointShift2;
   string N1ConfirmedPoint3="Not Found";

   while(N1ConfirmedPoint3!="Found")
     {
      double N1ZZ3=iCustom(NULL,PERIOD_M1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N1PointShift3);
      if(iHigh(NULL,PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagHigh")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M1,N1PointShift3)==N1ZZ3 && N1PointShiftDirection=="N1zigzagLow")
        {
         N1ConfirmedPoint3="Found";
         break;
        }
      N1PointShift3++;
     }

   double N1ZZZ1 = iCustom(NULL, PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift);
   double N1ZZZ2 = iCustom(NULL, PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift2);
   double N1ZZZ3 = iCustom(NULL, PERIOD_M1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N1PointShift3);

   double N1zigzagLongEntry=0;
   double N1zigzagShortEntry=0;

   if(N1ZZZ3<N1ZZZ2 && N1ZZZ2>N1ZZZ1 && N1ZZZ1>N1ZZZ3)
     {
      N1zigzagLongEntry=N1ZZZ2+(PipBuffer*Point);
      N1zigzagShortEntry=0;
     }

   if(N1ZZZ3>N1ZZZ2 && N1ZZZ2<N1ZZZ1 && N1ZZZ1<N1ZZZ3)
     {
      N1zigzagLongEntry=0;
      N1zigzagShortEntry=N1ZZZ2 -(PipBuffer*Point);
     }


   

   int N5PointShift=1;
   string N5ConfirmedPoint="Not Found";
   string N5PointShiftDirection="None";

   while(N5ConfirmedPoint!="Found")
     {
      double N5ZZ=iCustom(NULL,PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift);
      if(iHigh(NULL,PERIOD_M5,N5PointShift)==N5ZZ || iLow(NULL,PERIOD_M5,N5PointShift)==N5ZZ)
        {
         N5ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_M5,N5PointShift)==N5ZZ)
           {
            N5PointShiftDirection="N5zigzagLow";
            break;
           }
        }
      N5PointShift++;
     }

   int N5PointShift2=N5PointShift;
   string N5ConfirmedPoint2="Not Found";

   while(N5ConfirmedPoint2!="Found")
     {
      double N5ZZ2=iCustom(NULL,PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift2);
      if(iHigh(NULL,PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M5,N5PointShift2)==N5ZZ2 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint2="Found";
         break;
        }
      N5PointShift2++;
     }

   int N5PointShift3=N5PointShift2;
   string N5ConfirmedPoint3="Not Found";

   while(N5ConfirmedPoint3!="Found")
     {
      double N5ZZ3=iCustom(NULL,PERIOD_M5,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N5PointShift3);
      if(iHigh(NULL,PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagHigh")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M5,N5PointShift3)==N5ZZ3 && N5PointShiftDirection=="N5zigzagLow")
        {
         N5ConfirmedPoint3="Found";
         break;
        }
      N5PointShift3++;
     }

   double N5ZZZ1 = iCustom(NULL,  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift);
   double N5ZZZ2 = iCustom(NULL,  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift2);
   double N5ZZZ3 = iCustom(NULL,  PERIOD_M5, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N5PointShift3);

   double N5zigzagLongEntry=0;
   double N5zigzagShortEntry=0;

   if(N5ZZZ3<N5ZZZ2 && N5ZZZ2>N5ZZZ1 && N5ZZZ1>N5ZZZ3)
     {
      N5zigzagLongEntry=N5ZZZ2+(PipBuffer*Point);
      N5zigzagShortEntry=0;
     }

   if(N5ZZZ3>N5ZZZ2 && N5ZZZ2<N5ZZZ1 && N5ZZZ1<N5ZZZ3)
     {
      N5zigzagLongEntry=0;
      N5zigzagShortEntry=N5ZZZ2 -(PipBuffer*Point);
     }

   
   int N15PointShift=1;
   string N15ConfirmedPoint="Not Found";
   string N15PointShiftDirection="None";

   while(N15ConfirmedPoint!="Found")
     {
      double N15ZZ=iCustom(NULL,PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift);
      if(iHigh(NULL,PERIOD_M15,N15PointShift)==N15ZZ || iLow(NULL,PERIOD_M15,N15PointShift)==N15ZZ)
        {
         N15ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_M15,N15PointShift)==N15ZZ)
           {
            N15PointShiftDirection="N15zigzagLow";
            break;
           }
        }
      N15PointShift++;
     }

   int N15PointShift2=N15PointShift;
   string N15ConfirmedPoint2="Not Found";

   while(N15ConfirmedPoint2!="Found")
     {
      double N15ZZ2=iCustom(NULL,PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift2);
      if(iHigh(NULL,PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M15,N15PointShift2)==N15ZZ2 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint2="Found";
         break;
        }
      N15PointShift2++;
     }

   int N15PointShift3=N15PointShift2;
   string N15ConfirmedPoint3="Not Found";

   while(N15ConfirmedPoint3!="Found")
     {
      double N15ZZ3=iCustom(NULL,PERIOD_M15,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N15PointShift3);
      if(iHigh(NULL,PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagHigh")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M15,N15PointShift3)==N15ZZ3 && N15PointShiftDirection=="N15zigzagLow")
        {
         N15ConfirmedPoint3="Found";
         break;
        }
      N15PointShift3++;
     }

   double N15ZZZ1 = iCustom(NULL, PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift);
   double N15ZZZ2 = iCustom(NULL, PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift2);
   double N15ZZZ3 = iCustom(NULL, PERIOD_M15, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N15PointShift3);

   double N15zigzagLongEntry=0;
   double N15zigzagShortEntry=0;

   if(N15ZZZ3<N15ZZZ2 && N15ZZZ2>N15ZZZ1 && N15ZZZ1>N15ZZZ3)
     {
      N15zigzagLongEntry=N15ZZZ2+(PipBuffer*Point);
      N15zigzagShortEntry=0;
     }

   if(N15ZZZ3>N15ZZZ2 && N15ZZZ2<N15ZZZ1 && N15ZZZ1<N15ZZZ3)
     {
      N15zigzagLongEntry=0;
      N15zigzagShortEntry=N15ZZZ2 -(PipBuffer*Point);
     }


  
   int N30PointShift=1;
   string N30ConfirmedPoint="Not Found";
   string N30PointShiftDirection="None";

   while(N30ConfirmedPoint!="Found")
     {
      double N30ZZ=iCustom(NULL,PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift);
      if(iHigh(NULL,PERIOD_M30,N30PointShift)==N30ZZ || iLow(NULL,PERIOD_M30,N30PointShift)==N30ZZ)
        {
         N30ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_M30,N30PointShift)==N30ZZ)
           {
            N30PointShiftDirection="N30zigzagLow";
            break;
           }
        }
      N30PointShift++;
     }

   int N30PointShift2=N30PointShift;
   string N30ConfirmedPoint2="Not Found";

   while(N30ConfirmedPoint2!="Found")
     {
      double N30ZZ2=iCustom(NULL,PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift2);
      if(iHigh(NULL,PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M30,N30PointShift2)==N30ZZ2 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint2="Found";
         break;
        }
      N30PointShift2++;
     }

   int N30PointShift3=N30PointShift2;
   string N30ConfirmedPoint3="Not Found";

   while(N30ConfirmedPoint3!="Found")
     {
      double N30ZZ3=iCustom(NULL,PERIOD_M30,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,N30PointShift3);
      if(iHigh(NULL,PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagHigh")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M30,N30PointShift3)==N30ZZ3 && N30PointShiftDirection=="N30zigzagLow")
        {
         N30ConfirmedPoint3="Found";
         break;
        }
      N30PointShift3++;
     }

   double N30ZZZ1 = iCustom(NULL, PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift);
   double N30ZZZ2 = iCustom(NULL, PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift2);
   double N30ZZZ3 = iCustom(NULL, PERIOD_M30, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, N30PointShift3);

   double N30zigzagLongEntry=0;
   double N30zigzagShortEntry=0;

   if(N30ZZZ3<N30ZZZ2 && N30ZZZ2>N30ZZZ1 && N30ZZZ1>N30ZZZ3)
     {
      N30zigzagLongEntry=N30ZZZ2+(PipBuffer*Point);
      N30zigzagShortEntry=0;
     }

   if(N30ZZZ3>N30ZZZ2 && N30ZZZ2<N30ZZZ1 && N30ZZZ1<N30ZZZ3)
     {
      N30zigzagLongEntry=0;
      N30zigzagShortEntry=N30ZZZ2 -(PipBuffer*Point);
     }

   
   int G1PointShift=1;
   string G1ConfirmedPoint="Not Found";
   string G1PointShiftDirection="None";

   while(G1ConfirmedPoint!="Found")
     {
      double G1ZZ=iCustom(NULL,PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift);
      if(iHigh(NULL,PERIOD_H1,G1PointShift)==G1ZZ || iLow(NULL,PERIOD_H1,G1PointShift)==G1ZZ)
        {
         G1ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_H1,G1PointShift)==G1ZZ)
           {
            G1PointShiftDirection="G1zigzagLow";
            break;
           }
        }
      G1PointShift++;
     }

   int G1PointShift2=G1PointShift;
   string G1ConfirmedPoint2="Not Found";

   while(G1ConfirmedPoint2!="Found")
     {
      double G1ZZ2=iCustom(NULL,PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift2);
      if(iHigh(NULL,PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_H1,G1PointShift2)==G1ZZ2 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint2="Found";
         break;
        }
      G1PointShift2++;
     }

   int G1PointShift3=G1PointShift2;
   string G1ConfirmedPoint3="Not Found";

   while(G1ConfirmedPoint3!="Found")
     {
      double G1ZZ3=iCustom(NULL,PERIOD_H1,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G1PointShift3);
      if(iHigh(NULL,PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagHigh")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_H1,G1PointShift3)==G1ZZ3 && G1PointShiftDirection=="G1zigzagLow")
        {
         G1ConfirmedPoint3="Found";
         break;
        }
      G1PointShift3++;
     }

   double G1ZZZ1 = iCustom(NULL, PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift);
   double G1ZZZ2 = iCustom(NULL, PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift2);
   double G1ZZZ3 = iCustom(NULL, PERIOD_H1, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G1PointShift3);

   double G1zigzagLongEntry=0;
   double G1zigzagShortEntry=0;

   if(G1ZZZ3<G1ZZZ2 && G1ZZZ2>G1ZZZ1 && G1ZZZ1>G1ZZZ3)
     {
      G1zigzagLongEntry=G1ZZZ2+(PipBuffer*Point);
      G1zigzagShortEntry=0;
     }

   if(G1ZZZ3>G1ZZZ2 && G1ZZZ2<G1ZZZ1 && G1ZZZ1<G1ZZZ3)
     {
      G1zigzagLongEntry=0;
      G1zigzagShortEntry=G1ZZZ2 -(PipBuffer*Point);
     }

   
   int G4PointShift=1;
   string G4ConfirmedPoint="Not Found";
   string G4PointShiftDirection="None";

   while(G4ConfirmedPoint!="Found")
     {
      double G4ZZ=iCustom(NULL,PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift);
      if(iHigh(NULL,PERIOD_H4,G4PointShift)==G4ZZ || iLow(NULL,PERIOD_H4,G4PointShift)==G4ZZ)
        {
         G4ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_H4,G4PointShift)==G4ZZ)
           {
            G4PointShiftDirection="G4zigzagLow";
            break;
           }
        }
      G4PointShift++;
     }

   int G4PointShift2=G4PointShift;
   string G4ConfirmedPoint2="Not Found";

   while(G4ConfirmedPoint2!="Found")
     {
      double G4ZZ2=iCustom(NULL,PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift2);
      if(iHigh(NULL,PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_H4,G4PointShift2)==G4ZZ2 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint2="Found";
         break;
        }
      G4PointShift2++;
     }

   int G4PointShift3=G4PointShift2;
   string G4ConfirmedPoint3="Not Found";

   while(G4ConfirmedPoint3!="Found")
     {
      double G4ZZ3=iCustom(NULL,PERIOD_H4,"ZigZag",ExtDept,ExtDeviatn,ExtBakstep,0,G4PointShift3);
      if(iHigh(NULL,PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagHigh")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_H4,G4PointShift3)==G4ZZ3 && G4PointShiftDirection=="G4zigzagLow")
        {
         G4ConfirmedPoint3="Found";
         break;
        }
      G4PointShift3++;
     }

   double G4ZZZ1 = iCustom(NULL, PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift);
   double G4ZZZ2 = iCustom(NULL, PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift2);
   double G4ZZZ3 = iCustom(NULL, PERIOD_H4, "ZigZag", ExtDept, ExtDeviatn, ExtBakstep, 0, G4PointShift3);

   double G4zigzagLongEntry=0;
   double G4zigzagShortEntry=0;

   if(G4ZZZ3<G4ZZZ2 && G4ZZZ2>G4ZZZ1 && G4ZZZ1>G4ZZZ3)
     {
      G4zigzagLongEntry=G4ZZZ2+(PipBuffer*Point);
      G4zigzagShortEntry=0;
     }

   if(G4ZZZ3>G4ZZZ2 && G4ZZZ2<G4ZZZ1 && G4ZZZ1<G4ZZZ3)
     {
      G4zigzagLongEntry=0;
      G4zigzagShortEntry=G4ZZZ2 -(PipBuffer*Point);
     }

 




//--------------------------------------------------------------------------------------------------

// START M1 ZIGZAG TO H4 ZIGZAG


//--------------------------------------------------------------------------------------------------



   int M5PointShift=1;
   string M5ConfirmedPoint="Not Found";
   string M5PointShiftDirection="None";

   while(M5ConfirmedPoint!="Found")
     {
      double M5ZZ=iCustom(NULL,PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift);
      if(iHigh(NULL,PERIOD_M5,M5PointShift)==M5ZZ || iLow(NULL,PERIOD_M5,M5PointShift)==M5ZZ)
        {
         M5ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_M5,M5PointShift)==M5ZZ)
           {
            M5PointShiftDirection="M5zigzagLow";
            break;
           }
        }
      M5PointShift++;
     }

   int M5PointShift2=M5PointShift;
   string M5ConfirmedPoint2="Not Found";

   while(M5ConfirmedPoint2!="Found")
     {
      double M5ZZ2=iCustom(NULL,PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift2);
      if(iHigh(NULL,PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M5,M5PointShift2)==M5ZZ2 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint2="Found";
         break;
        }
      M5PointShift2++;
     }

   int M5PointShift3=M5PointShift2;
   string M5ConfirmedPoint3="Not Found";

   while(M5ConfirmedPoint3!="Found")
     {
      double M5ZZ3=iCustom(NULL,PERIOD_M5,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M5PointShift3);
      if(iHigh(NULL,PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagHigh")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M5,M5PointShift3)==M5ZZ3 && M5PointShiftDirection=="M5zigzagLow")
        {
         M5ConfirmedPoint3="Found";
         break;
        }
      M5PointShift3++;
     }

   double M5ZZZ1 = iCustom(NULL,  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift);
   double M5ZZZ2 = iCustom(NULL,  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift2);
   double M5ZZZ3 = iCustom(NULL,  PERIOD_M5, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M5PointShift3);

   double M5zigzagLongEntry=0;
   double M5zigzagShortEntry=0;

   if(M5ZZZ3<M5ZZZ2 && M5ZZZ2>M5ZZZ1 && M5ZZZ1>=M5ZZZ3)
     {
      M5zigzagLongEntry=M5ZZZ2+(PipBuffer*Point);
      M5zigzagShortEntry=0;
     }

   if(M5ZZZ3>M5ZZZ2 && M5ZZZ2<M5ZZZ1 && M5ZZZ1<=M5ZZZ3)
     {
      M5zigzagLongEntry=0;
      M5zigzagShortEntry=M5ZZZ2 -(PipBuffer*Point);
     }

   int M15PointShift=1;
   string M15ConfirmedPoint="Not Found";
   string M15PointShiftDirection="None";

   while(M15ConfirmedPoint!="Found")
     {
      double M15ZZ=iCustom(NULL,PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift);
      if(iHigh(NULL,PERIOD_M15,M15PointShift)==M15ZZ || iLow(NULL,PERIOD_M15,M15PointShift)==M15ZZ)
        {
         M15ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_M15,M15PointShift)==M15ZZ)
           {
            M15PointShiftDirection="M15zigzagLow";
            break;
           }
        }
      M15PointShift++;
     }

   int M15PointShift2=M15PointShift;
   string M15ConfirmedPoint2="Not Found";

   while(M15ConfirmedPoint2!="Found")
     {
      double M15ZZ2=iCustom(NULL,PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift2);
      if(iHigh(NULL,PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M15,M15PointShift2)==M15ZZ2 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint2="Found";
         break;
        }
      M15PointShift2++;
     }

   int M15PointShift3=M15PointShift2;
   string M15ConfirmedPoint3="Not Found";

   while(M15ConfirmedPoint3!="Found")
     {
      double M15ZZ3=iCustom(NULL,PERIOD_M15,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M15PointShift3);
      if(iHigh(NULL,PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagHigh")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M15,M15PointShift3)==M15ZZ3 && M15PointShiftDirection=="M15zigzagLow")
        {
         M15ConfirmedPoint3="Found";
         break;
        }
      M15PointShift3++;
     }

   double M15ZZZ1 = iCustom(NULL, PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift);
   double M15ZZZ2 = iCustom(NULL, PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift2);
   double M15ZZZ3 = iCustom(NULL, PERIOD_M15, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M15PointShift3);

   double M15zigzagLongEntry=0;
   double M15zigzagShortEntry=0;

   if(M15ZZZ3<M15ZZZ2 && M15ZZZ2>M15ZZZ1 && M15ZZZ1>=M15ZZZ3)
     {
      M15zigzagLongEntry=M15ZZZ2+(PipBuffer*Point);
      M15zigzagShortEntry=0;
     }

   if(M15ZZZ3>M15ZZZ2 && M15ZZZ2<M15ZZZ1 && M15ZZZ1<=M15ZZZ3)
     {
      M15zigzagLongEntry=0;
      M15zigzagShortEntry=M15ZZZ2 -(PipBuffer*Point);
     }


   int M30PointShift=1;
   string M30ConfirmedPoint="Not Found";
   string M30PointShiftDirection="None";

   while(M30ConfirmedPoint!="Found")
     {
      double M30ZZ=iCustom(NULL,PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift);
      if(iHigh(NULL,PERIOD_M30,M30PointShift)==M30ZZ || iLow(NULL,PERIOD_M30,M30PointShift)==M30ZZ)
        {
         M30ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_M30,M30PointShift)==M30ZZ)
           {
            M30PointShiftDirection="M30zigzagLow";
            break;
           }
        }
      M30PointShift++;
     }

   int M30PointShift2=M30PointShift;
   string M30ConfirmedPoint2="Not Found";

   while(M30ConfirmedPoint2!="Found")
     {
      double M30ZZ2=iCustom(NULL,PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift2);
      if(iHigh(NULL,PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M30,M30PointShift2)==M30ZZ2 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint2="Found";
         break;
        }
      M30PointShift2++;
     }

   int M30PointShift3=M30PointShift2;
   string M30ConfirmedPoint3="Not Found";

   while(M30ConfirmedPoint3!="Found")
     {
      double M30ZZ3=iCustom(NULL,PERIOD_M30,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,M30PointShift3);
      if(iHigh(NULL,PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagHigh")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M30,M30PointShift3)==M30ZZ3 && M30PointShiftDirection=="M30zigzagLow")
        {
         M30ConfirmedPoint3="Found";
         break;
        }
      M30PointShift3++;
     }

   double M30ZZZ1 = iCustom(NULL, PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift);
   double M30ZZZ2 = iCustom(NULL, PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift2);
   double M30ZZZ3 = iCustom(NULL, PERIOD_M30, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, M30PointShift3);

   double M30zigzagLongEntry=0;
   double M30zigzagShortEntry=0;

   if(M30ZZZ3<M30ZZZ2 && M30ZZZ2>M30ZZZ1 && M30ZZZ1>=M30ZZZ3)
     {
      M30zigzagLongEntry=M30ZZZ2+(PipBuffer*Point);
      M30zigzagShortEntry=0;
     }

   if(M30ZZZ3>M30ZZZ2 && M30ZZZ2<M30ZZZ1 && M30ZZZ1<=M30ZZZ3)
     {
      M30zigzagLongEntry=0;
      M30zigzagShortEntry=M30ZZZ2 -(PipBuffer*Point);
     }



   int H1PointShift=1;
   string H1ConfirmedPoint="Not Found";
   string H1PointShiftDirection="None";

   while(H1ConfirmedPoint!="Found")
     {
      double H1ZZ=iCustom(NULL,PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift);
      if(iHigh(NULL,PERIOD_H1,H1PointShift)==H1ZZ || iLow(NULL,PERIOD_H1,H1PointShift)==H1ZZ)
        {
         H1ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_H1,H1PointShift)==H1ZZ)
           {
            H1PointShiftDirection="H1zigzagLow";
            break;
           }
        }
      H1PointShift++;
     }

   int H1PointShift2=H1PointShift;
   string H1ConfirmedPoint2="Not Found";

   while(H1ConfirmedPoint2!="Found")
     {
      double H1ZZ2=iCustom(NULL,PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift2);
      if(iHigh(NULL,PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_H1,H1PointShift2)==H1ZZ2 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint2="Found";
         break;
        }
      H1PointShift2++;
     }

   int H1PointShift3=H1PointShift2;
   string H1ConfirmedPoint3="Not Found";

   while(H1ConfirmedPoint3!="Found")
     {
      double H1ZZ3=iCustom(NULL,PERIOD_H1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H1PointShift3);
      if(iHigh(NULL,PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagHigh")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_H1,H1PointShift3)==H1ZZ3 && H1PointShiftDirection=="H1zigzagLow")
        {
         H1ConfirmedPoint3="Found";
         break;
        }
      H1PointShift3++;
     }

   double H1ZZZ1 = iCustom(NULL, PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift);
   double H1ZZZ2 = iCustom(NULL, PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift2);
   double H1ZZZ3 = iCustom(NULL, PERIOD_H1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H1PointShift3);

   double H1zigzagLongEntry=0;
   double H1zigzagShortEntry=0;

   if(H1ZZZ3<H1ZZZ2 && H1ZZZ2>H1ZZZ1 && H1ZZZ1>=H1ZZZ3)
     {
      H1zigzagLongEntry=H1ZZZ2+(PipBuffer*Point);
      H1zigzagShortEntry=0;
     }

   if(H1ZZZ3>H1ZZZ2 && H1ZZZ2<H1ZZZ1 && H1ZZZ1<=H1ZZZ3)
     {
      H1zigzagLongEntry=0;
      H1zigzagShortEntry=H1ZZZ2 -(PipBuffer*Point);
     }


   int H4PointShift=1;
   string H4ConfirmedPoint="Not Found";
   string H4PointShiftDirection="None";

   while(H4ConfirmedPoint!="Found")
     {
      double H4ZZ=iCustom(NULL,PERIOD_H4,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H4PointShift);
      if(iHigh(NULL,PERIOD_H4,H4PointShift)==H4ZZ || iLow(NULL,PERIOD_H4,H4PointShift)==H4ZZ)
        {
         H4ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_H4,H4PointShift)==H4ZZ)
           {
            H4PointShiftDirection="H4zigzagLow";
            break;
           }
        }
      H4PointShift++;
     }

   int H4PointShift2=H4PointShift;
   string H4ConfirmedPoint2="Not Found";

   while(H4ConfirmedPoint2!="Found")
     {
      double H4ZZ2=iCustom(NULL,PERIOD_H4,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H4PointShift2);
      if(iHigh(NULL,PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_H4,H4PointShift2)==H4ZZ2 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint2="Found";
         break;
        }
      H4PointShift2++;
     }

   int H4PointShift3=H4PointShift2;
   string H4ConfirmedPoint3="Not Found";

   while(H4ConfirmedPoint3!="Found")
     {
      double H4ZZ3=iCustom(NULL,PERIOD_H4,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,H4PointShift3);
      if(iHigh(NULL,PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagHigh")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_H4,H4PointShift3)==H4ZZ3 && H4PointShiftDirection=="H4zigzagLow")
        {
         H4ConfirmedPoint3="Found";
         break;
        }
      H4PointShift3++;
     }

   double H4ZZZ1 = iCustom(NULL, PERIOD_H4, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H4PointShift);
   double H4ZZZ2 = iCustom(NULL, PERIOD_H4, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H4PointShift2);
   double H4ZZZ3 = iCustom(NULL, PERIOD_H4, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, H4PointShift3);

   double H4zigzagLongEntry=0;
   double H4zigzagShortEntry=0;

   if(H4ZZZ3<H4ZZZ2 && H4ZZZ2>H4ZZZ1 && H4ZZZ1>=H4ZZZ3)
     {
      H4zigzagLongEntry=H4ZZZ2+(PipBuffer*Point);
      H4zigzagShortEntry=0;
     }

   if(H4ZZZ3>H4ZZZ2 && H4ZZZ2<H4ZZZ1 && H4ZZZ1<=H4ZZZ3)
     {
      H4zigzagLongEntry=0;
      H4zigzagShortEntry=H4ZZZ2 -(PipBuffer*Point);
     }



   int PointShift=1;
   string ConfirmedPoint="Not Found";
   string PointShiftDirection="None";

   while(ConfirmedPoint!="Found")
     {
      double ZZ=iCustom(NULL,PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift);
      if(iHigh(NULL,PERIOD_M1,PointShift)==ZZ || iLow(NULL,PERIOD_M1,PointShift)==ZZ)
        {
         ConfirmedPoint="Found";
         if(iHigh(NULL,PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagHigh";
            break;
           }
         if(iLow(NULL,PERIOD_M1,PointShift)==ZZ)
           {
            PointShiftDirection="zigzagLow";
            break;
           }
        }
      PointShift++;
     }

   int PointShift2=PointShift;
   string ConfirmedPoint2="Not Found";

   while(ConfirmedPoint2!="Found")
     {
      double ZZ2=iCustom(NULL,PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift2);
      if(iHigh(NULL,PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint2="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M1,PointShift2)==ZZ2 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint2="Found";
         break;
        }
      PointShift2++;
     }

   int PointShift3=PointShift2;
   string ConfirmedPoint3="Not Found";

   while(ConfirmedPoint3!="Found")
     {
      double ZZ3=iCustom(NULL,PERIOD_M1,"ZigZag",ExtDepth,ExtDeviation,ExtBackstep,0,PointShift3);
      if(iHigh(NULL,PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagHigh")
        {
         ConfirmedPoint3="Found";
         break;
        }
      if(iLow(NULL,PERIOD_M1,PointShift3)==ZZ3 && PointShiftDirection=="zigzagLow")
        {
         ConfirmedPoint3="Found";
         break;
        }
      PointShift3++;
     }

   double ZZZ1 = iCustom(NULL, PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift);
   double ZZZ2 = iCustom(NULL, PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift2);
   double ZZZ3 = iCustom(NULL, PERIOD_M1, "ZigZag", ExtDepth, ExtDeviation, ExtBackstep, 0, PointShift3);

   double zigzagLongEntry=0;
   double zigzagShortEntry=0;

   if(ZZZ3<ZZZ2 && ZZZ2>ZZZ1 && ZZZ1>=ZZZ3)
     {
      zigzagLongEntry=ZZZ2+(PipBuffer*Point);
      zigzagShortEntry=0;
     }

   if(ZZZ3>ZZZ2 && ZZZ2<ZZZ1 && ZZZ1<=ZZZ3)
     {
      zigzagLongEntry=0;
      zigzagShortEntry=ZZZ2 -(PipBuffer*Point);
     }


/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

    /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

   ObjectDelete(" zigzagLongEntry ");
   ObjectCreate(" zigzagLongEntry ",OBJ_HLINE,0,0,zigzagLongEntry );
   ObjectSet(" zigzagLongEntry",OBJPROP_COLOR,Green );
   ObjectSet(" zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" zigzagLongEntry ",OBJPROP_BACK,True);



   ObjectDelete(" M5zigzagLongEntry ");
   ObjectCreate(" M5zigzagLongEntry ",OBJ_HLINE,0,0,M5zigzagLongEntry );
   ObjectSet(" M5zigzagLongEntry",OBJPROP_COLOR,Green );
   ObjectSet(" M5zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" M5zigzagLongEntry ",OBJPROP_BACK,True);


  ObjectDelete(" M15zigzagLongEntry ");
   ObjectCreate(" M15zigzagLongEntry ",OBJ_HLINE,0,0,M15zigzagLongEntry );
   ObjectSet(" M15zigzagLongEntry",OBJPROP_COLOR,Green );
   ObjectSet(" M15zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" M15zigzagLongEntry ",OBJPROP_BACK,True);


  ObjectDelete(" M30zigzagLongEntry ");
   ObjectCreate(" M30zigzagLongEntry ",OBJ_HLINE,0,0,M30zigzagLongEntry );
   ObjectSet(" M30zigzagLongEntry",OBJPROP_COLOR,Green );
   ObjectSet(" M30zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" M30zigzagLongEntry ",OBJPROP_BACK,True);

 ObjectDelete(" H1zigzagLongEntry ");
   ObjectCreate(" H1zigzagLongEntry ",OBJ_HLINE,0,0,H1zigzagLongEntry );
   ObjectSet(" H1zigzagLongEntry",OBJPROP_COLOR,Green );
   ObjectSet(" H1zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" H1zigzagLongEntry ",OBJPROP_BACK,True);

 ObjectDelete(" H4zigzagLongEntry ");
   ObjectCreate(" H4zigzagLongEntry ",OBJ_HLINE,0,0,H4zigzagLongEntry );
   ObjectSet(" H4zigzagLongEntry",OBJPROP_COLOR,Green );
   ObjectSet(" H4zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" H4zigzagLongEntry ",OBJPROP_BACK,True);
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
 ObjectDelete(" zigzagShortEntry ");
   ObjectCreate(" zigzagShortEntry ",OBJ_HLINE,0,0,zigzagShortEntry );
   ObjectSet(" zigzagShortEntry ",OBJPROP_COLOR,Red );
   ObjectSet(" zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" zigzagShortEntry",OBJPROP_BACK,True);


ObjectDelete(" M5zigzagShortEntry ");
   ObjectCreate(" M5zigzagShortEntry ",OBJ_HLINE,0,0,M5zigzagShortEntry );
   ObjectSet(" M5zigzagShortEntry ",OBJPROP_COLOR,Red );
   ObjectSet(" M5zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" M5zigzagShortEntry",OBJPROP_BACK,True);


ObjectDelete(" M15zigzagShortEntry ");
   ObjectCreate(" M15zigzagShortEntry ",OBJ_HLINE,0,0,M15zigzagShortEntry );
   ObjectSet(" M15zigzagShortEntry ",OBJPROP_COLOR,Red );
   ObjectSet(" M15zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" M15zigzagShortEntry",OBJPROP_BACK,True);


ObjectDelete(" M30zigzagShortEntry ");
   ObjectCreate(" M30zigzagShortEntry ",OBJ_HLINE,0,0,M30zigzagShortEntry );
   ObjectSet(" M30zigzagShortEntry ",OBJPROP_COLOR,Red );
   ObjectSet(" M30zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" M30zigzagShortEntry",OBJPROP_BACK,True);

ObjectDelete(" H1zigzagShortEntry ");
   ObjectCreate(" H1zigzagShortEntry ",OBJ_HLINE,0,0,H1zigzagShortEntry );
   ObjectSet(" H1zigzagShortEntry ",OBJPROP_COLOR,Red );
   ObjectSet(" H1zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" H1zigzagShortEntry",OBJPROP_BACK,True);


ObjectDelete(" H4zigzagShortEntry ");
   ObjectCreate(" H4zigzagShortEntry ",OBJ_HLINE,0,0,H4zigzagShortEntry );
   ObjectSet(" H4zigzagShortEntry ",OBJPROP_COLOR,Red );
   ObjectSet(" H4zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" H4zigzagShortEntry",OBJPROP_BACK,True);


/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

 
    
   
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

   ObjectDelete(" N1zigzagLongEntry ");
   ObjectCreate(" N1zigzagLongEntry ",OBJ_HLINE,0,0,N1zigzagLongEntry);
   ObjectSet(" N1zigzagLongEntry",OBJPROP_COLOR,Green);
   ObjectSet(" N1zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" N1zigzagLongEntry ",OBJPROP_BACK,True);



   ObjectDelete(" N5zigzagLongEntry ");
   ObjectCreate(" N5zigzagLongEntry ",OBJ_HLINE,0,0,N5zigzagLongEntry);
   ObjectSet(" N5zigzagLongEntry",OBJPROP_COLOR,Green);
   ObjectSet(" N5zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" N5zigzagLongEntry ",OBJPROP_BACK,True);


   ObjectDelete(" N15zigzagLongEntry ");
   ObjectCreate(" N15zigzagLongEntry ",OBJ_HLINE,0,0,N15zigzagLongEntry);
   ObjectSet(" N15zigzagLongEntry",OBJPROP_COLOR,Green);
   ObjectSet(" N15zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" N15zigzagLongEntry ",OBJPROP_BACK,True);


   ObjectDelete(" N30zigzagLongEntry ");
   ObjectCreate(" N30zigzagLongEntry ",OBJ_HLINE,0,0,N30zigzagLongEntry);
   ObjectSet(" N30zigzagLongEntry",OBJPROP_COLOR,Green);
   ObjectSet(" N30zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" N30zigzagLongEntry ",OBJPROP_BACK,True);

   ObjectDelete(" G1zigzagLongEntry ");
   ObjectCreate(" G1zigzagLongEntry ",OBJ_HLINE,0,0,G1zigzagLongEntry);
   ObjectSet(" G1zigzagLongEntry",OBJPROP_COLOR,Green);
   ObjectSet(" G1zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" G1zigzagLongEntry ",OBJPROP_BACK,True);

   ObjectDelete(" G4zigzagLongEntry ");
   ObjectCreate(" G4zigzagLongEntry ",OBJ_HLINE,0,0,G4zigzagLongEntry);
   ObjectSet(" G4zigzagLongEntry",OBJPROP_COLOR,Green);
   ObjectSet(" G4zigzagLongEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" G4zigzagLongEntry ",OBJPROP_BACK,True);
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
   ObjectDelete(" N1zigzagShortEntry ");
   ObjectCreate(" N1zigzagShortEntry ",OBJ_HLINE,0,0,N1zigzagShortEntry);
   ObjectSet(" N1zigzagShortEntry ",OBJPROP_COLOR,Red);
   ObjectSet(" N1zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" N1zigzagShortEntry",OBJPROP_BACK,True);


   ObjectDelete(" N5zigzagShortEntry ");
   ObjectCreate(" N5zigzagShortEntry ",OBJ_HLINE,0,0,N5zigzagShortEntry);
   ObjectSet(" N5zigzagShortEntry ",OBJPROP_COLOR,Red);
   ObjectSet(" N5zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" N5zigzagShortEntry",OBJPROP_BACK,True);


   ObjectDelete(" N15zigzagShortEntry ");
   ObjectCreate(" N15zigzagShortEntry ",OBJ_HLINE,0,0,N15zigzagShortEntry);
   ObjectSet(" N15zigzagShortEntry ",OBJPROP_COLOR,Red);
   ObjectSet(" N15zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" N15zigzagShortEntry",OBJPROP_BACK,True);


   ObjectDelete(" N30zigzagShortEntry ");
   ObjectCreate(" N30zigzagShortEntry ",OBJ_HLINE,0,0,N30zigzagShortEntry);
   ObjectSet(" N30zigzagShortEntry ",OBJPROP_COLOR,Red);
   ObjectSet(" N30zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" N30zigzagShortEntry",OBJPROP_BACK,True);

   ObjectDelete(" G1zigzagShortEntry ");
   ObjectCreate(" G1zigzagShortEntry ",OBJ_HLINE,0,0,G1zigzagShortEntry);
   ObjectSet(" G1zigzagShortEntry ",OBJPROP_COLOR,Red);
   ObjectSet(" G1zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" G1zigzagShortEntry",OBJPROP_BACK,True);


   ObjectDelete(" G4zigzagShortEntry ");
   ObjectCreate(" G4zigzagShortEntry ",OBJ_HLINE,0,0,G4zigzagShortEntry);
   ObjectSet(" G4zigzagShortEntry ",OBJPROP_COLOR,Red);
   ObjectSet(" G4zigzagShortEntry ",OBJPROP_STYLE,STYLE_DASHDOTDOT);
   ObjectSet(" G4zigzagShortEntry",OBJPROP_BACK,True);


/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

   
  //////////////////////////////////////////////////////////////////////////////////////
 //////////////////////////////////////////////////////////////////////////////////////
// --- Detect Conditions Only ---
bool potentialBuyCond307 =
   (
      ( N1zigzagLongEntry  > 0 && N1ZZZ3  == GetLowestPointAfterDailyHigh() && ( N1ZZZ1  < PivotAtLowestPointAfterDailyHighM1() || N1ZZZ1  < GetHighAtLowestPointAfterDailyHighM1() ) ) 
     
   
   );

if(potentialBuyCond307)
  {
   buyCondMet307  = true;
   buyCondLow307  = GetLowAtLowestPointAfterDailyHighM1();
   buyCondTime307 = GetLowestPointAfterDailyHighTimeOnM1();

   // Visual arrow on chart (upward arrow)
   string buyArrowLabel307 = "BuyArrow307_" + TimeToStr(buyCondTime307, TIME_SECONDS);
   ObjectCreate(0, buyArrowLabel307, OBJ_ARROW, 0, Time[0], Low[0] - 10 * Point);
   ObjectSetInteger(0, buyArrowLabel307, OBJPROP_ARROWCODE, 233); // Wingdings up arrow
   ObjectSetInteger(0, buyArrowLabel307, OBJPROP_COLOR, clrOrchid);
   ObjectSetInteger(0, buyArrowLabel307, OBJPROP_WIDTH, 3);
  }
//////////////////////////////////////////////////////////////////////////////////////

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

    // --- Detect Conditions Only ---
   bool potentialSellCond307 = 
       
       (( N1zigzagShortEntry > 0 && N1ZZZ3 == GetHighestPointAfterDailyLow() &&  (( N1ZZZ1 > PivotAtHighestPointAfterDailyLowM1() ) || (N1ZZZ1 > GetLowAtHighestPointAfterDailyLowM1()) )  ))
   
      
      
       
   
   
     
     
   
   

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
 
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

   ;
  if ( potentialSellCond307) {
   
    sellCondMet307 = true;
   sellCondHigh307 = GetHighAtHighestPointAfterDailyLowM1();
   
   sellCondTime307 = GetHighestPointAfterDailyLowTimeOnM1();

                     
  
    // Visual arrow on chart (red downward arrow)
   string sellArrowLabel307 = "SellArrow307_" + TimeToStr(sellCondTime307, TIME_SECONDS);
   ObjectCreate(0, sellArrowLabel307, OBJ_ARROW, 0, Time[0], High[0] + 2 * Point);
   ObjectSetInteger(0, sellArrowLabel307, OBJPROP_ARROWCODE, 234); // Wingdings Down Arrow
   ObjectSetInteger(0, sellArrowLabel307, OBJPROP_COLOR, clrMediumOrchid);
   ObjectSetInteger(0, sellArrowLabel307, OBJPROP_WIDTH, 3);

  
  } 
  
  
  /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////


  /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////



  
//////////////////////////////////////////////////////////////////////////////////////
// --- Detect Conditions Only ---
bool potentialBuyCond308 =
   (
      ( N1zigzagLongEntry  > 0 && N1ZZZ3  == M5LowerLow_on_M1timefame() && N1ZZZ1 < PivotatM5LowerLow_on_M1timefame()   ) ||
      ( N5zigzagLongEntry  > 0 && N5ZZZ3  == M5LowerLow_on_M1timefame() && N5ZZZ1 < PivotatM5LowerLow_on_M1timefame()  ) ||
      ( N15zigzagLongEntry > 0 && N15ZZZ3 == M5LowerLow_on_M1timefame() && N15ZZZ1 < PivotatM5LowerLow_on_M1timefame()   ) ||
      ( N30zigzagLongEntry > 0 && N30ZZZ3 == M5LowerLow_on_M1timefame() && N30ZZZ1 < PivotatM5LowerLow_on_M1timefame()  ) ||
      ( G1zigzagLongEntry  > 0 && G1ZZZ3  == M5LowerLow_on_M1timefame() && G1ZZZ1 < PivotatM5LowerLow_on_M1timefame()  ) ||
      ( G4zigzagLongEntry  > 0 && G4ZZZ3  == M5LowerLow_on_M1timefame() && G4ZZZ1 < PivotatM5LowerLow_on_M1timefame()  ) ||

      ( zigzagLongEntry    > 0 && ZZZ3    == M5LowerLow_on_M1timefame() && ZZZ1 < PivotatM5LowerLow_on_M1timefame()  ) ||
      ( M5zigzagLongEntry  > 0 && M5ZZZ3  == M5LowerLow_on_M1timefame() && M5ZZZ1 < PivotatM5LowerLow_on_M1timefame()  ) ||
      ( M15zigzagLongEntry > 0 && M15ZZZ3 == M5LowerLow_on_M1timefame() && M15ZZZ1 < PivotatM5LowerLow_on_M1timefame()  ) ||
      ( M30zigzagLongEntry > 0 && M30ZZZ3 == M5LowerLow_on_M1timefame() && M30ZZZ1 < PivotatM5LowerLow_on_M1timefame()  ) ||
      ( H1zigzagLongEntry  > 0 && H1ZZZ3  == M5LowerLow_on_M1timefame() && H1ZZZ1 < PivotatM5LowerLow_on_M1timefame()   ) ||
      ( H4zigzagLongEntry  > 0 && H4ZZZ3  == M5LowerLow_on_M1timefame() && H4ZZZ1 < PivotatM5LowerLow_on_M1timefame()   ) )
   
   
  ;

if(potentialBuyCond308)
  {
   buyCondMet308  = true;
   buyCondLow308  = M5LowerLow_on_M1timefame();
   buyCondTime308 = TIMEof_M5LowerLow_on_M1timefame() ;

   // Visual arrow on chart (upward arrow)
   string buyArrowLabel308 = "BuyArrow308_" + TimeToStr(buyCondTime308, TIME_SECONDS);
   ObjectCreate(0, buyArrowLabel308, OBJ_ARROW, 0, Time[0], Low[0] - 10 * Point);
   ObjectSetInteger(0, buyArrowLabel308, OBJPROP_ARROWCODE, 233); // Wingdings up arrow
   ObjectSetInteger(0, buyArrowLabel308, OBJPROP_COLOR, clrWheat);
   ObjectSetInteger(0, buyArrowLabel308, OBJPROP_WIDTH, 3);
  }
//////////////////////////////////////////////////////////////////////////////////////

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

    // --- Detect Conditions Only ---
   bool potentialSellCond308 = 
       
       (( N1zigzagShortEntry > 0 && N1ZZZ3 == M5HigherHigh_on_M1timeframe() && N1ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()   )||
    ( N5zigzagShortEntry > 0 && N5ZZZ3 == M5HigherHigh_on_M1timeframe() && N5ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()  )||
    ( N15zigzagShortEntry > 0 && N15ZZZ3 == M5HigherHigh_on_M1timeframe() && N15ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()  )||
    ( N30zigzagShortEntry > 0 && N30ZZZ3 == M5HigherHigh_on_M1timeframe() && N30ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()  )||
    ( G1zigzagShortEntry > 0 && G1ZZZ3 == M5HigherHigh_on_M1timeframe() && G1ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()  )||
    ( G4zigzagShortEntry > 0 && G4ZZZ3 == M5HigherHigh_on_M1timeframe() && G4ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()  )||

    ( zigzagShortEntry > 0 && ZZZ3 == M5HigherHigh_on_M1timeframe() && ZZZ1 > PivotatM5HigherHigh_on_M1timeframe() )||
    ( M5zigzagShortEntry > 0 && M5ZZZ3 == M5HigherHigh_on_M1timeframe() && M5ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()  )||
    ( M15zigzagShortEntry > 0 && M15ZZZ3 == M5HigherHigh_on_M1timeframe() && M15ZZZ1 > PivotatM5HigherHigh_on_M1timeframe() )||
    ( M30zigzagShortEntry > 0 && M30ZZZ3 == M5HigherHigh_on_M1timeframe() && M30ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()  )||
    ( H1zigzagShortEntry > 0 && H1ZZZ3 == M5HigherHigh_on_M1timeframe() && H1ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()  )||
    ( H4zigzagShortEntry > 0 && H4ZZZ3 == M5HigherHigh_on_M1timeframe() && H4ZZZ1 > PivotatM5HigherHigh_on_M1timeframe()  ))
    
      
      
       
   
   
     
     
   
   

/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
 
/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

   ;
  if ( potentialSellCond308) {
   
    sellCondMet308 = true;
   sellCondHigh308 = M5HigherHigh_on_M1timeframe();
   
   sellCondTime308 = TIMEof_M5HigherHigh_on_M1timeframe();

                     
  
    // Visual arrow on chart (red downward arrow)
   string sellArrowLabel308 = "SellArrow308_" + TimeToStr(sellCondTime308, TIME_SECONDS);
   ObjectCreate(0, sellArrowLabel308, OBJ_ARROW, 0, Time[0], High[0] + 2 * Point);
   ObjectSetInteger(0, sellArrowLabel308, OBJPROP_ARROWCODE, 234); // Wingdings Down Arrow
   ObjectSetInteger(0, sellArrowLabel308, OBJPROP_COLOR, clrWheat);
   ObjectSetInteger(0, sellArrowLabel308, OBJPROP_WIDTH, 3);

  
  } 
  
  
  /////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

  

  
  
  
  
  
  
   double pipMultiplier = GetPipMultiplier(Symbol());
   double point = Point;

   // MASTER BLOCK: BUY - opens on this EA's own conditions, then broadcasts "BUY"
   // via REAL315E_SendSignal() (same flow as REAL_315E_Master_V1).
   if(Ask - Bid <= 3 * pipMultiplier * point)
   {
      if(TradeTimeAllowed() == true 
        
          
            && iTime("GBPUSD", 0, iHighest("GBPUSD", 0, MODE_HIGH, DayStartShift + 1, 1)) > iTime("GBPUSD", 0, iLowest("GBPUSD", 0, MODE_LOW, DayStartShift + 1, 1))
  
               && iTime("EURUSD", 0, iHighest("EURUSD", 0, MODE_HIGH, DayStartShift + 1, 1)) > iTime("EURUSD", 0, iLowest("EURUSD", 0, MODE_LOW, DayStartShift + 1, 1))

     
          
          
             &&  GBPUSD_GetLargestBullishM1_Time(20) >= GBPUSD_GetLowestPointAfterDailyHighTimeOnM1() && EURUSD_GetLargestBullishM1_Time( 20) >= EURUSD_GetLowestPointAfterDailyHighTimeOnM1()
       
       
            &&  GBPUSD_GetLargestBullishM1_Close( 20) - GBPUSD_GetLargestBullishM1_Open(  20) > GBPUSD_GetLargestBearishM1_Open(  20) - GBPUSD_GetLargestBearishM1_Close(  20)
              
             &&  EURUSD_GetLargestBullishM1_Close(  20) - EURUSD_GetLargestBullishM1_Open(  20) > EURUSD_GetLargestBearishM1_Open(  20) - EURUSD_GetLargestBearishM1_Close(  20)

              
       
       && !AlreadyTraded_60Minutes() && M5LowerLow_on_M1timefame() > 0 && iClose("EURUSD", PERIOD_M1, 0) > iOpen("EURUSD", PERIOD_M1, 0) && iClose("GBPUSD", PERIOD_M1, 0) > iOpen("GBPUSD", PERIOD_M1, 0)
            
            && GBPUSD_GetLowestPointAfterDailyHighTimeOnM1() > 0 && EURUSD_GetLowestPointAfterDailyHighTimeOnM1() > 0 && MathAbs((double)(GBPUSD_GetLowestPointAfterDailyHighTimeOnM1() - EURUSD_GetLowestPointAfterDailyHighTimeOnM1())) <= 30 * 60
            
        
      && (  ( M5LowerLow_on_M1timefame() == N5ZZZ3 && GetLowestPointAfterDailyHigh() == N5ZZZ1 )||
        ( M5LowerLow_on_M1timefame() == N15ZZZ3 && GetLowestPointAfterDailyHigh() == N15ZZZ1 )|| 
       ( M5LowerLow_on_M1timefame() == N30ZZZ3 && GetLowestPointAfterDailyHigh() == N30ZZZ1 )||  
        ( M5LowerLow_on_M1timefame() == G1ZZZ3 && GetLowestPointAfterDailyHigh() == G1ZZZ1 )||
        ( M5LowerLow_on_M1timefame() == G4ZZZ3 && GetLowestPointAfterDailyHigh() == G4ZZZ1 ))
      
  

        
       
       
       
        &&   IsBuyCondLow307Unbreached(buyCondLow307, buyCondTime307) == true && buyCondMet307 == true 
         

        && ( ( N1zigzagLongEntry  > 0 && iOpen(NULL, PERIOD_M1, 0) < N1zigzagLongEntry  && iClose(NULL, PERIOD_M1, 0) > N1zigzagLongEntry  && N1ZZZ3  == GetLowestPointAfterDailyHigh() && ( N1ZZZ1  < PivotAtLowestPointAfterDailyHighM1() || N1ZZZ1  < GetHighAtLowestPointAfterDailyHighM1() ) ) )
          
        
        
        //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
// BUY SIDE  (price vs price: ZZZ3 == low at lowest point after daily high)
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  && ( ( GBPUSD_GetLowAtLowestPointAfterDailyHighM1() > 0 && GBPUSD_GetN1ZZZ3ZigZagStopLossLevel()  == GBPUSD_GetLowAtLowestPointAfterDailyHighM1() && GBPUSD_GetN1ZZZ2ZigZagStopLossLevel() > 0 && iOpen("GBPUSD", PERIOD_M1, 0) < GBPUSD_GetN1ZZZ2ZigZagStopLossLevel() && iClose("GBPUSD", PERIOD_M1, 0) > GBPUSD_GetN1ZZZ2ZigZagStopLossLevel()
        
         && GBPUSD_GetN1ZZZ2ZigZagStopLossLevel() > GBPUSD_GetN1ZZZ1ZigZagStopLossLevel() &&  GBPUSD_GetN1ZZZ2ZigZagStopLossLevel() > GBPUSD_GetN1ZZZ3ZigZagStopLossLevel()
        
       
                            &&  GBPUSD_GetN1ZZZ2ZigZagTime() > GBPUSD_GetLowestPointAfterDailyHighTimeOnM1() 
       
        &&  GBPUSD_GetN1ZZZ1ZigZagStopLossLevel() > GBPUSD_GetN1ZZZ3ZigZagStopLossLevel() ))
        
        
             
            
             
             
           
    
        

        
 //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
       
        
        
        
        
        
       
        
        
           
          
           &&  (  iClose(NULL, PERIOD_M1, 0) - GetLowAtLowestPointAfterDailyHighM1() <= 20 * pipMultiplier * point )
          
          
        
        
        
        
        
        
        
       &&  !HasOpenTrade())
      {
      // Capture the current, unique 1-minute opening bar timestamp
         datetime currentBarM1Time = iTime(NULL, PERIOD_M1, 0);
         double sl =  GetFrozenStopLossSnapshot_GetLowestPointAfterDailyHigh(currentBarM1Time) - (StopLossPadding * pipMultiplier * point);
         double tp = Ask + (TakeProfitPoints * pipMultiplier * point);

         int ticket = OrderSend(Symbol(), OP_BUY, LotSize, Ask, 3, sl, tp, "315 Buy 315E", MagicNumber, 0, clrOrchid);

         if(ticket < 0)
         {
            Alert("CRITICAL ERROR: Buy Strategy 315E failed to open on ", Symbol(), ". Error Code: ", GetLastError());
         }
         else
         {
            if(OrderSelect(ticket, SELECT_BY_TICKET))
            {
               Alert("SUCCESS: Buy 315E opened at: ", DoubleToString(OrderOpenPrice(), Digits));
               DrawTradeSignal(OP_BUY, OrderOpenPrice(), sl, tp);
               REAL315E_SendSignal("BUY");
            }
         }
      }
   }

   // MASTER BLOCK: SELL - opens on this EA's own conditions, then broadcasts "SELL"
   // via REAL315E_SendSignal() (same flow as REAL_315E_Master_V1).
   if(Ask - Bid <= 3 * pipMultiplier * point)
   {
      if(TradeTimeAllowed() == true 
        
          
         
          && iTime("GBPUSD", 0, iLowest("GBPUSD", 0, MODE_LOW, DayStartShift + 1, 1)) > iTime("GBPUSD", 0, iHighest("GBPUSD", 0, MODE_HIGH, DayStartShift + 1, 1))
  
        && iTime("EURUSD", 0, iLowest("EURUSD", 0, MODE_LOW, DayStartShift + 1, 1)) > iTime("EURUSD", 0, iHighest("EURUSD", 0, MODE_HIGH, DayStartShift + 1, 1))

  &&  ( ( M5HigherHigh_on_M1timeframe() == N5ZZZ3 && GetHighestPointAfterDailyLow() == N5ZZZ1 ) ||
 ( M5HigherHigh_on_M1timeframe() == N15ZZZ3 && GetHighestPointAfterDailyLow() == N15ZZZ1 ) || 
 ( M5HigherHigh_on_M1timeframe() == N30ZZZ3 && GetHighestPointAfterDailyLow() == N30ZZZ1 ) ||  
 ( M5HigherHigh_on_M1timeframe() == G1ZZZ3 && GetHighestPointAfterDailyLow() == G1ZZZ1 ) ||
 ( M5HigherHigh_on_M1timeframe() == G4ZZZ3 && GetHighestPointAfterDailyLow() == G4ZZZ1 ) )

     
       
        &&  GBPUSD_GetLargestBearishM1_Time(20) >= GBPUSD_GetHighestPointAfterDailyLowTimeOnM1() && EURUSD_GetLargestBearishM1_Time( 20) >= EURUSD_GetHighestPointAfterDailyLowTimeOnM1()
       
       
            &&  GBPUSD_GetLargestBearishM1_Open( 20) - GBPUSD_GetLargestBearishM1_Close(  20) > GBPUSD_GetLargestBullishM1_Close(  20) - GBPUSD_GetLargestBearishM1_Open(  20)
              
            &&  EURUSD_GetLargestBearishM1_Open( 20) - EURUSD_GetLargestBearishM1_Close(  20) > EURUSD_GetLargestBullishM1_Close(  20) - EURUSD_GetLargestBearishM1_Open(  20)

       
       
        && iClose(NULL, PERIOD_M1, 0) < iOpen(NULL, PERIOD_M1, 0) && M5HigherHigh_on_M1timeframe() > 0 && iClose("EURUSD", PERIOD_M1, 0) < iOpen("EURUSD", PERIOD_M1, 0) && iClose("GBPUSD", PERIOD_M1, 0) < iOpen("GBPUSD", PERIOD_M1, 0)
            
        && GBPUSD_GetHighestPointAfterDailyLowTimeOnM1() > 0 && EURUSD_GetHighestPointAfterDailyLowTimeOnM1() > 0 && MathAbs((double)(GBPUSD_GetHighestPointAfterDailyLowTimeOnM1() - EURUSD_GetHighestPointAfterDailyLowTimeOnM1())) <= 30 * 60
        
        
    
         &&  IsSellCondHigh307Unbreached(sellCondHigh307, sellCondTime307) == true && sellCondMet307 == true 
         

        && ( ( N1zigzagShortEntry  > 0 && iOpen(NULL, PERIOD_M1, 0) > N1zigzagShortEntry  && iClose(NULL, PERIOD_M1, 0) < N1zigzagShortEntry  && N1ZZZ3  == GetHighestPointAfterDailyLow() && ( N1ZZZ1  > PivotAtHighestPointAfterDailyLowM1() || N1ZZZ1  > GetLowAtHighestPointAfterDailyLowM1() ) ) )
          
        
        
        
        
        //////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
// SELL SIDE (price vs price: ZZZ3 == high at highest point after daily low)
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
  && ( ( GBPUSD_GetHighAtHighestPointAfterDailyLowM1() > 0 && GBPUSD_GetN1ZZZ3ZigZagStopLossLevel()  == GBPUSD_GetHighAtHighestPointAfterDailyLowM1() && GBPUSD_GetN1ZZZ2ZigZagStopLossLevel() > 0 && iOpen("GBPUSD", PERIOD_M1, 0) > GBPUSD_GetN1ZZZ2ZigZagStopLossLevel() && iClose("GBPUSD", PERIOD_M1, 0) < GBPUSD_GetN1ZZZ2ZigZagStopLossLevel()
        
         && GBPUSD_GetN1ZZZ2ZigZagStopLossLevel() < GBPUSD_GetN1ZZZ1ZigZagStopLossLevel() &&  GBPUSD_GetN1ZZZ2ZigZagStopLossLevel() < GBPUSD_GetN1ZZZ3ZigZagStopLossLevel()
        
       
                            &&  GBPUSD_GetN1ZZZ2ZigZagTime() > GBPUSD_GetHighestPointAfterDailyLowTimeOnM1() 
       
        &&  GBPUSD_GetN1ZZZ1ZigZagStopLossLevel() < GBPUSD_GetN1ZZZ3ZigZagStopLossLevel() ) )
        
       
//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

         
          && (  GetHighestPointAfterDailyLow() - iClose(NULL, PERIOD_M1, 0) <= 20 * pipMultiplier * point )
          
                                                                    
         
         
      &&   !HasOpenTrade())
    
    
    
    
    
    
    
      {
        // Capture the current, unique 1-minute opening bar timestamp
         datetime currentBarM1Time = iTime(NULL, PERIOD_M1, 0);
         double sl = GetFrozenStopLossSnapshot_GetHighestPointAfterDailyLow(currentBarM1Time) + (StopLossPadding * pipMultiplier * point) + (Ask - Bid);
         double tp = Bid - (TakeProfitPoints * pipMultiplier * point);

         int ticket = OrderSend(Symbol(), OP_SELL, LotSize, Bid, 3, sl, tp, "315 Sell 315E", MagicNumber, 0, clrMediumOrchid);

         if(ticket < 0)
         {
            Alert("CRITICAL ERROR: Sell Strategy 315E failed to open on ", Symbol(), ". Error Code: ", GetLastError());
         }
         else
         {
            if(OrderSelect(ticket, SELECT_BY_TICKET))
            {
               Alert("SUCCESS: Sell 315E opened at: ", DoubleToString(OrderOpenPrice(), Digits));
               DrawTradeSignal(OP_SELL, OrderOpenPrice(), sl, tp);
               REAL315E_SendSignal("SELL");
            }
         }
      }
   }
}
