// Define input parameters for the strategy
input ENUM_TIMEFRAMES TimeFrame = PERIOD_M1; // Timeframe for the strategy
input double LotSize = 0.1; // Lot size for orders
input double Slippage = 3; // Slippage for orders
input int MagicNumber = 123456; // Unique identifier for orders

// Global variable to track the last processed time
datetime lastTime = 0;

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit()
{
    // Set up the timer to check every new candlestick
    EventSetTimer(60); // Check every minute (adjust based on the timeframe)
    return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
    // Cleanup on deinitialization
    EventKillTimer();
}

//+------------------------------------------------------------------+
//| Expert timer function                                           |
//+------------------------------------------------------------------+
void OnTimer()
{
    // Check if a new candlestick has opened
    if (iTime(NULL, TimeFrame, 0) != lastTime)
    {
        lastTime = iTime(NULL, TimeFrame, 0);
        ProcessOrders();
    }
}

//+------------------------------------------------------------------+
//| Function to process orders based on the latest closed candlestick |
//+------------------------------------------------------------------+
void ProcessOrders()
{
    // Close all existing orders
    CloseAllOrders();
    
    // Get the price data of the last closed candlestick
    double high = iHigh(NULL, TimeFrame, 1);
    double low = iLow(NULL, TimeFrame, 1);

    // Place buy order
    PlaceOrder(ORDER_BUY, high, low);

    // Place sell order
    PlaceOrder(ORDER_SELL, low, high);
}

//+------------------------------------------------------------------+
//| Function to place an order                                      |
//+------------------------------------------------------------------+
void PlaceOrder(int orderType, double openPrice, double stopLoss)
{
    MqlTradeRequest request = {};
    MqlTradeResult result = {};

    request.action = TRADE_ACTION_DEAL;
    request.symbol = Symbol();
    request.volume = LotSize;
    request.type = orderType;
    request.price = openPrice;
    request.sl = stopLoss;
    request.tp = 0; // No take profit
    request.deviation = Slippage;
    request.magic = MagicNumber;
    request.comment = "Order";

    if (!OrderSend(request, result))
    {
        Print("Error sending order: ", GetLastError());
    }
}

//+------------------------------------------------------------------+
//| Function to close all orders                                    |
//+------------------------------------------------------------------+
void CloseAllOrders()
{
    for (int i = OrdersTotal() - 1; i >= 0; i--)
    {
        if (OrderSelect(i, SELECT_BY_POS))
        {
            // Check if the order belongs to this expert (MagicNumber)
            if (OrderMagicNumber() == MagicNumber)
            {
                bool closed = false;
                if (OrderType() == ORDER_BUY)
                {
                    closed = OrderClose(OrderTicket(), OrderLots(), Bid, Slippage, clrRed);
                }
                else if (OrderType() == ORDER_SELL)
                {
                    closed = OrderClose(OrderTicket(), OrderLots(), Ask, Slippage, clrRed);
                }
                if (!closed)
                {
                    Print("Error closing order: ", GetLastError());
                }
            }
        }
    }
}
