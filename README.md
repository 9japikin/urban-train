//+------------------------------------------------------------------+
//|            Tick Velocity + Body Filter Breakout EA               |
//+------------------------------------------------------------------+
#property strict

//---- Inputs
input double LotSize = 0.10;
input int StopLossPips = 5;
input double TickMultiplier = 7.0;
input double BodyMultiplier = 3.0;

input double RiskReward = 2.0;
input double BreakEvenTrigger = 1.5;




input ENUM_TIMEFRAMES TrendTF       = PERIOD_H4;
input int    TrendSMA               = 50;




//---- Tick tracking
datetime lastTime = 0;
int tickCount = 0;

double currTickRate = 0;

//---- Tick history
#define MAX_SAMPLES 50
double tickHistory[MAX_SAMPLES];
int tickIndex = 0;
int tickSamples = 0;

//---- Pip handling
double PipPoint;





//--- Inputs (add these with your other EA inputs)
input int    MaxSpikeDetectionMilliseconds = 5000; // Detection window (5 seconds)
input double HighTolerancePips             = 0.2;  // Close must be within this many pips of High





//+------------------------------------------------------------------+
//| Initialize                                                      |
//+------------------------------------------------------------------+
int OnInit()
{
   if(Digits == 3 || Digits == 5)
      PipPoint = Point * 10;
   else
      PipPoint = Point;

   return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Average Tick Rate                                               |
//+------------------------------------------------------------------+
double GetAverageTickRate()
{
   double sum = 0;
   for(int i=0; i<tickSamples; i++)
      sum += tickHistory[i];

   if(tickSamples == 0) return 0;

   return sum / tickSamples;
}

//+------------------------------------------------------------------+
void StoreTickRate(double rate)
{
   tickHistory[tickIndex] = rate;

   tickIndex++;
   if(tickIndex >= MAX_SAMPLES)
      tickIndex = 0;

   if(tickSamples < MAX_SAMPLES)
      tickSamples++;
}

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

//+------------------------------------------------------------------+
//| Daily Structure Functions                                       |
//+------------------------------------------------------------------+
double GetLowestPointAfterDailyHigh()
{
   double dailyHigh = iHigh(Symbol(), PERIOD_D1, 0);
   int dailyHighBar = -1;

   for(int x=0; x<Bars; x++)
   {
      if(High[x] == dailyHigh)
      {
         dailyHighBar = x;
         break;
      }
   }

   if(dailyHighBar == -1) return EMPTY_VALUE;

   double lowestLow = dailyHigh;

   for(int z = dailyHighBar - 1; z >= 0; z--)
   {
      if(Low[z] < lowestLow)
         lowestLow = Low[z];
   }

   return lowestLow;
}

//+------------------------------------------------------------------+
double GetHighestPointAfterDailyLow()
{
   double dailyLow = iLow(Symbol(), PERIOD_D1, 0);
   int dailyLowBar = -1;

   for(int y=0; y<Bars; y++)
   {
      if(Low[y] == dailyLow)
      {
         dailyLowBar = y;
         break;
      }
   }

   if(dailyLowBar == -1) return EMPTY_VALUE;

   double highestHigh = dailyLow;

   for(int m = dailyLowBar - 1; m >= 0; m--)
   {
      if(High[m] > highestHigh)
         highestHigh = High[m];
   }

   return highestHigh;
}


 //+------------------------------------------------------------------+
//| Returns datetime of Max Bullish Spike on M1 if criteria met      |
//+------------------------------------------------------------------+
datetime MaxBullSpikeTime_OnM1()
{
   static datetime lastCheckedBarTime = 0;
   static ulong    barStartMicrosec   = 0;
   static datetime validSpikeTime     = 0;

   datetime currentBarTime = iTime(Symbol(), PERIOD_M1, 0);

   // 1. Detect New M1 Bar and anchor the initial microsecond timestamp
   if(currentBarTime != lastCheckedBarTime)
   {
      lastCheckedBarTime = currentBarTime;
      barStartMicrosec   = GetMicrosecondCount(); // Microsecond timestamp at bar start
      validSpikeTime     = 0;
   }

   // Return cached result if condition already met for this bar
   if(validSpikeTime != 0)
   {
      return validSpikeTime;
   }

   // 2. Microsecond Constraint Check: Must occur within 1,000,000 microseconds (1 sec)
   if(GetMicrosecondCount() - barStartMicrosec > 1000000)
   {
      return 0;
   }

   // 3. High-to-Open Spread Evaluation across preceding bars (Shift 1 to 5)
   double currentExpansion = iHigh(Symbol(), PERIOD_M1, 0) - iOpen(Symbol(), PERIOD_M1, 0);

   bool isDominantExpansion = 
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 1) - iOpen(Symbol(), PERIOD_M1, 1))) &&
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 2) - iOpen(Symbol(), PERIOD_M1, 2))) &&
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 3) - iOpen(Symbol(), PERIOD_M1, 3))) &&
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 4) - iOpen(Symbol(), PERIOD_M1, 4))) &&
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 5) - iOpen(Symbol(), PERIOD_M1, 5)));

   // 4. Close at High Check (Ensures price is actively printing at the absolute peak)
   bool isAtHigh = (iClose(Symbol(), PERIOD_M1, 0) == iHigh(Symbol(), PERIOD_M1, 0));

   // Validate conditions
   if(isDominantExpansion && isAtHigh)
   {
      validSpikeTime = currentBarTime;
      return validSpikeTime;
   }

   return 0;
}


