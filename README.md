# The Money Printer

The Money Printer is a simple a basic algorithmic trading strategy that buys call options when the price of the underlying equity is higher than its 21-day high and liquidates any existing options positions that are close to expiration.

## Usage

Created and testing in the site https://www.quantconnect.com

```bash
https://www.quantconnect.com
```

## Initialize

• Sets the start and end dates for the algorithm using SetStartDate() and SetEndDate() functions.

• Sets the initial amount of cash available for trading using SetCash().

• Adds an equity with the ticker symbol "SPY" to the algorithm using AddEquity() function, which specifies a resolution of minute and raw data normalization mode. It assigns the Symbol property of the equity object to the self.equity variable.

• Sets the benchmark of the algorithm to SPY using SetBenchmark() function.

• Adds an option with the ticker symbol "SPY" to the algorithm using AddOption() function, which specifies a resolution of minute and a filter for option contracts. The filter is set to only include contracts with a strike price between -3 and 3 standard deviations from the current underlying price, and with an expiration date between 20 and 40 days from the current date.

• Calculates the highest high price for the equity symbol "SPY" over the past 21 days using MAX() function with a resolution of daily and sets it to the self.high variable.

```python
def Initialize(self):
        self.SetStartDate(2018, 1, 1)
        self.SetEndDate(2023, 1, 1)
        self.SetCash(100000)
        equity = self.AddEquity("SPY", Resolution.Minute)
        equity.SetDataNormalizationMode(DataNormalizationMode.Raw)
        self.equity = equity.Symbol
        self.SetBenchmark(self.equity)
        
        option = self.AddOption("SPY", Resolution.Minute)
        option.SetFilter(-3, 3, timedelta(20), timedelta(40))

        self.high = self.MAX(self.equity, 21, Resolution.Daily, Field.High)
```
## OnData

• It first checks whether the self.high variable is ready or not. self.high stores the highest value of the equity symbol SPY over the last 21 days. If it's not ready, then the function returns without doing anything.

• It checks whether any options are already invested or not. If any options are already invested, it checks whether the time to expiration is less than or equal to four days. If so, it calls the Liquidate function to sell the option.

• If no options are invested or all the invested options are far away from expiration, it checks whether the current price of the SPY equity is greater than or equal to the self.high value. If it is, the function iterates over all the option chains and calls the BuyCall function to purchase the call option with the closest strike price to the current underlying asset price.

```python
 def OnData(self,data):
        if not self.high.IsReady:
            return
        
        option_invested = [x.Key for x in self.Portfolio if x.Value.Invested and x.Value.Type==SecurityType.Option]
        
        if option_invested:
            if self.Time + timedelta(4) > option_invested[0].ID.Date:
                self.Liquidate(option_invested[0], "Too close to expiration")
            return
        
        if self.Securities[self.equity].Price >= self.high.Current.Value:
            for i in data.OptionChains:
                chains = i.Value
                self.BuyCall(chains)
```

## OnData

The BuyCall function is called when the price of the self.equity is greater than or equal to its 21-day high value. It first sorts the chains option chains by the expiry date of the options in descending order and selects the latest expiry date. It then selects only the call options that have the same expiry date as the latest expiry date and stores them in the calls list. The calls list is then sorted by the absolute difference between the strike price and the underlying asset's last price. The first element of the sorted call_contracts list is then selected, and the quantity of the option contract to be bought is calculated as 5% of the total portfolio value divided by the ask price of the option contract. Finally, the Buy function is called with the option contract's symbol and the calculated quantity to buy the option.


```python
   def BuyCall(self,chains):
        expiry = sorted(chains,key = lambda x: x.Expiry, reverse=True)[0].Expiry
        calls = [i for i in chains if i.Expiry == expiry and i.Right == OptionRight.Call]
        call_contracts = sorted(calls,key = lambda x: abs(x.Strike - x.UnderlyingLastPrice))
        if len(call_contracts) == 0: 
            return
        self.call = call_contracts[0]
        
        quantity = self.Portfolio.TotalPortfolioValue / self.call.AskPrice
        quantity = int( 0.05 * quantity / 100 )
        self.Buy(self.call.Symbol, quantity)
```

## OnOrderEvent

The OnOrderEvent function is a method that gets called whenever an order event occurs, such as when an order is filled or partially filled.

In this particular case, the function first retrieves the order object associated with the order event using the GetOrderById method. It then checks whether the order type is an option exercise using order.Type == OrderType.OptionExercise. If the order is indeed an option exercise, the function calls self.Liquidate(), which is a method defined earlier in the class and is responsible for liquidating the current position.


```python

   def OnOrderEvent(self, orderEvent):
        order = self.Transactions.GetOrderById(orderEvent.OrderId)
        if order.Type == OrderType.OptionExercise:
            self.Liquidate()            
    
```
## Contributing

Pull requests are welcome. Money printer go brrrr! Go Flames! #QUANTCLUB