//+------------------------------------------------------------------+
//| Returns datetime of Max Bullish Spike on M1 if criteria met      |
//+------------------------------------------------------------------+
datetime MaxBullSpikeTime_OnM1_Version2()
{
   static datetime lastCheckedBarTime = 0;
   static ulong    barStartMicrosec   = 0;
   static datetime validSpikeTime     = 0;

   datetime currentBarTime = iTime(Symbol(), PERIOD_M1, 0);

   // 1. Detect new M1 bar and reset state
   if(currentBarTime != lastCheckedBarTime)
   {
      lastCheckedBarTime = currentBarTime;
      barStartMicrosec   = GetMicrosecondCount();
      validSpikeTime     = 0;
   }

   // Return cached result if already detected for this candle
   if(validSpikeTime != 0)
      return(validSpikeTime);

   // 2. Detection must occur within configurable time window
   if(GetMicrosecondCount() - barStartMicrosec >
      (ulong)MaxSpikeDetectionMilliseconds * 1000)
   {
      return(0);
   }

   // 3. Current bullish expansion
   double currentExpansion =
      iHigh(Symbol(), PERIOD_M1, 0) -
      iOpen(Symbol(), PERIOD_M1, 0);

   // Compare with previous five completed M1 candles
   bool isDominantExpansion =
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 1) - iOpen(Symbol(), PERIOD_M1, 1))) &&
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 2) - iOpen(Symbol(), PERIOD_M1, 2))) &&
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 3) - iOpen(Symbol(), PERIOD_M1, 3))) &&
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 4) - iOpen(Symbol(), PERIOD_M1, 4))) &&
      (currentExpansion > (iHigh(Symbol(), PERIOD_M1, 5) - iOpen(Symbol(), PERIOD_M1, 5)));

   // 4. Close must be at or very near the current High
   double tolerance = HighTolerancePips * PipPoint ;

   bool isAtHigh =
      (MathAbs(iHigh(Symbol(), PERIOD_M1, 0) -
               iClose(Symbol(), PERIOD_M1, 0))
       <= tolerance);

   // 5. Validate conditions
   if(isDominantExpansion && isAtHigh)
   {
      validSpikeTime = currentBarTime;
      return(validSpikeTime);
   }

   return(0);
}



//+------------------------------------------------------------------+
//| Returns datetime of Max Bearish Spike on M1 if criteria met      |
//+------------------------------------------------------------------+
datetime MaxBearSpikeTime_OnM1()
{
   static datetime lastCheckedBarTime = 0;
   static ulong    barStartMicrosec   = 0;
   static datetime validSpikeTime     = 0;

   datetime currentBarTime = iTime(Symbol(), PERIOD_M1, 0);

   // 1. Detect New M1 Bar and anchor the initial microsecond timestamp
   if(currentBarTime != lastCheckedBarTime)
   {
      lastCheckedBarTime = currentBarTime;
      barStartMicrosec   = GetMicrosecondCount();
      validSpikeTime     = 0;
   }

   // Return cached result if condition already met for this bar
   if(validSpikeTime != 0)
   {
      return validSpikeTime;
   }

   // 2. Microsecond Constraint Check: Must occur within 1,000,000 microseconds (1 sec)
   if(GetMicrosecondCount() - barStartMicrosec > 1000000)
   {
      return 0;
   }

   // 3. Open-to-Low Spread Evaluation across preceding bars (Shift 1 to 5)
   double currentExpansion = iOpen(Symbol(), PERIOD_M1, 0) - iLow(Symbol(), PERIOD_M1, 0);

   bool isDominantExpansion =
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 1) - iLow(Symbol(), PERIOD_M1, 1))) &&
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 2) - iLow(Symbol(), PERIOD_M1, 2))) &&
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 3) - iLow(Symbol(), PERIOD_M1, 3))) &&
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 4) - iLow(Symbol(), PERIOD_M1, 4))) &&
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 5) - iLow(Symbol(), PERIOD_M1, 5)));

   // 4. Close at Low Check (Ensures price is actively printing at the absolute low)
   bool isAtLow = (iClose(Symbol(), PERIOD_M1, 0) == iLow(Symbol(), PERIOD_M1, 0));

   // Validate conditions
   if(isDominantExpansion && isAtLow)
   {
      validSpikeTime = currentBarTime;
      return validSpikeTime;
   }

   return 0;
}

//+------------------------------------------------------------------+
//| Returns datetime of Max Bearish Spike on M1 if criteria met      |
//| Version 2                                                        |
//+------------------------------------------------------------------+
datetime MaxBearSpikeTime_OnM1_Version2()
{
   static datetime lastCheckedBarTime = 0;
   static ulong    barStartMicrosec   = 0;
   static datetime validSpikeTime     = 0;

   datetime currentBarTime = iTime(Symbol(), PERIOD_M1, 0);

   // 1. Detect new M1 bar and reset state
   if(currentBarTime != lastCheckedBarTime)
   {
      lastCheckedBarTime = currentBarTime;
      barStartMicrosec   = GetMicrosecondCount();
      validSpikeTime     = 0;
   }

   // Return cached result if already detected for this candle
   if(validSpikeTime != 0)
      return(validSpikeTime);

   // 2. Detection must occur within configurable time window
   if(GetMicrosecondCount() - barStartMicrosec >
      (ulong)MaxSpikeDetectionMilliseconds * 1000)
   {
      return(0);
   }

   // 3. Current bearish expansion
   double currentExpansion =
      iOpen(Symbol(), PERIOD_M1, 0) -
      iLow(Symbol(), PERIOD_M1, 0);

   // Compare with previous five completed M1 candles
   bool isDominantExpansion =
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 1) - iLow(Symbol(), PERIOD_M1, 1))) &&
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 2) - iLow(Symbol(), PERIOD_M1, 2))) &&
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 3) - iLow(Symbol(), PERIOD_M1, 3))) &&
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 4) - iLow(Symbol(), PERIOD_M1, 4))) &&
      (currentExpansion > (iOpen(Symbol(), PERIOD_M1, 5) - iLow(Symbol(), PERIOD_M1, 5)));

   // 4. Close must be at or very near the current Low
   double tolerance = HighTolerancePips * PipPoint ;

   bool isAtLow =
      (MathAbs(iClose(Symbol(), PERIOD_M1, 0) -
               iLow(Symbol(), PERIOD_M1, 0))
       <= tolerance);

   // 5. Validate conditions
   if(isDominantExpansion && isAtLow)
   {
      validSpikeTime = currentBarTime;
      return(validSpikeTime);
   }

   return(0);
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


/////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////



//+------------------------------------------------------------------+
//| Trade Check                                                     |
//+------------------------------------------------------------------+
bool HasOpenTrade()
{
   for(int i=0; i<OrdersTotal(); i++)
   {
      if(OrderSelect(i, SELECT_BY_POS, MODE_TRADES))
      {
         if(OrderSymbol() == Symbol())
            return true;
      }
   }
   return false;
}

//+------------------------------------------------------------------+
void OpenBuy()
{
   double sl = Bid - StopLossPips * PipPoint;
   double tp = Bid + (StopLossPips * PipPoint * RiskReward);

   OrderSend(Symbol(), OP_BUY, LotSize, Ask, 5, sl, tp, "Buy Breakout", 0, 0, clrBlue);
}

//+------------------------------------------------------------------+
void OpenSell()
{
   double sl = Ask + StopLossPips * PipPoint;
   double tp = Bid - (StopLossPips * PipPoint * RiskReward);

   OrderSend(Symbol(), OP_SELL, LotSize, Bid, 5, sl, tp, "Sell Breakout", 0, 0, clrRed);
}

//+------------------------------------------------------------------+
//| OnTick                                                          |
//+------------------------------------------------------------------+
void OnTick()
{
   datetime now = TimeCurrent();

   if(lastTime == 0)
   {
      lastTime = now;
      return;
   }

   tickCount++;

   double elapsed = now - lastTime;

   if(elapsed >= 1)
   {
      currTickRate = tickCount / elapsed;

      double avgTickRate = GetAverageTickRate();
      StoreTickRate(currTickRate);

      if(avgTickRate == 0)
      {
         tickCount = 0;
         lastTime = now;
         return;
      }

      bool strongVelocity = (currTickRate > avgTickRate * TickMultiplier);

      // Candle data
      double openM1  = iOpen(NULL, PERIOD_M1, 0);
      double closeM1 = iClose(NULL, PERIOD_M1, 0);

      double body = MathAbs(closeM1 - openM1);
      double avgBody = GetAverageBody(20);

      bool strongBody = (body > avgBody * BodyMultiplier);

      // Levels
      double resistance = GetHighestPointAfterDailyLow();
      double support    = GetLowestPointAfterDailyHigh();

      if(!HasOpenTrade())
      {
         // BUY
         if(strongVelocity && strongBody 
              
         && iTime(NULL, PERIOD_M1,  0 ) ==  MaxBullSpikeTime_OnM1_Version2()  
          
         && iClose(NULL, PERIOD_M1,  0 ) - iLow(NULL, PERIOD_M1,  0 ) >= 2* PipPoint 
              
&& ( iHigh(NULL, PERIOD_M1,  0 ) - iOpen(NULL, PERIOD_M1,  0 ) > iHigh(NULL, PERIOD_M1,  1 ) - iOpen(NULL, PERIOD_M1,  1 )&&
iHigh(NULL, PERIOD_M1, 0 ) - iOpen(NULL, PERIOD_M1,  0 ) > iHigh(NULL, PERIOD_M1,  2 ) - iOpen(NULL, PERIOD_M1,  2 )&&
iHigh(NULL, PERIOD_M1,  0 ) - iOpen(NULL, PERIOD_M1,  0 ) > iHigh(NULL, PERIOD_M1, 3 ) - iOpen(NULL, PERIOD_M1,  3 )&&
iHigh(NULL, PERIOD_M1,  0 ) - iOpen(NULL, PERIOD_M1,  0 ) > iHigh(NULL, PERIOD_M1,  4 ) - iOpen(NULL, PERIOD_M1,  4 )&&
iHigh(NULL, PERIOD_M1,  0 ) - iOpen(NULL, PERIOD_M1,  0 ) > iHigh(NULL, PERIOD_M1,  5 ) - iOpen(NULL, PERIOD_M1,  5 ) )

            
             )
         {
            OpenBuy();
         }

         // SELL
         if(strongVelocity && strongBody 
            
            
                     && iTime(NULL, PERIOD_M1,  0 ) ==  MaxBearSpikeTime_OnM1_Version2() 

             && iHigh(NULL, PERIOD_M1,  0 ) - iClose(NULL, PERIOD_M1,  0 ) >= 2* PipPoint 
            
            
             && ( iOpen(NULL, PERIOD_M1,  0 ) - iLow(NULL, PERIOD_M1,  0 ) > iOpen(NULL, PERIOD_M1,  1 ) - iLow(NULL, PERIOD_M1,  1 )
 && iOpen(NULL, PERIOD_M1,  0 ) - iLow(NULL, PERIOD_M1,  0 ) > iOpen(NULL, PERIOD_M1,  2 ) - iLow(NULL, PERIOD_M1,  2 )
 && iOpen(NULL, PERIOD_M1,  0 ) - iLow(NULL, PERIOD_M1,  0 ) > iOpen(NULL, PERIOD_M1,  3 ) - iLow(NULL, PERIOD_M1,  3 )
 && iOpen(NULL, PERIOD_M1,  0 ) - iLow(NULL, PERIOD_M1,  0 ) > iOpen(NULL, PERIOD_M1,  4 ) - iLow(NULL, PERIOD_M1,  4 )
 && iOpen(NULL, PERIOD_M1,  0 ) - iLow(NULL, PERIOD_M1,  0 ) > iOpen(NULL, PERIOD_M1,  5 ) - iLow(NULL, PERIOD_M1,  5 ) )
  
            
            
            )
         {
            OpenSell();
         }
      }

      tickCount = 0;
      lastTime = now;
   }
}




